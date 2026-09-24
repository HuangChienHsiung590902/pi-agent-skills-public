---
name: funasr-voice-chat-https
description: 管理遠端 GPU 主機 10.145.119.19 上的 FunASR Paraformer 中文串流 ASR + llama.cpp Qwen3.8-27B + Edge TTS 瀏覽器語音對話測試系統。當使用者要求部署、啟動、停止、檢查、除錯或修改這個語音對話頁面、FunASR WebSocket、瀏覽器麥克風、HTTPS、語音對話卡在 LLM/TTS、或提到 https://10.145.119.19:8096/ 時使用。這個 skill 固定使用主機自簽 HTTPS，不使用 Cloudflare Tunnel/Quick Tunnel。
compatibility: 需要 SSH 金鑰連線 hch@10.145.119.19、遠端 Docker Compose、NVIDIA Container Toolkit；瀏覽器測試需接受 10.145.119.19 的自簽憑證警告。
---

# FunASR Voice Chat HTTPS

## When to Use

使用者要處理下列任務時載入本 skill：

- 部署或維護瀏覽器語音對話測試頁。
- 操作 FunASR Paraformer-zh-streaming、WebSocket ASR 或 `funasr-paraformer` 容器。
- 問「語音網頁在哪裡」「麥克風不能用」「LLM 思考中卡住」「TTS 沒聲音」。
- 要把這套服務改成 HTTPS、移除 Cloudflare Tunnel，或使用 `https://10.145.119.19:8096/`。
- 要確認這個語音助理使用哪個 ASR、LLM、TTS 或 GPU。

不要把本 skill 與以下服務混用：

- `/home/hch/xiaoai-voice-test`：另一套 CosyVoice/SenseVoice 測試環境，目前不作為本系統來源。
- 正式 `xiaoai` 音箱堆疊：本 skill 不操作小愛音箱、open-xiaoai-bridge 或 agent-proxy。
- `whisper-service`：那是錄音檔/字幕轉錄服務，不是本語音對話頁的 ASR。
- `audiocpp-realtime-web`：另一套 audiocpp ASR/TTS 網頁。

## Inputs and Outputs

### 輸入

- 遠端主機：`hch@10.145.119.19`
- 遠端專案：`/home/hch/funasr-voice-chat`
- ASR 專案：`/home/hch/funasr-paraformer`
- 使用者要做的操作、瀏覽器錯誤或容器日誌。

### 輸出

回報時至少包含：

1. 目前要做的事情與影響範圍；若要修改或重建，先明確告知，不要連續盲目重試。
2. 實際變更的檔案、容器與 port。
3. 實際驗證結果：HTTP/HTTPS health、容器狀態、WebSocket pipeline 或錯誤日誌。
4. 使用者可開啟的 URL，以及自簽憑證需要如何處理。
5. 若失敗，清楚指出卡在哪一層，不要把 health check 當成完整語音流程成功。

## Current Architecture

```text
Browser HTTPS
  https://10.145.119.19:8096/
        |
        | WSS /ws/voice
        v
funasr-voice-chat :8096
  |-- browser PCM16 audio -> FunASR WebSocket
  |-- transcript -> llama.cpp OpenAI-compatible API
  |-- answer -> Edge TTS
  `-- MP3 -> browser audio playback
        |
        +--> funasr-paraformer :10095
        |      FunASR Paraformer streaming + VAD/2pass models, GPU 0
        |
        `--> llama-cpp-qwen38-27b-abliterated :8080
               Qwen3.8-27B Abliterated Q4_K_M, GPUs 0/1
```

### 服務與模型

