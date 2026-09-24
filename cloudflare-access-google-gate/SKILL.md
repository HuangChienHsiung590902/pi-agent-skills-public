---
name: cloudflare-access-google-gate
description: Add a Cloudflare Access login gate (Google-account-only) in front of a hostname already routed through the shared Cloudflare Tunnel `hch` (see skill `cloudflare-tunnel`). Use when the user asks to require Google login before a public HTTPS page/service is reachable — e.g. "加登入"、"要有 Google 認證才能進"、"擋掉陌生人"、"限制只有我能用" for a URL under `*.james-huang.org`. Covers both the one-time account-level setup (Google identity provider + its backing OAuth client) and the per-hostname setup (Access Application + policy) — as of 2026-07-22 the Google IdP already exists for this account, so most future uses only need the per-hostname half.
---

# cloudflare-access-google-gate

## What this produces

Requests to the target hostname get intercepted by Cloudflare's edge (before ever reaching the origin/tunnel) and redirected to a "Continue with Google" login page. Only emails on an explicit allow-list can get through. For a browser already signed into an allowed Google account, this is close to invisible — Google auto-approves and redirects back, no visible prompt. A fresh/incognito session sees the interstitial.

**No app code changes required** — this is entirely Cloudflare/Google dashboard configuration sitting in front of the existing tunnel route.

## Account facts (this Cloudflare account, confirmed 2026-07-22)

| What | Value |
|---|---|
| Cloudflare account | `Hch590902@gmail.com's Account`, account ID `26e1d6671a45c7e243fda877a761e3b9` |
| Zero Trust team domain | `hch-n8n.cloudflareaccess.com` |
| Google IdP | Already configured, named "Google" (plain Google, not "Google Workspace") |
| Backing OAuth client | Google Cloud project `gas-mcp-503101` (reused existing project — see skill `gas-mcp`), OAuth client "Cloudflare Access - audiocpp-web", type **Web application** |
| OAuth redirect URI on that client | `https://hch-n8n.cloudflareaccess.com/cdn-cgi/access/callback` |
| OAuth publish status | Testing (100-user cap) — test-user list already includes `hch.new@gmail.com` and `hch590902@gmail.com` |

**Since the Google IdP already exists, gating a NEW hostname is just Part B below** — skip Part A unless `dash.cloudflare.com/{account}/one/integrations/identity-providers` no longer shows a "Google" row (e.g. fresh account, or it got deleted).

## Part A: One-time Google IdP setup (skip if "Google" already listed)

1. **Google Cloud Console** — create the OAuth client first, you need its Client ID/Secret for Part A step 2.
   - `https://console.cloud.google.com/apis/credentials?project={gcp-project}` → 建立憑證 → OAuth 用戶端 ID
   - Application type: **網頁應用程式 / Web application** (not Desktop — Cloudflare needs the redirect-URI callback flow)
   - Redirect URI: `https://{team-domain}/cdn-cgi/access/callback` (get `{team-domain}` from Cloudflare Zero Trust → 設定, see Part B step 0)
   - Save the Client ID and Client Secret shown once after creation — Google will not show the secret again.
   - If the project's OAuth consent screen is in "Testing" status, add every allowed email under `https://console.cloud.google.com/auth/audience?project={gcp-project}` → 測試使用者 → 新增使用者, or their login will be rejected by Google before Cloudflare even sees it.

2. **Cloudflare Zero Trust** — `https://dash.cloudflare.com/{account-id}/one/integrations/identity-providers/add/google`
   - Fill 用戶端識別碼 (Client ID) / 用戶端密碼 (Client Secret) with the values from step 1.
   - **Gotcha, hit twice in production:** Chrome autofill silently pre-fills these two fields with the Cloudflare *login* password (not an OAuth secret) on page load. Before saving, verify actual field values via:
     ```js
     () => Array.from(document.querySelectorAll('input')).map(i => ({ label: i.getAttribute('aria-label') || i.name, value: i.type === 'password' ? (i.value ? i.value.slice(0,6)+'...' : '') : i.value }))
     ```
     If wrong, use `browser_type` (fill, which replaces content) to overwrite with the real values before clicking 儲存.
   - Save.

