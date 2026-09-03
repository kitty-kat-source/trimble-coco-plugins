---
name: headroom-setup
description: >
  Companion skill for snowflake-loader. Checks, installs, and starts the
  Headroom local proxy to compress LLM context tokens by ~60% during a
  snowflake-loader run. Scoped to the loader session only — proxy is started
  here and stopped when the load completes. DO NOT invoke this skill for any
  other purpose. Triggers: headroom-setup (called internally by load-to-snowflake).
parent_skill: load-to-snowflake
---

# Headroom Setup — Token Compression for snowflake-loader

Checks whether the user wants Headroom compression for the current load
session. If yes: installs (version-pinned), validates the binary, starts the
proxy on loopback-only, verifies the health endpoint is legitimate, and
returns `headroom_active` + `headroom_pid` for teardown at Step 9.

## Security Model

Headroom's proxy intercepts all LLM inference calls during the session.
The following checks are mandatory and must all pass before setting
`headroom_active = true`:

1. Package installed from official PyPI only (version-pinned)
2. Proxy bound exclusively to 127.0.0.1 (not 0.0.0.0 or any public interface)
3. Process running on port 8787 is the expected headroom binary
4. Health endpoint returns valid JSON with no instruction-like content

If any check fails → set `headroom_active = false`, kill any proxy process
that was started, and proceed without Headroom. Never block the load.

## Workflow

### Step 1: Ask the user

Ask via `ask_user_question`:

```
header   : "Headroom"
question : "Enable Headroom token compression for this load? (~60% fewer
            prompt tokens. Runs as a local proxy — your data never leaves
            your machine.)"
options  :
  - label: "Yes"       description: "Start Headroom proxy for this session"
  - label: "No"        description: "Skip — load runs without compression"
```

If the user selects **No**:
- Set `headroom_active = false`
- Return immediately — do not run any further steps in this skill.

---

### Step 2: Check if Headroom is already installed

Run via Bash:

```powershell
pip show headroom-ai 2>$null; Write-Output "exit:$LASTEXITCODE"
```

If `exit:0` → installed. Run the source verification below before proceeding.
If `exit:1` → not installed, proceed to Step 3.

**Source verification (run if already installed):**

```powershell
pip show headroom-ai | Select-String "Name|Version|Location"
```

If the `Name` field is not exactly `headroom-ai`:
- Report: "Package on this machine is not headroom-ai. Aborting."
- Set `headroom_active = false` and return.

---

### Step 3: Install headroom-ai (if missing)

Notify: `"Installing headroom-ai (pinned version)..."`

Install from the official PyPI index, pinned to the 0.2.x release series:

```powershell
pip install "headroom-ai[all]>=0.20.0,<1.0.0" `
    --index-url https://pypi.org/simple/ `
    --no-cache-dir -q