| 元件 | 實際值 |
|---|---|
| Browser URL | `https://10.145.119.19:8096/` |
| Voice app container | `funasr-voice-chat` |
| Voice app project | `/home/hch/funasr-voice-chat` |
| Voice app port | `8096` |
| ASR container | `funasr-paraformer` |
| ASR WebSocket | `ws://10.145.119.19:10095`，只供後端容器連接 |
| ASR model | FunASR Paraformer 中文 streaming，搭配 VAD/punctuation 模型 |
| LLM container | `llama-cpp-qwen38-27b-abliterated` |
| LLM API | `http://10.145.119.19:8080/v1` |
| LLM model | `Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf` |
| LLM image | `ghcr.io/ggml-org/llama.cpp:server-cuda` |
| TTS | Edge TTS |
| TTS voice | `zh-TW-HsiaoChenNeural` |
| HTTPS | Uvicorn TLS + host self-signed certificate |
| Public tunnel | **禁止使用 Cloudflare Tunnel/Quick Tunnel** |

## Canonical Files

```text
/home/hch/funasr-voice-chat/
├── Dockerfile
├── docker-compose.yml
├── app/
│   ├── main.py
│   └── static/index.html
└── certs/
    ├── server.crt
    └── server.key
```

ASR 專案：

```text
/home/hch/funasr-paraformer/
├── Dockerfile
├── docker-compose.yml
├── funasr_wss_server.py
└── models/
```

`certs/server.key` 是私密金鑰，不要貼到回答、commit、公開儲存庫或 log。若需要備份，只備份在遠端受保護的目錄，並維持 `chmod 600`。

## Procedure

### 1. 先檢查現況，不要直接重建

```bash
ssh hch@10.145.119.19 '
  docker ps -a --filter name=funasr-voice-chat \
                  --filter name=funasr-paraformer \
                  --filter name=llama-cpp-qwen38-27b-abliterated
  ss -ltnp | grep -E ":(8096|10095|8080)\\b" || true
  pgrep -af "cloudflared.*8096" || true
'
```

健康檢查：

```bash
ssh hch@10.145.119.19 \
  'curl -ksS --max-time 10 https://127.0.0.1:8096/health'
```

正常應包含：

```json
{"ok":true,"llm":true,"tts":true}
```

檢查 LLM：

```bash
ssh hch@10.145.119.19 \
  'curl -sS --max-time 10 http://127.0.0.1:8080/health; curl -sS http://127.0.0.1:8080/v1/models'
```

### 2. 啟動或重建語音 Web App

這個容器不是 GPU 推理容器，可保留 `restart: unless-stopped`。修改 Compose 後必須重建容器才能載入 command、volume 或環境變數：

```bash
ssh hch@10.145.119.19 \
  'cd /home/hch/funasr-voice-chat && \
   docker compose config >/dev/null && \
   docker compose up -d --force-recreate'
```

如果只需正常啟動：

```bash
ssh hch@10.145.119.19 \
  'cd /home/hch/funasr-voice-chat && docker compose up -d'
```

### 3. HTTPS 憑證

目前使用自簽憑證，必須包含 IP SAN：

```bash
openssl x509 -in /home/hch/funasr-voice-chat/certs/server.crt \
  -noout -subject -dates -ext subjectAltName
```

應看到：

```text
subject=CN = 10.145.119.19
IP Address:10.145.119.19
```

若要重新產生，先確認使用者同意，因為瀏覽器會需要重新信任新憑證：

```bash
cd /home/hch/funasr-voice-chat
mkdir -p certs
openssl req -x509 -newkey rsa:2048 -nodes -sha256 \
  -days 825 \
  -keyout certs/server.key \
  -out certs/server.crt \
  -subj '/CN=10.145.119.19' \
  -addext 'subjectAltName=IP:10.145.119.19'
chmod 600 certs/server.key
chmod 644 certs/server.crt
```

Compose 必須將憑證以唯讀方式掛載，並讓 Uvicorn 使用：

```yaml
volumes:
  - ./certs:/certs:ro
command:
  - uvicorn
  - app.main:app
  - --host
  - 0.0.0.0
  - --port
  - "8096"
  - --ssl-keyfile
  - /certs/server.key
  - --ssl-certfile
  - /certs/server.crt
```

### 4. Cloudflare Tunnel 禁止使用

本系統不要啟動：

```bash
cloudflared tunnel --url http://127.0.0.1:8096
```

如果目前有指向這個服務的 Quick Tunnel，先確認 command 確實是該 port，再停止，不要使用寬泛的 `pkill cloudflared` 影響其他服務：

