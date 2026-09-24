---
name: whisper-service
description: >-
  遠端 whisper ASR 服務（onerahmet/openai-whisper-asr-webservice，跑在 10.145.119.19:9000）的完整管理——
  Docker 容器啟停/狀態/log（正常應保持關閉，只在需要轉錄時開）、GPU 共用協調（跟 llama.cpp/ollama 搶
  VRAM，優先用 gpu-switch 而非直接 docker start/stop）、CLI 轉錄呼叫（音檔轉文字/字幕）、逐句字幕播放器
  player.html。當使用者要求「轉錄音檔」、「whisper 轉文字」、「音檔轉字幕」、「檢查/啟動/停止 whisper 服務」、
  「whisper 掛了」時使用。合併自原本的 `whisper-cli-transcribe`（呼叫）與 `whisper-docker-service`（容器管理）
  兩個 skill。
triggers:
  - whisper
  - 轉錄音檔
  - 音檔轉字幕
  - whisper 服務
  - whisper 轉文字
---

# Whisper ASR 服務（容器管理 + CLI 轉錄）

遠端 GPU 主機 `hch@10.145.119.19` 上的 Docker 容器 `whisper-service`
（image: `onerahmet/openai-whisper-asr-webservice:latest-gpu`），提供 `POST /asr` REST API。

**該服務跟 llama.cpp / ollama 共用同一台機器的 GPU VRAM**（見 `omc-learned` 的
`llama-cpp-server-setup.md` Step 7 / `gpu-switch`），同一時間只能有一個服務吃 GPU。

---

## Part A — 容器管理

### 快速參考

| Task | Command |
|---|---|
| Status | `ssh hch@10.145.119.19 "docker ps -a --filter name=whisper-service"` |
| Start | `ssh hch@10.145.119.19 "docker start whisper-service"` |
| Stop | `ssh hch@10.145.119.19 "docker stop whisper-service"` |
| Logs | `ssh hch@10.145.119.19 "docker logs -f whisper-service"` |
| GPU usage | `ssh hch@10.145.119.19 "nvidia-smi"` |

### Known State

- Container: `whisper-service`
- Image: `onerahmet/openai-whisper-asr-webservice:latest-gpu`
- Restart policy: `unless-stopped`
- **正常狀態：停止**，這樣才不會佔用 GPU
- 跑轉錄時會跟 `ollama`、`llama-server`、`unsloth` 搶 GPU/VRAM

### Common Mistakes

- 不要假設「已停止的容器」在用 GPU——`Exited` 就是沒在用。
- 一次性轉錄完後不要讓 Whisper 繼續開著，除非使用者要求常駐 ASR。
- `docker stop whisper-service` 之後，`restart=unless-stopped` **不會**立刻重啟它。
- **優先用 `gpu-switch`，而非直接 `docker start/stop`**（當 llama.cpp/ollama 可能也在用時）。
  這台主機有 `gpu-switch` script（見 `omc-learned/llama-cpp-server-setup.md` Step 7），會協調停掉另一個
  GPU consumer 並等 VRAM 真的釋放（有真實的 race condition——VRAM 不會在服務回報停止的瞬間就釋放）。
  直接 `docker start whisper-service`（llama-server 還握著 VRAM 時）可能 CUDA-OOM。
  改用 `gpu-switch whisper` / `gpu-switch stop`（遠端）或 `D:\BIN\gpu-switch-remote.bat`（本機 Windows
  wrapper），除非確定要繞過協調機制。

### 健康檢查 / smoke test script

`scripts/test-audiocpp.sh`（用途：測相鄰的 audiocpp-server TTS/ASR 服務，非 whisper-service 本身，
但用法可參考——健康檢查、TTS 生成、ASR 回饋測試三段式）：
```bash
bash scripts/test-audiocpp.sh [remote_host]
```

---

## Part B — CLI 轉錄呼叫

直接呼叫遠端 whisper ASR REST API（`POST /asr`）做語音轉文字/字幕，不用自己重新組 curl multipart 指令。

