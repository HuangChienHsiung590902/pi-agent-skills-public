---
name: ue-cmdlink-mcp
description: >-
  當使用者要從外部對正在跑的 Unreal Engine 5.8 Editor 灌 Console 命令（CmdLink / cmdlink.py）、
  用官方 Experimental MCP（unreal-mcp、http://127.0.0.1:8000/mcp、EditorToolset）、
  或分不清 ConsoleHelp.html、啟動參數、-ExecCmds、CmdLink、MCP 時使用。
  不用於編輯器繁中本地化、SAM 3D Body／IK Rig 角色管線、或 UnrealEditor.exe 完整啟動參數手冊。
---

# UE 5.8 CmdLink + 官方 MCP

從外部控制**正在跑的** Unreal Editor，有兩條路，不要混：

| 通道 | 做什麼 | 本機入口 |
|---|---|---|
| **CmdLink** | 任意 Console 字串（CVar、`stat`、`PY`）並拿 log 回覆 | `D:\BIN\cmdlink.py` / `D:\BIN\cmdlink.cmd` |
| **官方 MCP** | 結構化 Toolset（搜 CVar、截 viewport、選 actor、PIE、資產） | pi `unreal-mcp` → `http://127.0.0.1:8000/mcp` |

`ConsoleHelp.html` 是 Console 下 `Help` 匯出的 CVar／命令百科，**不是**啟動參數手冊，也**不能**代替 CmdLink。  
`UnrealEditor-Cmd.exe` 是無頭／commandlet 編輯器，**不是** CmdLink 客戶端。Launcher 安裝**沒有**官方 `CmdLink.exe`。

## When to Use

- 使用者說 CmdLink、`cmdlink`、`console.CmdLink`、named pipe、從 CLI 灌 UE Console。
- 使用者說 UE 5.8 MCP、`unreal-mcp`、`ModelContextProtocol`、EditorToolset、PluginToolset、`list_toolsets`。
- 要分清：`ConsoleHelp.html` vs `-ExecCmds` vs CmdLink vs MCP。
- CmdLink pipe 連不上、MCP 只有 AgentSkill、或 8000 沒在聽。

不要用於：

- 編輯器繁中（`ue-editor-zh-hant-localization`）。
- SAM 3D Body／Rigify／IK Rig（`sam3d-body-rigify-unreal-pipeline`）。
- cooked game／Standalone `-game`（CmdLink 與 MCP Editor 模組都是 Editor-only）。
- 把 1 萬筆 CVar 全抄進 skill。

## Inputs and Outputs

### Inputs

- 正在跑的 UE 5.8 Editor，專案預設：`C:\Users\HCH\Documents\Unreal Projects\MyProject\MyProject.uproject`
- CmdLink 客戶端：`D:\BIN\cmdlink.py`（wrapper：`D:\BIN\cmdlink.cmd`）
- pi MCP：`D:\.system\.pi\agent\mcp.json` 的 `unreal-mcp` → `http://127.0.0.1:8000/mcp`

### Outputs

