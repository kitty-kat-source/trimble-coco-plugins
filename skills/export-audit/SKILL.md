---
name: Export-audit
description: >
  Export the current CoCo session's chat history to Snowflake for governance and audit.
  Automatically detects the source application (Snowsight, Desktop, CLI) and pushes
  the current session transcript to GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY.
  Use when: export chat, audit chat history, log conversations, Export-audit,
  governance export, save chat to snowflake, chat audit log.
---

# Export-audit

Exports the **current CoCo session** transcript to `GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY`.
Automatically detects which application the user is chatting from (Snowsight, Desktop,
CLI, etc.) and records it. No manual source selection needed.

## Prerequisites

The following objects MUST already exist in Snowflake:

- Database: `GOVERNANCE_DB`
- Schema: `GOVERNANCE_DB.AUDIT`
- Stage: `GOVERNANCE_DB.AUDIT.AUDIT_STAGE`
- File Format: `GOVERNANCE_DB.AUDIT.JSON_FF` (TYPE = JSON, STRIP_OUTER_ARRAY = TRUE)
- Table: `GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY`

Expected table DDL:

```sql
CREATE TABLE GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY (
    SESSION_ID          VARCHAR(20)    NOT NULL,
    TITLE               VARCHAR(500),
    TRANSCRIPT          VARIANT,
    CREATED_DATE        TIMESTAMP_NTZ,
    CREATED_BY          VARCHAR(100),
    COMMENTS            VARCHAR(500),
    SOURCE              VARCHAR(50),
    SESSION_UPDATED_AT  TIMESTAMP_NTZ,
    EXPORTED_AT         TIMESTAMP_NTZ,
    MSG_COUNT           NUMBER DEFAULT 0,
    CONTENT_HASH        VARCHAR(64)
);
```

## Constants

```
TARGET_TABLE   = GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY
STAGE          = GOVERNANCE_DB.AUDIT.AUDIT_STAGE
STAGE_FOLDER   = COCO_EXPORT/<YYYY-MM-DD>/
FILE_FORMAT    = GOVERNANCE_DB.AUDIT.JSON_FF
SIZE_THRESHOLD = 10000   (characters; above this, use stage-based loading)
```

## Workflow

### Step 0: Validate Infrastructure

Run via `snowflake_sql_execute`:

```sql
SELECT COUNT(*) FROM GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY LIMIT 0;
```

If this fails, report the error and stop:
"Table GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY not found or not accessible.
Please ensure prerequisites exist before running this skill."

If it succeeds, proceed silently.

---

### Step 1: Identify Current Session and Auto-Detect Source

The skill exports the session it is currently running in. To identify the
current session and its source, run the following CLI commands via Bash:

**1a — Get the current session ID:**

```powershell
cortex conversations list --output csv --origin all -n 1
```

The first (most recent) row is the current active session. Parse the `id` field.
This is the `CURRENT_SESSION_ID`.

**1b — Auto-detect source by checking which origin owns this session:**

Run each origin-specific query and check if the current session ID appears:

```powershell
cortex conversations list --output csv --origin coding_agent -n 5
```

```powershell
cortex conversations list --output csv --origin coco:desktop -n 5
```

```powershell
cortex conversations list --output csv --origin coding_agent_cli -n 5
```

```powershell
cortex conversations list --output csv --origin sql_function -n 5
```

Check which origin list contains `CURRENT_SESSION_ID`. Apply this mapping:

| --origin containing session | SOURCE     | COMMENTS                       |
|-----------------------------|------------|--------------------------------|
| coding_agent                | Snowsight  | Triggered from coco-snowsight  |
| coco:desktop                | Desktop    | Triggered from coco-desktop    |
| coding_agent_cli            | CLI        | Triggered from coco-cli        |
| sql_function                | Automation | Triggered from coco-automation |
| (none matched)              | (Others - Unknown) | Triggered from coco-unknown |

**1c — Extract session metadata:**

From the conversation list row for the current session, extract:
- `SESSION_ID` = the `id` field
- `TITLE` = the `title` field (may be empty)
- `SESSION_UPDATED_AT` = the `updated` field (format: YYYY-MM-DD HH:MM)

**1d — Prompt for title if missing:**

If the `title` field is empty or blank, ask the user via `ask_user_question`:

- header: "Session Title"
- question: "This session has no title. Please provide a title for the audit record:"
- type: text
- defaultValue: "Untitled session - <SESSION_UPDATED_AT>"

Use the user's response as the `TITLE` value. If the user accepts the default,
use it as-is.

Report to user:
```
Current session detected:
  Session ID : <SESSION_ID>
  Source     : <SOURCE>
  Title      : <TITLE>
  Updated    : <SESSION_UPDATED_AT>
```

---

### Step 2: Incremental Check

Query the table for this specific session:

```sql
SELECT SESSION_UPDATED_AT, MSG_COUNT, CONTENT_HASH
FROM GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY
WHERE SESSION_ID = '<SESSION_ID>';
```

