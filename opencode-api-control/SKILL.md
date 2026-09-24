---
name: opencode-api-control
description: Use when you need to drive opencode (D:\BIN or C:\Users\HCH\.opencode) programmatically instead of the TUI — creating sessions, sending non-blocking prompts, polling status, aborting a stuck/runaway run, or coordinating multiple opencode tasks in parallel. Triggers on "驅動 opencode"、"opencode API"、"opencode 卡住"、"opencode session"、"平行跑 opencode 任務"、opencode serve headless control.
---

# opencode API 驅動

## 概觀

opencode 有兩種被控制的方式:`opencode run "..."` 是阻塞式黑盒(等到跑完才有輸出,中途看不到狀態、停不掉);`opencode serve`(預設 port 4096)開的是完整 REST API(OpenAPI 規格見 `http://127.0.0.1:4096/doc`),可以非阻塞送出任務、隨時輪詢、平行開多個 session、卡住時 abort/換 session 重試。需要監控/介入/平行執行時一律走這條路。

**一律用 CLI `opencode-api`(D:\BIN 上,已在 PATH),不要重新手動組 curl/jq、也不要繞過它直接呼叫 .ps1。** 已對照 opencode 1.17.15 的 API 實測驗證過(建 session → 送 prompt → 輪詢 → abort 全部跑通),PowerShell/cmd/Git Bash 三種殼都能直接呼叫。

- CLI 入口(工作副本):`D:\BIN\opencode-api.bat` → 轉呼叫 → 正本 `C:\Users\HCH\.claude\skills\opencode-api-control\scripts\scripts/opencode-api.ps1`(單一正本,兩邊不用手動同步,跟 `llama-vision-test` 那種要手動同步兩份的模式不一樣)。
- 改動一律改正本 `.ps1`,`.bat` 只是薄轉發層,通常不用碰。

## 用法

```bash
# 一次到位:確保 server 在跑、開新 session、送 prompt、輪詢到有結果為止
opencode-api run -Provider llama_cpp -Model qwythos -Text "你的任務" -TimeoutSec 120

# 續用同一個 session(多輪對話/延續任務)
opencode-api run -SessionId ses_xxx -Provider ollama -Model qwen3.6:35b-a3b -Text "接著做..."

# 分開控制:自己管 session 生命週期
opencode-api new -Title "my-task" -Directory "C:\path\to\project"   # 回傳含 id 的 JSON
opencode-api prompt -SessionId ses_xxx -Text "任務內容"               # 非阻塞送出
opencode-api poll -SessionId ses_xxx -TimeoutSec 120                 # 輪詢直到有回覆
opencode-api abort -SessionId ses_xxx                                # 中途中止
opencode-api list                                                    # 列出所有 session
opencode-api messages -SessionId ses_xxx                             # 該 session 完整訊息(JSON)
opencode-api status                                                  # server 是否存活
opencode-api start                                                   # server 沒開就幫你拉起來
```

## 參數速查

| 參數 | 預設 | 說明 |
|---|---|---|
| `-BaseUrl` | `http://127.0.0.1:4096` | server 位址 |
| `-Provider`/`-Model` | `llama_cpp`/`qwythos` | 對照 `opencode.jsonc` 裡註冊的 provider,另一個常用組合是 `ollama`/`qwen3.6:35b-a3b` |
| `-Agent` | `build` | opencode agent 名稱 |
| `-Directory` | 目前工作目錄 | 新 session 的工作目錄(影響它能操作的檔案範圍) |
| `-TimeoutSec` / `-PollIntervalSec` | 180 / 3 | 輪詢等待上限與間隔 |

## 已知的坑:「卡住沒反應」不等於 opencode 壞了

實測踩過:送出 prompt 後 assistant 訊息一直是 `tokens.output=0`、`error=null`,像卡死——**真正原因是遠端 GPU 主機(見 `omc-learned/llama-cpp-server-setup.md` Step 7)的模型還在中途載入,回應 `503 Loading model`,而 opencode 沒有把這個狀態 surface 出來**。排查順序:

1. 先直接 curl 模型自己的 endpoint 確認活著:`curl http://<host>:<port>/v1/models`。
2. 如果那個模型是 llama.cpp 跑的,check `gpu-switch-remote.sh status`(見同一份 GPU 記憶),確認 GPU 現在被哪個服務(llama.cpp/ollama/whisper)占用——三選一,同時只能一個。
3. 不是名稱打錯:單模型 llama.cpp server 對任何 model 欄位值都會用當前載入的那顆回應,不會因為名稱對不上而卡住。
4. 確認模型端 OK 之後,用 `abort` 中止卡住的 session、開新 session 重送即可(舊 session 卡住不會自動清空佇列,同一個 session 後面排的 prompt 會一直卡在它後面)。

## 常見錯誤

- `Format-Table -AutoSize` 在這個 script 的多行管線裡對這台機器的終端捕捉環境會顯示錯亂(資料底層是對的,只是排版故障)——`list`/`poll` 因此改用純文字逐行輸出,不要再改回 Format-Table。
- `prompt_async` 回 204 空內容,正常,不代表訊息已經有結果,一定要接著 `poll`。
- 同一個 session 若還有前一個 prompt 在跑,新送的 prompt 會**排隊**,不會取代或平行跑——要平行,開不同的 session。
- **`-Text` 內容有換行(多行 prompt)時,絕對不要透過 `.bat` 呼叫,一定要直接 `pwsh -NoProfile -File C:\Users\HCH\.claude\skills\opencode-api-control\scripts\scripts/opencode-api.ps1 ...`。** 實測踩過:透過 `opencode-api.bat` 送一個兩行的 `-Text`,結果 session 裡的 user message 只收到第一行,第二行(含後面全部內容)整個消失——根本原因是 cmd.exe 在啟動 batch 檔那一刻就已經按換行把命令列切開了,batch 腳本(`%*`)壓根沒機會看到第一行以後的內容,**這是 cmd.exe 命令列解析階段的天生限制,`.bat` 檔本身無法修好**。單行 prompt 完全沒事,只有多行才會中招,所以很容易漏測。判斷準則:只要 prompt 是用 heredoc/多行字串組出來的(例如批次跑一系列題目、從檔案讀 prompt),一律直接呼叫 `.ps1`,不要用 `opencode-api` 這個 `.bat` 別名。
- **`.bat` 檔絕對不要寫中文註解/任何非 ASCII 字元。** 踩過一次真的很痛的坑:`opencode-api.bat` 的 `REM` 註解寫了中文(UTF-8 多位元組),cmd.exe 的批次檔解析器直接壞掉,陷入無窮迴圈狂印 `'ipt' is not recognized`/`'r' is not recognized` 之類的錯誤片段,幾分鐘內洗出十幾萬行、進程占著不退出。跟 `D:\BIN\Vision\vision.bat`(純 ASCII,正常)對照後才抓到差異只在中文字元(換行符號兩邊都是 LF,不是問題)。**`.bat` wrapper 只能寫英文註解**,中文說明寫在 SKILL.md 或 `.ps1` 本體(pwsh 執行期讀 UTF-8 沒問題,壞的只有 cmd.exe 批次檔解析階段)。中招時的急救:`Get-CimInstance Win32_Process` 查出卡住的 `cmd.exe`/`pwsh.exe` 真實 PID 跟 CommandLine(不要瞎猜亂殺,MCP server 那些 `npx`/`codegraph`/`playwright-mcp` 常駐 cmd.exe 是正常的,只殺剛好對得上時間戳跟指令內容的那個),`Stop-Process -Force` 收掉。

---

## Conformance Addendum

## When to Use
Use when you need to drive opencode (D:\BIN or C:\Users\HCH\.opencode) programmatically instead of the TUI — creating sessions, sending non-blocking prompts, polling status, aborting a stuck/runaway run, or coordinating multiple opencode tasks in parallel. Triggers on "驅動 opencode"、"opencode API"、"opencode 卡住"、"opencode session"、"平行跑 opencode 任務"、opencode serve headless control.

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
