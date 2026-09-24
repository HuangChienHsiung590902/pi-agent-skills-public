---
name: frp-vps
description: Set up frp (fast reverse proxy) with frps on a VPS and frpc on internal hosts — deploy, configure, connect
triggers:
  - frp
  - frps
  - frpc
  - reverse proxy vps
argument-hint: "[setup|status|add-proxy]"
---

# frp-vps Skill

## Purpose

Expose internal/NAT'd hosts (e.g. `portable-sshd-pro`) to the internet via a VPS
running `frps`, so remote `frpc` clients can tunnel services (SSH, etc.) through it.
Chosen over Cloudflare Tunnel because frp is a raw TCP protocol — routing it through
Cloudflare Tunnel requires `cloudflared access tcp` on every client, which is more
moving parts than just pointing frpc at a VPS with a public IP.

## Status: NOT YET DEPLOYED

This skill currently documents the plan and local binaries only. Still to do:
1. Decide/provision the VPS (see options below)
2. Deploy `frps` on the VPS, open port 7000 (+ any proxy ports)
3. Configure `frpc` on `portable-sshd-pro` (and any other client) to connect
4. Add auth token to both sides
5. Add Windows Defender exclusion for `D:\frp_0.70.1` (needs admin — see below)

## Local Files (this machine)

| Path | Purpose |
|------|---------|
| `D:\frp_0.70.1\frpc.exe` / `frpc.toml` | Client binary + example config (currently just the default TCP/22 example) |
| `D:\frp_0.70.1\frps.exe` / `frps.toml` | Server binary + example config (currently just `bindPort = 7000`, no auth) |

These are the reference binaries/configs for this machine's own frpc if it also
needs to be a client. The actual `frps` deployment target is the VPS, not this machine.

## VPS Decision

Evaluated GCP Free Tier (e2-micro, us-west1/central1/east1) vs a paid low-cost VPS:

- **GCP Free Tier**: free forever, but only **1GB/month egress**, US-based (latency
  from Taiwan), 1 vCPU/1GB RAM. Fine for occasional low-traffic SSH access; not
  fine for frequent file transfers or sustained throughput — risks overage billing.
- **Paid VPS** (Vultr/DigitalOcean/Linode ~$5/mo, or a Taiwan/Asia-region provider):
  no tight traffic cap, better latency, straightforward frp deployment.

Decision: user is leaning toward renting a VPS (not GCP free tier) — pick this up
next session by asking which provider/region, then provision + deploy frps.

## Deployment Plan (once VPS exists)

### 1. On the VPS (Linux, systemd assumed unless told otherwise)

```bash
# download frp release matching VPS arch from https://github.com/fatedier/frp/releases
mkdir -p /opt/frp && cd /opt/frp
tar -xzf frp_0.70.1_linux_amd64.tar.gz --strip-components=1

cat > frps.toml <<'EOF'
bindPort = 7000
auth.method = "token"
auth.token = "REPLACE_WITH_STRONG_TOKEN"

# optional dashboard — ask user before enabling, adds attack surface
# webServer.addr = "0.0.0.0"
# webServer.port = 7500
# webServer.user = "admin"
# webServer.password = "REPLACE"
EOF

# run as systemd service
cat > /etc/systemd/system/frps.service <<'EOF'
[Unit]
Description=frp server
After=network.target

[Service]
Type=simple
ExecStart=/opt/frp/frps -c /opt/frp/frps.toml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now frps
```

Open firewall/security-group: TCP 7000 (control port) + whatever `remotePort`s
proxies use (e.g. 6000 for the SSH example).

### 2. On each frpc client (e.g. portable-sshd-pro)

```toml
serverAddr = "VPS_PUBLIC_IP"
serverPort = 7000
auth.token = "REPLACE_WITH_STRONG_TOKEN"   # must match frps

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
```

Run: `frpc.exe -c frpc.toml`

### 3. Connect from anywhere

```
ssh -p 6000 user@VPS_PUBLIC_IP
```

## Windows Defender Exclusion (still pending — needs admin)

`frpc.exe`/`frps.exe` commonly get flagged by AV as tunneling tools. Run in an
elevated PowerShell (this session couldn't — not admin):

```powershell
Add-MpPreference -ExclusionPath "D:\frp_0.70.1"
```

## Open Questions For Next Session

- VPS provider/region choice
- Whether to enable frps dashboard (convenience vs attack surface)
- Auth token value (generate + store securely, don't hardcode in committed configs)
- Full list of clients/services to proxy beyond the SSH example

---

## Conformance Addendum

## When to Use
Set up frp (fast reverse proxy) with frps on a VPS and frpc on internal hosts — deploy, configure, connect

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
