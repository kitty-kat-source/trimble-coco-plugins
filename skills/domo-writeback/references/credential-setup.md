# Credential & Config Setup

## Step 1 — Fill in .env

Open this file and replace both placeholder values:
```
C:\Users\saryan\.snowflake\domo-writeback\.env
```

```
DOMO_CLIENT_ID=your-actual-client-id
DOMO_CLIENT_SECRET=your-actual-client-secret
```

Where to find these values:
- Log into DOMO → top-right menu → Admin → Access Tokens → API Credentials
- Copy the Client ID and Client Secret for your service account

## Step 2 — Secure the .env file (Windows)

The `.env` file is a plain text file. Restrict who can read it:

1. Right-click `.env` → Properties → Security tab
2. Click Advanced → Disable Inheritance → Convert
3. Remove all users except your own Windows account
4. Set your account to Read only (not Full Control)

If this directory syncs to OneDrive or SharePoint, use Windows Credential
Manager instead (see below) — plain files in synced folders are not safe.

## Step 3 — Populate DEVRP.REVPRO.DOMO_WRITEBACK_CONFIG

Insert a row for each DOMO dataset you want to writeback to, plus the ETL dataflow ID:

```sql
INSERT INTO DEVRP.REVPRO.DOMO_WRITEBACK_CONFIG (DATASET_ID, LABEL, ETL_DATAFLOW_ID)
VALUES
  ('xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx', 'My Dataset Name',    'your-etl-dataflow-id'),
  ('yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy', 'Another Dataset',    'your-etl-dataflow-id');
```

Where to find Dataset UUIDs in DOMO:
- Open a dataset in DOMO Data Center
- The UUID is in the browser URL: `domo.com/datasources/{UUID}/details`

Where to find the ETL Dataflow ID:
- Open the dataflow in DOMO
- The ID is in the URL: `domo.com/dataflows/{ID}/details`

To disable a dataset without deleting it:
```sql
UPDATE DEVRP.REVPRO.DOMO_WRITEBACK_CONFIG SET IS_ACTIVE = FALSE WHERE DATASET_ID = 'xxxx...';
```

## Step 4 — Verify setup

**Check DOMO credentials:**
```powershell
python -c "
import os
from dotenv import load_dotenv
load_dotenv(r'C:\Users\saryan\.snowflake\domo-writeback\.env', override=False)
c = os.environ.get('DOMO_CLIENT_ID', '')
s = os.environ.get('DOMO_CLIENT_SECRET', '')
print('Credentials OK' if c and s and len(c) >= 10 else 'MISSING or too short')
"
```

**Check Snowflake config table has rows:**
```powershell
python -c "import snowflake.connector; conn=snowflake.connector.connect(connection_name='default'); cur=conn.cursor(); cur.execute('SELECT COUNT(*) FROM DEVRP.REVPRO.DOMO_WRITEBACK_CONFIG WHERE IS_ACTIVE=TRUE'); n=cur.fetchone()[0]; conn.close(); print(f'{n} active dataset(s) found' if n>0 else 'EMPTY — insert rows first')"
```

---

## Optional: Windows Credential Manager (stronger security)

If the script directory is on a shared or synced drive, store credentials in
the Windows encrypted vault instead of a plain `.env` file.

**One-time setup (run once in PowerShell):**

```powershell
pip install keyring
python -c "
import keyring
keyring.set_password('domo_writeback', 'client_id',     'your-actual-client-id')
keyring.set_password('domo_writeback', 'client_secret', 'your-actual-client-secret')
print('Stored in Windows Credential Manager')
"
```

**Update the script** to read from keyring instead of dotenv:

Replace the `load_dotenv` + `os.environ.get` block with:

```python
import keyring
CLIENT_ID     = keyring.get_password("domo_writeback", "client_id") or ""
CLIENT_SECRET = keyring.get_password("domo_writeback", "client_secret") or ""
```

Credentials are then stored encrypted in the Windows Credential vault,
not in any file on disk.
