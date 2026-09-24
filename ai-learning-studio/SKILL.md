---
name: ai-learning-studio
description: 部署、維護與修改 AI Learning Studio（https://10.145.119.19:8097/）——這是一套在遠端 GPU 主機 10.145.119.19 上以 Docker Compose 執行的互動式 AI 語音課程系統，使用本地 LLM 生成課程簡報、Qwen3-TTS 提供語音講解、FunASR 處理語音提問。當使用者提到「AI 學習工作室」「學習系統」「語音課程」「簡報生成」「learnroom」「AI Learning Studio」「8097」「修改學習介面」「課程投影片」「Qwen TTS 設定」「更換 LLM」「調整 GPU 配置」「部署教學系統」或任何與這套學習系統的部署、修改、除錯、功能新增相關的操作時使用。也涵蓋前端 UI、語音導師球體、localStorage、彈窗定位、課程導航等前端調整。
compatibility: 需要 SSH 金鑰連線 hch@10.145.119.19、遠端 Docker Compose、NVIDIA Container Toolkit；LLM 目前使用 llama.cpp Spark-X2.5-4B（port 8080，GPU 0）；TTS 使用 Qwen3-TTS-1.7B-CustomVoice（port 7862，GPU 1）；ASR 使用 FunASR Paraformer（port 10095，GPU 0）。
---

# AI Learning Studio

## When to Use

使用者提到以下任一關鍵字時必用：

- AI Learning Studio、學習工作室、learnroom
- 語音課程、語音簡報、投影片生成
- 8097、ai-learning-studio、Docker Compose
- 課程投影片、語音導師、導師球體
- Qwen TTS、Qwen3-TTS、Spark LLM、Spark-X2.5
- 修改學習介面、前端 UI 調整
- 部署學習系統、GPU 配置、TTS 換模型
- 遠端 `hch@10.145.119.19` 的 ai-learning-studio 容器

不適用於：

- 原本的 `funasr-voice-chat`（port 8096）
- 小愛音箱／xiaoai 系統
- whisper 轉錄服務
- vLLM qwythos 部署
- ECP／aipower 系統

## Inputs and Outputs

### 輸入

- 遠端主機：`hch@10.145.119.19`
- 遠端專案：`/home/hch/ai-learning-studio`
- 前端暫存本機目錄：`C:\Users\HCH\AppData\Local\Temp\ai-learning-studio-staging`
- GitHub Repository：`https://github.com/HuangChienHsiung590902/ai-learning-studio`（Private）
- 使用者對功能、介面、語音、部署等的修改需求。

### 輸出

回報時至少包含：

1. 實際變更的檔案（Dockerfile、docker-compose.yml、app/main.py、app/static/index.html）。
2. 容器重建與健康檢查結果（`docker compose up -d --build --force-recreate`）。
3. 實際驗證：`https://127.0.0.1:8097/health`、頁面連線、LLM 連線、TTS 連線。
4. GPU 記憶體使用變化。
5. 使用者可開啟的 URL（`https://10.145.119.19:8097/`）與快取清除提醒。
6. 若失敗，清楚指出卡在哪一層。

## Architecture

```text
AI Learning Studio (port 8097, HTTPS)
  │
  ├── LLM：Spark-X2.5-4B（llama.cpp，port 8080，GPU 0）
  ├── TTS：Qwen3-TTS-12Hz-1.7B-CustomVoice（port 7862，GPU 1）
  ├── ASR：FunASR Paraformer（port 10095，GPU 0）
  └── 前端：單一 HTML 頁面（FastAPI + 靜態檔案）
```

### 服務列表

| 容器 | Port | GPU | 用途 |
|---|---|---|---|
| `ai-learning-studio` | 8097 (HTTPS) | 無 | 主應用 |
| `llama-cpp-spark-x25-4b` | 8080 | GPU 0 | LLM 課程生成 |
| `qwen3-tts` | 7862 | GPU 1 | TTS 語音合成 |
| `funasr-paraformer` | 10095 | GPU 0 | ASR 語音辨識 |

## Canonical Files

### 本機端（UI 開發）

```text
C:\Users\HCH\AppData\Local\Temp\ai-learning-studio-staging\
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── app/
│   ├── main.py
│   └── static/index.html
└── .gitignore
```

### 遠端端（正式部署）

```text
/home/hch/ai-learning-studio/
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── app/
│   ├── main.py
│   └── static/index.html
└── .gitignore
```

### TTS 服務

```text
/home/hch/qwen3-tts/
├── Dockerfile
├── docker-compose.yml
└── server.py
```

TTS staging：`C:\Users\HCH\AppData\Local\Temp\qwen3-tts-staging`

## Procedure

### 1. 修改前端 UI（99% 的任務都在這裡）

所有 UI 修改都在本機 staging 目錄進行：

```text
C:\Users\HCH\AppData\Local\Temp\ai-learning-studio-staging\app\static\index.html
```

修改後使用以下流程部署：

```bash
tar -C /c/Users/HCH/AppData/Local/Temp/ai-learning-studio-staging -czf /c/Users/HCH/AppData/Local/Temp/ai-learning-studio-staging.tgz . && \
cat /c/Users/HCH/AppData/Local/Temp/ai-learning-studio-staging.tgz | \
ssh hch@10.145.119.19 'rm -rf /home/hch/ai-learning-studio && mkdir -p /home/hch/ai-learning-studio && tar -xzf - -C /home/hch/ai-learning-studio && rm -f /home/hch/ai-learning-studio/app/static/check.js && cd /home/hch/ai-learning-studio && docker compose config >/dev/null && docker compose up -d --build --force-recreate >/tmp/ai-learning-build.log && sleep 3 && curl -ksS --max-time 15 https://127.0.0.1:8097/health'
```

