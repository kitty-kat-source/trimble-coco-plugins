# trimble-coco-plugins

A collection of Cortex Code (CoCo) desktop plugins used at Trimble RMS.

## Plugins

### `snowflake-loader`

Four skills for data loading, DOMO orchestration, and governance:

| Skill | What it does |
|---|---|
| `load-to-snowflake` | Upload local CSV files to Snowflake via SnowSQL `PUT` + `COPY INTO` (Okta SSO, `trimble_rms` connection) |
| `domo-writeback` | Trigger DOMO Snowflake writeback connectors and optionally fire a downstream ETL dataflow. Config driven by `DEVRP.REVPRO.DOMO_WRITEBACK_CONFIG` |
| `headroom-setup` | Sub-skill: starts a local LLM token-compression proxy during large load sessions (called automatically by `load-to-snowflake`) |
| `export-audit` | Export the current CoCo chat transcript to `GOVERNANCE_DB.AUDIT.COCO_CHAT_HISTORY` for audit/governance |

---

## Installation on a new machine

### Prerequisites

- [Cortex Code Desktop](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) installed and signed in
- [uv](https://docs.astral.sh/uv/getting-started/installation/) installed (`winget install astral-sh.uv`)
- [SnowSQL](https://docs.snowflake.com/en/user-guide/snowsql-install-config) installed with a `trimble_rms` connection configured (for `load-to-snowflake`)
- [gh CLI](https://cli.github.com/) (optional, only needed to re-push updates)

### Install via CoCo

In a CoCo chat session, run:

```
install plugin from github kitty-kat-source/trimble-coco-plugins
```

CoCo's `github-plugin-installer` skill will clone this repo and register all skills automatically.

### First-time setup for `domo-writeback`

After installing the plugin:

1. Create `C:\Users\<you>\.snowflake\domo-writeback\.env` with your DOMO API credentials:
   ```
   DOMO_CLIENT_ID=your_client_id_here
   DOMO_CLIENT_SECRET=your_client_secret_here
   ```
2. Place `Orchestrated_python_for_all_writebacks.py` at `C:\Users\<you>\.snowflake\domo-writeback\`
3. Ensure `DEVRP.REVPRO.DOMO_WRITEBACK_CONFIG` is populated in Snowflake

See [`skills/domo-writeback/references/credential-setup.md`](skills/domo-writeback/references/credential-setup.md) for full setup instructions.

---

## Repo structure

```
snowflake-loader/
├── .cortex-plugin/
│   └── plugin.json          # CoCo plugin manifest
└── skills/
    ├── load-to-snowflake/
    │   └── SKILL.md
    ├── domo-writeback/
    │   ├── SKILL.md
    │   ├── pyproject.toml   # Python deps (uv)
    │   ├── uv.lock
    │   └── references/
    │       └── credential-setup.md
    ├── headroom-setup/
    │   └── SKILL.md
    └── export-audit/
        └── SKILL.md
```
