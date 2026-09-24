---
name: ecp-schema
description: Research and troubleshoot Chainsea ECP/Aipower embedded MariaDB schema, metadata links, Unit/Menu/Page issues, and safe DB fixes.
triggers:
  - ecp schema
  - aipower db
  - unit 不存在
  - Unit does not exist
  - TsUnit
  - TsMenu
  - TsPage
  - TsRelation
argument-hint: "[uuid|menu name|unit/page/code/table]"
---

# ECP Schema Skill

## Purpose

Investigate Chainsea ECP/Aipower's embedded MariaDB schema and diagnose metadata problems such as `ID 為 "..." 的 Unit 不存在`, broken menus, missing pages, invalid relations, and orphaned configuration rows.

Use this skill for DB structure research and safe metadata troubleshooting. For starting/restarting the app or fixing MariaDB4j startup, use `ecp-server-startup` first.

## Key Local Facts

- App root: `C:\com\chainsea`
- Real persistent DB data dir: `C:\com\chainsea\mariadb\data`
- Normal app startup: `C:\com\chainsea\server.bat`
- Aipower webapp starts embedded MariaDB4j itself on an auto-picked free port, not usually `3306`.
- Standalone `C:\com\chainsea\mariadb\database.bat` is for maintenance and uses port `3306`; do not run it while `server.bat` is using the same data dir.
- Bundled clients usually exist at:
  - `C:\com\chainsea\apache-tomcat\temp\MariaDB4j\base\bin\mysql.exe`
  - `C:\com\chainsea\apache-tomcat\temp\MariaDB4j\base\bin\mysqldump.exe`
- Main application schema observed locally: `default`

## How To Find The Actual DB Port

The active Aipower DB port is visible in the embedded `mariadbd.exe` command line:

```powershell
Get-CimInstance Win32_Process -Filter "Name='mariadbd.exe' OR Name='mysqld.exe' OR Name='java.exe'" |
  Select-Object ProcessId,ParentProcessId,Name,CommandLine |
  Format-List
```

Look for:

```text
mariadbd.exe ... --datadir=C:\com\chainsea\mariadb\data --port=<PORT> ...
```

Then connect with:

```powershell
$port = 52651  # replace with discovered port
$mysql = 'C:/com/chainsea/apache-tomcat/temp/MariaDB4j/base/bin/mysql.exe'
& $mysql --protocol=TCP -h 127.0.0.1 -P $port -u root --password= -N -e "SHOW DATABASES;"
```

If `127.0.0.1:3306` fails with `ERROR 2002 ... 10061`, that only means the standalone maintenance DB is not running; it does not prove Aipower's embedded DB is down.

## Observed Schema Shape

Snapshot from local `default` schema on 2026-06-16:

- Tables: 485
- Columns: 5099
- Largest/highest-row metadata tables include:
  - `TsI18nText`
  - `TsI18nMapping`
  - `TsField`
  - `TsListField`
  - `TsTextResource`
  - `TsFormField`
  - `TsToolItem`
  - `TsDictionaryItem`
  - `TsPage`
  - `TsRelation`
  - `TsMenu`
  - `TsUnit`

### Table Prefixes

- `Ts*`: system metadata/configuration. This is where menus, pages, units, fields, relations, privileges, query schemas, forms, lists, tools, i18n mappings, dictionaries, reports, charts, and scripts live.
- `Tc*`: business/master data tables. Examples: `TcCustomer`, `TcCallLog`, `TcTask`, `TcServiceRequest`, `TcChatRoom`, `TcDocument`.
- `Tt*`: transaction/task/temp-like data. Examples: `TtPhonePlan`, `TtLastCall`, activity/service tracking tables.
- `Tw*`: workflow data. Examples: `TwWorkItem`, `TwNode`, `TwEvent`, workflow line/process data.
- `~lenus*`: English language/shadow metadata tables.
- `~lzhcn*`: Simplified Chinese language/shadow metadata tables.

Many metadata relationships are soft UUID references stored in varchar columns, not enforced by database foreign keys. Broken references can exist and cause runtime UI errors.

## Core Metadata Model

The most important runtime path is:

```text
TsMenu.FPageId
  -> TsPage.FId
  -> TsPage.FUnitId
  -> TsUnit.FId
  -> TsField.FUnitId / TsRelation.FUnitId1,FUnitId2 / TsQuerySchema.FUnitId / TsPrivilege.FUnitId
```

### Core Tables

#### `TsMenu`

Menu tree and UI navigation.

Important columns:

- `FId`: menu UUID
- `FParentId`: parent menu UUID
- `FIndex`, `FTreeLevel`, `FTreeSerial`: ordering/tree placement
- `FName`: visible menu name
- `FType`: often `Directory`, `InternalPage`, `Function`, or null
- `FPageId`: page opened by the menu
- `FQuerySchemaId`: optional query schema override
- `FArguments`: page/menu runtime arguments
- `FEnabled`, `FHideInMainMenu`: visibility flags
- `FCountUnitId`: Unit used for count badge metadata

