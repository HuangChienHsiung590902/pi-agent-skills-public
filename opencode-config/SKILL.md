---
name: opencode-config
description: 管理 opencode 的設定檔 C:\Users\HCH\.config\opencode\opencode.jsonc——新增/修改 provider（接哪個 LLM 後端）、agent（例如省 context 的 local-qa）、mcp server（本機工具，如 playwright、llm-wiki）。當使用者要求「幫 opencode 加一個 provider/model」「opencode 加 MCP server」「opencode 設定」「opencode 連不上/報錯」時使用。
---

# opencode 設定管理

檔案只有一份：`C:\Users\HCH\.config\opencode\opencode.jsonc`（JSONC，可以寫
註解）。改完永遠先跑 `opencode models`（驗證 provider/model 有正確載入）
跟／或 `opencode mcp list`（驗證 MCP server），**不要只改完檔案就假設它會動**
——JSON 語法錯誤、schema 打錯欄位名稱都不會有明顯提示，靠這兩個指令才會
真的看到結果。

## 三種區塊

### 1. `provider`：接哪個 LLM 後端

用 `@ai-sdk/openai-compatible` 接任何 OpenAI 相容的 API（本機 llama.cpp、
遠端 vLLM 都是這樣接）：

```jsonc
"provider": {
  "<provider-id>": {
    "name": "<provider-id>",
    "npm": "@ai-sdk/openai-compatible",
    "options": { "baseURL": "http://<host>:<port>/v1" },
    "models": {
      "<model-id>": {
        "name": "<model-id>",
        "tool_call": true,           // 後端有沒有支援 function calling
        "attachment": true,          // 能不能吃圖片附件
        "modalities": { "input": ["text","image"], "output": ["text"] },
        "limit": { "context": 65536, "output": 4096 }
      }
    }
  }
}
```

**`limit.context` 一定要跟後端伺服器實際的 context window 對齊，不要憑感覺
填**——填太大，opencode 自己的 context 管理邏輯會誤判還有空間，等到真的
超過後端上限才會報錯（`vllm-remote/qwythos-9b` 這個 provider 就是用這個
模式接 [[vllm-qwythos-deploy]] skill 管的那台 vLLM，改了 `--max-model-len`
記得回來同步這裡）。

現有 provider（2026-07-20）：
- `llama-cpp-local`：本機 `D:\_暫時保留\llama.cpp` 的 llama-server（port
  8080），只剩 `qwen3.6-35b` 這顆模型還在用（要手動重啟 exe 才能用）
- `vllm-remote`：接 `10.145.119.19:8182` 的 vLLM，見
  [[vllm-qwythos-deploy]] skill

### 2. `agent`：自訂行為模式

範例：`local-qa`——給小 context 本機模型用的精簡問答 agent，關掉所有內建
工具（省 system prompt token，opencode 預設 `build` agent 光工具定義就要
吃掉近 1.9 萬 token）：

```jsonc
"agent": {
  "local-qa": {
    "mode": "primary",
    "model": "<provider-id>/<model-id>",
    "prompt": "系統提示詞",
    "tools": {
      "bash": false, "edit": false, "write": false, "read": false,
      "glob": false, "grep": false, "list": false, "webfetch": false,
      "websearch": false, "task": false, "todowrite": false, "patch": false
    }
  }
}
```

用法：`opencode run --agent local-qa "問題"`。

### 3. `mcp`：本機 MCP server（例如給瀏覽器工具）

```jsonc
"mcp": {
  "<server-name>": {
    "type": "local",
    "command": ["<exe或.cmd完整路徑>", "arg1", "arg2", ...],
    "enabled": true
  }
}
```

`environment` 欄位（`McpLocalConfig` schema 的合法欄位，物件、字串值）可以
用來給 MCP server 塞一般環境變數（例如 API token），完全沒問題。

**⚠️ 但不要拿它來蓋 `PATH`**，尤其不要寫
`"environment": {"PATH": "...;${PATH}"}` 這種寫法——opencode 不支援
`${PATH}` 變數展開，會把整段當字面字串蓋掉子行程的 PATH，連
`C:\Windows\System32`（`cmd.exe` 所在位置）都不見了，導致 spawn `.cmd` 檔案
失敗噴 `Executable not found in $PATH: "cmd.exe"`。只要 command 陣列裡的
執行檔本來就在系統 PATH 上（`where <exe>` 找得到），完全不需要碰 `PATH`，
子行程會自然繼承完整環境變數；`environment` 欄位留給其他 key（像
token/base URL）用就好。

現有 MCP server：
- `playwright`（2026-07-20）：接 9222 debug Chrome，完整細節（含上面這個
  `${PATH}` 坑的完整說明）在 [[connect-chrome]] skill
- `llm-wiki`（2026-07-27）：LLM Wiki 桌面 App 的知識庫 API，完整設定細節在
  [[connect-llm-wiki]] skill

## 驗證指令

```bash
opencode models              # 列出所有 provider/model，確認新增的有出現
opencode mcp list             # 列出 MCP server 連線狀態
opencode run --agent <name> "測試訊息"     # 實際跑一次驗證 provider+agent 正常
opencode run -m <provider>/<model> "測試訊息"   # 指定 model 測試（略過預設 build agent 的 prompt 開銷可以用 local-qa）
```

**⚠️ 這台機器（2026-07-27 確認過）PATH 上完全沒有 `opencode` CLI 執行檔**，
只有 `D:\APP\opencode-desktop-win-x64.exe`（GUI 桌面版）。拿它硬套 CLI 語法
（例如 `opencode-desktop-win-x64.exe mcp list`）不會印出任何東西到
stdout——它會直接無聲地整個 GUI 開起來（背景會冒出好幾個 `OpenCode` 進
程）。誤觸發的話用這行關掉：
```powershell
Get-Process | Where-Object { $_.ProcessName -eq "OpenCode" } | Stop-Process -Confirm:$false
```
在確認找到真正的 CLI 之前，改完設定後只能請使用者自己開 opencode（或
之後在別台有裝 CLI 的機器上）跑上面幾個指令驗證，不要自己亂試著跑
桌面版 exe。

**mcp 的 "connected" 狀態只代表 MCP stdio handshake 成功**，不代表底層資源
（例如 debug Chrome）真的可用——要驗證整條鏈路，得實際跑一次會呼叫該工具
的請求（例如 `opencode run "用 playwright 工具查看分頁列表"`），才會踩到
底層連線失敗的錯誤（如 `ECONNREFUSED`）。

## 已知：小模型對模糊指令常常不呼叫工具

即使 provider/agent/mcp 全部設定正確、工具本身能正常運作，8-9B 這個量級
的模型對「模糊、沒給具體參數」的請求（例如「幫我打開新聞網頁」，沒給
URL）常常會直接用文字回答（甚至編造內容）而不是呼叫工具；給具體參數
（例如明確網址、明講「用 XX 工具」）才會可靠觸發工具呼叫。這是模型能力
限制，不是設定問題，遇到「工具明明有掛，模型卻不用」的狀況，先確認是不是
指令本身不夠具體，不要急著懷疑 mcp/provider 設定壞了。

---

## Conformance Addendum

## When to Use
管理 opencode 的設定檔 C:\Users\HCH\.config\opencode\opencode.jsonc——新增/修改 provider（接哪個 LLM 後端）、agent（例如省 context 的 local-qa）、mcp server（本機工具，如 playwright、llm-wiki）。當使用者要求「幫 opencode 加一個 provider/model」「opencode 加 MCP server」「opencode 設定」「opencode 連不上/報錯」時使用。

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
