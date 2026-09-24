---
name: gcp-billing-rainmeter-dashboard
description: Build, install, repair, or update a Windows Rainmeter desktop dashboard that displays Google Cloud Billing amounts and refreshes automatically (default 60 seconds). Use when the user mentions Rainmeter plus GCP/GCloud billing/cost dashboard, wants API-based billing data, wants to change refresh interval, or needs to switch between Cloud Billing Budget API, BigQuery Billing Export, and the fallback Chrome-CDP scraper.
compatibility: Windows, Rainmeter, Google Cloud SDK gcloud/bq. BigQuery Billing Export is required for accurate actual cost/credits totals; Cloud Billing Budget API only exposes budget metadata.
---

# GCP Billing Rainmeter Dashboard

This skill maintains a Rainmeter skin that shows Google Cloud Billing amounts on the Windows desktop.

Installed/default skin path:

```text
%USERPROFILE%\Documents\Rainmeter\Skins\GCPBillingDashboard\GCPBillingDashboard.ini
```

Bundled template path, relative to this skill:

```text
templates\GCPBillingDashboard\
```

## When to use

Use this skill when the user asks to:

- create/write/install a Rainmeter dashboard for GCP costs
- show GCP Billing amount, GPU charges, credits, or monthly total on desktop
- make the dashboard refresh automatically, e.g. every 60 seconds
- switch from browser scraping to Google API data
- configure BigQuery Billing Export table for accurate billing totals
- fix the Rainmeter skin or update its interval/configuration

## Important GCP billing API facts

Do **not** claim there is a simple real-time Cloud Billing REST API that returns the exact billing-console total.

There are three data-source modes:

1. **BigQuery Billing Export — recommended**
   - Best for actual cost / credits / net total.
   - User must enable Billing Export in the Cloud Billing console first.
   - Then set `bigQueryBillingExportTable` in `gcp-billing-config.json`.
   - The updater queries BigQuery using `bq query`.

2. **Cloud Billing Budget API — limited fallback**
   - Requires `billingbudgets.googleapis.com` enabled.
   - Useful for budget metadata, not precise actual spend totals.
   - If there is no budget, it cannot show current actual spend.

3. **Chrome CDP scraper — old fallback only**
   - Reads an already-open Chrome debug port 9222 billing page.
   - Does not require API setup, but depends on logged-in Chrome.
   - Included as `scripts\update-gcp-billing.ps1` but the active skin uses the API script.

## Current implementation

The skin uses:

```text
scripts\update-gcp-billing-api.ps1
```

The API updater writes this JSON file:

```text
billing.json
```

Rainmeter reads that JSON via `WebParser` measures.

Default refresh interval is **60 seconds** in `GCPBillingDashboard.ini`:

```ini
[MeasureRefreshTimer]
Measure=Calc
Formula=MeasureRefreshTimer + 1
IfCondition=(MeasureRefreshTimer >= 60)
IfTrueAction=[!SetOption MeasureRefreshTimer Formula 0][!CommandMeasure MeasureUpdateBilling "Run"]
```

The user can also click the skin's refresh button to run the updater immediately.

## Config file

Target config file:

```text
%USERPROFILE%\Documents\Rainmeter\Skins\GCPBillingDashboard\gcp-billing-config.json
```

Default/example:

```json
{
  "billingAccountId": "019062-F4FEB4-30EE89",
  "projectId": "project-fbcea0c4-05a8-4472-b9d",
  "bigQueryBillingExportTable": "",
  "currencySymbol": "$",
  "timeZone": "Asia/Taipei"
}
```

If BigQuery Billing Export is enabled, set:

```json
"bigQueryBillingExportTable": "PROJECT.DATASET.TABLE"
```

Example shape:

```json
"bigQueryBillingExportTable": "project-fbcea0c4-05a8-4472-b9d.billing_export.gcp_billing_export_v1_019062_F4FEB4_30EE89"
```

Do not invent the actual dataset/table name. Verify with `bq ls` / Cloud Console, or ask the user.

## Install / reinstall

Run from the skill directory:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts/install.ps1
```

With explicit values:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts/install.ps1 `
  -BillingAccountId '019062-F4FEB4-30EE89' `
  -ProjectId 'project-fbcea0c4-05a8-4472-b9d' `
  -RefreshSeconds 60
```

With BigQuery Billing Export:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts/install.ps1 `
  -BillingAccountId '019062-F4FEB4-30EE89' `
  -ProjectId 'project-fbcea0c4-05a8-4472-b9d' `
  -BigQueryBillingExportTable 'PROJECT.DATASET.TABLE' `
  -RefreshSeconds 60
```