#### `TsPage`

Page definitions.

Important columns:

- `FId`: page UUID
- `FCode`: unique page code, e.g. `Ecp.CallLog.List`
- `FTitle`: page title
- `FType`: page type
- `FUrl`, `FActionMethodName`, `FLoadHandler`: implementation hooks
- `FUnitId`: primary Unit for the page
- `FMasterUnitId`: master Unit for slave/detail pages
- `FRelationId`: relation for slave/detail pages
- `FQuerySchemaId`: default query schema
- `FEditId`: edit/form metadata

#### `TsUnit`

Entity/module metadata. A missing row here commonly causes `Unit 不存在`.

Important columns:

- `FId`: Unit UUID
- `FCode`: unique code, e.g. `Ecp.CallLog`
- `FName`: display/entity name, e.g. `通話記錄`
- `FModuleId`: module/category UUID-like value; no `TsModule` table exists in this local schema
- `FEditId`: default edit/form id
- `FTable`: backing business table, e.g. `TcCallLog`
- `FKeyField`: primary key field in backing table, usually `FId`
- `FNameField`: display/name field in backing table
- `FDataSource`: optional data source
- class hook columns: `FHomeClassName`, `FDaoClassName`, `FServiceClassName`, `FActionClassName`, `FApiClassName`

#### `TsField`

Fields/columns belonging to a Unit.

Important columns:

- `FId`: field UUID
- `FUnitId`: owner Unit UUID
- `FName`: field name, unique with `FUnitId`
- `FTitle`: UI title
- `FType`, `FSize`, `FVisible`, `FRequired`, `FReadOnly`, `FQueryable`
- `FEntityUnitId`: referenced Unit for entity/select fields
- `FEntityEditId`: edit metadata for referenced entity
- `FRelationId`: relation used by field
- `FDictionaryId`: dictionary metadata
- source/filter columns: `FSourceType`, `FSource`, `FSelectListFilterSql`, `FSelectListConstantFilterSql`, `FSelectListVariableFilterSql`

#### `TsRelation`

Soft relationship metadata between Units.

Important columns:

- `FId`: relation UUID
- `FOppositeId`: opposite-direction relation UUID
- `FName`, `FOppositeName`: directional names
- `FUnitId1`, `FUnitId2`: related Unit UUIDs
- `FType`: relation type
- `FTable`: join table if any
- `FField1`, `FField2`: fields used for joining
- delete/privilege columns: `FDeleteAction1`, `FDeleteAction2`, `FPrivilegeTypeId1`, `FPrivilegeTypeId2`

#### Other Important Metadata Tables

- `TsEdit`: edit/form definitions for Units.
- `TsForm`, `TsFormField`, `TsEditField`: form/edit layout and field placement.
- `TsList`, `TsListField`: list/grid layout.
- `TsQuerySchema`: saved/public query schemas, unit-specific filters and SQL.
- `TsPrivilege`, `TsRole`, `TsRoleMenu`: permissions and role-menu binding.
- `TsDictionary`, `TsDictionaryItem`: enumerations/dropdowns.
- `TsI18nText`, `TsI18nMapping`, `TsTextResource`: localization/resource mapping.
- `TsPageGroup`, `TsFieldGroup`, `TsIconMenuCatalog`, `TsReportCatalog`, `TsChartCatalog`: grouping/catalog metadata.

## Common Investigations

### 1. Menu -> Page -> Unit Trace

Use when a menu opens the wrong page, blank page, or `Unit 不存在`.

```powershell
$port = 52651
$name = '通話記錄'
$mysql = 'C:/com/chainsea/apache-tomcat/temp/MariaDB4j/base/bin/mysql.exe'
$sql = @"
SELECT
  m.FId AS menu_id,
  m.FName AS menu_name,
  m.FType AS menu_type,
  m.FPageId AS page_id,
  p.FCode AS page_code,
  p.FTitle AS page_title,
  p.FUnitId AS page_unit_id,
  u.FCode AS unit_code,
  u.FName AS unit_name,
  u.FTable AS unit_table
FROM `default`.TsMenu m
LEFT JOIN `default`.TsPage p ON p.FId = m.FPageId
LEFT JOIN `default`.TsUnit u ON u.FId = p.FUnitId
WHERE m.FName LIKE '%$name%'
ORDER BY m.FTreeSerial, m.FIndex;
"@
& $mysql --protocol=TCP -h 127.0.0.1 -P $port -u root --password= -t -e $sql
```

### 2. Find A Unit By Name/Code/Table

