---
name: pi-sdk-development
description: >
  Use when the user asks what the Pi SDK is for, whether it is used to make an
  agent, or how to develop Node.js/TypeScript integrations with
  `@earendil-works/pi-coding-agent` SDK: embedding Pi as an agent service,
  sub-agent, custom UI backend, custom tools/extensions/providers, sessions,
  streaming events, or RPC-vs-SDK choice.
---

# Pi SDK Development

## When to Use

使用者提到以下任一情境時使用本 skill：

- 「Pi SDK 是做什麼」、「Pi SDK 是不是要做成 agent」、「如何開發 Pi SDK」。
- 想把 `pi` 包成 Web/API/後端服務、聊天框、子 agent、agent pipeline。
- 想在 Node.js / TypeScript 程式中呼叫 `@earendil-works/pi-coding-agent`。
- 想開發 Pi custom tool、extension、provider、模型 adapter、或串流 UI。
- 需要判斷 SDK、CLI、RPC mode 哪個適合。
- 需要為 Pi SDK 專案建立最小可跑骨架。

## Core Rule

**Pi SDK 的核心用途不是重新發明一個 agent framework，而是把已經存在的 Pi coding agent 嵌入、控制、包裝或擴充成你自己的 agent / 子 agent / agent service。**

回答此類問題時要先說清楚：

- `pi` CLI：給人直接在終端機使用的現成 coding agent。
- Pi SDK：給 Node.js / TypeScript 程式建立與控制 `AgentSession`，把 Pi agent 嵌入產品或自動化流程。
- Pi custom tool / extension / provider：擴充 Pi agent 的工具、事件、命令、模型供應商。
- Pi RPC mode：給非 Node.js 系統用 subprocess + JSON protocol 控制 Pi，隔離性較好但型別與狀態控制較少。

## Procedure

1. **先判斷使用者的開發目標**
   - 只是手動叫 Pi 改程式：建議用 `pi` CLI。
   - Node.js / TypeScript 程式內要直接控制 agent：建議用 SDK。
   - Java / Python / Go / 其他語言整合：優先考慮 `pi --mode rpc --no-session`。
   - Web 聊天框或後端 API：可用 SDK 包一層 HTTP server。
   - 要新增工具能力：用 `defineTool()` 或 extension 的 `pi.registerTool()`。
   - 要接自訂模型：開發 provider / extension，或先用 OpenAI-compatible/OmniRoute。

2. **最小安裝**
   ```bash
   mkdir pi-sdk-demo
   cd pi-sdk-demo
   npm init -y
   npm install @earendil-works/pi-coding-agent typebox
   npm install -D typescript tsx @types/node
   ```

3. **建議 `package.json`**
   ```json
   {
     "type": "module",
     "scripts": {
       "dev": "tsx src/index.ts"
     }
   }
   ```

4. **建議 `tsconfig.json`**
   ```json
   {
     "compilerOptions": {
       "target": "ES2022",
       "module": "NodeNext",
       "moduleResolution": "NodeNext",
       "strict": true,
       "esModuleInterop": true,
       "skipLibCheck": true
     }
   }
   ```

5. **最小 `createAgentSession()` 範例**
   ```ts
   import {
     createAgentSession,
     ModelRuntime,
     SessionManager,
   } from "@earendil-works/pi-coding-agent";

   const modelRuntime = await ModelRuntime.create();

   const { session } = await createAgentSession({
     modelRuntime,
     sessionManager: SessionManager.inMemory(),
   });

   session.subscribe((event) => {
     if (
       event.type === "message_update" &&
       event.assistantMessageEvent.type === "text_delta"
     ) {
       process.stdout.write(event.assistantMessageEvent.delta);
     }
   });

   await session.prompt("請列出目前資料夾有哪些檔案。");
   session.dispose();
   ```

6. **依風險設定工具權限**
   - 讀取/搜尋模式：
     ```ts
     tools: ["read", "grep", "find", "ls"]
     ```
   - 可執行命令與改檔：
     ```ts
     tools: ["read", "bash", "edit", "write", "grep", "find", "ls"]
     ```
   - 對外提供 Web/API 時，預設先用 read-only；除非使用者明確需要並接受風險，才開 `bash/edit/write`。

7. **自訂 tool 範例**
   ```ts
   import { Type } from "typebox";
   import { defineTool } from "@earendil-works/pi-coding-agent";

   const statusTool = defineTool({
     name: "status",
     label: "Status",
     description: "取得目前服務狀態",
     parameters: Type.Object({}),
     execute: async () => ({
       content: [{ type: "text", text: `服務正常，uptime=${process.uptime()} 秒` }],
       details: {},
     }),
   });
   ```

   啟用時：
   ```ts
   const { session } = await createAgentSession({
     customTools: [statusTool],
     tools: ["status"],
   });
   ```

   若同時開內建工具：
   ```ts
   tools: ["read", "bash", "status"]
   ```

