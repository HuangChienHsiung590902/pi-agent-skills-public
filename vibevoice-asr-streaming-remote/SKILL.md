---
name: vibevoice-asr-streaming-remote
description: 在遠端 GPU 主機 10.145.119.19（gigabyte，雙 RTX 4060 Ti 16GB）以 Docker + vLLM 部署 microsoft/VibeVoice-ASR-Streaming-7B 串流語音辨識（speaker-attributed streaming ASR + hotwords + 10 語言），含 HTTPS 麥克風 demo 頁（https://10.145.119.19:7860）、API（http://10.145.119.19:8000）、vibevoice-ctl.sh 管理腳本、GPU/VRAM 參數規劃與除錯。當使用者提到「VibeVoice」「VibeVoice-ASR」「streaming ASR 7B」「8000 轉錄」「7860 麥克風頁面」「vibevoice-ctl」「speaker diarization 串流」「hotwords 轉錄」或要啟動/停止/重測這套服務時使用。不要用於：本機 Qwen3-ASR 閃電說（windows-asr-shandianshuo）、whisper 轉錄（whisper-service）、FunASR 語音對話頁（funasr-voice-chat-https）、audiocpp（audiocpp-realtime-web）。
triggers:
  - VibeVoice
  - VibeVoice-ASR-Streaming
  - streaming ASR 7B
  - vibevoice-ctl
  - 8000 轉錄
  - 7860 麥克風
  - speaker attributed transcription
  - hotwords 轉錄
---

# VibeVoice-ASR-Streaming-7B（10.145.119.19）

Microsoft 的統一串流 ASR：邊收音邊輸出「誰（Speaker N）在什麼時候說了什麼」，支援 hotwords 與 10 種語言（en/zh/es/pt/de/ja/ko/fr/ru/it）。2026-09-24 在本環境完成部署與實測。

## 現況（已驗證，2026-09-24）

| 項目 | 值 |
|---|---|
| 主機 | `hch@10.145.119.19`（gigabyte，雙 RTX 4060 Ti 16GB，key-based SSH） |
| 程式碼 | `/home/hch/vibevoice-asr-streaming`（github microsoft/VibeVoice main @ `1541f59`） |
| 模型快取 | `/home/hch/vibevoice-model-cache`（HF cache，約 17.3 GB BF16，**已持久化，重啟不會重抓**） |
| 推理容器 | `vibevoice-asr-streaming`（image `vllm/vllm-openai:v0.14.1`，TP=2） |
| ASR API | `http://10.145.119.19:8000`（內部/後端用，plain HTTP） |
| Demo 頁 | **`https://10.145.119.19:7860`**（容器 `vibevoice-demo`，host network，自簽 TLS；麥克風需 secure context 所以必須 HTTPS） |
| 憑證 | `/home/hch/vibevoice-certs/server.{crt,key}`（自簽 10 年，SAN：IP 10.145.119.19 / 192.168.2.90 / 127.0.0.1、DNS localhost；key 權限 600，不要貼進回答或 log） |
| 管理腳本 | `/home/hch/vibevoice-asr-streaming/vibevoice-ctl.sh` |
| Logs | `/home/hch/vibevoice-asr-streaming/vibevoice-server.log`、`vibevoice-demo.log`（寫在掛載的 /app 內，host 可直接看） |
| GPU 佔用 | 兩卡各約 13.6 / 16 GB（**兩卡都被這個服務佔滿**，剩約 2.7 GB/卡） |

### 啟動參數（依 16GB 卡實測調過，不要照抄官方預設）

```
--skip-deps --tp 2 --port 8000 --max-model-len 16384
--max-audio-windows 512 --mm-processor-cache-gb 8 --gpu-memory-utilization 0.80
```

- 官方預設 `--gpu-memory-utilization 0.85` + `--mm-processor-cache-gb 16` 在 16GB 卡會 OOM；0.80 + 8GB cache 實測可跑，KV cache 21,792 tokens。
- `--tp 2` 必要：BF16 權重 17.35 GB > 單卡 16 GB。
- `--dp` 不支援（streaming session 的 KV/prefix cache 綁單一 replica）。
- 模型載入 + torch.compile + CUDA graph 約 1.5 分鐘；首次啟動還要先 `pip install -e /app`（約 2 分鐘）與模型下載（約 25 分鐘，僅首次）。

## 管理指令