```bash
pgrep -af 'cloudflared.*8096'
```

停止後確認：

```bash
pgrep -af 'cloudflared.*8096' || true
```

使用者應使用固定的主機 HTTPS URL，而不是 `trycloudflare.com` URL：

```text
https://10.145.119.19:8096/
```

### 5. 瀏覽器測試

第一次開啟自簽 HTTPS 會出現憑證警告。告知使用者選擇瀏覽器的「進階 → 繼續前往 10.145.119.19」。

瀏覽器在 HTTPS 頁面會自動將前端 WebSocket 變成：

```text
wss://10.145.119.19:8096/ws/voice
```

前端麥克風流程：

```text
getUserMedia
→ AudioContext
→ 取 mono audio
→ 重採樣 16 kHz
→ Int16 PCM
→ WSS binary frames
```

不要把 `MediaRecorder` 產生的 WebM/Opus blob 直接送給 FunASR；本服務的 FunASR WebSocket 需要 16 kHz mono PCM16 binary。

## Troubleshooting

### A. 瀏覽器顯示 `invalid Connection header: keep-alive`

原因是把 WebSocket port 當普通 HTTP 頁面或網址列直接開啟。現在應開：

```text
https://10.145.119.19:8096/
```

不要開 ASR WebSocket port `10095`。瀏覽器頁面由 `8096` 提供，頁面內部才建立 WSS。

### B. 頁面卡在「LLM 思考中」

先看：

```bash
ssh hch@10.145.119.19 \
  'docker logs --since 10m funasr-voice-chat 2>&1 | tail -160'
```

應看到：

```text
turn transcript='...'
llm_start
llm_done answer='...'
```

如果只有 `llm_start`，直接測 app 容器內的 LLM 呼叫：

```bash
ssh hch@10.145.119.19 \
  'docker exec funasr-voice-chat python -c "import asyncio; from app.main import ask_llm; print(asyncio.run(asyncio.wait_for(ask_llm(\"請簡短回答：測試成功\"), timeout=120)))"'
```

已知根因：早期版本在收到 FunASR `is_end` 後立刻結束前端 WebSocket，沒有等待 LLM/TTS。正確流程必須等待：

```text
ASR is_end
→ LLM 完成
→ TTS 完成
→ audio event 送出
→ turn_end
→ 關閉 WebSocket
```

程式應用 `turn_done` event 或同等同步機制，不能在 ASR `is_end` 後立即 `break`。

### C. ASR 文字重複

使用 `2pass` 時會同時有：

- `2pass-online` partial transcript
- `2pass-offline` final transcript

顯示給使用者可以傳送 final 文字，但組合送給 LLM 時不能把 online partial 和 final 再重複串接。應在後端依 `mode` 分開處理。

### D. LLM health 正常但完整流程失敗

`/health` 只表示 HTTP endpoint 可連線，不代表完整 pipeline 成功。必須驗證：

```text
ready
→ asr
→ transcript
→ assistant
→ audio
→ turn_end
```

可用 WebSocket client 做 smoke test；完整成功標準是收到以上事件，不能只看 `docker ps` 或 `/health`。

### E. TTS 沒有音訊

先獨立測試：

```bash
ssh hch@10.145.119.19 \
  'docker exec funasr-voice-chat python -c "import asyncio; from app.main import make_tts; b=asyncio.run(make_tts(\"語音測試成功\")); print(len(b))"'
```

Edge TTS 依賴外部網路；如果 TTS timeout，應回報是外部 TTS 連線問題，不要誤判為 FunASR 或 LLM 問題。

### F. GPU/OOM

查 GPU：

```bash
ssh hch@10.145.119.19 \
  'nvidia-smi --query-gpu=index,name,memory.used,memory.free,utilization.gpu --format=csv,noheader'
```

`funasr-paraformer` 與 llama.cpp 都是 GPU 服務。除非使用者明確要求，不要替 GPU 容器設定 `restart: always` 或 `restart: unless-stopped`，也不要自行設定開機啟動。

### G. HTTPS 連不上