呼叫本腳本前 whisper 必須是目前「切到」的那個服務，否則會連不上——腳本內建 `-AutoSwitch` 可以自動處理
這件事（見下方）。**不要重新手動組 curl 指令，直接跑這裡的腳本。**

### 用法

```powershell
# 基本轉錄（純文字輸出）
pwsh -NoProfile -File C:\Users\HCH\.claude\skills\whisper-service\scripts\scripts/whisper-transcribe.ps1 -AudioPath "D:\meeting.mp3"

# 指定語言 + 輸出 srt 字幕 + 存檔
C:\Users\HCH\.claude\skills\whisper-service\scripts\scripts/whisper-transcribe.ps1 -AudioPath "D:\meeting.mp3" -Language zh -OutputFormat srt -OutFile "D:\meeting.srt"

# whisper 目前沒開也沒關係，自動切換 GPU 過去再轉錄
C:\Users\HCH\.claude\skills\whisper-service\scripts\scripts/whisper-transcribe.ps1 "D:\meeting.mp3" -AutoSwitch

# 也有 .bat 版本，參數原樣往下傳
C:\Users\HCH\.claude\skills\whisper-service\scripts\scripts/whisper-transcribe.bat -AudioPath "D:\meeting.mp3"
```

參數：
| 參數 | 說明 |
|------|------|
| `-AudioPath`（位置 0，必填） | 音檔路徑（mp3/wav/m4a 等） |
| `-Language`（選填） | ISO-639-1 語言碼（如 `zh`、`en`、`ja`），不填則自動偵測 |
| `-Task`（選填） | `transcribe`（預設）或 `translate`（翻成英文） |
| `-OutputFormat`（選填） | `txt`（預設）/ `vtt` / `srt` / `tsv` / `json` |
| `-OutFile`（選填） | 額外存一份到檔案 |
| `-ServerUrl`（選填） | 預設 `http://10.145.119.19:9000` |
| `-AutoSwitch`（開關） | whisper 連不上時自動 ssh 切換遠端 GPU（會停掉當時在跑的 llama-server/ollama）再重試。預設關閉，因為會中斷別人正在用的 llama.cpp/ollama session，屬於有感知風險的操作 |

### 邊聽邊看逐字稿的網頁播放器（player.html）

如果使用者要「一邊放音一邊顯示字幕」（不是只要純文字稿），流程是：

1. 音檔要用 `-OutputFormat vtt` 轉錄（不是 `txt`），才會有逐句時間戳：
   ```powershell
   scripts/whisper-transcribe.ps1 -AudioPath "D:\BIN\Whisper\xxx.mp3" -Language zh -OutputFormat vtt -OutFile "D:\BIN\Whisper\xxx.vtt"
   ```
   （若同一支音檔已經有 `-OutputFormat txt` 存過 `.md` 逐字稿，`.vtt` 另外存，兩者並存不衝突）
2. 跑 `scripts/build-data.ps1` 把該目錄下所有 mp3+vtt 配對掃出來，解析 VTT 時間戳，
   產生 `recordings-data.js`（內嵌 JS 陣列，不用額外 fetch，file:// 開啟也能動）：
   ```powershell
   pwsh -NoProfile -File D:\BIN\Whisper\scripts/build-data.ps1
   ```
3. 直接用瀏覽器開 `D:\BIN\Whisper\player.html`：左上角選錄音、播放器放音、
   下方逐句字幕會依播放進度自動高亮+捲動，點任一句可跳轉播放位置，
   旁邊也有連結開對應的完整 `.md` 逐字稿。

新增錄音時只要重複步驟 1-2（轉 vtt、重跑 scripts/build-data.ps1），`player.html` 本身**不用改**，
它是純資料驅動的（讀 `recordings-data.js`）。

**不要每次都重新手動組 HTML/JS 逐句同步邏輯** —— `player.html` 的高亮演算法已經寫好
並用 Playwright 驗證過（含 VTT 時間戳跨小時進位 `MM:SS.mmm` → `HH:MM:SS.mmm` 的格式切換
都有處理），有新需求就是改這個檔案，不要重新從零推導。