8. **自訂 system prompt / context**
   用 `DefaultResourceLoader` 覆寫或追加 agent 規則：
   ```ts
   import { DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

   const loader = new DefaultResourceLoader({
     systemPromptOverride: () => `
   你是內部系統維運助手。
   一律使用繁體中文回答。
   執行破壞性操作前必須先確認。
   `,
   });
   await loader.reload();
   ```

   或加入虛擬 `AGENTS.md`：
   ```ts
   const loader = new DefaultResourceLoader({
     agentsFilesOverride: (current) => ({
       agentsFiles: [
         ...current.agentsFiles,
         {
           path: "/virtual/AGENTS.md",
           content: "# 規則\n\n- 一律用繁體中文回答。\n- 修改檔案前先讀檔。",
         },
       ],
     }),
   });
   await loader.reload();
   ```

9. **HTTP API 包裝模式**
   - Express/Fastify endpoint 內建立 session。
   - 用 `session.subscribe()` 收集或串流 `text_delta`。
   - `await session.prompt(message)` 後回傳結果。
   - `finally` 中 `unsubscribe()` 與 `session.dispose()`。
   - 需要長對話時使用持久化 `SessionManager.create(cwd)` 或自行把 `sessionId` 映射到 session file。

10. **Session 選擇**
    - 一次性、不落地：`SessionManager.inMemory()`。
    - 新增持久 session：`SessionManager.create(process.cwd())`。
    - 接續最近 session：`SessionManager.continueRecent(process.cwd())`。
    - 指定 session file：`SessionManager.open("/path/to/session.jsonl")`。

11. **SDK vs RPC 決策**
    - SDK 適合：Node.js 同 process、要型別安全、要直接存取 `AgentSession`/state、要程式化註冊 tool/extension、要細緻事件串流。
    - RPC 適合：非 Node.js 語言、需要 subprocess 隔離、語言無關 JSON protocol、降低主程式被 agent 依賴污染。
    - RPC 啟動：
      ```bash
      pi --mode rpc --no-session
      ```

## Community / Upstream Patterns

上網查到的 Pi SDK / Pi runtime 常見開發方向，之後回答「可以做什麼」或規劃專案時可優先引用這些模式：

1. **Web UI / Browser Client**
   - 代表：`agegr/pi-web`（Web UI for the pi coding agent）。
   - 模式：瀏覽器 UI + backend，底層讀 Pi session files / resources，讓使用者在瀏覽器中聊天、接續 session、檢視 tool calls、管理模型與專案檔案。
   - 適合延伸成內部 `pi-agent-service` + web console。

2. **Desktop GUI / Electron Shell**
   - 代表：`minghinmatthewlam/pi-gui`（Electron GUI app for the pi coding agent runtime）。
   - 模式：桌面 UI shell 包住 `@earendil-works/pi-coding-agent`；不是重寫 runtime，而是使用 Pi 的 sessions、models、auth、tools。
   - 常見功能：session timeline、inline diff、terminal、git worktree、多 agent orchestration。

3. **Protocol Adapter**
   - 代表：`svkozak/pi-acp`（ACP adapter for pi coding agent），使用 `pi --mode rpc` 接 ACP / Zed 等 client。
   - 模式：外部 client protocol ↔ adapter ↔ Pi RPC / runtime。
   - 適合接 IDE、aipower、自製 WebSocket agent hub、或非 Node.js 系統。

4. **MCP / Tool Gateway**
   - 代表：`nicobailon/pi-mcp-adapter`（Token-efficient MCP adapter for Pi coding agent）。
   - 模式：不要把大量 MCP tool schemas 一次塞進 context；先給 Pi 一個輕量 proxy/router tool，需要時再查詢/呼叫實際 MCP server。
   - 適合本機 HCH 環境：把 AnyTXT、Obsidian、Playwright、ComfyUI、Docker、DB 查詢等工具包成一個省 token gateway。

5. **Web Access / Search Extension**
   - 代表：`nicobailon/pi-web-access`（Web search and content extraction extension）。
   - 模式：用 Pi extension/custom tools 增加搜尋、網頁擷取、影片理解等能力，可支援多 provider（Exa、Brave、Tavily、Firecrawl、Jina、Kagi、SearXNG、DuckDuckGo 等）。
   - 適合自製：`search_internal_docs`、`search_obsidian`、`search_anytxt`、`read_webpage`、`query_aipower_db`。

