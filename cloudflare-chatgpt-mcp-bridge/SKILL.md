---
name: cloudflare-chatgpt-mcp-bridge
description: Configure and verify the Cloudflare Tunnel bridge from Jetson/ZeroTier phone MCP servers into ChatGPT developer plugins, especially the ESP32 HID Bridge MCP endpoint.
---

# Cloudflare ChatGPT MCP Bridge

Use this skill when working on the user's Jetson-hosted Cloudflare Tunnel that exposes a phone MCP server to ChatGPT plugins/connectors.

## Known Local Topology

- Windows host workspace: `C:\Users\HCH`
- User skill folder: `D:\OB\skills`
- HID project: `D:\Github\esp32_hid_bridge`
- Related skill: `D:\OB\skills\phone-bluetooth-hid-t25`
- Chrome CDP helper skill: `D:\OB\skills\connect-chrome`
- Jetson SSH target: `hch@10.145.119.12`
- Jetson ZeroTier network: `AI3`
- Phone ZeroTier IP: `10.145.119.96`
- Phone MCP port: `5588`
- Phone bridge relay port: `5590`
- Phone command port: `5566`
- MCP token/path secret: `<KB_BEARER_TOKEN>`
- Cloudflare MCP URL:
  `https://mcp.james-huang.org/mcp/<KB_BEARER_TOKEN>`

## Cloudflare Tunnel State

The Jetson uses `cloudflared` service with config at `/etc/cloudflared/config.yml`.

Expected tunnel config:

```yaml
tunnel: 1ec91570-0cd8-4ce2-908a-3a1713de3114
credentials-file: /etc/cloudflared/1ec91570-0cd8-4ce2-908a-3a1713de3114.json

ingress:
  - hostname: test.james-huang.org
    service: http://127.0.0.1:18081
  - hostname: mcp.james-huang.org
    service: http://10.145.119.96:5588
  - service: http_status:404
```

Do not remove or break the existing `test.james-huang.org` rule when changing the MCP rule.

## Verification Workflow

Before changing ChatGPT plugin settings, verify the network path.

From Windows, public Cloudflare URL should reach the MCP server:

```powershell
curl.exe -i --max-time 15 https://mcp.james-huang.org/mcp/<KB_BEARER_TOKEN>
```

For a GET request, `405 Method Not Allowed` is acceptable and means the request reached the MCP server. `502 Bad Gateway` means Cloudflare cannot reach the phone MCP origin.

From Jetson, direct phone connectivity should show ports open:

```bash
ssh hch@10.145.119.12 "cloudflared tunnel --config /etc/cloudflared/config.yml ingress rule https://mcp.james-huang.org/mcp/<KB_BEARER_TOKEN>; (timeout 3 bash -c '</dev/tcp/10.145.119.96/5588' && echo '5588 open' || echo '5588 closed'); (timeout 3 bash -c '</dev/tcp/10.145.119.96/5590' && echo '5590 open' || echo '5590 closed'); (timeout 3 bash -c '</dev/tcp/10.145.119.96/5566' && echo '5566 open' || echo '5566 closed')"
```

Test MCP initialization through Cloudflare:

```powershell
$body = '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl-test","version":"0.1"}}}'
curl.exe -i --max-time 20 -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" -X POST --data $body https://mcp.james-huang.org/mcp/<KB_BEARER_TOKEN>
```

Successful response:

```json
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{}},"serverInfo":{"name":"esp32-hid-bridge","version":"1.0.0"}}}
```

## ChatGPT Plugin Setup

Existing developer plugin URLs may not be editable in ChatGPT. If the existing plugin is `ESP32 HID Bridge` with old URL `https://hid.james-huang.org/...`, preserve it unless the user explicitly approves deletion.

Create a new ChatGPT plugin/connector instead:

- Name: `ESP32 HID Bridge CF`
- Description: `透過 BLE 控制電腦的鍵盤與滑鼠 (ESP32-S3 HID)`
- Server URL: `https://mcp.james-huang.org/mcp/<KB_BEARER_TOKEN>`
- Connection mode: `伺服器 URL`
- Authentication: `無驗證` (`NONE`)
- Risk acknowledgement: checked

Known successful ChatGPT plugin ID:

```text
plugin_asdk_app_6a89a08260048191a5db93b35187d445
```

If ChatGPT returns `連接器名稱已存在`, use a distinct name or manage the existing connector. If it returns a generic creation error while public curl gives `502`, fix the phone/ZeroTier origin first.

## Chrome CDP Notes

When the user asks to control ChatGPT in Chrome, read and follow `D:\OB\skills\connect-chrome\SKILL.md`.

Chrome is expected to expose CDP on `127.0.0.1:9222`. If raw WebSocket receives an origin rejection, Python `websocket-client` can connect with `suppress_origin=True`.

The ChatGPT create-plugin modal uses:

- `#custom-connector-name`
- `#custom-connector-description`
- `#custom-connector-url`
- `#custom-connector-auth`, where `NONE` means `無驗證`
- `#trust-checkbox`

Submitting with `OAUTH` may show OAuth discovery UI and fail for this MCP server. Set `#custom-connector-auth` to `NONE` before submitting.

---

## Conformance Addendum

## When to Use
Configure and verify the Cloudflare Tunnel bridge from Jetson/ZeroTier phone MCP servers into ChatGPT developer plugins, especially the ESP32 HID Bridge MCP endpoint.

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
