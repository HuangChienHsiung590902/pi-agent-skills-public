---
name: local-oss-llm-team
description: 用 opencode serve + qwen cli 這兩個本地開源 LLM(跑在 10.145.119.19,GPU 共用)組成序列化 team 分工任務。當使用者說「開源 LLM team」「用本地模型分工」「GPU 不夠用但想組 team」「opencode 跟 qwen cli 一起做事」時使用。
---

# Local OSS LLM Team

用 [[opencode-api-control]] 驅動的 opencode serve,加上 qwen cli,在
`10.145.119.19`(hostname gigabyte)這台**單張共用 GPU** 上分工做任務。

## 前提:為什麼不能真平行

這台機器的 GPU 一次只能餵一個服務(llama.cpp / ollama / whisper 三選一,見
memory `omc-learned/llama-cpp-server-setup.md` Step 7)。目前的配置:

| 後端 | port | 別名 | 誰在用 |
|---|---|---|---|
| llama.cpp(gguf, 8.9B) | 8181 | `qwythos`(opencode、qwen cli 兩邊命名已統一) | **同一個 server,同一顆模型** |
| ollama(qwen3.6:35b-a3b) | 11434 | opencode 的 `qwen3.6` | 另一顆更大的模型 |

**重要認知:** opencode 跟 qwen cli 打的 `qwythos` 是同一台 llama.cpp
server、同一顆權重——不是兩個獨立大腦(原本 opencode 這邊叫 `qwython` 是命名
不一致造成的錯誤,已改名統一成 `qwythos`)。真正「第二顆不同的大腦」是 ollama
上的 `qwen3.6:35b-a3b`,但它跟 llama.cpp 搶同一張卡,不能同時開。

所以這個 skill 的策略是**序列分工,依後端分組換卡**,不是真平行多工:同一個
後端的任務盡量排在一起做,只有換後端的時候才真的付 GPU 切換的代價(卸載/重載
模型有幾秒到十幾秒的 race condition,腳本已內建等待緩衝)。

## 用法

一律用 CLI `oss-team-dispatch`(D:\BIN 上,已在 PATH),不要重新手動組
`opencode-api` / `qwen -p` / `gpu-switch-remote` 三邊呼叫。

```powershell
oss-team-dispatch <tasks.json>
oss-team-dispatch <tasks.json> -ReportPath D:\path\report.md
oss-team-dispatch <tasks.json> -KeepGpuLoaded   # 跑完不自動釋放 GPU(預設跑完會 gpu-switch stop)
```

`tasks.json` 範例(見 `examples/sample-tasks.json`):

```json
[
  { "worker": "opencode", "model": "qwythos", "prompt": "...", "timeoutSec": 60 },
  { "worker": "qwencli",  "model": "qwythos", "prompt": "...", "timeoutSec": 60, "directory": "C:\\some\\project" },
  { "worker": "opencode", "model": "qwen3.6", "prompt": "...", "timeoutSec": 180, "sessionId": "ses_xxx" }
]
```

| 欄位 | 必填 | 說明 |
|---|---|---|
| `worker` | 是 | `opencode` 或 `qwencli` |
| `model` | 是 | `qwythos`(-> llama.cpp 後端)或 `qwen3.6`(-> ollama 後端)。**只用來決定要不要切 GPU**,qwen cli 實際跑哪顆模型仍由它自己的設定決定,不會真的把這個值傳給 `qwen` 指令 |
| `prompt` | 是 | 要問的任務內容 |
| `directory` | 否 | opencode 的 session 工作目錄 / qwencli 執行時的 cwd |
| `sessionId` | 否 | 只給 opencode 用,延續同一個 session 做多輪對話 |
| `timeoutSec` | 否 | 預設 opencode 180 秒、qwen cli 由 `qwen` 自己控制(此欄位對 qwencli 目前無效,保留欄位給未來用) |

