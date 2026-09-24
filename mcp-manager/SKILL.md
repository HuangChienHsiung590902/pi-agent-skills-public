---
name: mcp-manager
version: 1.0.0
description: 手動管理 Claude Code MCP 開關與低 token profiles。使用者要求開關 MCP、切換 MCP profile、恢復 MCP、關閉 MCP 時使用。
---

# MCP Manager

## 目的

管理 Claude Code MCP server 開關，優先降低 token/schema 成本。

## 重要規則

- 關 MCP 後，**目前已開啟的 session 不會立刻卸載已載入 schema**；請提醒使用者重開 Claude Code 才會真正省 token。
- 修改設定前先讀現有設定，保留非 MCP 設定。
- 不要測試性亂啟 MCP；只依使用者指定 profile 開。
- 若要完全關閉 MCP，除了移除直接 MCP servers，也要擋 plugin/connector MCP：
  - `disableClaudeAiConnectors: true`
  - `allowedMcpServers: []`
- `claude mcp list --verbose` 不支援；用 `claude mcp list`。

## 目前已知 MCP server 與恢復指令

### Direct MCP

```powershell
claude mcp add anytxt -- node D:/MCP/anytxt-mcp/index.js
claude mcp add llm-wiki -- node D:/MCP/llm-wiki-mcp/index.js
claude mcp add es -- node D:/MCP/es-mcp/index.js
claude mcp add playwright-vrs -- npx @playwright/mcp@latest --cdp-endpoint http://127.0.0.1:9222
```

### Plugin MCP

來自 `~/.claude/settings.json` 的：

```json
"enabledPlugins": {
  "oh-my-claudecode@omc": true,
  "playwright@claude-plugins-official": true,
  "ponytail@ponytail": true,
  "drawio@365-skills": true
}
```

其中 `oh-my-claudecode@omc` 與 `playwright@claude-plugins-official` 會帶 MCP server。若要完全無 MCP，使用 `allowedMcpServers: []` 阻擋；不要輕易關掉整個 plugin，避免技能一起消失。

## Profile 建議

### off / none

用途：最省 token。

操作：

1. `claude mcp remove <name>` 移除 direct MCP。
2. 對同名多 scope server，分別執行：
   - `claude mcp remove <name> -s local`
   - `claude mcp remove <name> -s user`
3. 在 `~/.claude/settings.json` 合併：

```json
{
  "disableClaudeAiConnectors": true,
  "allowedMcpServers": []
}
```

4. 移除 permissions allow 裡的 `mcp__...` 規則。
5. 提醒使用者重開 Claude Code。

### basic

用途：本機搜尋 / 查 code，較省。

```powershell
claude mcp add es -- node D:/MCP/es-mcp/index.js
claude mcp add anytxt -- node D:/MCP/anytxt-mcp/index.js
```

`allowedMcpServers` 應改成允許這些 server，或移除空 allowlist。

### wiki

用途：查知識庫。

```powershell
claude mcp add llm-wiki -- node D:/MCP/llm-wiki-mcp/index.js
```

### browser

用途：瀏覽器自動化。只開一套 Playwright，避免重複 schema。

```powershell
claude mcp add playwright-vrs -- npx @playwright/mcp@latest --cdp-endpoint http://127.0.0.1:9222
```

或使用 plugin Playwright，但不要兩套同時開。

## 常用流程

### 關全部 MCP

```powershell
claude mcp list
claude mcp remove playwright-vrs
claude mcp remove anytxt -s local
claude mcp remove anytxt -s user
claude mcp remove llm-wiki -s local
claude mcp remove llm-wiki -s user
claude mcp remove es -s local
claude mcp remove es -s user
```

然後編輯 `~/.claude/settings.json`：

```json
"disableClaudeAiConnectors": true,
"allowedMcpServers": []
```

### 驗證

```powershell
python -m json.tool "$env:USERPROFILE/.claude/settings.json" > $null
python -m json.tool "$env:USERPROFILE/.claude/settings.local.json" > $null
claude mcp list
```

若 `claude mcp list` 仍看到 plugin MCP，通常是 plugin 仍啟用或目前 session 未重載；重開 Claude Code 後再確認。

## 回覆模板

完成時簡短回覆：

```text
已切到 <profile>。目前 session 已載入的 MCP schema 不會消失；請重開 Claude Code 才會真的省 token。
驗證：<claude mcp list 結果摘要>
```

---

## Conformance Addendum

## When to Use
手動管理 Claude Code MCP 開關與低 token profiles。使用者要求開關 MCP、切換 MCP profile、恢復 MCP、關閉 MCP 時使用。

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
