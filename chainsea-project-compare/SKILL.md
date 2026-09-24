---
name: chainsea-project-compare
description: Compare two Chainsea/ECP/aipower local deployment directories and identify portable vs non-portable differences.
triggers:
  - chainsea compare
  - ecp compare
  - aipower compare
  - project migration report
  - 功能互移
argument-hint: "<source-dir> <target-dir>"
---

# Chainsea Project Compare Skill

## Purpose

Use this skill to compare two local Chainsea / ECP / aipower deployment directories before deciding whether functionality can be moved between them.

## When to Activate

Use when the task mentions:

- Comparing `C:\com\chainsea`, `C:\chainsea\ecp`, or similar Chainsea deployment folders
- Moving features between `aipower` and `ecp`
- Checking whether two ECP packages are compatible
- Producing a migration feasibility report

## Workflow

1. Confirm both directories exist and stay read-only.
2. Compare top-level layout:
   - `apache-tomcat`
   - `jre` / `jdk`
   - `mariadb`
   - `redis`
   - `tool`
   - `backup`
   - `server.bat`
   - `HttpLogin.url`
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
5. Compare config files under `extension\<app>\config`:
   - `application.properties`
   - `datasource.xml`
   - `cache-caffeine.xml`
   - `cache-redis.xml`
   - `redis.xml`
   - custom files such as `cbm-lite.properties`
6. Compare JARs under `WEB-INF\lib`:
   - Main product module (`aipower-module-*`, `ecp-module-*`)
   - Internal module metadata such as `QS-MODULE/module.xml` when available
   - Module `depends`, `formerNames`, and file operations
   - `quicksilver-module-main-*`
   - `quicksilver-lib-*`
   - MariaDB4j / local driver
   - Redisson Tomcat libs
   - large third-party dependency families
7. Compare runtime versions:
   - `apache-tomcat\RELEASE-NOTES`
   - `jre\release`
   - `jdk\release` if present
   - `cache-caffeine.xml` TTL
   - `redisson.yaml` and Redisson jar versions
8. Compare embedded MariaDB schema shape without modifying data:
   - `mariadb\data\default`
   - Count `.ibd` / `.frm` files
   - Sample source-only and target-only table names
8. Compare extension resources:
   - `attachment`
   - `i18n`
   - `image`
   - `report`
   - `temp`
9. Produce a Chinese report with:
   - Similarity level
   - Hard differences
   - Portable feature categories
   - Non-portable components
   - Safe migration sequence
   - Verification checklist

## Key Heuristics

- Same directory skeleton does not mean module compatibility.
- Same `quicksilver-module-main` does not mean same `quicksilver-lib` compatibility.
- `WEB-INF\lib` should never be wholesale copied between deployments.
- `mariadb\data` should never be wholesale copied as a feature migration method.
- A function is portable only after its DB metadata, business tables, Java classes, config, resources, and permissions are all identified.

## Evidence to Capture

Always cite concrete paths and observed versions, such as:

- `WEB-INF\lib\ecp-module-main-*.jar`
- `WEB-INF\lib\aipower-module-base-*.jar`
- `WEB-INF\lib\quicksilver-lib-basic-*.jar`
- `extension\<app>\config\datasource.xml`
- `conf\Catalina\localhost\<app>.xml`
- `server.bat`

## Gotchas

- `aipower` may have Redis and Redisson session manager enabled even if `application.properties` uses Caffeine cache.
- `ecp` may contain Redisson jars/config but not actually enable Redisson session manager.
- `datasource.xml` can contain additional datasource IDs like `vrm`; reports or DAO code may depend on them.
- `extension\<app>\temp` and log files are runtime artifacts, not migration source unless explicitly needed for investigation.

---

## Conformance Addendum

## When to Use
Compare two Chainsea/ECP/aipower local deployment directories and identify portable vs non-portable differences.

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