If the session exists in the table:
- Compare `SESSION_UPDATED_AT` from CSV with stored value
- If timestamps match → report "Session already exported and unchanged. Nothing to do." and **stop**
- If timestamps differ → proceed (will check content hash after fetching transcript)

If the session does NOT exist in the table → proceed (new session).

---

### Step 3: Fetch and Process Transcript

#### 3a — Fetch transcript

```powershell
cortex conversations transcript <SESSION_ID> --output json
```

Capture output as an array of strings (one JSONL line per message).

#### 3b — Compute MSG_COUNT

```
MSG_COUNT = number of lines in transcript output
```

#### 3c — Compute CONTENT_HASH

```powershell
$raw = $transcript_lines -join "`n"
$md5 = [System.Security.Cryptography.MD5]::Create()
$bytes = [System.Text.Encoding]::UTF8.GetBytes($raw)
$hash = [BitConverter]::ToString($md5.ComputeHash($bytes)).Replace("-","").ToLower()
```

**Deduplication check:** If the session already exists in the table AND the
computed `CONTENT_HASH` matches the stored hash, skip and report:
"Session content unchanged (hash match). Nothing to export."

#### 3d — Convert JSONL to JSON Array

```powershell
$json_array = "[" + ($transcript_lines -join ",") + "]"
```

---

### Step 4: Load to Snowflake

#### 4a — Determine today's date for stage folder

```powershell
$today = Get-Date -Format "yyyy-MM-dd"
```

Stage path will be: `@GOVERNANCE_DB.AUDIT.AUDIT_STAGE/COCO_EXPORT/<today>/`

#### 4b — Determine loading path

Check the character length of `$json_array`:

- If **under 10,000 characters** → use inline MERGE (Step 4c)
- If **10,000 characters or more** → use stage-based MERGE (Step 4d)

#### 4c — Inline MERGE (small transcripts only)

Escape single quotes in `$json_array` and `$title` by doubling them (`'` → `''`).

Run via `snowflake_sql_execute`:

```sql
MERGE INTO GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY AS tgt
USING (
    SELECT
        '<SESSION_ID>'                           AS SESSION_ID,
        '<ESCAPED_TITLE>'                        AS TITLE,
        PARSE_JSON('<ESCAPED_JSON_ARRAY>')       AS TRANSCRIPT,
        '<SESSION_UPDATED_AT>'::TIMESTAMP_NTZ    AS CREATED_DATE,
        CURRENT_USER()                           AS CREATED_BY,
        '<COMMENTS>'                             AS COMMENTS,
        '<SOURCE>'                               AS SOURCE,
        '<SESSION_UPDATED_AT>'::TIMESTAMP_NTZ    AS SESSION_UPDATED_AT,
        CURRENT_TIMESTAMP()                      AS EXPORTED_AT,
        <MSG_COUNT>                              AS MSG_COUNT,
        '<CONTENT_HASH>'                         AS CONTENT_HASH
) AS src
ON tgt.SESSION_ID = src.SESSION_ID
WHEN MATCHED THEN UPDATE SET
    tgt.TRANSCRIPT         = src.TRANSCRIPT,
    tgt.TITLE              = src.TITLE,
    tgt.SESSION_UPDATED_AT = src.SESSION_UPDATED_AT,
    tgt.EXPORTED_AT        = src.EXPORTED_AT,
    tgt.MSG_COUNT          = src.MSG_COUNT,
    tgt.CONTENT_HASH       = src.CONTENT_HASH,
    tgt.COMMENTS           = src.COMMENTS,
    tgt.SOURCE             = src.SOURCE
WHEN NOT MATCHED THEN INSERT
    (SESSION_ID, TITLE, TRANSCRIPT, CREATED_DATE, CREATED_BY, COMMENTS,
     SOURCE, SESSION_UPDATED_AT, EXPORTED_AT, MSG_COUNT, CONTENT_HASH)
VALUES
    (src.SESSION_ID, src.TITLE, src.TRANSCRIPT, src.CREATED_DATE,
     src.CREATED_BY, src.COMMENTS, src.SOURCE, src.SESSION_UPDATED_AT,
     src.EXPORTED_AT, src.MSG_COUNT, src.CONTENT_HASH);
```

Proceed to Step 4e (retry logic).

#### 4d — Stage-based MERGE (large transcripts, default path)

**Write to temp file:**

```powershell
$tempPath = "C:\temp\coco_session_<SESSION_ID>.json"
$json_array | Set-Content $tempPath -Encoding UTF8
```

**PUT to stage with date-based folder** via `snowflake_sql_execute`:

```sql
PUT 'file://C:/temp/coco_session_<SESSION_ID>.json'
    @GOVERNANCE_DB.AUDIT.AUDIT_STAGE/COCO_EXPORT/<YYYY-MM-DD>/
    AUTO_COMPRESS=FALSE OVERWRITE=TRUE;
```

**MERGE from stage** via `snowflake_sql_execute`:

```sql
MERGE INTO GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY AS tgt
USING (
    SELECT
        '<SESSION_ID>'                           AS SESSION_ID,
        '<ESCAPED_TITLE>'                        AS TITLE,
        $1                                       AS TRANSCRIPT,
        '<SESSION_UPDATED_AT>'::TIMESTAMP_NTZ    AS CREATED_DATE,
        CURRENT_USER()                           AS CREATED_BY,
        '<COMMENTS>'                             AS COMMENTS,
        '<SOURCE>'                               AS SOURCE,
        '<SESSION_UPDATED_AT>'::TIMESTAMP_NTZ    AS SESSION_UPDATED_AT,
        CURRENT_TIMESTAMP()                      AS EXPORTED_AT,
        <MSG_COUNT>                              AS MSG_COUNT,
        '<CONTENT_HASH>'                         AS CONTENT_HASH
    FROM @GOVERNANCE_DB.AUDIT.AUDIT_STAGE/COCO_EXPORT/<YYYY-MM-DD>/coco_session_<SESSION_ID>.json
        (FILE_FORMAT => 'GOVERNANCE_DB.AUDIT.JSON_FF')
) AS src
ON tgt.SESSION_ID = src.SESSION_ID
WHEN MATCHED THEN UPDATE SET
    tgt.TRANSCRIPT         = src.TRANSCRIPT,
    tgt.TITLE              = src.TITLE,
    tgt.SESSION_UPDATED_AT = src.SESSION_UPDATED_AT,
    tgt.EXPORTED_AT        = src.EXPORTED_AT,
    tgt.MSG_COUNT          = src.MSG_COUNT,
    tgt.CONTENT_HASH       = src.CONTENT_HASH,
    tgt.COMMENTS           = src.COMMENTS,
    tgt.SOURCE             = src.SOURCE
WHEN NOT MATCHED THEN INSERT
    (SESSION_ID, TITLE, TRANSCRIPT, CREATED_DATE, CREATED_BY, COMMENTS,
     SOURCE, SESSION_UPDATED_AT, EXPORTED_AT, MSG_COUNT, CONTENT_HASH)
VALUES
    (src.SESSION_ID, src.TITLE, src.TRANSCRIPT, src.CREATED_DATE,
     src.CREATED_BY, src.COMMENTS, src.SOURCE, src.SESSION_UPDATED_AT,
     src.EXPORTED_AT, src.MSG_COUNT, src.CONTENT_HASH);
```

**Cleanup local temp file (keep stage file for audit trail):**

```powershell
Remove-Item "C:\temp\coco_session_<SESSION_ID>.json" -ErrorAction SilentlyContinue
```

Note: Stage files are kept organized by date under `COCO_EXPORT/<YYYY-MM-DD>/`
for audit purposes. They can be cleaned up periodically if storage is a concern.

#### 4e — Retry Logic

If the MERGE statement (4c or 4d) fails:

1. Wait 2 seconds: `Start-Sleep -Seconds 2`
2. Retry the MERGE statement exactly once
3. If retry succeeds → mark as success
4. If retry also fails → report the error to user and stop

Do NOT retry PUT failures — report immediately and stop.

---

### Step 5: Summary

After successful export, print:

```
Export Complete
==============
Session ID      : <SESSION_ID>
Title           : <TITLE or "(untitled)">
Source          : <SOURCE>
Messages        : <MSG_COUNT>
Action          : <Inserted | Updated>
Target table    : GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY
Stage path      : @GOVERNANCE_DB.AUDIT.AUDIT_STAGE/COCO_EXPORT/<YYYY-MM-DD>/
Exported by     : <CURRENT_USER>
Export timestamp : <CURRENT_TIMESTAMP>
```

---

## Stopping Points

- Step 0: If infrastructure validation fails → stop with error
- Step 2: If session unchanged (timestamp match) → stop (nothing to do)
- Step 3c: If content hash unchanged → stop (nothing to do)
- Step 4e: If MERGE fails after retry → stop with error

## Notes

- This skill exports the CURRENT session only — the session in which the
  skill is invoked. It does not batch-export multiple sessions.
- Source is auto-detected from the environment. No manual selection needed.
  The detection works by checking which `--origin` filter returns the
  current session ID.
- One row per SESSION_ID in the table. MERGE enforces this (upsert pattern).
- If the user triggers this skill multiple times in the same session, only
  the latest transcript state is kept (MERGE UPDATE overwrites).
- Stage files are organized by export date: `COCO_EXPORT/2026-08-24/coco_session_123.json`
- CREATED_DATE is set only on first INSERT; subsequent updates preserve it.
- EXPORTED_AT reflects the most recent export timestamp.
- Temp files are written to C:\temp\ — ensure this directory exists.
- Required Snowflake privileges:
  - SELECT/INSERT/UPDATE on GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY
  - READ/WRITE on @GOVERNANCE_DB.AUDIT.AUDIT_STAGE
