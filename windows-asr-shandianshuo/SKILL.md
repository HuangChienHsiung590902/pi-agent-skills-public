---
name: windows-asr-shandianshuo
description: Windows 本機 llama.cpp Qwen3-ASR + 閃電說自定義語音辨識服務，含轉碼 proxy、開機自動啟動
triggers:
  - 閃電說 ASR
  - Qwen3-ASR
  - 語音辨識
  - llama cpp asr
  - shandianshuo asr
  - 開機啟動 ASR
  - asr proxy
---

# Windows 本機 llama.cpp Qwen3-ASR + 閃電說

讓閃電說使用本機 llama.cpp 跑的 Qwen3-ASR 語音辨識模型。

## 架構

```
閃電說 ──→ scripts/asr_proxy.py (8091) ──→ llama-server (8090)
                │                        │
           webm→WAV 轉碼            Qwen3-ASR-1.7B
           剝除 data: 前綴
```

閃電說發出 POST `/v1/chat/completions`（OpenAI multimodal 格式，`input_audio` content part），proxy 在 8091 攔截、轉碼、剝 Data URL 前綴後轉發給 8090 的 llama-server。

## 模型

| 檔案 | 大小 | 來源 |
|------|------|------|
| `Qwen3-ASR-1.7B-Q8_0.gguf` | ~2.16 GB | `https://huggingface.co/ggml-org/Qwen3-ASR-1.7B-GGUF` |
| `mmproj-Qwen3-ASR-1.7B-Q8_0.gguf` | ~356 MB | 同上 |

放在 `D:\llama.cpp\models\` 目錄。

## 服務啟動

### 手動（PowerShell）

```powershell
# 1. llama-server
Start-Process -FilePath "D:\llama.cpp\llama-server.exe" `
    -ArgumentList "-m models/Qwen3-ASR-1.7B-Q8_0.gguf --mmproj models/mmproj-Qwen3-ASR-1.7B-Q8_0.gguf --port 8090" `
    -WorkingDirectory "D:\llama.cpp" `
    -RedirectStandardOutput "D:\llama.cpp\server_asr_stdout.log" `
    -RedirectStandardError "D:\llama.cpp\server_asr_stderr.log" `
    -WindowStyle Hidden

# 2. asr_proxy
Start-Process -FilePath "C:\Users\HCH\AppData\Local\Programs\Python\Python312\python.exe" `
    -ArgumentList "D:\llama.cpp\scripts/asr_proxy.py" `
    -RedirectStandardOutput "D:\llama.cpp\proxy_stdout.log" `
    -RedirectStandardError "D:\llama.cpp\proxy_stderr.log" `
    -WindowStyle Hidden
```

### 開機自動啟動

已建立 **工作排程任務**「ASR-Services-AutoStart」，登入時自動觸發 `D:\llama.cpp\scripts/start_asr_services.ps1`。

啟動腳本備份：`scripts/start_asr_services.ps1`（本 skill 目錄）。

重建排程（若遺失）：
```powershell
$action = New-ScheduledTaskAction -Execute "powershell.exe" `
    -Argument '-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File "D:\llama.cpp\scripts/start_asr_services.ps1"'
$trigger = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERNAME"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
Register-ScheduledTask -TaskName "ASR-Services-AutoStart" -Action $action -Trigger $trigger -Settings $settings -RunLevel Limited -Force
```

## 閃電說設定

| 欄位 | 值 |
|------|-----|
| 地址 | `http://127.0.0.1:8091/v1`（**proxy**，不是 8090） |
| API Key | 任意字串（不驗證） |
| 模型 | `models/Qwen3-ASR-1.7B-Q8_0.gguf` |

> **注意**：開機後模型載入約需 40~60 秒（CPU 模式），這段時間 ASR 會回 503 "Loading model"，等一下再試即可。

## scripts/asr_proxy.py 關鍵邏輯

原始腳本：`scripts/asr_proxy.py`，部署到 `D:\llama.cpp\scripts/asr_proxy.py`。

### 做兩件事

1. **音訊格式轉碼**：用 ffmpeg pipe stdin→stdout 把 webm/opus 等格式轉成 16kHz mono WAV
2. **剝除 Data URL 前綴**：閃電說送的 `input_audio.data` 是 `data:audio/wav;base64,...` 格式，必須剝掉 `data:audio/wav;base64,` 前綴才能 base64 decode

## 已知踩坑

| 坑 | 原因 | 解法 |
|----|------|------|
| HTTP 400（第一次） | llama.cpp 的 `is_audio_file()` 只認 WAV/MP3/FLAC magic bytes，webm 落入 video-probe fallback 而失敗 | 用 proxy 轉碼 |
| HTTP 400（第二次） | 閃電說送的 `input_audio.data` 是 `data:audio/wav;base64,...` Data URL，不是純 base64 | 剝除前綴再 decode |
| ffprobe failed on buffer | 誤導訊息：非 ffprobe/PATH 問題，是音訊格式不認得 | 同上，轉碼即可 |
| Python 指令找不到 | `python` / `python3` 在 PowerShell 可能指向 Microsoft Store stub | 用完整路徑 `C:\Users\HCH\AppData\Local\Programs\Python\Python312\python.exe` |
| 模型載入慢 | CPU 模式，無 CUDA（`ggml_cuda_init: failed to initialize CUDA`） | 開機後等 ~1 分鐘 |

## 驗證

```powershell
# 確認兩個 port 都在監聽
netstat -ano | findstr "8090" | findstr "LISTENING"
netstat -ano | findstr "8091" | findstr "LISTENING"

# 用捕獲的真實請求重放
curl -s -X POST http://127.0.0.1:8091/v1/chat/completions `
    -H "Content-Type: application/json" `
    -d @D:\llama.cpp\last_request.json
```

成功應回傳 200，`content` 欄位含 `<asr_text>...</asr_text>`。

---

## Conformance Addendum

## When to Use
Windows 本機 llama.cpp Qwen3-ASR + 閃電說自定義語音辨識服務，含轉碼 proxy、開機自動啟動

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