## Part B: Per-hostname setup (do this every time, even after Part A is done once)

0. **Find the team domain** (needed for verification later, and for Part A step 1 if doing it fresh):
   ```js
   () => { const el = Array.from(document.querySelectorAll('*')).find(e => e.textContent && e.textContent.includes('.cloudflareaccess.com') && e.children.length === 0); return el ? el.textContent : 'not found'; }
   ```
   run on `https://dash.cloudflare.com/{account-id}/one/settings`.

1. **Create the Access Application** — `https://dash.cloudflare.com/{account-id}/one/access-controls/apps` → 建立新應用程式 → tab "自我裝載和私有" → sub-tab "公開 DNS" (Public Hostname — use this one, not "私有目的地", for a hostname already publicly resolvable via the tunnel) → 繼續使用.
   - 子網域 (subdomain) + 網域 (domain, pick from the dropdown — it's populated from zones on the account, e.g. `james-huang.org`).
   - Scroll to 認證 (Credentials) section: turn OFF "接受所有可用的識別提供者" (Accept all available IdPs), then in "選擇此應用程式可用的識別提供者" select only **Google** — this excludes the "One-time PIN" email-code fallback so Google is the only way in.
   - Under Access 原則 (Access Policy) → 建立新原則:
     - 原則名稱: anything, e.g. "Allow me"
     - 選取器是... → **電子郵件** (Email), then type each allowed address into the resulting multi-value chip input and press Enter after each — this is an OR-within-one-rule list, not one rule per email.
     - 動作 stays 允許 (Allow, default).
     - 儲存政策.
   - 名稱 (application name) auto-fills from the subdomain — fine to leave.
   - 建立 to finish.

2. **Verify — wait for edge propagation first.** New Access policies/IdP bindings take roughly 30–90 seconds to reach Cloudflare's edge. An immediate `curl` right after saving may still show the OLD (unprotected) behavior — don't conclude failure from that; wait ~60s and retest:
   ```bash
   curl -sv -o /dev/null https://{hostname}/ 2>&1 | grep -E "^< HTTP|^< [Ll]ocation"
   ```
   Success looks like:
   ```
   < HTTP/1.1 302 Found
   < Location: https://{team-domain}/cdn-cgi/access/login/{hostname}?kid=...
   ```
   A `200` with real page content means the gate isn't active yet (propagation) or wasn't actually saved — re-check the Application/Policy in the dashboard before assuming propagation is still the cause.

3. **Verify a real login works** by navigating an authenticated browser (one already logged into an allowed Google account) to `https://{hostname}/` via Playwright — if the IdP/policy are right, `page.goto` follows the whole redirect chain (site → Access login → Google OAuth → Access callback → site) transparently and lands on the real page with no visible interstitial, since the browser is already Google-authenticated. Confirm getUserMedia/whatever the app needs now works too, e.g.:
   ```js
   () => window.isSecureContext
   ```

## Adding more allowed emails later

Don't create a second policy — edit the existing one (`https://dash.cloudflare.com/{account-id}/one/access-controls/apps` → click the app → 原則 tab → edit "Allow me") and add another email chip to the same Email rule.

## Related

- `cloudflare-tunnel` skill — owns the underlying ingress routing (`config.yml`) that gets this hostname to the tunnel in the first place; this skill's Access gate sits in front of that, doesn't replace it.
- `gas-mcp` skill — the Google Cloud project reused as the OAuth client backing this, if you need to touch it for unrelated reasons.

---

## Conformance Addendum

## When to Use
Add a Cloudflare Access login gate (Google-account-only) in front of a hostname already routed through the shared Cloudflare Tunnel `hch` (see skill `cloudflare-tunnel`). Use when the user asks to require Google login before a public HTTPS page/service is reachable — e.g. "加登入"、"要有 Google 認證才能進"、"擋掉陌生人"、"限制只有我能用" for a URL under `*.james-huang.org`. Covers both the one-time account-level setup (Google identity provider + its backing OAuth client) and the per-hostname setup (Access Application + policy) — as of 2026-07-22 the Google IdP already exists for this account, so most future uses only need the per-hostname half.

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
