---
name: chainsea-migration
description: Compare two Chainsea/ECP/aipower deployments, migrate a single bounded feature between them (without wholesale-copying jars or databases), and verify the result across startup, DB, UI, permissions, resources, logs, and integrations. Covers the full compare → migrate → verify workflow.
triggers:
  - chainsea migration
  - ecp migration
  - aipower migration
  - migrate feature
  - 搬功能
  - 功能遷移
  - chainsea compare
  - ecp compare
  - aipower compare
  - project migration report
  - 功能互移
  - verify chainsea migration
  - verify ecp migration
  - verify aipower migration
  - migration verification
  - 驗證遷移
argument-hint: "<feature-name> from <source-dir> to <target-dir>"
---

# Chainsea / ECP / aipower Feature Migration

End-to-end skill for moving one bounded feature between Chainsea / ECP / aipower
deployments. Three phases, run in order:

1. **Compare** the two deployments to judge feasibility (Part A).
2. **Migrate** the single feature by package, not by whole app (Part B).
3. **Verify** startup, DB, UI, permissions, resources, logs, integrations (Part C).

Golden rule for all phases: **migrate by feature package, never wholesale-copy
`WEB-INF\lib`, `mariadb\data`, `webapps\<app>`, or `extension\<app>\config`.**

---

# Part A — Compare two deployments (feasibility)

## When to activate

Comparing `C:\com\chainsea`, `C:\chainsea\ecp`, or similar deployment folders;
moving features between `aipower` and `ecp`; checking whether two ECP packages
are compatible; producing a migration feasibility report.

## Workflow

1. Confirm both directories exist and stay read-only.
2. Compare top-level layout: `apache-tomcat`, `jre`/`jdk`, `mariadb`, `redis`,
   `tool`, `backup`, `server.bat`, `HttpLogin.url`.
3. Identify webapp context names:
   - `apache-tomcat\webapps\<app>`
   - `apache-tomcat\extension\<app>`
   - `ROOT\index.jsp` redirect
   - `HttpLogin.url`
   - Tomcat connector ports in `conf\server.xml`
4. Compare startup behavior:
   - Whether `server.bat` starts Redis
   - Whether Tomcat context XML exists under `conf\Catalina\localhost`
   - Whether Redisson session manager is enabled
5. Compare config files under `extension\<app>\config`: `application.properties`,
   `datasource.xml`, `cache-caffeine.xml`, `cache-redis.xml`, `redis.xml`,
   custom files such as `cbm-lite.properties`.
6. Compare JARs under `WEB-INF\lib`:
   - Main product module (`aipower-module-*`, `ecp-module-*`)
   - Internal module metadata such as `QS-MODULE/module.xml` when available
   - Module `depends`, `formerNames`, and file operations
   - `quicksilver-module-main-*`
   - `quicksilver-lib-*`
   - MariaDB4j / local driver
   - Redisson Tomcat libs
   - large third-party dependency families
7. Compare runtime versions: `apache-tomcat\RELEASE-NOTES`, `jre\release`,
   `jdk\release`, `cache-caffeine.xml` TTL, `redisson.yaml` and Redisson jar versions.
8. Compare embedded MariaDB schema shape without modifying data:
   `mariadb\data\default`, count `.ibd`/`.frm` files, sample source-only and
   target-only table names.
9. Compare extension resources: `attachment`, `i18n`, `image`, `report`, `temp`.
10. Produce a Chinese report with: similarity level, hard differences, portable
    feature categories, non-portable components, safe migration sequence,
    verification checklist.

## Key heuristics

- Same directory skeleton ≠ module compatibility.
- Same `quicksilver-module-main` ≠ same `quicksilver-lib` compatibility.
- `WEB-INF\lib` should never be wholesale copied.
- `mariadb\data` should never be wholesale copied as a feature migration method.
- A function is portable only after its DB metadata, business tables, Java classes,
  config, resources, and permissions are all identified.

## Evidence to capture

Cite concrete paths/versions: `WEB-INF\lib\ecp-module-main-*.jar`,
`WEB-INF\lib\aipower-module-base-*.jar`, `WEB-INF\lib\quicksilver-lib-basic-*.jar`,
`extension\<app>\config\datasource.xml`, `conf\Catalina\localhost\<app>.xml`,
`server.bat`.

## Gotchas (compare)

- `aipower` may have Redis + Redisson session manager enabled even if
  `application.properties` uses Caffeine cache.
- `ecp` may contain Redisson jars/config but not actually enable Redisson.
- `datasource.xml` can contain extra datasource IDs like `vrm`; reports/DAO may depend on them.
- `extension\<app>\temp` and logs are runtime artifacts, not migration source.

---

# Part B — Migrate one feature

## Rule

Migrate by feature package, not by whole application package. Do not start by
copying whole `WEB-INF\lib`, whole `mariadb\data`, whole `webapps\<app>`, or
whole `extension\<app>\config`.

## Workflow

### 1. Freeze, back up, define rollback