Optionally load Rainmeter after install:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts/install.ps1 -LoadRainmeter
```

If Rainmeter is not found automatically, tell the user to open Rainmeter manually, click **Refresh all**, then load:

```text
GCPBillingDashboard\GCPBillingDashboard.ini
```

## Change refresh interval

Edit the installed INI:

```text
%USERPROFILE%\Documents\Rainmeter\Skins\GCPBillingDashboard\GCPBillingDashboard.ini
```

Change:

```ini
IfCondition=(MeasureRefreshTimer >= 60)
```

Examples:

- 30 seconds: `>= 30`
- 60 seconds: `>= 60`
- 5 minutes: `>= 300`

Then refresh the skin in Rainmeter.

## Test updater manually

```powershell
$skin = "$env:USERPROFILE\Documents\Rainmeter\Skins\GCPBillingDashboard"
powershell -NoProfile -ExecutionPolicy Bypass -File "$skin\scripts\update-gcp-billing-api.ps1" `
  -OutputPath "$skin\billing.json" `
  -ConfigPath "$skin\gcp-billing-config.json"
Get-Content "$skin\billing.json"
```

Expected JSON keys:

```json
{
  "status": "...",
  "totalCost": "...",
  "cost": "...",
  "credits": "...",
  "period": "...",
  "url": "...",
  "source": "...",
  "updated": "..."
}
```

## Enable Cloud Billing Budget API, if using budget fallback

Use `gcloud.cmd` rather than `gcloud.ps1` if PowerShell execution policy blocks scripts.

```powershell
& "$env:LOCALAPPDATA\Google\Cloud SDK\google-cloud-sdk\bin\gcloud.cmd" services enable billingbudgets.googleapis.com --project=project-fbcea0c4-05a8-4472-b9d --quiet
```

Then test:

```powershell
& "$env:LOCALAPPDATA\Google\Cloud SDK\google-cloud-sdk\bin\gcloud.cmd" billing budgets list --billing-account=019062-F4FEB4-30EE89 --format=json --quiet
```

If it still says permission denied, the active account lacks billing permissions. Check:

```powershell
& "$env:LOCALAPPDATA\Google\Cloud SDK\google-cloud-sdk\bin\gcloud.cmd" config list --format=json
& "$env:LOCALAPPDATA\Google\Cloud SDK\google-cloud-sdk\bin\gcloud.cmd" billing accounts list --format=json
```

## Enable BigQuery Billing Export, recommended

This cannot always be fully automated because it is a billing-account console setting.

High-level steps:

1. Open Cloud Console → Billing → Billing export.
2. Enable **Detailed usage cost** export to BigQuery.
3. Choose or create a BigQuery dataset.
4. Wait until data begins to arrive. This may take hours after first enablement.
5. Find the generated table name.
6. Put the full table id into `gcp-billing-config.json` as `bigQueryBillingExportTable`.
7. Test the updater manually.

Useful commands:

```powershell
& "$env:LOCALAPPDATA\Google\Cloud SDK\google-cloud-sdk\bin\bq.cmd" ls --format=json
& "$env:LOCALAPPDATA\Google\Cloud SDK\google-cloud-sdk\bin\bq.cmd" ls --format=json PROJECT:DATASET
```

## Troubleshooting

### Skin shows `--`

Open `billing.json` and read `status`:

```powershell
Get-Content "$env:USERPROFILE\Documents\Rainmeter\Skins\GCPBillingDashboard\billing.json"
```

Common causes:

- BigQuery table not configured.
- BigQuery Billing Export not enabled or no rows yet.
- Cloud Billing Budget API disabled.
- Active `gcloud` account lacks billing permissions.
- `bq.cmd` / `gcloud.cmd` missing from Google Cloud SDK.

### PowerShell says gcloud.ps1 cannot be loaded

Use `gcloud.cmd` explicitly. The updater already searches for `gcloud.cmd` and `bq.cmd`.

### User wants old browser-based values

Switch the RunCommand parameter in the INI back to:

```ini
Parameter=-NoProfile -ExecutionPolicy Bypass -File "#CURRENTPATH#scripts\update-gcp-billing.ps1" -OutputPath "#DataFile#"
```

Then Chrome must be opened manually with remote debugging port 9222 and a GCP Billing tab must already be open.

---

## Conformance Addendum

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