```sql
SELECT FId, FCode, FName, FTable, FKeyField, FNameField, FModuleId
FROM `default`.TsUnit
WHERE FName LIKE '%通話記錄%'
   OR FCode LIKE '%CallLog%'
   OR FTable LIKE '%CallLog%';
```

### 3. Trace A Page Code

```sql
SELECT p.FId, p.FCode, p.FTitle, p.FType, p.FUnitId, u.FCode, u.FName, u.FTable
FROM `default`.TsPage p
LEFT JOIN `default`.TsUnit u ON u.FId = p.FUnitId
WHERE p.FCode LIKE '%CallLog%';
```

### 4. Trace All References To A UUID

Use this first when an error contains a UUID.

```powershell
$uuid = 'e7218656-38c0-4c01-8f16-c29b5e356f9f'
$port = 52651
$mysql = 'C:/com/chainsea/apache-tomcat/temp/MariaDB4j/base/bin/mysql.exe'
$sql = @'
SELECT CONCAT(
  'SELECT ', QUOTE(TABLE_NAME), ' AS table_name, ', QUOTE(COLUMN_NAME),
  ' AS column_name, COUNT(*) AS hits FROM ', CHAR(96), TABLE_SCHEMA, CHAR(96), '.', CHAR(96), TABLE_NAME, CHAR(96),
  ' WHERE ', CHAR(96), COLUMN_NAME, CHAR(96), ' LIKE ', QUOTE(CONCAT('%', @uuid, '%')),
  ' HAVING hits > 0;'
)
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA='default'
  AND DATA_TYPE IN ('char','varchar','text','tinytext','mediumtext','longtext');
'@
$query = "SET @uuid='$uuid'; $sql"
$queries = & $mysql --protocol=TCP -h 127.0.0.1 -P $port -u root --password= -N -e $query
$full = "SET @uuid='$uuid';`n" + ($queries -join "`n")
$full | & $mysql --protocol=TCP -h 127.0.0.1 -P $port -u root --password= -N
```

### 5. Orphan Metadata Checks

Run after imports/upgrades or before diagnosing UI metadata errors.

```sql
SELECT 'orphan_page_unit' AS check_name, COUNT(*) AS hits
FROM `default`.TsPage p LEFT JOIN `default`.TsUnit u ON u.FId=p.FUnitId
WHERE p.FUnitId IS NOT NULL AND u.FId IS NULL;

SELECT 'orphan_menu_page' AS check_name, COUNT(*) AS hits
FROM `default`.TsMenu m LEFT JOIN `default`.TsPage p ON p.FId=m.FPageId
WHERE m.FPageId IS NOT NULL AND p.FId IS NULL;

SELECT 'orphan_field_unit' AS check_name, COUNT(*) AS hits
FROM `default`.TsField f LEFT JOIN `default`.TsUnit u ON u.FId=f.FUnitId
WHERE f.FUnitId IS NOT NULL AND u.FId IS NULL;

SELECT 'orphan_field_entity_unit' AS check_name, COUNT(*) AS hits
FROM `default`.TsField f LEFT JOIN `default`.TsUnit u ON u.FId=f.FEntityUnitId
WHERE f.FEntityUnitId IS NOT NULL AND u.FId IS NULL;

SELECT 'orphan_relation_unit1' AS check_name, COUNT(*) AS hits
FROM `default`.TsRelation r LEFT JOIN `default`.TsUnit u ON u.FId=r.FUnitId1
WHERE r.FUnitId1 IS NOT NULL AND u.FId IS NULL;

SELECT 'orphan_relation_unit2' AS check_name, COUNT(*) AS hits
FROM `default`.TsRelation r LEFT JOIN `default`.TsUnit u ON u.FId=r.FUnitId2
WHERE r.FUnitId2 IS NOT NULL AND u.FId IS NULL;

SELECT 'orphan_queryschema_unit' AS check_name, COUNT(*) AS hits
FROM `default`.TsQuerySchema q LEFT JOIN `default`.TsUnit u ON u.FId=q.FUnitId
WHERE q.FUnitId IS NOT NULL AND u.FId IS NULL;

SELECT 'orphan_privilege_unit' AS check_name, COUNT(*) AS hits
FROM `default`.TsPrivilege p LEFT JOIN `default`.TsUnit u ON u.FId=p.FUnitId
WHERE p.FUnitId IS NOT NULL AND u.FId IS NULL;
```

### 6. Inspect Orphan Details