Confirm backups exist or create test copies. Minimum backup targets: source and
target root folders, `mariadb\data`, `apache-tomcat\webapps\<app>\WEB-INF\lib`,
`apache-tomcat\extension\<app>\config`, `...\attachment`, `...\i18n`, `...\image`,
`...\report`.

Define rollback before editing: which DB dump/folder copy restores the target DB,
which copied JARs must be removed, which config files must be restored, which
extension resources must be removed/restored, what startup/login checks prove
rollback succeeded.

### 2. Define one feature boundary

Document: feature name, source URL/menu path, target location, unit code,
Page/Form/List/Edit/Report codes, business table names, required roles/privileges,
external integrations, expected golden-path behavior. If the user cannot name the
boundary, inspect menu/page/unit metadata first and propose a minimal unit.

### 3. Inventory dependencies (walk the metadata graph)

```text
TsMenu.FPageId
  -> TsPage.FId
  -> TsPage.FUnitId / FMasterUnitId / FRelationId
  -> TsUnit.FId
  -> TsUnit.FTable / FDataSource / class hook fields
  -> TsField.FUnitId / FEntityUnitId / FRelationId / FDictionaryId
  -> TsRelation.FUnitId1 / FUnitId2 / FTable
  -> TsQuerySchema.FUnitId
  -> TsPrivilege.FUnitId
  -> TsRoleMenu.FMenuId
  -> TsList / TsListField
  -> TsForm / TsFormField
  -> TsEdit / related edit fields if present
```

Identify:
- Metadata tables: `TsMenu`, `TsPage`, `TsPageGroup`, `TsPageSet`, `TsPageSetItem`,
  `TsUnit`, `TsForm`, `TsEdit`, `TsField`, `TsList`, `TsReport`, `TsDictionary`,
  `TsDictionaryItem`, `TsRole`, `TsPrivilege`, `TsTextResource`
- Business tables, usually `Tc*`, `Tp*`, `Tr*`, or module-specific names
- Extension resources: report templates, images, attachments, i18n resources
- Config: datasource IDs (`default`, `vrm`), Redis/Redisson/session needs,
  `conf\Catalina\localhost\<app>.xml`, `conf\redisson.yaml`, cbm-lite/LINE/webhook
- Sensitive settings to reconfigure on target (not blindly copy): `license.lic`,
  OAuth tokens, LINE channel secret/access token, webhook secrets, encrypted
  datasource credentials, API keys, admin/user passwords
- Java dependencies: action/service/DAO class names, owning module JAR, third-party
  jars only if proven required

### 4. Prefer SQL export/import over file-level DB copying

Use SQL-level migration for metadata and business rows. Avoid copying `.ibd`,
`.frm`, `ibdata1`, or the whole `mariadb\data` folder.

### 5. Resolve identity conflicts

Check whether codes/IDs already exist or collide; prefer stable business codes;
preserve logical FK references between metadata rows; map menu parent/location
intentionally; recreate role/privilege links for the target roles instead of
overwriting target permissions.

### 6. Add resources

Copy only feature-owned resources (specific report files, images/icons, attachment
dirs, i18n files/rows). Do not overwrite target logos/license/config unless required.

### 7. Handle JAR/class issues last

Only add/change JARs after logs prove missing classes/methods
(`ClassNotFoundException`, `NoClassDefFoundError`, `NoSuchMethodError`, bean/action/
service not found). When JAR work is required: identify class owner, prefer a
product-compatible module version, avoid mixing `quicksilver-lib-*` versions, add
one JAR/group at a time, restart and verify after each change.

### 8. Update context-specific values

Adapt `/aipower` vs `/ecp`; 22821/22822 vs 12821/12822 or current target ports;
webhook/callback URLs; datasource IDs; Redis host/port; extension path assumptions.

### 9. Verify the feature

Proceed to Part C.

## Migration decision matrix

| Feature type | Approach |
|---|---|
| Menu/Page/Unit/Form/List only | DB metadata + permissions |
| CRUD backed by new table | DDL + metadata + seed rows + permissions |
| Report | metadata + report file + SQL/datasource validation |
| i18n/display text | metadata/text rows + i18n resources |
| Attachment/image | metadata reference + exact resource files |
| cbm-lite/LINE | config + tables + webhook + Redis/session + logs |
| Java-backed service | metadata + tables + compatible module/JAR dependency |

## Gotchas (migrate)

- `aipower` may have a `vrm` datasource that does not exist in `ecp`.
- `aipower` may have cbm-lite files absent from `ecp`.
- `ecp` has many tables absent from `aipower` (helpdesk/knowledge/AI chat/campaign/
  contract/invoice/lead domains).
- Metadata rows can reference classes inside product modules; UI metadata alone may
  be insufficient.
- A page opening ≠ save/delete/report/permission working.

---

# Part C — Verify the migration

## When to activate

After importing metadata rows, adding business tables/rows, copying extension
resources, adding/changing JARs, modifying datasource/cache/Redis/cbm-lite config,
or changing menu/page/unit/report settings.