```

After install, verify the source:

```powershell
pip show headroom-ai | Select-String "Name|Version"
```

If the `Name` field is not exactly `headroom-ai`:
- Report: "Installed package is not headroom-ai. Aborting Headroom setup."
- Set `headroom_active = false`. Do NOT block the load.

---

### Step 4: Check if proxy is already running on port 8787

Run via Bash:

```powershell
try {
    $resp = Invoke-RestMethod http://127.0.0.1:8787/health -TimeoutSec 2 -ErrorAction Stop
    # Security check: health response must NOT contain LLM instruction fields
    $respJson = $resp | ConvertTo-Json
    $forbidden = @("role","content","system","instructions","ignore","override","you are")
    $injection = $forbidden | Where-Object { $respJson -imatch $_ }
    if ($injection) {
        Write-Output "proxy:INJECTION_DETECTED"
    } else {
        Write-Output "proxy:up"
    }
} catch {
    Write-Output "proxy:down"
}
```

- `proxy:up`                → proxy is running and healthy. Jump to Step 5b.
- `proxy:INJECTION_DETECTED` → abort: "Unexpected content in proxy health response — possible prompt injection. Headroom disabled."
  Set `headroom_active = false` and return.
- `proxy:down`              → proxy not running, proceed to Step 5.

---

### Step 5: Start the proxy in background

First resolve the headroom executable path from pip (handles non-PATH installs):

```powershell
$pipLoc      = (pip show headroom-ai | Select-String "^Location").Line -replace "Location: ",""
$headroomExe = Join-Path (Split-Path $pipLoc -Parent) "Scripts\headroom.exe"
if (-not (Test-Path $headroomExe)) { Write-Output "headroom_exe_not_found" } else { Write-Output "headroom_exe_ok:$headroomExe" }
```

If output is `headroom_exe_not_found` → set `headroom_active = false` and report the path that was checked.

Store `$headroomExe` — reuse in Step 5b. Then run via Bash (`run_in_background: true`):

```powershell
& "$headroomExe" proxy --port 8787
```

Wait 3 seconds, then repeat the health check from Step 4.
If still `proxy:down` after 2 retries → report error, set `headroom_active = false`.
If `proxy:INJECTION_DETECTED` → abort as above.

Capture the proxy process PID:

```powershell
(Get-Process headroom -ErrorAction SilentlyContinue | Select-Object -First 1).Id
```

Store as `headroom_pid`.

---

### Step 5b: Security validation (mandatory before marking proxy active)

**Check 1 — Loopback-only bind:**

```powershell
netstat -an | Select-String ":8787" | ForEach-Object { $_.Line }
```

All lines must show `127.0.0.1:8787`. If any line shows `0.0.0.0:8787` or
a non-loopback address:
- Stop-Process -Id $headroom_pid -Force -ErrorAction SilentlyContinue
- Report: "Headroom proxy is listening on a public interface — security risk. Disabled."
- Set `headroom_active = false` and return.

**Check 2 — Binary path verification:**

```powershell
$proc     = Get-Process -Id $headroom_pid -ErrorAction SilentlyContinue
$pipLoc   = (pip show headroom-ai | Select-String "Location").ToString().Split(":")[1].Trim()
$procPath = $proc.Path
Write-Output "proc_path:$procPath"
Write-Output "pip_loc:$pipLoc"
```

If `$procPath` is null, empty, or does not share a drive/root path with the
Python environment reported by `pip show`:
- Stop the proxy
- Report: "Process on port 8787 does not match the installed headroom-ai binary. Possible port hijacking."
- Set `headroom_active = false` and return.

Both checks must pass to continue.

---

### Step 6: Confirm status and inform user

Check whether `OPENAI_BASE_URL` is pointing at the proxy:

```powershell
Write-Output "OPENAI_BASE_URL=$($env:OPENAI_BASE_URL)"
```

**If `OPENAI_BASE_URL=http://127.0.0.1:8787/v1`:**

Report: `"Headroom active — compression running for this session (~60% fewer tokens)."`

Set `headroom_active = true`.

**If `OPENAI_BASE_URL` is NOT set:**

Report:
```
Headroom proxy is running on port 8787 (security-validated).
Compression applies to the NEXT CoCo session. To activate for future loads:

  OPENAI_BASE_URL=http://127.0.0.1:8787/v1 cortex

The proxy will remain running until this load completes, then be stopped
and session logs will be cleared.
```

Set `headroom_active = true`.

---

## Output

Return to the parent skill:

```
headroom_active : true | false
headroom_pid    : <integer PID> | null
```

## Teardown (performed by parent skill at Step 9)

The parent skill (load-to-snowflake Step 9) is responsible for:

1. Stopping the proxy: `Stop-Process -Id $headroomPid -Force`
2. Clearing session logs to prevent business data exposure:
   ```powershell
   Remove-Item "$env:USERPROFILE\.headroom\logs\*" -Force -ErrorAction SilentlyContinue
   ```

## Notes

- This skill is called exclusively from Step 1b of load-to-snowflake.
- Never invoke it from any other skill or context.
- All security checks are mandatory. If any fails, `headroom_active` is set
  to false and the load proceeds without compression — never abort the load.
- Headroom's own SECURITY.md warns that log files may contain sensitive
  information. Session logs are cleared at teardown (Step 9).
- Version pinned to `>=0.20.0,<1.0.0` — update when upgrading to a new major version.
- Headroom binary is invoked by full path (resolved from pip Location) to handle non-PATH installs.