```bash
ssh hch@10.145.119.19 \
  'docker ps --filter name=funasr-voice-chat; \
   ss -ltnp | grep -E ":8096\\b"; \
   docker logs --tail 80 funasr-voice-chat 2>&1'
```

Uvicorn 正常 log 應為：

```text
Uvicorn running on https://0.0.0.0:8096
```

若是：

```text
Uvicorn running on http://0.0.0.0:8096
```

代表 Compose 的 TLS command 沒載入，需 `docker compose up -d --force-recreate`。

## Rules and Limitations

- 所有遠端操作先查現況；不要未說明就反覆重建、重啟或測試。
- 操作 GPU 主機前先列出將影響的 container、port、GPU 與檔案範圍。
- 不碰正式 xiaoai、CosyVoice 測試目錄或其他 compose project，除非使用者明確要求。
- 不啟動或重新建立 Cloudflare Tunnel；此系統固定走 `https://10.145.119.19:8096/`。
- 不把自簽私鑰 `server.key` 放入回答、Skill、git 或公開檔案。
- 不把 `https://10.145.119.19:8096/` 宣稱為公開網際網路網址；它通常只能由能連到該 GPU Server 的內網/ZeroTier 客戶端使用。
- 自簽憑證只提供加密與瀏覽器安全 context，不等於使用者驗證或存取控制。若需要防止同網路其他人使用，必須另外加認證或網路 ACL，不能把 HTTPS 當成登入防護。
- 若健康檢查正常但使用者報卡住，優先取得實際 `docker logs` 和 WebSocket event sequence，再修改程式。

## Pitfalls

- 不要用瀏覽器網址列開 `ws://10.145.119.19:10095`；那會得到一般 HTTP `keep-alive` 錯誤。
- 不要用 Cloudflare Quick Tunnel 取代主機 HTTPS；使用者已明確要求避免被外部盜用。
- 不要在 FunASR `is_end` 後立刻關閉前端 WebSocket；LLM/TTS 仍可能尚未完成。
- 不要只測 `/health` 就宣稱語音對話成功。
- 不要把 `2pass-online` 與 `2pass-offline` 文字盲目拼接，否則 LLM 會收到重複句子。
- 不要把 Edge TTS timeout 歸咎於 ASR；分層查證 ASR、LLM、TTS。
- 不要為 GPU 容器加自動重啟策略來「修復」偶發卡住；先檢查 VRAM 與 log。

## Verification

完成部署或修改後，依序驗證：

1. `docker compose config` 成功。
2. `docker ps` 顯示 `funasr-voice-chat` 為 `Up`。
3. `curl -ksS https://127.0.0.1:8096/health` 回傳 `ok=true`、`llm=true`、`tts=true`。
4. log 顯示 `https://0.0.0.0:8096`。
5. `curl -ksS https://10.145.119.19:8096/` 回傳 HTTP 200 且頁面存在。
6. `pgrep -af 'cloudflared.*8096'` 沒有本服務的 tunnel process。
7. 完整 WebSocket smoke test 收到：`ready`、`transcript`、`assistant`、`audio`、`turn_end`。
8. 回報實際驗證輸出與尚未驗證的限制。

## Conformance Addendum

### Inputs and Outputs
- Input: 遠端 GPU 主機、Docker Compose、瀏覽器錯誤、容器 log 與目前服務狀態。
- Output: 安全的現況檢查、最小必要變更、HTTPS URL、分層驗證結果與剩餘限制。

### Procedure
1. 先查現況與確認影響範圍。
2. 只修改 `/home/hch/funasr-voice-chat` 對應檔案，除非使用者明確要求其他服務。
3. 修改後重建必要容器，執行 HTTPS health 與完整 WebSocket pipeline 驗證。
4. 若失敗，保存 log 證據並停止重試，先回報卡點。

### Rules and Limitations
- 遠端主機與版本細節以現場檢查為準；Skill 中的狀態可能過時。
- 自簽 HTTPS 不提供身份驗證；不要把它描述成安全的公開服務。
- 不把 credentials、私鑰或未驗證的成功結果寫入 Skill。