```sql
SELECT p.FId, p.FCode, p.FTitle, p.FUnitId
FROM `default`.TsPage p LEFT JOIN `default`.TsUnit u ON u.FId=p.FUnitId
WHERE p.FUnitId IS NOT NULL AND u.FId IS NULL;

SELECT f.FId, f.FUnitId, owner.FCode AS owner_code, owner.FName AS owner_name,
       f.FName, f.FTitle, f.FEntityUnitId
FROM `default`.TsField f
LEFT JOIN `default`.TsUnit u ON u.FId=f.FEntityUnitId
LEFT JOIN `default`.TsUnit owner ON owner.FId=f.FUnitId
WHERE f.FEntityUnitId IS NOT NULL AND u.FId IS NULL;

SELECT r.FId, r.FName, r.FOppositeName, r.FUnitId1, u1.FCode AS unit1_code,
       r.FUnitId2, u2.FCode AS unit2_code, r.FField1, r.FField2
FROM `default`.TsRelation r
LEFT JOIN `default`.TsUnit u1 ON u1.FId=r.FUnitId1
LEFT JOIN `default`.TsUnit u2 ON u2.FId=r.FUnitId2
WHERE (r.FUnitId1 IS NOT NULL AND u1.FId IS NULL)
   OR (r.FUnitId2 IS NOT NULL AND u2.FId IS NULL);
```

## Safe Fix Workflow

Do not mutate first. ECP metadata is soft-linked and easy to make worse.

1. Find actual embedded DB port from `mariadbd.exe` command line.
2. Confirm schema:
   ```sql
   SHOW DATABASES;
   ```
3. Trace the bad UUID or menu/page/unit path with read-only SQL.
4. Decide the fix:
   - If a valid replacement Unit/Page exists and existing metadata already points to it, delete only the obsolete orphan rows.
   - If the Unit/Page is genuinely missing and should exist, restore it from a backup/shadow table/versioned SQL instead of hand-inventing a row.
   - If the menu points to the wrong page, update `TsMenu.FPageId` only after verifying the target `TsPage` and `TsUnit` exist.
5. Back up exact rows before any mutation with `mysqldump --where`.
6. Apply changes in a transaction.
7. Re-run orphan checks and UUID search.
8. Refresh/re-login/restart Tomcat if the UI still shows cached metadata.

### Backup Exact Rows Before Deleting Or Updating

```powershell
$port = 52651
$uuid = 'BAD-UUID-HERE'
$dump = 'C:/com/chainsea/apache-tomcat/temp/MariaDB4j/base/bin/mysqldump.exe'
$ts = Get-Date -Format 'yyyyMMdd_HHmmss'
$dir = 'C:/com/chainsea/backup'
if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Path $dir | Out-Null }

& $dump --protocol=TCP -h 127.0.0.1 -P $port -u root --password= \
  --no-create-info --skip-triggers --where="FEntityUnitId='$uuid'" default TsField \
  > "$dir/unit_fix_TsField_$ts.sql"

& $dump --protocol=TCP -h 127.0.0.1 -P $port -u root --password= \
  --no-create-info --skip-triggers --where="FUnitId1='$uuid' OR FUnitId2='$uuid'" default TsRelation \
  > "$dir/unit_fix_TsRelation_$ts.sql"
```

### Example: Fix Missing Obsolete Unit References

This is the pattern used for `e7218656-38c0-4c01-8f16-c29b5e356f9f` on 2026-06-16:

- `TsUnit` had 0 rows for the UUID.
- `TsField` still had 3 rows with `FEntityUnitId` pointing to it.
- `TsRelation` still had 6 rows with `FUnitId1`/`FUnitId2` pointing to it.
- Existing valid `Ecp.Customer / 企業` metadata already existed for `Ecp.CallLog` and `Ecp.LastCall`.
- Safe fix was to back up and delete obsolete broken `TsField`/`TsRelation` rows, not invent a new `TsUnit` row.

```sql
START TRANSACTION;
DELETE FROM `default`.TsField
WHERE FEntityUnitId='BAD-UUID-HERE';
SET @field_deleted = ROW_COUNT();

DELETE FROM `default`.TsRelation
WHERE FUnitId1='BAD-UUID-HERE' OR FUnitId2='BAD-UUID-HERE';
SET @relation_deleted = ROW_COUNT();

SELECT 'deleted' AS q, @field_deleted AS fields, @relation_deleted AS relations;
COMMIT;
```

Then verify:

```sql
SELECT 'remaining_TsField' AS q, COUNT(*) AS hits
FROM `default`.TsField WHERE FEntityUnitId='BAD-UUID-HERE';

SELECT 'remaining_TsRelation' AS q, COUNT(*) AS hits
FROM `default`.TsRelation WHERE FUnitId1='BAD-UUID-HERE' OR FUnitId2='BAD-UUID-HERE';
```

## Useful Inventory Queries

### Schema Size

