---
name: chainsea-feature-migration
description: Plan and execute a single-feature migration between Chainsea/ECP/aipower deployments without wholesale copying jars or databases.
triggers:
  - chainsea migration
  - ecp migration
  - aipower migration
  - migrate feature
  - 搬功能
  - 功能遷移
argument-hint: "<feature-name> from <source-dir> to <target-dir>"
---

# Chainsea Feature Migration Skill

## Purpose

Use this skill to migrate one bounded feature between Chainsea / ECP / aipower deployments safely.

## When to Activate

Use when the user asks to move, copy, port, or merge a specific feature between deployments such as:

- `C:\com\chainsea` and `C:\chainsea\ecp`
- `aipower` and `ecp`
- Two packaged ECP/Tomcat/MariaDB directories

## Rule

Migrate by feature package, not by whole application package.

Do not start by copying:

- Whole `WEB-INF\lib`
- Whole `mariadb\data`
- Whole `webapps\<app>`
- Whole `extension\<app>\config`

## Workflow

### 1. Freeze, back up, and define rollback

Before changes, ask the user to confirm backups exist or create test copies.

Minimum backup targets:

- Source and target root folders
- `mariadb\data`
- `apache-tomcat\webapps\<app>\WEB-INF\lib`
- `apache-tomcat\extension\<app>\config`
- `apache-tomcat\extension\<app>\attachment`
- `apache-tomcat\extension\<app>\i18n`
- `apache-tomcat\extension\<app>\image`
- `apache-tomcat\extension\<app>\report`

Define rollback before editing:

- Which DB dump or folder copy restores the target DB
- Which copied JARs must be removed
- Which config files must be restored
- Which extension resources must be removed or restored
- What startup/login checks prove rollback succeeded

### 2. Define one feature boundary

Document:

- Feature name
- Source URL/menu path
- Target location where it should appear
- Unit code
- Page/Form/List/Edit/Report codes
- Business table names
- Required roles/privileges
- External integrations
- Expected golden-path behavior

If the user cannot name the feature boundary, inspect menu/page/unit metadata first and propose a minimal migration unit.

### 3. Inventory dependencies

For the chosen feature, identify dependencies by walking the metadata graph, not by guessing table names:

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

For the chosen feature, identify:

- Metadata tables:
  - `TsMenu`
  - `TsPage`
  - `TsPageGroup`
  - `TsPageSet`
  - `TsPageSetItem`
  - `TsUnit`
  - `TsForm`
  - `TsEdit`
  - `TsField`
  - `TsList`
  - `TsReport`
  - `TsDictionary`
  - `TsDictionaryItem`
  - `TsRole`
  - `TsPrivilege`
  - `TsTextResource`
- Business tables, usually `Tc*`, `Tp*`, `Tr*`, or module-specific names
- Extension resources:
  - report templates
  - images
  - attachments
  - i18n spreadsheets/resources
- Config:
  - datasource IDs such as `default` or `vrm`
  - Redis/Redisson/session needs
  - `conf\Catalina\localhost\<app>.xml`
  - `conf\redisson.yaml`
  - cbm-lite/LINE/webhook settings
- Sensitive settings that must be reconfigured on the target, not blindly copied:
  - `license.lic`
  - OAuth tokens
  - LINE channel secret/access token
  - webhook secrets
  - encrypted datasource credentials
  - API keys
  - administrator/user passwords
- Java dependencies:
  - action/service/DAO class names if visible in metadata/logs
  - owning module JAR
  - third-party jars only if proven required

### 4. Prefer SQL export/import over file-level DB copying

Use SQL-level migration for metadata and business rows.

Avoid copying `.ibd`, `.frm`, `ibdata1`, or the whole `mariadb\data` folder for a feature migration.

### 5. Resolve identity conflicts

Before importing into target:

- Check whether codes already exist.
- Check whether IDs collide.
- Prefer stable business codes where available.
- Preserve foreign-key-like logical references between metadata rows.
- Map menu parent/location intentionally.
- Recreate role/privilege links for the target roles instead of blindly overwriting target permissions.

### 6. Add resources

Copy only feature-owned resources:

- Specific report files
- Specific images/icons
- Specific attachment directories
- Specific i18n files or rows

Do not overwrite target logos/license/config unless the feature explicitly requires it.

### 7. Handle JAR/class issues last

Only add or change JARs after logs prove missing Java classes or methods.

Common evidence:

- `ClassNotFoundException`
- `NoClassDefFoundError`
- `NoSuchMethodError`
- Bean/action/service not found

When JAR work is required:

1. Identify the class owner.
2. Prefer a product-compatible module version.
3. Avoid mixing `quicksilver-lib-*` versions between deployments.
4. Add one JAR/dependency group at a time.
5. Restart and verify after each change.

### 8. Update context-specific values

Adapt:

- `/aipower` vs `/ecp`
- 22821/22822 vs 12821/12822 or current target ports
- Webhook/callback URLs
- datasource IDs
- Redis host/port
- extension path assumptions

### 9. Verify the feature

Use the `chainsea-migration-verify` skill after migration.

## Migration Decision Matrix

| Feature type | Approach |
|---|---|
| Menu/Page/Unit/Form/List only | DB metadata + permissions |
| CRUD backed by new table | DDL + metadata + seed rows + permissions |
| Report | metadata + report file + SQL/datasource validation |
| i18n/display text | metadata/text rows + i18n resources |
| Attachment/image | metadata reference + exact resource files |
| cbm-lite/LINE | config + tables + webhook + Redis/session + logs |
| Java-backed service | metadata + tables + compatible module/JAR dependency |

## Gotchas

- `aipower` may have a `vrm` datasource that does not exist in `ecp`.
- `aipower` may have cbm-lite files that are absent from `ecp`.
- `ecp` has many tables absent from `aipower`, especially helpdesk/knowledge/AI chat/campaign/contract/invoice/lead domains.
- Metadata rows can reference classes inside product modules; UI metadata alone may not be sufficient.
- A page opening successfully is not enough; test save/delete/report/permission behavior too.

---

## Conformance Addendum

## When to Use
Plan and execute a single-feature migration between Chainsea/ECP/aipower deployments without wholesale copying jars or databases.

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
