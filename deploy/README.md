# deploy/

Deployment assets for the Windows VPS (VPS-Plan2: 2 vCPU / 4 GB / 50 GB, Windows Server 2022).

## 1. `DB_Pangya.bak` — original project database (phase 0 ✅)

Recovered from this repo's own git history:

```bash
git show a9e9568:Src/Py_DataBase/DB_Pangya.bak > deploy/DB_Pangya.bak
```

| Field | Value |
|---|---|
| Size | 4,313,088 bytes |
| SHA-256 | `a8a5b283e865bddfb1a8b790f6bd117660c94fe9a9d4b87bcabddc90ea8db615` |
| Backup label | `DB_Pangya-Completo Banco de Dados Backup` |
| Source instance | `CLEUDIANA\SQL` (SQL Server 2017, MSSQL14) |
| Logical files | `DB_Pangya` (mdf), `DB_Pangya_log` (ldf) |
| Original paths | `C:\Program Files\Microsoft SQL Server\MSSQL14.SQL\MSSQL\DATA\` |

> The commit `a9e9568` ("DataBase Clear !") truncated row data (46 MB → 4.3 MB),
> so expect schema, stored procedures and seed/config tables — not player data.

## 2. Restore runbook (VPS phase 2)

Requires **SQL Server Express 2017 or newer** + Mixed Mode auth enabled.
The database must be restored under the name **`pangya`** — the code calls
procs with three-part names (e.g. `pangya.ProcRegisterLogonServer`).

```sql
-- a) verify logical file names
RESTORE FILELISTONLY
FROM DISK = 'C:\deploy\DB_Pangya.bak';

-- b) restore as 'pangya', relocating files to the local DATA dir
RESTORE DATABASE pangya
FROM DISK = 'C:\deploy\DB_Pangya.bak'
WITH
  MOVE 'DB_Pangya'     TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.SQLEXPRESS\MSSQL\DATA\pangya.mdf',
  MOVE 'DB_Pangya_log' TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.SQLEXPRESS\MSSQL\DATA\pangya.ldf';
-- (adjust MSSQL16.SQLEXPRESS path to the installed instance version)

-- c) sanity checks
SELECT name FROM sys.databases;                               -- 'pangya' present
SELECT name FROM pangya.sys.procedures WHERE name LIKE 'Proc%';  -- procs exist
SELECT COUNT(*) FROM pangya.sys.tables;                       -- tables exist

-- d) login for the servers (Mixed Mode auth must be on)
USE pangya;
CREATE LOGIN pangya_app WITH PASSWORD = '<STRONG_PASSWORD>', CHECK_POLICY = ON;
CREATE USER pangya_app FOR LOGIN pangya_app;
EXEC sp_addrolemember 'db_datareader', 'pangya_app';
EXEC sp_addrolemember 'db_datawriter', 'pangya_app';
GRANT EXECUTE TO pangya_app;
```

## 3. ODBC DSN (code detail — important)

Both `mysql.cs` and `mssql.cs` connect via **ODBC** and build the connection
string as `DSN={DBIP};DATABASE={db_name};UID={user};PWD={pass};` — therefore
the `DBIP` field in each `server.ini` is an **ODBC System DSN name**, not a host.

On the VPS:

1. Install **Microsoft ODBC Driver 17 or 18 for SQL Server**.
2. `odbcad32.exe` (64-bit) → System DSN → **Add** → ODBC Driver 17/18 for SQL Server:
   - Name: `pangyax64` (matches current server.ini `DBIP`, no ini edit needed)
   - Server: `localhost\SQLEXPRESS` (or the instance name)
   - Authentication: **do not** enable trusted connection and do not store credentials in the DSN — the connection string supplies `UID`/`PWD` at runtime from `server.ini` (`DSN=DBIP;DATABASE=DBNAME;UID=DBUSER;PWD=DBPASS`)
   - Default database: leave blank (`DATABASE=pangya` comes from the connection string) or set to `pangya`.
3. In both `server.ini` set:

```ini
[NORMAL_DB]
DBENGINE = MSSQL
DBIP     = pangyax64
DBNAME   = pangya
DBUSER   = pangya_app
DBPASS   = <STRONG_PASSWORD>
```

## 4. Known mismatches to fix in the repo (runtime)

| Issue | Location | Fix |
|---|---|---|
| IFF filename mismatch: code expects `data/pangya_gb.iff`, repo ships `data/pangya_jp.iff` | `PangyaAPI/PangyaAPI.IFF.GB/Extensions/IFFHandle.cs:76` | rename the data file or change the constant |
| `[AUTHSERVER] 127.0.0.1:7777` has no listener; LoginServer listens on `10103` | `GameServer/server.ini` | set `[AUTHSERVER] PORT = 10103` (verify at first boot from logs) |
| `LoginServer.cs:53` calls `CommandDB.GetAuthKeyGame(14641, 20202)` | DB seed | ensure auth-key rows exist for server UIDs `10103`/`20202` |