```sql
SELECT 'table_count' AS metric, COUNT(*) AS value
FROM information_schema.TABLES
WHERE TABLE_SCHEMA='default';

SELECT 'column_count' AS metric, COUNT(*) AS value
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA='default';

SELECT TABLE_NAME, TABLE_ROWS,
       ROUND(DATA_LENGTH/1024/1024,2) AS data_mb,
       ROUND(INDEX_LENGTH/1024/1024,2) AS index_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA='default'
ORDER BY TABLE_ROWS DESC, DATA_LENGTH DESC
LIMIT 40;
```

### Table Prefix Groups

```sql
SELECT
  CASE
    WHEN TABLE_NAME LIKE 'Ts%' THEN 'Ts: system metadata/config'
    WHEN TABLE_NAME LIKE 'Tc%' THEN 'Tc: business/master data'
    WHEN TABLE_NAME LIKE 'Tt%' THEN 'Tt: transaction/temp/task data'
    WHEN TABLE_NAME LIKE 'Tw%' THEN 'Tw: workflow data'
    WHEN TABLE_NAME LIKE '~lenus%' THEN '~lenus: English i18n/shadow metadata'
    WHEN TABLE_NAME LIKE '~lzhcn%' THEN '~lzhcn: Simplified Chinese i18n/shadow metadata'
    ELSE 'other'
  END AS table_group,
  COUNT(*) AS tables,
  SUM(TABLE_ROWS) AS approx_rows,
  ROUND(SUM(DATA_LENGTH+INDEX_LENGTH)/1024/1024,2) AS total_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA='default'
GROUP BY table_group
ORDER BY tables DESC;
```

### Columns For Any Core Table

```sql
SELECT TABLE_NAME, COLUMN_NAME, COLUMN_TYPE, IS_NULLABLE, COLUMN_KEY, COLUMN_DEFAULT, EXTRA
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA='default'
  AND TABLE_NAME IN ('TsUnit','TsField','TsRelation','TsMenu','TsPage','TsEdit','TsQuerySchema','TsRoleMenu','TsPrivilege','TsAccount','TsRole')
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

### Indexes For Core Tables

```sql
SELECT TABLE_NAME, INDEX_NAME, NON_UNIQUE, SEQ_IN_INDEX, COLUMN_NAME
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA='default'
  AND TABLE_NAME IN ('TsUnit','TsField','TsRelation','TsMenu','TsPage','TsEdit','TsQuerySchema','TsRoleMenu','TsPrivilege','TsAccount','TsRole')
ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;
```

## System Parameter Metadata Model (TsParameterDefinition / TsSystemParameter / TsParameterGroup)

System-wide settings (Session timeout, license/crowded rules, password policy, CTI, chat...)
do **not** follow the Menu→Page→Unit path above. They render inside a single fixed screen
(`系統管理 > 參數管理 > 系統參數`) whose tabs/sections are driven by a separate self-referencing
tree table. Use this path whenever the symptom is "where in the UI do I toggle/see parameter X"
or "what does DB key X mean/do".

```text
TsSystemParameter.FKey (actual stored value, 1 row per overridden parameter)
  -> TsParameterDefinition.FCode (matches FKey; defines type/required/UI row)
  -> TsParameterDefinition.FParameterGroupId
  -> TsParameterGroup.FId (a 區塊/section header, e.g. "Session 設定")
  -> TsParameterGroup.FParentId -> top-level TsParameterGroup (FTreeLevel=1, FParentId IS NULL)
     = the visible **tab** on the 系統參數 screen (e.g. 基本設定/登入設定/客戶關係/其他服務)
```

- `TsSystemParameter`: `FKey` (varchar, = `TsParameterDefinition.FCode`), `FValue` (varchar, stored as
  text even for CheckBox: `'1'`/`'0'`). **No row here = parameter is unset**, the app falls back to a
  hardcoded Java default (not visible in DB) — don't assume "no row" means off/false without checking.
- `TsParameterDefinition`: `FCode` (stable key used everywhere, e.g. `QsSessionTimeout`), `FName`
  (Chinese label shown as the field caption), `FType` (`CheckBox`, `InputBox-Integer`,
  `InputBox-Text`, `ComboBox-SelectOnly`, ...), `FScale` = `'System'`/`'Tenant'`/`'Account'`/... scope.
- `TsParameterGroup`: tree via `FParentId`/`FTreeLevel`/`FTreeSerial`. Level 1 rows (no parent) are
  tabs; level 2 rows are the collapsible section headers within a tab.

### Trace any parameter code to its screen location

```sql
SELECT pd.FCode, pd.FName AS field_label, pd.FType, g.FName AS section_group, top.FName AS tab_name
FROM `default`.TsParameterDefinition pd
JOIN `default`.TsParameterGroup g ON g.FId = pd.FParameterGroupId
LEFT JOIN `default`.TsParameterGroup top
  ON top.FId = (CASE WHEN g.FTreeLevel = 1 THEN g.FId ELSE g.FParentId END)
