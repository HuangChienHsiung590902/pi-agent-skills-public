---
name: llama-vision-test
description: 測試自建 llama.cpp server（provider llama_cpp/qwythos，透過 opencode CLI）的多模態影像辨識能力——OCR 圖片文字、描述圖片內容、數人數等。當使用者要求「測試 llama.cpp 的圖片辨識」、「用 opencode 傳圖片給 llama.cpp」、「llama.cpp 能不能看圖」、「qwythos OCR」、「測試 llama.cpp vision」時使用。
---

# llama.cpp 視覺能力測試

透過 opencode CLI 呼叫本機設定的 `llama_cpp` provider（見
`C:\Users\HCH\.config\opencode\opencode.jsonc`，baseURL 指向
`http://10.145.119.19:8181/v1`，模型 `qwythos`,跟 qwen cli 那邊叫的名字已統一)測試圖片辨識/OCR 能力。

**不要重新手動組 `opencode run` 指令，直接跑這裡的腳本。**

## 用法

```powershell
# OCR（預設 prompt，抽取圖片所有文字）
pwsh -NoProfile -File C:\Users\HCH\.claude\skills\llama-vision-test\scripts\scripts/llama-vision-ocr-test.ps1 -ImagePath "D:\xxx.jpg"

# 自訂問題（位置參數也可以）
C:\Users\HCH\.claude\skills\llama-vision-test\scripts\scripts/llama-vision-ocr-test.ps1 D:\xxx.jpg "照片有多少人"

# 加 -Pure 關閉外部插件（更保險，但影響整個 opencode run）
C:\Users\HCH\.claude\skills\llama-vision-test\scripts\scripts/llama-vision-ocr-test.ps1 -ImagePath "D:\xxx.jpg" -Pure

# 也有 .bat 版本，參數原樣往下傳
C:\Users\HCH\.claude\skills\llama-vision-test\scripts\scripts/llama-vision-ocr-test.bat D:\xxx.jpg "描述這張圖片的內容"
```

參數：
| 參數 | 說明 |
|------|------|
| `-ImagePath`（位置 0，必填） | 圖片路徑 |
| `-Prompt`（位置 1，選填） | 問題/任務，預設為 OCR 抽字。**不管有沒有自訂，安全指令一律自動附加**（見下方坑） |
| `-Model`（選填） | 預設 `llama_cpp/qwythos` |
| `-Pure`（開關） | 加 opencode 的 `--pure`，關閉全部外部插件 |
| `-NoSafeguard`（開關） | 跳過自動附加安全指令，只在明確想讓模型自由呼叫工具時使用 |

## 已知踩坑

> **坑 1：opencode 的 build agent 會透過 superpowers 插件拿到 Claude 的全部
> skills（含 `vision-llm`）當工具用。** 如果 model 設定沒標記
> `attachment: true` + `modalities.input: ["text","image"]`，opencode 只會把
> 圖片路徑當純文字傳給模型，模型「看不到」圖片內容，於是會亂猜著去呼叫
> `vision-llm` 之類的工具改用別台機器（10.145.119.234 Ollama Qwen3-VL）做
> OCR，等於根本沒測到 llama.cpp 自己的能力。
>
> **兩種解法**（腳本都支援）：
> 1. prompt 裡明講「不要呼叫任何工具」——輕量，單次生效。
> 2. `-Pure`（對應 opencode `--pure`）關閉所有外部插件——較保險但整個 run
>    都不能用其他插件功能。
>
> 另外 `opencode.jsonc` 裡 `llama_cpp.models.qwythos` 必須加：
> ```jsonc
> "attachment": true,
> "modalities": { "input": ["text", "image"], "output": ["text"] }
> ```
> 否則圖片附件不會被當成多模態內容送出。此設定已經修好，不用再改。

> **坑 2：安全指令絕對不能寫死在 `-Prompt` 的預設值裡。** 第一版腳本把
> 「不要呼叫任何工具」直接寫進 `-Prompt` 參數的預設字串。只要使用者自己帶了
> `-Prompt`（或位置參數）問自訂問題，整段預設值連同安全指令一起被換掉，
> 模型立刻故態復萌去呼叫 `vision-llm`。**已修正**：安全指令是獨立變數
> `$NoToolsClause`，不管有沒有帶自訂 Prompt 都會自動附加在後面（除非明確加
> `-NoSafeguard`）。**改這支腳本時絕對不要把安全指令塞回 Prompt 預設值裡。**

> **坑 3：`.bat` 檔的 REM 註解裡不要放中文（UTF-8）。** 會跟 cmd.exe
> 非 UTF-8 codepage 衝突，把後面幾行命令解析炸掉（症狀：檔名被離奇截斷、
> 報「不是內部或外部命令」）。批次檔一律用純 ASCII 註解。

> **坑 4：透過 Claude Code 的 PowerShell 工具直接呼叫腳本，曾經卡住 20+
> 分鐘沒有任何輸出**（`opencode run` process 掛著不動，但 llama.cpp server
> 本身用 curl 測還是正常回應）。改用 `pwsh -File ...`（例如透過 Bash 工具
> 執行，或使用者自己在終端機打）則每次都在 1-3 分鐘內正常完成。原因不明，
> 懷疑是背景執行時 stdin 處理方式的問題。**若透過 PowerShell 工具呼叫卡住
> 超過 3 分鐘，直接砍掉該 process 重試（找 `opencode run ...` 的 process，
> 不要砍到常駐的 `opencode serve`）。**

## 相關

- llama.cpp server 本身的安裝/部署見 `omc-learned` 的
  `llama-cpp-server-setup.md`（CUDA、build、systemd 等）。這個 skill 只管
  「怎麼測試已經在跑的 server 有沒有視覺能力」。
- 使用者在 `D:\BIN\Vision\`（原本在 `Desktop\Vision\`，後來搬過去了）有一份
  工作副本（`vision.bat` + `scripts/llama-vision-ocr-test.ps1`），跟這裡的正本邏輯
  一致，修 bug 或更新腳本時記得兩邊都要同步。`D:\BIN` 本身在系統 PATH 上，
  但 `D:\BIN\Vision` 子目錄目前不在 PATH，所以還是要用相對/完整路徑呼叫
  `vision.bat`，不能直接打 `vision` 裸指令。
- **同一台主機（10.145.119.19）的 GPU 是三個服務（llama.cpp / ollama /
  whisper）共用，靠 `gpu-switch` 手動切換的**（見 `llama-cpp-server-setup.md`
  Step 7）。如果這裡的腳本連不上 server、或回應異常慢/逾時，先確認當下是不是
  GPU 被切去跑 ollama 或 whisper 了（遠端 `gpu-switch status`，或本機
  `D:\BIN\gpu-switch-remote.bat status`），不要一開始就當成 llama.cpp/opencode
  設定壞了去查半天。
- 如果需求是「轉錄音檔」而不是「看圖片」，那是另一個獨立 skill
  `whisper-service`，不要混用這裡的腳本。

---

## Conformance Addendum

## When to Use
測試自建 llama.cpp server（provider llama_cpp/qwythos，透過 opencode CLI）的多模態影像辨識能力——OCR 圖片文字、描述圖片內容、數人數等。當使用者要求「測試 llama.cpp 的圖片辨識」、「用 opencode 傳圖片給 llama.cpp」、「llama.cpp 能不能看圖」、「qwythos OCR」、「測試 llama.cpp vision」時使用。

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