**限制:** qwen cli 目前只接了 `qwythos`(llama.cpp 後端),`worker: qwencli` +
`model: qwen3.6` 會直接報錯拒絕——qwen cli 沒有註冊 ollama 後端。要用
`qwen3.6:35b-a3b` 只能透過 `worker: opencode`。

## 運作流程

1. 用 `gpu-switch-remote.bat status` 讀目前 GPU 是誰在用。
2. 依序處理 `tasks.json` 裡的每個 task:
   - 算出這個 task 需要的後端(llama / ollama)。
   - 如果跟目前後端不同,才呼叫 `gpu-switch-remote.bat <llama|ollama>` 切換,
     並等 `-SwitchSettleSec`(預設 8 秒)緩衝。
   - 依 `worker` 呼叫 `opencode-api.bat run ...` 或 `qwen -p ...`,拿到回覆
     (兩邊都會把 `<think>...</think>` 內心獨白濾掉,只留最終答案)。
3. 全部跑完,預設呼叫 `gpu-switch-remote.bat stop` 釋放 VRAM(除非
   `-KeepGpuLoaded`)。
4. 寫一份 markdown 報告(每個 task 的 prompt + 回覆/錯誤),存在 tasks.json
   旁邊(或 `-ReportPath` 指定的位置)。

## 已知的坑

- **多行 prompt 一定要直接呼叫 `opencode-api.ps1`(`pwsh -File`),不能透過 `.bat`。**
  cmd.exe 在啟動 batch 檔那一刻就會按換行把命令列切開,`-Text` 只要有換行,第二行
  以後全部消失——這是 cmd.exe 的天生限制,不是 bug 能修的。`scripts/oss-team-dispatch.ps1`
  的 opencode worker 已經改成直接呼叫 `.ps1`,不用擔心;但如果你自己在別的地方
  手動組 `opencode-api.bat run -Text "...多行..."`,一樣會中招。詳見
  [[opencode-api-control]] 的已知錯誤章節。
- **`.bat` 檔絕對不要寫中文註解**(見 [[opencode-api-control]] 記過的教訓,
  `oss-team-dispatch.bat` 已確認只用英文註解)。
- 換後端後第一個請求可能吃到「模型還在載入」的延遲(opencode 那邊會自動輪詢
  等,qwen cli 那邊目前沒有這層保護,如果切換後 qwen cli 任務失敗/空白,先加大
  `-SwitchSettleSec` 再重跑那個 task)。
- 排 `tasks.json` 的順序時,把同後端的 task 排在一起,能省掉重複切換 GPU 的
  時間成本——腳本不會自動重排順序(尊重你給的執行順序,因為任務之間可能有前後
  依賴)。
- 改動一律改正本 `C:\Users\HCH\.claude\skills\local-oss-llm-team\scripts\scripts/oss-team-dispatch.ps1`,
  `D:\BIN\oss-team-dispatch.bat` 只是薄轉發層。

## 相關

- [[opencode-api-control]] — opencode serve 的 session/prompt/poll CLI,這裡
  的 opencode worker 底層就是呼叫它。
- `omc-learned/llama-cpp-server-setup.md`（Step 7）— GPU 共用機制與 `gpu-switch-remote` 用法。
- [[reference-llama-vision-test]] — 同一台 llama.cpp server 的多模態測試,
  同樣受 GPU 切換限制。

## 變更記錄

- 2026-07-09:使用者把 `opencode.jsonc` 裡 `llama_cpp` provider 的 model 別名
  從 `qwython` 改成 `qwythos`,跟 qwen cli 統一。這裡的 SKILL.md、
  `scripts/oss-team-dispatch.ps1`、`examples/sample-tasks.json` 已同步更新,不用再認
  `qwython` 這個舊名字。

---

## Conformance Addendum

## When to Use
用 opencode serve + qwen cli 這兩個本地開源 LLM(跑在 10.145.119.19,GPU 共用)組成序列化 team 分工任務。當使用者說「開源 LLM team」「用本地模型分工」「GPU 不夠用但想組 team」「opencode 跟 qwen cli 一起做事」時使用。

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