> 測試這個網頁時，Playwright MCP 的 `browser_navigate` 會擋 `file://` 協定
> （只在自動化測試環境裡擋，一般瀏覽器打開沒事），要驗證時用
> `python -m http.server` 之類的臨時 server 起在該目錄再 navigate 過去測。
> 另外 headless 瀏覽器沒有音訊裝置，`audio.currentTime` 設值會被無聲吃掉（seek 不會生效），
> 要測「逐句高亮邏輯」本身要直接呼叫 JS 邏輯或 dispatch `timeupdate` 事件驗證資料對應關係，
> 不能依賴 headless 環境裡真的把音檔播放/拖曳進度來驗證。

### 已知踩坑

> **坑 1：GPU 共用導致連不上，不是腳本壞了。** whisper 服務跟 llama.cpp／ollama
> 搶同一台機器的 VRAM，若目前 GPU 切在 llama/ollama，直接呼叫本腳本會在
> `Test-WhisperReachable` 這關失敗並印出提示訊息，要求你先手動
> `gpu-switch-remote.ps1 whisper` 或加 `-AutoSwitch`。不要以為是網路或服務掛了。

> **坑 2：API 端點與欄位名稱是問 `/openapi.json` 現場確認出來的，不是憑印象猜的。**
> `POST /asr`，query string `encode`/`task`/`language`/`output`，
> multipart form 檔案欄位固定叫 `audio_file`（不是 `file`）。改版時如果懷疑
> 欄位變了，先 `curl http://10.145.119.19:9000/openapi.json` 撈 schema 出來看，
> 不要用其他版本 whisper-asr-webservice 的文件瞎猜。

> **坑 3：測試用假音檔（無語音內容）也能驗證整條管線。** 沒有真人語音樣本時，
> 用 `ffmpeg -f lavfi -i "sine=frequency=1000:duration=2" -ar 16000 test.wav`
> 產生一段測試音，whisper 會回傳類似 `"Beep"` 這種幻覺文字 —— 這樣就能確認
> 上傳/API 呼叫/回傳整條路徑通了，不用真的準備語音檔才能測。

## 相關

- whisper 服務本身的 docker-compose 部署、GPU 切換機制見 `omc-learned` 的
  `llama-cpp-server-setup.md`（Step 7：`gpu-switch` / `gpu-switch-remote`）。
- 使用者在 `D:\BIN\Whisper\` 有一份工作副本（`scripts/whisper-transcribe.ps1` +
  `scripts/whisper-transcribe.bat` + `player.html` + `scripts/build-data.ps1`），跟這裡
  `scripts/` 底下的正本邏輯一致，修 bug 或更新腳本/網頁時記得兩邊都要同步
  （比照 `D:\BIN\Vision\` 跟 `llama-vision-test` skill 的作法）。
  `recordings-data.js` 是 `scripts/build-data.ps1` 在 `D:\BIN\Whisper\` 現場產生的資料檔，
  不用同步進 skill（skill 裡放的是產生資料的腳本本身）。

---

## Conformance Addendum

## When to Use
遠端 whisper ASR 服務（onerahmet/openai-whisper-asr-webservice，跑在 10.145.119.19:9000）的完整管理—— Docker 容器啟停/狀態/log（正常應保持關閉，只在需要轉錄時開）、GPU 共用協調（跟 llama.cpp/ollama 搶 VRAM，優先用 gpu-switch 而非直接 docker start/stop）、CLI 轉錄呼叫（音檔轉文字/字幕）、逐句字幕播放器 player.html。當使用者要求「轉錄音檔」、「whisper 轉文字」、「音檔轉字幕」、「檢查/啟動/停止 whisper 服務」、 「whisper 掛了」時使用。合併自原本的 `whisper-cli-transcribe`（呼叫）與 `whisper-docker-service`（容器管理） 兩個 skill。

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