### 2. 修改後端 API

修改 `app/main.py` 後用同一條部署指令。修改前先用 `python -m py_compile` 檢查語法。

後端 API 端點：

- `GET /` — 靜態頁面
- `GET /health` — 健康檢查
- `POST /api/course` — 生成課程
- `POST /api/tts` — 文字轉語音
- `POST /api/answer` — 文字問答
- `POST /api/voice-answer` — 語音問答
- `WS /ws/voice` — 語音辨識 WebSocket

### 3. 修改 TTS 服務

TTS 設定在 `C:\Users\HCH\AppData\Local\Temp\qwen3-tts-staging`。

部署：

```bash
tar -C /c/Users/HCH/AppData/Local/Temp/qwen3-tts-staging -czf /c/Users/HCH/AppData/Local/Temp/qwen3-tts-staging.tgz . && \
cat /c/Users/HCH/AppData/Local/Temp/qwen3-tts-staging.tgz | \
ssh hch@10.145.119.19 'rm -rf /home/hch/qwen3-tts && mkdir -p /home/hch/qwen3-tts && tar -xzf - -C /home/hch/qwen3-tts && cd /home/hch/qwen3-tts && docker compose up -d --force-recreate'
```

### 4. 檢查服務狀態

```bash
ssh hch@10.145.119.19 'docker ps --format "{{.Names}}\t{{.Status}}\t{{.Ports}}" | sort'
```

健康檢查：

```bash
ssh hch@10.145.119.19 'curl -ksS --max-time 15 https://127.0.0.1:8097/health'
```

### 5. 切換 LLM

目前使用 Spark-X2.5-4B。若要換成 Qwen3.8-27B：

```bash
# 停止 Spark
ssh hch@10.145.119.19 'cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.spark-x25-4b.yml down'
# 啟動 Qwen
ssh hch@10.145.119.19 'cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwen38-27b-abliterated.yml up -d'
```

Qwen3.8-27B 使用雙卡，切換時可能需要停止 FunASR 或 TTS。

### 6. GitHub 上傳

本機 staging 目錄已初始化 git：

```bash
cd /c/Users/HCH/AppData/Local/Temp/ai-learning-studio-staging
git add -A && git commit -m "描述修改內容" && git push
```

### 7. 使用 gh 工具

本機已登入 `gh`（帳號：HuangChienHsiung590902），可以直接建立／推送 repo。

## UI 功能清單

- 課程生成（Spark LLM）
- 語音導師球體（可拖曳，localStorage 記憶位置）
- 導師彈窗（文字輸入、語音提問、閱讀投影片）
- 彈窗自動依照球體位置定位
- 側欄滑鼠 hover 展示
- 鍵盤快捷鍵：`←` `→` 換頁、`P` 閱讀、`Esc` 停止
- TTS 一律走 Qwen3-TTS（不 fallback 到瀏覽器語音）
- 投影片內容過長時顯示向下箭頭（無捲軸）
- 臺灣華語 TTS 腔調設定
- 開始說話／停止合併為單一按鈕

## Rules and Limitations

- 不要在回答中暴露 `/home/hch/funasr-voice-chat/certs/server.key` 或其他金鑰內容。
- Qwen3-TTS 與 llama.cpp 都使用 GPU，切換模型前先檢查 VRAM。
- 不要在 `main.py` 中 fallback 到 `edge_tts` 或 `speechSynthesis`（使用者明確要求只使用 Qwen3-TTS）。
- 不要覆蓋 `funasr-voice-chat`（port 8096）。
- 不要修改 `spark-x25-config` 下的 MCP 設定（這是 Spark 自己的工具鏈）。
- 部署後提醒使用者使用 `Ctrl + F5` 強制清除快取。
- `ai-learning-studio` 本身不使用 GPU，可設定 `restart: unless-stopped`。
- `qwen3-tts` 使用 GPU，`restart: "no"`。
- 修改 `index.html` 後務必用 `node --check` 驗證 JS 語法。
- `render()` 函數中不要對已不存在的 DOM 元素（如 `$('prev')`, `$('next')`, `$('pageHint')`）設定屬性，否則會中斷課程生成。
- 修改 `main.py` 後務必用 `python -m py_compile` 驗證語法。

## Pitfalls

- 不要把「AI Learning Studio」和「funasr-voice-chat」（8096）混淆。
- 不要刪除簡報的 `#lesson` 元素或改變其 `hidden` 屬性的控制邏輯。
- 修改 `render()` 前先確認該函數中引用的所有 DOM id 都存在。
- 不要把 `check.js` 或 `__pycache__` 提交到 GitHub。
- TTS 服務的 container 內只能用 `cuda:0`（因為只有 GPU 1 被映射進去）。
- `qwen3-tts` 的 image 名稱仍是 `qwen3-tts:0.6b-customvoice`，即使現在實際使用 1.7B 模型。

## Verification

1. `curl -ksS https://10.145.119.19:8097/health` 應回 `{"ok":true,"llm":true,"tts":true}`。
2. 頁面 `https://10.145.119.19:8097/` 正常載入。
3. 生成課程後可換頁，鍵盤 `←` `→` 正常。
4. 語音導師球體可拖曳，位置在刷新後保持不變。
5. TTS 播放時不應出現 Microsoft 語音。
6. `docker ps` 四個關鍵容器都在執行中。
7. GPU 記憶體：GPU 0 約 13.5 GiB（Spark + FunASR），GPU 1 約 5–6 GiB（Qwen3-TTS 1.7B）。
8. GitHub repo 與遠端檔案一致。