6. **Multi-agent Communication / Orchestration**
   - 代表：`nicobailon/pi-messenger`、`jayminwest/overstory`、`ZY-LI-F/pi-workbench`。
   - 模式：多個 agent 共享任務、訊息、檔案鎖定、狀態；或用 Kanban/DAG 管控 agent team。
   - 適合：Planner / Coder / Browser Tester / DB Agent / Doc Agent 分工。

7. **Docs / Skills / Playbook**
   - 代表：`enderzcx/pi-docs-playbook`、`badlogic/pi-skills`。
   - 模式：把 Pi 官方文件、經驗規則、技能路由整理成 agent-readable playbook，避免 agent 混淆 SDK、RPC、extension、session、compaction。
   - 本機對應：`D:\OB\skills` 與 `obsidian-mcp-skill-router`。

8. **官方 SDK examples 方向**
   - 上游 repo：`earendil-works/pi/packages/coding-agent/examples/sdk`。
   - 範例包含：
     - `01-minimal.ts`：最小 SDK 使用。
     - `02-custom-model.ts`：選模型與 thinking level。
     - `03-custom-prompt.ts`：替換或修改 system prompt。
     - `04-skills.ts`：discover/filter/replace skills。
     - `05-tools.ts`：built-in tool allowlist。
     - `06-extensions.ts`：logging、blocking、修改結果。
     - `07-context-files.ts`：AGENTS.md context files。
     - `08-prompt-templates.ts`：prompt templates。
     - `09-api-keys-and-oauth.ts`：API key / OAuth。
     - `10-settings.ts`：compaction、retry、terminal settings。
     - `11-sessions.ts`：in-memory / persistent / continue / list sessions。
     - `12-full-control.ts`：完全自訂，不走 discovery。
     - `13-session-runtime.ts`：new/resume/fork/import 等 session replacement。

## Recommended Project Direction for HCH

若使用者問「我該先做什麼」，建議路線：

1. **第一階段：`pi-agent-service` HTTP API**
   - `/chat`
   - `/sessions`
   - `/tools`
   - `/run-task`
   - SSE streaming
   - cwd 限制、tool allowlist、session persistence

2. **第二階段：本機工具 Gateway**
   - 單一 `hch_tool_router` / `local_tool_gateway` custom tool。
   - 內部再分派到 AnyTXT、Obsidian、Playwright、ComfyUI、Docker、usql、aipower/ECP 等。
   - 目標：降低 token schema 成本，不直接暴露所有 MCP tools。

3. **第三階段：Web Console / Desktop UI**
   - 接上 `pi-agent-service`。
   - 顯示 streaming 回覆、tool calls、session list、project files、diff、terminal output。

4. **第四階段：Multi-agent Orchestration**
   - Planner / Coder / Browser Tester / DB Agent / Doc Agent。
   - 先用 file-based queue 或 SQLite queue，之後再做 Kanban/DAG UI。

## Pitfalls

- 不要把「用 SDK 開 agent session」誤解成「從零重寫 agent loop」。Pi 已經內建 agent loop、tools、session、skills、extensions、compaction。
- 對 Web/API 開 `bash/edit/write` 風險很高；等於讓能呼叫該 endpoint 的人透過 agent 執行命令或改檔。必須先做身分驗證、授權、cwd 限制、工具 allowlist。
- `session.subscribe()` 是綁定特定 `AgentSession`；若用 `AgentSessionRuntime` 切換/建立新 session，要重新 subscribe，extension 也可能要重新 bind。
- 如果 prompt 在 streaming 中又送入，要指定 `streamingBehavior: "steer" | "followUp"`，否則會報錯。
- `ModelRuntime.create()` 預設會使用 `~/.pi/agent` 的 auth/model 設定；在本機 HCH 要優先檢查 `D:\.system\.pi\agent`，`C:\Users\HCH\.pi` 多半是 junction。
- 客製化 system prompt、skills、context files 時，注意 `DefaultResourceLoader` 的 `cwd` 與 `agentDir` 會影響 discovery 路徑。
- 如果要保留完整隔離，尤其主系統不是 Node.js，不要硬塞 SDK；用 RPC mode 通常更安全。

## Verification

1. `npm run dev` 可以成功啟動最小 `createAgentSession()` 範例。
2. `session.subscribe()` 能收到 `message_update` / `text_delta` 串流。
3. 設定 `tools: ["read", "grep", "find", "ls"]` 時，agent 不應能改檔或執行任意 shell。
4. 自訂 tool 被模型呼叫後，能收到 `tool_execution_start` / `tool_execution_end` 事件。
5. 若使用 HTTP API 包裝，endpoint 必須在 `finally` 釋放 `unsubscribe()` 與 `session.dispose()`。
6. 若使用本機 HCH 設定，確認 Pi 實際設定來源是 `D:\.system\.pi\agent\settings.json`，不是只看 `C:\Users\HCH\.pi`。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
