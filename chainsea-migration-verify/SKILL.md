---
name: chainsea-migration-verify
description: Verify a Chainsea/ECP/aipower feature migration across startup, database, UI, permissions, resources, logs, and integrations.
triggers:
  - verify chainsea migration
  - verify ecp migration
  - verify aipower migration
  - migration verification
  - 驗證遷移
argument-hint: "<target-dir> <feature-name>"
---

# Chainsea Migration Verify Skill

## Purpose

Use this skill after moving a feature between Chainsea / ECP / aipower deployments to prove the migration works and did not break the target system.

## When to Activate

Use after:

- Importing metadata rows
- Adding business tables or rows
- Copying extension resources
- Adding or changing JARs
- Modifying datasource/cache/Redis/cbm-lite config
- Changing menu/page/unit/report settings

## Verification Workflow

### 1. Pre-start checks

Confirm:

- Target directory is a test copy unless the user explicitly approved production changes.
- Ports do not collide with other running instances.
- `server.bat` matches the target runtime needs.
- `extension\<app>\config\datasource.xml` contains every datasource ID the feature uses.
- If Redisson session manager is enabled, Redis server/config exists and starts.
- If cbm-lite/LINE is involved, callback URL and signature config are target-specific.

### 2. Startup verification

Start the target deployment using the project’s expected command.

Watch for terminal startup failures in:

- Tomcat console output
- `apache-tomcat\logs`
- `apache-tomcat\extension\<app>\log`
- MariaDB/Atomikos errors
- Redis errors if used

Blockers:

- `ClassNotFoundException`
- `NoClassDefFoundError`
- `NoSuchMethodError`
- SQL table missing
- SQL column missing
- datasource not found
- license/config fatal errors
- Atomikos pool startup failure
- MariaDB data directory failure

### 3. Login and shell checks

Verify:

- Root redirect points to the correct app context.
- Direct URL opens, e.g. `/aipower` or `/ecp`.
- Login works.
- Main frame/dashboard loads.
- No browser-visible 404/500 loop.
- Session survives navigation; if Redisson is enabled, no session serialization errors.

### 4. Menu and permission checks

Verify the migrated feature:

- Appears under intended menu location.
- Is visible to intended roles.
- Is hidden from roles without permission.
- Has correct create/read/update/delete/report/export permissions.
- Does not overwrite or displace unrelated target menu items.

### 5. UI golden path

For each migrated page/unit:

- Open the page.
- Run default query.
- Search/filter.
- Open detail/form.
- Create a test record if safe.
- Edit a test record if safe.
- Delete or rollback test record if safe.
- Export/report if applicable.
- Confirm labels/i18n render correctly.
- Confirm images/icons/attachments render correctly.

### 6. Database verification

Check that target DB has:

- Required metadata rows.
- Required business tables.
- Required seed rows.
- No duplicate code collisions.
- No orphaned menu/page/unit/form/list/report references.
- No references to source-only datasource IDs unless intentionally added.
- No hard-coded source context path or source port.

Run read-only orphan checks where possible:

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

Also verify:

- `TsUnit.FTable` exists in `information_schema.TABLES`.
- `TsListField.FFieldName` exists on the Unit field/table side.
- `TsFormField.FFieldName` exists on the Unit field/table side.
- `TsUnit.FDataSource` exists in target `datasource.xml`.
- `TsUnit` class hook fields exist in target JAR classpath.

### 7. Resource verification

Check copied resources:

- `extension\<app>\report` report templates
- `extension\<app>\attachment` files
- `extension\<app>\image` icons/logos used by the feature
- `extension\<app>\i18n` resources
- `webapps\<app>\quicksilver` custom JS/page assets if any

Confirm only feature-owned files were copied.

### 8. Integration verification

If the feature uses external systems, verify separately:

- LINE/cbm-lite webhook reaches the correct target URL.
- Signature validation passes.
- Reply flow works.
- Redis/session behavior works if enabled.
- Google/OAuth/Drive/Calendar credentials are target-specific.
- SFTP/SSH/7zip/report integrations load required classes.

### 9. Regression sweep

Verify unrelated target basics still work:

- Login/logout.
- Existing menu pages.
- Existing CRUD page.
- Existing report page.
- Existing attachment/image access.
- Logs stay clean for several minutes after feature use.

## Completion Criteria

Only claim migration complete when:

- Startup is clean.
- Login works.
- Migrated menu/page is visible to intended roles.
- Golden path works.
- Required DB rows/tables/resources exist.
- Logs show no unresolved class, method, datasource, SQL, or permission errors.
- At least one unrelated existing feature still works.

## Failure Handling

When verification fails:

1. Capture the exact error text and log path.
2. Classify the failure:
   - DB schema/data
   - Metadata reference
   - Permission
   - Resource path
   - Config/datasource/cache
   - JAR/class/version
   - External integration
3. Fix the smallest missing dependency.
4. Restart only if config/JAR/classpath changed.
5. Re-run the failed check and then the regression sweep.

## Gotchas

- A clean Tomcat startup does not prove feature-level success.
- A menu item appearing does not prove underlying Unit/Form/List/DAO works.
- A page opening does not prove save/delete/report/permission paths work.
- Copying JARs can fix `ClassNotFoundException` but introduce `NoSuchMethodError` if quicksilver libs are mismatched.
- Runtime `temp` files and logs should not be treated as source assets.

---

## Conformance Addendum

## When to Use
Verify a Chainsea/ECP/aipower feature migration across startup, database, UI, permissions, resources, logs, and integrations.

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