WHERE pd.FCode IN ('QsOneSessionPerUser','QsCrowdedSessionTimeout','QsSessionTimeout');
```

Result confirms all three live under tab **登入設定** → section **Session 設定**.

### Read/write current values

```sql
-- current effective value (absent row = using hardcoded default, check TsParameterDefinition.FType to know what "off" looks like)
SELECT * FROM `default`.TsSystemParameter WHERE FKey='QsCrowdedSessionTimeout';

-- set/override a value (INSERT if no row exists yet; CheckBox uses '1'/'0' strings)
INSERT INTO `default`.TsSystemParameter (FId, FKey, FValue)
VALUES ('35a46b93-fa4e-4b74-836b-e689a9e6b3c0', 'QsOneSessionPerUser', '1')
ON DUPLICATE KEY UPDATE FValue='1';
```

Changing via raw SQL takes effect after a Tomcat restart (parameters are cached at startup); changing
via the UI screen itself + clicking 保存 is the safer path and usually applies without a restart.

### Related monitoring tables and their UI location

Quick reference for tables that came up while chasing session/login behavior — these are plain
`Ts*` operational tables with a dedicated monitor screen, not Unit-driven CRUD pages:

| Table | UI location | Notes |
|---|---|---|
| `TsOnlineUser` | `系統管理 > 系統監控 > 線上使用者` | One row per live session (`FSessionId`/`FTokenId`). Toolbar "踢出" button force-disconnects a selected row. |
| `TsLoginLog` | `系統管理 > 系統監控 > 登入日誌` | Audit trail; `FAction` values seen: `Login`, `Logout`, `Timeout`, `LoginFalure` (sic), `ReloginOut` (kicked by a newer login under `QsOneSessionPerUser`). |
| `TsSystemParameter` / `TsParameterDefinition` | `系統管理 > 參數管理 > 系統參數` (values) / `... > 參數定義` (definitions/labels) | See model above. |

## Employee / Account / Identity / Duty Metadata Model

Creating one 員工 (Employee) + 帳號 (Account) from the UI (`基礎設定 > 組織架構 > 員工`) touches
4 tables. Verified by diffing `TsAccount` before/after creating 3 employees on 2026-07-02.

```text
TsAccount (login credentials: FLoginName/FPassword/FEmail, 1 row per login)
  │ FAccountId
  ▼