- CmdLink：命令的 Console log 文字（失敗時 stderr + exit 2）
- MCP：`list_toolsets` / `describe_toolset` / `call_tool` 的 JSON／文字結果
- 測試產物放 `%TEMP%\pi-work\ue-cmdlink-mcp\`，不要寫進專案或 skill 目錄

## 本機現況（2026-09-10 已驗證）

```text
引擎：C:\Program Files\Epic Games\UE_5.8
專案：C:\Users\HCH\Documents\Unreal Projects\MyProject
CmdLink 客戶端：D:\BIN\cmdlink.py
自動開 pipe：MyProject\Config\ConsoleVariables.ini  [Startup] console.CmdLink.enable=1
MCP URL：http://127.0.0.1:8000/mcp
pi mcp.json：unreal-mcp / lifecycle lazy / 不必再改
```

`MyProject.uproject` 已啟用：`ModelContextProtocol`、`MCPClientToolset`、`EditorToolset`、`PluginToolset`。  
**不要**為了「工具多一點」就開 `AllToolsets`（會一次掛約 20 個 Experimental 插件）。

## Procedure

### 0. 先判斷要用哪條路

1. **啟動時跑一次** → `UnrealEditor.exe <uproject> -ExecCmds="stat unit,r.VSync 0"`（本 skill 不展開啟動參數）。
2. **Editor 已開、要灌任意 Console／拿 log** → CmdLink。
3. **給 agent 選取、截圖、改資產、查 CVar 目前值** → 官方 MCP。
4. **Python API** → Console 或 CmdLink 送 `PY ...`。

兩條路可同時開。MCP **沒有** `ExecuteConsole("stat fps")`；`SearchCVars` 只查變數。

### 1. CmdLink：開 pipe

Editor Console：

```text
console.CmdLink.enable 1
console.CmdLink.key None
```

或啟動參數 `-cmdlink`。MyProject 已在 `Config\ConsoleVariables.ini` 設 startup enable，下次開專案會開：

```text
\\.\pipe\UnrealEngine-CLI
```

`console.CmdLink.key foo` → `\\.\pipe\UnrealEngine-CLI-foo`。多開 Editor 必須用不同 key。建置機（`GIsBuildMachine`）會強制關掉。

插件：`Engine\Plugins\CmdLinkServer\`（Editor-only、Win64、`EnabledByDefault: true`）。模組會載，**pipe 仍要 enable**。

### 2. CmdLink：送命令

```text
cmdlink help
cmdlink r.VSync
cmdlink r.VSync 0
cmdlink stat unit
cmdlink PY print("hi")
cmdlink --key foo help
cmdlink -t 15 DumpCVars r.Lumen
```

協定（`CmdLinkServer.cpp`）：客戶端送 `int32 ArgC`，再重複 ArgC 次 `int32 ArgLen` + `char[ArgLen]`（含 NUL）。`ArgV[0]` 丟掉；`ArgV[1:]` 空白接成一行。回覆：`int32 responseLen` + ANSI 字串（含 NUL）。沒命令就當 `help`。伺服器會試所有 `IConsoleCommandExecutor`（引擎 + Python）。

客戶端原始碼只維護 `D:\BIN\cmdlink.py`，不要在 skill 目錄再放一份。

### 3. 官方 MCP：確認伺服器

預設 `bAutoStartServer` 在 MyProject 的 `DefaultEditorPerProjectUserSettings.ini` 已是 `True`。Console：

```text
ModelContextProtocol.StartServer
ModelContextProtocol.StartServer 8000
ModelContextProtocol.StopServer
ModelContextProtocol.RefreshTools
```

啟動參數：`-ModelContextProtocolStartServer`、`-ModelContextProtocolPort=8000`。  
舊的 `-StartModelContextProtocolServer` 仍可用但 deprecated。

設定類：`UModelContextProtocolSettings`（EditorPerProjectUserSettings）

- `ServerUrlPath=/mcp`
- `ServerPortNumber=8000`
- `bAutoStartServer`
- `bEnableToolSearch=true` → `tools/list` **只**暴露 `list_toolsets`、`describe_toolset`、`call_tool`

pi 已指向同一個 URL，不必跑 `GenerateClientConfig`（那是給 Claude Code／Cursor／VS Code／Gemini／Codex 寫專案根設定檔）。

### 4. 官方 MCP：呼叫工具

順序固定：

1. `unreal-mcp_list_toolsets`
2. `unreal-mcp_describe_toolset`（`toolset_name`）
3. `unreal-mcp_call_tool`（`toolset_name` + `tool_name` **不含** toolset 前綴 + `arguments`）

`tool_name` 用短名，例如 `SearchCVars`，不要送 `EditorToolset.EditorAppToolset.SearchCVars`。

MyProject 重開且 EditorToolset／PluginToolset 已啟用後，至少應看到：

- `EditorToolset.EditorAppToolset` — SearchCVars、CaptureViewport、CaptureEditorImage、選 actor、camera、Content Browser、StartPIE／StopPIE
- `EditorToolset.LogsToolset`
- `PluginToolset.PluginToolset`
- `editor_toolset.toolsets.actor|asset|blueprint|scene|...`
- `ToolsetRegistry.AgentSkillToolset` — 這是**專案內 AgentSkill 資產**，不是 `D:\OB\skills`

若 `list_toolsets` 只剩 AgentSkill：`.uproject` 沒啟用 EditorToolset／PluginToolset，或啟用後還沒重開 Editor。改完插件必須重載／重開，再 `ModelContextProtocol.RefreshTools`。

### 5. 啟用更多 Toolset（最小變更）

只在 `.uproject` 的 `Plugins` 加需要的名字，例如：

```json
{ "Name": "EditorToolset", "Enabled": true, "TargetAllowList": ["Editor"] }
```

引擎裡還有 `AnimationAssistantToolset`、`NiagaraToolsets`、`PCGToolset`、`SlateInspectorToolset` 等，**要哪個加哪個**。不要開 `AllToolsets`。

## Pitfalls

- CmdLink pipe 沒開時 `cmdlink` 會 Win32 2「系統找不到指定的檔案」。先 `console.CmdLink.enable 1` 或重開專案吃 `ConsoleVariables.ini`。
- 正在跑的 Editor **不會**自動吃剛改的 `.uproject`／`ConsoleVariables.ini`；要重開或 Console 手動 enable。
- `bEnableToolSearch=true` 時不要指望 `tools/list` 出現上百個原生 tool。
- `call_tool` 不能呼叫自己。
- MCP HTTP 綁 `127.0.0.1`，不是對外服務。
- 官方 MCP 是 Experimental；Epic 更新可能改 port／schema／tool 名稱。
- `ShowFlag.` / `Slate.` 在 `ConsoleHelp.html` 快捷鈕大小寫不一致，查 CVar 用實際名稱。
- 測試檔、截圖、Dump 輸出放 `%TEMP%\pi-work\ue-cmdlink-mcp\`，不要寫進 MyProject 或本 skill 目錄。

## Verification

1. `Get-Process UnrealEditor` 存在；`Get-NetTCPConnection -LocalPort 8000 -State Listen` 有列。
2. `python D:\BIN\cmdlink.py -t 8 help` 印出 `Console Help:`；`cmdlink.py r.VSync` 印出目前值。
3. MCP `list_toolsets` 含 `EditorToolset.EditorAppToolset`。
4. `call_tool` → toolset `EditorToolset.EditorAppToolset`、tool `SearchCVars`、`arguments.name=console.CmdLink` → `enable` 為 true。
5. 本 skill：資料夾名 = frontmatter `name` = `ue-cmdlink-mcp`；YAML 可解析；`SKILLS_INDEX.md` 搜得到。

## Rules and Limitations

- CmdLink、官方 MCP 與 `ConsoleHelp.html` 是不同通道，不可互相替代或混稱。
- 只操作正在執行且已授權的 UE Editor；不要把 Editor-only 工具用於 cooked game 或 Standalone。
- 修改 `.uproject`、插件或 Console 設定後要重開 Editor；不得把尚未重啟的狀態宣稱為已生效。
- 測試腳本、截圖與 dump 放在 `%TEMP%\pi-work\ue-cmdlink-mcp\`，不要寫入專案或 Skill 目錄。