```bash
CTL=/home/hch/vibevoice-asr-streaming/vibevoice-ctl.sh
ssh hch@10.145.119.19 "$CTL start"        # 重啟 ASR 引擎（docker rm -f + run）
ssh hch@10.145.119.19 "$CTL stop"
ssh hch@10.145.119.19 "$CTL status"       # 容器狀態 + /v1/config
ssh hch@10.145.119.19 "$CTL logs 200"
ssh hch@10.145.119.19 "$CTL test"         # 官方串流測試（demo1-chat.mp3）
ssh hch@10.145.119.19 "$CTL demo-start"   # 重啟 HTTPS demo（7860）
ssh hch@10.145.119.19 "$CTL demo-stop"
```

**主機有每日自動關機排程（約 06:30 開、18:00-18:25 關）**：關機後容器不會自動重啟，開機後要手動 `$CTL start` + `$CTL demo-start`（模型已快取，約 3-4 分鐘就緒）。不要設 `restart: always`（與其他 GPU 服務協調原則一致）。

## API 用法

`GET /v1/config` 回 chunk 幾何：`chunk_seconds 2.933`、`lookahead_frames 4`、`sample_rate 24000`、`max_audio_windows 512`（約 25 分鐘/ session 上限，另受 `--max-model-len` 限制）。

| Endpoint | 用途 |
|---|---|
| `WS /v1/stream` | 即時 session：JSON config → float32 PCM frames → `"end"` |
| `POST /v1/transcribe`、`/v1/transcribe_batch` | 整檔轉錄 |
| `POST /v1/chat/completions` | OpenAI-compatible（整檔） |

測試腳本（容器內，音檔須在掛載的 /app 下）：

```bash
# 串流 + hotwords
docker exec vibevoice-asr-streaming python3 /app/vllm_plugin/tests/test_api_streaming.py \
  /app/demo/asr_demo/demo3-hotwords.wav --hotwords "VibeVoice,Microsoft"
# 整檔
docker exec vibevoice-asr-streaming python3 /app/vllm_plugin/tests/test_api.py \
  /app/demo/asr_demo/demo2-song.mp3
```

內建 demo 音檔：`/app/demo/asr_demo/demo1-chat.mp3`（英文雙人）、`demo2-song.mp3`（歌曲）、`demo3-hotwords.wav`（中文）。

實測成績（2026-09-24）：demo1 RTF 0.24x、demo3 中文 RTF 0.16x、demo2 六分鐘歌曲整檔 RTF 0.17x；speaker 歸因與時間戳正確，中文 + hotwords 正常。

## HTTPS / 麥克風（重要）

- 瀏覽器麥克風只在 secure context 可用 → demo 必須走 HTTPS。
- 做法：demo 容器內 uvicorn 直接掛 TLS（`/app/run_demo_https.py` 呼叫 `create_app("http://localhost:8000", 256)` 後 `uvicorn.run(..., ssl_keyfile="/certs/server.key", ssl_certfile="/certs/server.crt")`），同一 port 7860 同時服務 https 頁面與 `wss://host/ws/asr`。demo 頁 JS 依 `location.protocol` 自動切 wss，**不要**在頁面外另架 proxy 改協議。
- 後端 8000 維持 plain HTTP（只給容器/後端呼叫，不給瀏覽器）。
- 自簽憑證：瀏覽器第一次會警告，選「進階 → 繼續前往」即可；憑證只給加密與 secure context，**不是存取控制**，同網段任何人都能用。要對外或防他人使用需另加認證/ACL。
- 重建憑證（若過期或換 IP）：
  ```bash
  openssl req -x509 -newkey rsa:2048 -nodes -sha256 -days 3650 \
    -keyout server.key -out server.crt -subj "/CN=10.145.119.19" \
    -addext "subjectAltName=IP:10.145.119.19,IP:192.168.2.90,IP:127.0.0.1,DNS:localhost"
  ```
  （SAN 必須含實際使用的 IP，否則瀏覽器仍判不安全。）

## 完整重建流程（從零）