## Verification workflow

### 1. Pre-start checks

Target is a test copy unless production approved; ports don't collide; `server.bat`
matches target runtime; `datasource.xml` has every datasource ID the feature uses;
if Redisson enabled, Redis server/config exists; if cbm-lite/LINE, callback URL and
signature config are target-specific.

### 2. Startup verification

Start with the project's expected command. Watch Tomcat console, `apache-tomcat\logs`,
`apache-tomcat\extension\<app>\log`, MariaDB/Atomikos errors, Redis errors.
Blockers: `ClassNotFoundException`, `NoClassDefFoundError`, `NoSuchMethodError`, SQL
table/column missing, datasource not found, license/config fatal, Atomikos pool
startup failure, MariaDB data directory failure.

### 3. Login and shell checks

Root redirect points to correct context; direct URL opens (`/aipower` or `/ecp`);
login works; main frame loads; no 404/500 loop; session survives navigation (no
Redisson serialization errors if enabled).

### 4. Menu and permission checks

Migrated feature appears under intended menu; visible to intended roles; hidden from
others; correct CRUD/report/export permissions; does not displace unrelated target
menu items.

### 5. UI golden path

Per page/unit: open, default query, search/filter, open detail/form, create test
record if safe, edit if safe, delete/rollback if safe, export/report if applicable,
confirm i18n labels and images/icons/attachments render.

### 6. Database verification

Target DB has required metadata rows, business tables, seed rows; no duplicate code
collisions; no orphaned menu/page/unit/form/list/report references; no source-only
datasource IDs unless intentional; no hard-coded source context path/port.

Read-only orphan checks:

```sql
SELECT COUNT(*)
FROM `default`.TsMenu m
LEFT JOIN `default`.TsPage p ON p.FId = m.FPageId
WHERE m.FPageId IS NOT NULL AND p.FId IS NULL;

SELECT COUNT(*)
FROM `default`.TsPage p
LEFT JOIN `default`.TsUnit u ON u.FId = p.FUnitId
WHERE p.FUnitId IS NOT NULL AND u.FId IS NULL;

SELECT COUNT(*)
FROM `default`.TsField f
LEFT JOIN `default`.TsUnit u ON u.FId = f.FUnitId
WHERE f.FUnitId IS NOT NULL AND u.FId IS NULL;

SELECT COUNT(*)
FROM `default`.TsRelation r
LEFT JOIN `default`.TsUnit u1 ON u1.FId = r.FUnitId1
LEFT JOIN `default`.TsUnit u2 ON u2.FId = r.FUnitId2
WHERE (r.FUnitId1 IS NOT NULL AND u1.FId IS NULL)
   OR (r.FUnitId2 IS NOT NULL AND u2.FId IS NULL);
```

Also verify: `TsUnit.FTable` exists in `information_schema.TABLES`;
`TsListField.FFieldName` / `TsFormField.FFieldName` exist on the Unit field/table
side; `TsUnit.FDataSource` exists in target `datasource.xml`; `TsUnit` class hook
fields exist in target JAR classpath.

### 7. Resource verification

Check copied `extension\<app>\report`, `...\attachment`, `...\image`, `...\i18n`,
and `webapps\<app>\quicksilver` custom JS/page assets. Confirm only feature-owned
files were copied.

### 8. Integration verification

If external systems used: LINE/cbm-lite webhook reaches correct target URL;
signature validation passes; reply flow works; Redis/session works if enabled;
Google/OAuth/Drive/Calendar credentials target-specific; SFTP/SSH/7zip/report
integrations load required classes.

### 9. Regression sweep

Verify unrelated basics: login/logout, existing menu pages, existing CRUD page,
existing report page, existing attachment/image access, clean logs for several
minutes after feature use.

## Completion criteria

Claim complete only when: startup clean; login works; migrated menu/page visible to
intended roles; golden path works; required DB rows/tables/resources exist; logs show
no unresolved class/method/datasource/SQL/permission errors; at least one unrelated
existing feature still works.

## Failure handling

1. Capture exact error text and log path.
2. Classify: DB schema/data, metadata reference, permission, resource path,
   config/datasource/cache, JAR/class/version, external integration.
3. Fix the smallest missing dependency.
4. Restart only if config/JAR/classpath changed.
5. Re-run the failed check, then the regression sweep.

## Gotchas (verify)

- Clean Tomcat startup ≠ feature-level success.
- A menu item appearing ≠ underlying Unit/Form/List/DAO working.
- A page opening ≠ save/delete/report/permission paths working.
- Copying JARs can fix `ClassNotFoundException` but introduce `NoSuchMethodError` if
  quicksilver libs mismatch.
- Runtime `temp` files and logs are not source assets.

---

## Conformance Addendum

## When to Use
Compare two Chainsea/ECP/aipower deployments, migrate a single bounded feature between them (without wholesale-copying jars or databases), and verify the result across startup, DB, UI, permissions, resources, logs, and integrations. Covers the full compare → migrate → verify workflow.

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

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
