---
name: line-login-static-site-gate
description: Protect a static website Docker/Nginx deployment behind LINE Login QR-code authentication. Use when the user asks to make a website require LINE QR Code login/LINE 認證/LINE Login before viewing content, especially the 10.145.119.12 wanglai site or similar static sites served from Docker.
---

# LINE Login Static Site Gate

This skill documents the reusable pattern used to protect a static website with LINE Login. It replaces a plain `nginx` static container with a small Node/Express auth gateway container that:

1. shows a login page to unauthenticated visitors,
2. redirects to LINE Login (`https://access.line.me/oauth2/v2.1/authorize`),
3. lets LINE display its QR-code login screen,
4. handles `/auth/line/callback`,
5. stores the LINE profile in an HTTP-only session cookie,
6. serves the original static site only after login,
7. appends successful login events to a JSON-lines audit log.

## Current known deployment: 旺來 site

Remote host:

```bash
ssh hch@10.145.119.12
```

Host info discovered during setup:

```text
hostname: Jetson
OS: Ubuntu 18.04.6 / aarch64
Docker is local to this host.
```

Static website content:

```text
/home/hch/docker-webs/wanglai
```

Auth gateway app:

```text
/home/hch/docker-webs/wanglai-auth
```

Current container name:

```text
wanglai-line-gate
```

Current internal/public port mapping on Jetson:

```text
0.0.0.0:18081 -> container 3000
```

Current Cloudflare public URL:

```text
https://test.james-huang.org/
```

Current Cloudflare named tunnel:

```text
tunnel name: test
tunnel id: 1ec91570-0cd8-4ce2-908a-3a1713de3114
hostname: test.james-huang.org
origin service: http://127.0.0.1:18081
```

Cloudflare config on Jetson:

```text
/etc/cloudflared/config.yml
/etc/cloudflared/1ec91570-0cd8-4ce2-908a-3a1713de3114.json
```

Systemd service:

```bash
sudo systemctl status cloudflared
sudo systemctl restart cloudflared
cloudflared tunnel info test
```

Current LINE Login settings for this deployment:

```text
BASE_URL=https://test.james-huang.org
LINE_CHANNEL_ID=2011163472
LINE_CHANNEL_SECRET is set in the Docker container env; do not print it in final answers.
LINE callback URL=https://test.james-huang.org/auth/line/callback
```

## When invoked

Use this skill when the user asks any of the following:

- 「讓這個網頁要 LINE QRCode 認證後才能看」
- 「LINE Login gate」
- 「用 LINE 登入保護網站」
- 「LINE 掃 QR code 才能進」
- 「把 nginx 靜態站改成需要 LINE 認證」
- 「test.james-huang.org LINE 登入」
- 「看 LINE 登入記錄」
- 「限制只有某些 LINE 帳號能看」

## Required LINE Developers settings

Ask the user for these values, or create them if you have access to their LINE Developers console:

```text
LINE_CHANNEL_ID=...
LINE_CHANNEL_SECRET=...
BASE_URL=https://your-public-domain.example
```

In LINE Developers console, create or use a **LINE Login channel**, then set Callback URL exactly to:

```text
${BASE_URL}/auth/line/callback
```

For the current 旺來 deployment:

```text
https://test.james-huang.org/auth/line/callback
```

Scopes used by the gateway:

```text
profile openid
```

Optional allowlist:

```text
ALLOWED_LINE_USER_IDS=Uxxxxxxxx,Uyyyyyyyy
```

If empty, any successful LINE Login user can view the site. To lock the website down to only the owner, first have the owner login once, read the audit log to get their `userId`, then recreate the container with `ALLOWED_LINE_USER_IDS=<that userId>`.

## Browser / LINE Developers gotcha from this setup

When trying to operate LINE Developers through the existing Chrome CDP profile, `access.line.me` may be blocked by Fortinet Secure DNS or fail certificate validation:

```text
net::ERR_CERT_AUTHORITY_INVALID
Fortinet Secure DNS Service Portal
HTTP 403
```

Do **not** type LINE credentials into a browser session with certificate errors. Fastest workaround: ask the user to manually copy the LINE Channel ID and Channel Secret from a browser/profile that can login successfully, then configure Docker directly.

## Gateway files

The gateway folder should contain:

```text
package.json
server.js
.env.example
node_modules/
login-events.log          # created after first successful login
```

Minimal `package.json`:

```json
{"name":"wanglai-line-gate","version":"1.0.0","private":true,"type":"module","scripts":{"start":"node server.js"},"dependencies":{"express":"^4.19.2","express-session":"^1.18.0"}}
```

Core app behavior in `server.js`:

- `GET /login`: render the locked login page
- `GET /auth/line`: create `state` and `nonce`, redirect to LINE Login
- `GET /auth/line/callback`: exchange code for access token, fetch profile, create session, append audit log
- `GET /logout`: destroy session
- `GET /me`: show logged-in LINE profile JSON
- all static content: guarded by `requireLogin`, then served from `SITE_DIR`

The current `server.js` must include:

```js
import fs from 'fs';
// ...
console.log('LINE_LOGIN', JSON.stringify(loginEvent));
fs.appendFileSync('/app/login-events.log', JSON.stringify(loginEvent) + '\n');
```

Important escaping gotcha: when generating this JS from scripts/heredocs, make sure the JavaScript source contains literal backslash-n (`'\\n'` in Python/raw-generation terms, displayed in JS as `'\n'`), not a real newline inside the string. A broken line like this crashes Node:

```js
fs.appendFileSync('/app/login-events.log', JSON.stringify(loginEvent) + '
');
```

where the newline is physically split across two source lines. Verify with:

```bash
ssh hch@10.145.119.12 "nl -ba /home/hch/docker-webs/wanglai-auth/server.js | sed -n '75,85p'"
```

## Deploy / redeploy the auth gateway

On the remote host:

```bash
ssh hch@10.145.119.12
```

Install/update dependencies:

```bash
cd ~/docker-webs/wanglai-auth
npm install --omit=dev
```

Recreate the auth gateway container. Docker does not let you change env vars in place, so recreate it whenever `BASE_URL`, channel credentials, or allowlist changes.

```bash
SESSION_SECRET=$(docker inspect wanglai-line-gate --format '{{range .Config.Env}}{{println .}}{{end}}' 2>/dev/null | awk -F= '/^SESSION_SECRET=/{print $2; exit}')
[ -n "$SESSION_SECRET" ] || SESSION_SECRET=$(openssl rand -hex 32)

docker rm -f wanglai-line-gate 2>/dev/null || true

docker run -d \
  --name wanglai-line-gate \
  --restart unless-stopped \
  -p 18081:3000 \
  -e SITE_DIR=/site \
  -e BASE_URL='https://test.james-huang.org' \
  -e LINE_CHANNEL_ID='2011163472' \
  -e LINE_CHANNEL_SECRET='REPLACE_WITH_SECRET' \
  -e SESSION_SECRET="$SESSION_SECRET" \
  -e ALLOWED_LINE_USER_IDS='' \
  -v /home/hch/docker-webs/wanglai:/site:ro \
  -v /home/hch/docker-webs/wanglai-auth:/app \
  -w /app \
  node:20-alpine node server.js
```

Never print the real `LINE_CHANNEL_SECRET` in final answers. If the user provides a secret, use it in the command but summarize it as “已回填”.

For a temporary unconfigured gate, omit LINE env values and use HTTP base URL. It will block content and show a setup warning:

```bash
docker run -d \
  --name wanglai-line-gate \
  --restart unless-stopped \
  -p 18081:3000 \
  -e SITE_DIR=/site \
  -e BASE_URL='http://10.145.119.12:18081' \
  -e SESSION_SECRET="$(openssl rand -hex 32)" \
  -v /home/hch/docker-webs/wanglai:/site:ro \
  -v /home/hch/docker-webs/wanglai-auth:/app \
  -w /app \
  node:20-alpine node server.js
```

## Verify the auth gateway

Container:

```bash
ssh hch@10.145.119.12 "docker ps --filter name=wanglai-line-gate --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'"
```

Unauthenticated request should be blocked:

```bash
ssh hch@10.145.119.12 "curl -sI http://127.0.0.1:18081/ | head"
```

Expected before login:

```text
HTTP/1.1 401 Unauthorized
```

Login page should not show setup warning after credentials are set:

```bash
curl -s https://test.james-huang.org/login | grep -o 'LINE Login 尚未設定完成\|使用 LINE 登入 / 掃 QR Code'
```

Expected:

```text
使用 LINE 登入 / 掃 QR Code
```

`/auth/line` should redirect to LINE with the correct callback:

```bash
curl -sI https://test.james-huang.org/auth/line | grep -i '^location:'
```

Expected location contains:

```text
client_id=2011163472
redirect_uri=https%3A%2F%2Ftest.james-huang.org%2Fauth%2Fline%2Fcallback
scope=profile+openid
```

Container logs:

```bash
ssh hch@10.145.119.12 "docker logs --tail=100 wanglai-line-gate"
```

## Login records / audit log

Successful LINE Login creates/appends:

```text
/home/hch/docker-webs/wanglai-auth/login-events.log
```

Read latest logins:

```bash
ssh hch@10.145.119.12 "tail -50 /home/hch/docker-webs/wanglai-auth/login-events.log"
```

If it says `No such file or directory`, either:

1. nobody has completed LINE Login since logging was added, or
2. `server.js` does not contain the `LINE_LOGIN` / `appendFileSync` lines, or
3. the container is not using the expected `/home/hch/docker-webs/wanglai-auth:/app` bind mount.

Check logging is installed:

```bash
ssh hch@10.145.119.12 "grep -n 'import fs\|LINE_LOGIN\|appendFileSync' /home/hch/docker-webs/wanglai-auth/server.js"
```

Also check Docker logs for successful logins:

```bash
ssh hch@10.145.119.12 "docker logs --tail=200 wanglai-line-gate | grep LINE_LOGIN || true"
```

Audit event JSON-lines fields:

```json
{
  "time": "2026-08-19T01:23:45.000Z",
  "userId": "Uxxxxxxxxxxxxxxxx",
  "displayName": "LINE display name",
  "pictureUrl": "https://...",
  "ip": "client ip or CF forwarded ip",
  "userAgent": "Mozilla/5.0 ..."
}
```

### Confirmed working (2026-08-19): first real logins

The `\n`-escaping bug documented above was fixed and the logging pipeline is confirmed working end-to-end. `docker logs wanglai-line-gate` and `login-events.log` contain identical entries (both are written from the same `console.log('LINE_LOGIN', ...)` + `appendFileSync` call), so either source can be used to audit logins.

As of 2026-08-19 the log contains exactly 2 entries, both the site owner (`🐶James`), ~90s apart from two different devices — first from a desktop browser, then from the LINE in-app browser on Android:

```text
userId: U6efbfa52ff5e78157672bc3fb9280ece   <- this is the owner's LINE userId
2026-08-19T01:28:11Z  ip=2402:7500:...(IPv6)      Windows, Chrome 151 (regular browser)
2026-08-19T01:29:41Z  ip=101.12.149.25            Android, LINE in-app browser (Line/26.11.0/IAB)
```

Because `login-events.log` currently only ever contains the owner's own logins, `U6efbfa52ff5e78157672bc3fb9280ece` is the value to use for `ALLOWED_LINE_USER_IDS` if/when the owner asks to lock the site down to just themselves (see "Configure allowlist after first login" below) — no need to ask them to log in again just to get this value.

### Direct (non-Cloudflare) access still works

The site is reachable two ways simultaneously, both hitting the same container:

```text
http://10.145.119.12:18081/        <- direct LAN/WAN IP:port, via docker-proxy
https://test.james-huang.org/      <- Cloudflare named tunnel
```

`docker-proxy` binds `18081` on both `0.0.0.0` and `::` (check with `ss -ltnp | grep 18081`), so the direct URL is not just for internal testing — it is publicly reachable if the host's network allows inbound 18081. Keep this in mind for security review: closing the Cloudflare tunnel alone does not take the site offline.

## Configure allowlist after first login

After the owner logs in, read `userId` from `login-events.log`, then recreate container with:

```bash
-e ALLOWED_LINE_USER_IDS='Uxxxxxxxxxxxxxxxx'
```

Multiple allowed users:

```bash
-e ALLOWED_LINE_USER_IDS='Uaaa,Ubbb,Uccc'
```

If a non-allowed user logs in, callback returns a 403-style login page error:

```text
你的 LINE 帳號尚未被允許瀏覽此網站
```

## Cloudflare named tunnel setup for a new host

For this deployment, `cloudflared` was installed directly on Jetson, not as a Docker container.

Install on Ubuntu aarch64:

```bash
cd /tmp
wget -q -O cloudflared-linux-arm64.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared-linux-arm64.deb
cloudflared --version
```

Login needs browser authorization and creates `~/.cloudflared/cert.pem`:

```bash
cloudflared tunnel login
```

Create named tunnel and route DNS:

```bash
cloudflared tunnel create test
cloudflared tunnel route dns test test.james-huang.org
```

Write `/etc/cloudflared/config.yml`:

```yaml
tunnel: 1ec91570-0cd8-4ce2-908a-3a1713de3114
credentials-file: /etc/cloudflared/1ec91570-0cd8-4ce2-908a-3a1713de3114.json

ingress:
  - hostname: test.james-huang.org
    service: http://127.0.0.1:18081
  - service: http_status:404
```

Install/start service:

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl restart cloudflared
```

Verify:

```bash
systemctl is-active cloudflared
cloudflared tunnel info test
curl -sI https://test.james-huang.org/ | head
```

Expected external status before auth:

```text
HTTP/1.1 401 Unauthorized
```

## Roll back to plain nginx

If the user wants to remove LINE Login and restore static nginx:

```bash
docker rm -f wanglai-line-gate 2>/dev/null || true

docker run -d \
  --name nginx-wanglai \
  --restart unless-stopped \
  -p 18081:80 \
  -v /home/hch/docker-webs/wanglai:/usr/share/nginx/html:ro \
  nginx:alpine
```

Verify:

```bash
curl -sI http://127.0.0.1:18081/ | head
```

Expected:

```text
HTTP/1.1 200 OK
```

## Security notes

- Keep `LINE_CHANNEL_SECRET` and `SESSION_SECRET` out of git and public logs.
- Use HTTPS in production. Current public URL `https://test.james-huang.org` satisfies LINE callback requirements.
- `ALLOWED_LINE_USER_IDS` is the only built-in authorization filter. Without it, all LINE users who can login are allowed.
- The session store is the default in-memory Express store. It is acceptable for a small personal/demo site but not for high-traffic production. For production, use Redis or another persistent session store.
- If `SESSION_SECRET` changes, all users are logged out.

---

## Conformance Addendum

## When to Use
Protect a static website Docker/Nginx deployment behind LINE Login QR-code authentication. Use when the user asks to make a website require LINE QR Code login/LINE 認證/LINE Login before viewing content, especially the 10.145.119.12 wanglai site or similar static sites served from Docker.

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