```bash
# 1. 程式碼
git clone --depth=1 https://github.com/microsoft/VibeVoice.git /home/hch/vibevoice-asr-streaming
mkdir -p /home/hch/vibevoice-model-cache /home/hch/vibevoice-certs   # 憑證見上節

# 2. 引擎容器（首次啟動會在容器內 pip install + 下載模型，約 30 分鐘）
docker run -d --name vibevoice-asr-streaming --gpus all --ipc=host \
  --ulimit memlock=-1:-1 --ulimit stack=67108864:67108864 -p 8000:8000 \
  -e HF_HOME=/root/.cache/huggingface -e VIBEVOICE_FFMPEG_MAX_CONCURRENCY=64 \
  -e PYTORCH_ALLOC_CONF=expandable_segments:True \
  -v /home/hch/vibevoice-asr-streaming:/app \
  -v /home/hch/vibevoice-model-cache:/root/.cache/huggingface \
  --entrypoint bash vllm/vllm-openai:v0.14.1 \
  -lc "python3 /app/vllm_plugin/scripts/start_streaming_server.py --skip-deps --tp 2 --port 8000 --max-model-len 16384 --max-audio-windows 512 --mm-processor-cache-gb 8 --gpu-memory-utilization 0.80 > /app/vibevoice-server.log 2>&1"

# 3. HTTPS demo（run_demo_https.py 內容見「HTTPS / 麥克風」節）
docker run -d --network host --name vibevoice-demo \
  -v /home/hch/vibevoice-asr-streaming:/app -v /home/hch/vibevoice-certs:/certs:ro \
  -w /app --entrypoint bash vllm/vllm-openai:v0.14.1 \
  -lc "python3 /app/run_demo_https.py > /app/vibevoice-demo.log 2>&1"
```

## Pitfalls

- **vllm image 的 entrypoint 是 `vllm serve`**：`docker run vllm/vllm-openai ... python3 -c ...` 會被當成 serve 參數而報奇怪錯誤（`--compilation-config invalid JSON`）。跑其他指令一律加 `--entrypoint`。
- **`pip install -e /app[vllm]` 的 `[vllm]` extra 不存在**（pyproject 只有 `streamingtts`）：pip 只裝 base deps，無害；但會升級容器內 safetensors/cryptography（容器層，不持久，無妨）。`--skip-deps` 只跳過 apt，不跳過 pip。
- **SSH 前景跑長作業會超時**：`docker pull`（約 9GB）、模型下載（約 17GB）都超過一般 tool timeout。用 `docker run -d` 讓下載在容器內背景跑，再輪詢 log / cache 大小；不要重複觸發多個下載 process。
- **16GB 卡不要照官方預設參數**（見「啟動參數」節）；OOM 症狀是啟動 log 的 CUDA out of memory，降 `--gpu-memory-utilization` 或 `--mm-processor-cache-gb`。
- **啟動 log 的 tokenizer 警告**（`Qwen2Tokenizer` vs `VibeVoiceASRTextTokenizerFast`）是正常訊息，不是錯誤。
- **log 檔在掛載目錄**：`vibevoice-server.log` / `vibevoice-demo.log` 寫在 `/home/hch/vibevoice-asr-streaming/`（容器內 /app），`docker logs` 幾乎是空的，除錯要看這兩個檔。
- **GPU 協調**：此服務吃滿雙卡。啟動其他 GPU 服務（ComfyUI、openjev 訓練等）前先確認 VRAM；本機沒有 gpu-switch 涵蓋它，需手動 `$CTL stop` 釋放。
- **session 路由**：多副本擴充時不能對單一 chunk round-robin（prefix cache 綁 replica）；要擴 throughput 就一卡一 server、整 session 路由。
- 不要把它接進本機閃電說/8091 proxy（那是 Qwen3-ASR 的 OpenAI multimodal 格式）；VibeVoice 的介面是自家 WS/REST。

## Verification

1. `ssh hch@10.145.119.19 "$CTL status"` → 容器 Up 且 `/v1/config` 回 `chunk_seconds 2.933`。
2. `nvidia-smi` → 兩卡各約 13.6 GB used。
3. `$CTL test` → 24 chunks、RTF < 0.5、segments 含 `Speaker 0/1` 與時間戳。
4. `curl -skS -o /dev/null -w "%{http_code}" https://127.0.0.1:7860/` → 200；同 port 純 http 應被拒。
5. 瀏覽器開 `https://10.145.119.19:7860`（接受自簽警告）→ 允許麥克風 → 說話約 3 秒後出現帶 speaker 標籤的字幕。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** 使用者對 VibeVoice-ASR-Streaming 服務的部署/啟停/測試/除錯請求；SSH 可連的 `hch@10.145.119.19`。
- **Output:** 服務狀態變更 + 實際驗證輸出（config、RTF、segments、HTTP code）。

## Rules and Limitations
- 相對路徑以本 SKILL.md 目錄為基準；遠端路徑以本檔「現況」表為準，實機證據優先。
- 不停止/刪除本服務以外的容器；不設 `restart: always`。
- 不公開 `server.key` 內容；憑證變更只在遠端受保護目錄進行。
- 不保證中文/台語/遠場效果等同官方 benchmark；實測資料僅代表 demo 音檔。

## Pitfalls
- 見上方 Pitfalls 節；另：不要把 8000 直接暴露給瀏覽器當 demo（plain HTTP 無麥克風權限）。