TsUser  (the person's profile: FName/FDepartmentId/FEmail/FMobile...)
  │  ⚠ 職務 (Duty/position) is NOT a separate table — a Duty is just another TsUser row,
  │    distinguished only by which TsAccountIdentity rows point at it, not by FIsDuty
  │    (observed NULL/unset even for a row used as a Duty in this dataset — don't rely on it).
  │
  ├─ TsAccountIdentity (FIdentityTypeId='員工', FEntityId → this TsUser row itself)
  ├─ TsAccountIdentity (FIdentityTypeId='職務', FEntityId → the assigned Duty's TsUser row)
  └─ TsEmployeeDuty    (FEmployeeId → this TsUser row, FDutyId → the assigned Duty's TsUser row)
```

- `TsAccount`: `FId`, `FName`, `FLoginName`, `FPassword` (hashed), `FEmail`, `FCreateTime`, `FEnabled`.
- `TsUser`: backs both "員工" and "職務" UI concepts. Key columns: `FDepartmentId`, `FAccountId`
  (FK back to `TsAccount`), `FManagerId`, `FTitle`, `FEmail`. **No `FCreateTime` column** — to find
  recently-created employees, join through `TsAccount.FCreateTime` via `FAccountId` instead.
- `TsAccountIdentity`: the actual "身份" (Identity) rows a login screen shows in 身份 selector — one
  account can hold multiple identities (e.g. one 員工 identity + one or more 職務 identities).
  `FIdentityTypeId` resolves via `TsIdentityType` (dictionary-like: 員工/職務/...).
- `TsEmployeeDuty`: pure link table, `FEmployeeId`/`FDutyId` both point into `TsUser`.

### Find what a UI-created employee touched

```sql
-- newest accounts (employee creation always creates one of these)
SELECT FId, FName, FLoginName, FEmail, FCreateTime FROM `default`.TsAccount
ORDER BY FCreateTime DESC LIMIT 10;

-- the linked employee profile + department
SELECT u.FId, u.FName, u.FDepartmentId, u.FEmail, u.FAccountId
FROM `default`.TsUser u WHERE u.FAccountId = '<account FId from above>';

-- all identities (员工/职务/...) granted to that account
SELECT ai.FId, ai.FName, it.FName AS identity_type, ai.FEntityId, ai.FIsMainDuty
FROM `default`.TsAccountIdentity ai
LEFT JOIN `default`.TsIdentityType it ON it.FId = ai.FIdentityTypeId
WHERE ai.FAccountId = '<account FId>';

-- employee/duty assignment link rows
SELECT * FROM `default`.TsEmployeeDuty WHERE FEmployeeId = '<TsUser FId>';
```

## Root Cause: A Whole Module Is Not Installed

`Unit 不存在` is often not a single broken row but a **whole module missing** from this
deployment. A reduced build can ship one module (e.g. `aipower.base`, which includes
`Ecp.CallLog`) while excluding another (`aipower.crm`, which defines `Ecp.TargetCustomer`,
`Ecp.TargetCustomerResult`, `Ecp.Lead`, `Ecp.ServiceRequest`, `Ecp.ProductCatalog`...).
The shipped module's upgrade SQL still inserts cross-module fields/relations (via
`runOnEmpty`, keyed by each row's own `FId`), so they become orphans pointing at units the
DB never got. Deleting those orphans is NOT durable — the next startup re-inserts them
(the `runOnEmpty` guard sees the `FId` gone). Re-pointing (`UPDATE FEntityUnitId`) survives
because the `FId` is unchanged. The proper fix is to **install the missing module**, not
hand-insert `TsUnit` rows (which fail at runtime because the module's Home/Dao/Service
classes are absent).

Confirm the diagnosis:

```sql
-- which modules are installed
SELECT FModuleName, FVersion FROM `default`.TsModuleDataVersion;   -- e.g. only quicksilver.main + aipower.base
SELECT FName, FMajorVersion FROM `default`.TsModuleVersion;
-- all orphan entity fields (their owners reveal which feature needs the missing module)
SELECT owner.FCode AS owner, f.FName, f.FTitle, f.FEntityUnitId AS missing_unit
FROM `default`.TsField f
LEFT JOIN `default`.TsUnit u ON u.FId=f.FEntityUnitId
LEFT JOIN `default`.TsUnit owner ON owner.FId=f.FUnitId
WHERE f.FEntityUnitId IS NOT NULL AND f.FEntityUnitId<>'' AND u.FId IS NULL;
```

Check the install package modules (each module = a folder with `application/webroot/WEB-INF/lib/<module>.jar`);
grep the jars for `com/chainsea/ecp/<feature>/` classes to find which jar defines the missing units.

### Installing a missing Quicksilver module

1. Copy the module jar (e.g. `aipower-module-crm-7.3.12.5.jar`) into the **running webapp**
   `apache-tomcat/webapps/aipower/WEB-INF/lib/`. Verify its `module.xml` dependency
   (`aipower.crm` depends on `aipower.base >= x`) is satisfied.
2. **Dropping the jar is NOT enough.** Module discovery scans all classpath `QS-MODULE/module.xml`
   (`QsModuleManager.initialize` → `ClasspathUtil.getResourceTrees("QS-MODULE")`), but the
   SQL init phase only runs when `qs.autoinit.enabled=true`. `AutoInitializer` reads
   `Boolean.parseBoolean(System.getProperty("qs.autoinit.enabled"))` → **null/unset = false =
   no module SQL runs at all**. So start ONCE with the flag on.
3. `server.bat` overwrites `CATALINA_OPTS`, so inject via `JAVA_OPTS` instead (catalina.bat
   execs `%JAVA_OPTS% %CATALINA_OPTS%` and only appends to JAVA_OPTS):
   ```powershell
   $cmd = 'set "JAVA_OPTS=-Dqs.autoinit.enabled=true" && C:\Lab\chainsea\server.bat > C:\Temp\crm_install.log 2>&1'
   Start-Process cmd.exe -ArgumentList '/c',$cmd -WorkingDirectory 'C:\Lab\chainsea' -WindowStyle Hidden
   ```
4. Success log: `SQL modules: [..., aipower.crm]` → `Run SQL file: init.sql ...` + version
   scripts → a new `TsModuleDataVersion` row for the module. It creates the module's units,
   tables, and menus. Later normal `server.bat` boots (no flag) won't re-run it — version is
   recorded. **Always full-DB `mysqldump` before this step** (it runs a lot of SQL).
5. Re-point fixes you made earlier (orphan `FEntityUnitId` → some stand-in unit) may need
   reverting so they point at the real restored unit; the migration is `FId`-gated and won't
   fix them for you.

This was used to restore `aipower.crm` on `C:\Lab\chainsea` (2026-06-30) where 通話記錄
errored on `e7218656`/`883eb4ae`. After install, only base-side missing backing tables
remained (next section).

## Repair Pattern: Missing Backing Table For A Unit

Use this when a menu/page resolves cleanly to a `TsUnit`, but the generated SQL fails with `ERROR 1146 ... Table 'default.<table>' doesn't exist`.

Example observed on 2026-06-16:

- Menu: `聯絡人`
- Page: `Ecp.Contact.List`
- Unit: `Ecp.Contact / 聯絡人`
- Unit id: `00000000-0000-0000-0001-020000001002`
- Backing table: `TcContact`
- Symptom: list SQL selected from `TcContact`, but `information_schema.TABLES` had no `TcContact` row.

### Verify The Unit/Table Mismatch

```sql
SELECT m.FName AS menu_name, p.FCode AS page_code, u.FCode AS unit_code,
       u.FName AS unit_name, u.FTable AS unit_table
FROM `default`.TsMenu m
LEFT JOIN `default`.TsPage p ON p.FId=m.FPageId
LEFT JOIN `default`.TsUnit u ON u.FId=p.FUnitId
WHERE m.FName LIKE '%聯絡人%';

SELECT COUNT(*) AS table_exists
FROM information_schema.TABLES
WHERE TABLE_SCHEMA='default' AND TABLE_NAME='TcContact';
```

### Safe Empty-Table Creation Pattern

If the feature should remain available and there is no source table to restore, create an empty table from `TsField` metadata. For wide Units, do not translate every string field into large `varchar`; MariaDB/InnoDB can fail with row-size errors such as `ERROR 1118`. Use:

- `FId varchar(36) NOT NULL PRIMARY KEY`
- numeric/date/boolean fields as their native DB types
- most string/entity/multiselect/free-text fields as `longtext`
- `ROW_FORMAT=DYNAMIC`
- add any list-only fields missing from `TsField`, such as `FChannel` or display helper columns, before testing list SQL

After creating the table, verify:

```sql
SHOW TABLES LIKE 'TcContact';
SELECT COUNT(*) AS contact_rows FROM `default`.TcContact;

SELECT 'contact_missing_list_columns' AS check_name, COUNT(*) AS hits
FROM (
  SELECT lf.FFieldName
  FROM `default`.TsList l
  JOIN `default`.TsListField lf ON lf.FListId=l.FId
  JOIN `default`.TsUnit u ON u.FId=l.FUnitId
  LEFT JOIN information_schema.COLUMNS c
    ON c.TABLE_SCHEMA='default' AND c.TABLE_NAME=u.FTable AND c.COLUMN_NAME=lf.FFieldName
  WHERE u.FCode='Ecp.Contact' AND c.COLUMN_NAME IS NULL
) x;

SELECT 'contact_missing_form_columns' AS check_name, COUNT(*) AS hits
FROM (
  SELECT ff.FFieldName
  FROM `default`.TsForm f
  JOIN `default`.TsFormField ff ON ff.FFormId=f.FId
  JOIN `default`.TsUnit u ON u.FId=f.FUnitId
  LEFT JOIN information_schema.COLUMNS c
    ON c.TABLE_SCHEMA='default' AND c.TABLE_NAME=u.FTable AND c.COLUMN_NAME=ff.FFieldName
  WHERE u.FCode='Ecp.Contact' AND c.COLUMN_NAME IS NULL
) x;
```

## Gotchas

- `default` is a reserved-ish identifier; always wrap it as `` `default` `` in SQL.
- PowerShell treats backticks as escapes. In PowerShell here-strings, prefer generating SQL backticks with `CHAR(96)` for dynamic SQL, or put SQL in single-quoted here-strings carefully.
- Many relationships are soft links; `information_schema.KEY_COLUMN_USAGE` may show few or no foreign keys even when metadata links are required by the app.
- `~lenus*` and `~lzhcn*` shadow/i18n tables can still contain old UUIDs. Do not edit them first; fix active `Ts*` metadata unless the runtime is proven to read the shadow table.
- If UI still errors after DB fix, close the tab, logout/login, or restart Tomcat because metadata can be cached.
- Do not delete `C:\com\chainsea\mariadb\data`; it is the real database.
- Do not run standalone `database.bat` while `server.bat` is running; they compete for the same data dir.

## When To Ask Before Mutating

Always ask the user before executing `UPDATE`, `DELETE`, `INSERT`, schema changes, process kills, or DB restarts. Before mutation, show:

- the exact symptom and UUID/menu/page being fixed,
- the rows/tables affected,
- backup file paths,
- the planned SQL at a high level,
- rollback approach if available.

---

## Conformance Addendum

## When to Use
Research and troubleshoot Chainsea ECP/Aipower embedded MariaDB schema, metadata links, Unit/Menu/Page issues, and safe DB fixes.

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Procedure
1. Confirm the current environment and read the task-specific instructions already documented in this Skill.
2. Apply the smallest safe change that satisfies the request; confirm before destructive operations.
3. Run the checks in the Verification section and report actual results.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.

## Pitfalls
- Do not guess configuration paths or claim success without checking the resulting state.
- Do not execute copied commands or scripts before reviewing their targets and side effects.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
