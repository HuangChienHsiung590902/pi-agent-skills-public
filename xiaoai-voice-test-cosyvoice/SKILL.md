---
name: xiaoai-voice-test-cosyvoice
description: 在遠端 GPU 主機 10.145.119.19 的 `/home/hch/xiaoai-voice-test` 維護「不碰正式 xiaoai、單一測試容器」的瀏覽器語音助理實驗：SenseVoice/sherpa-onnx ASR + CosyVoice-300M-SFT TTS + 既有 agent-proxy/小黑後端。用在使用者要求「測試小黑語音網頁」「CosyVoice 小黑」「xiaoai voice test」「不要碰正式 xiaoai」「8095 語音測試」「SenseVoice + CosyVoice」時。此 skill 專門處理測試容器、HTTPS quick tunnel、CosyVoice 依賴/模型下載與驗證；正式小愛音箱/open-xiaoai-bridge 流程仍用 xiaoai skill。
triggers:
  - xiaoai-voice-test
  - CosyVoice 小黑
  - cosyvoice 小黑
  - SenseVoice CosyVoice
  - 語音測試頁
  - 8095
  - 小黑網頁語音
  - 不要碰正式 xiaoai
  - 不要拆服務
argument-hint: "[status|logs|start|stop|rebuild|tts-test|health|tunnel|cleanup]"
---

# xiaoai-voice-test-cosyvoice — 小黑瀏覽器語音助理測試容器

## 核心原則

這個 skill 只管理 **測試環境**：

```text
遠端主機：hch@10.145.119.19
測試專案：/home/hch/xiaoai-voice-test
測試容器：xiaoai-voice-test
測試 image：xiaoai-voice-test:cosy
測試 port：8095
```

**不要碰正式 xiaoai stack**：

```text
/home/hch/xiaoai/compose.yaml
/home/hch/open-xiaoai-bridge
/home/hch/agent-proxy
/home/hch/llm-wiki
/home/hch/llama-cpp-docker
```

除非使用者明確改口，否則不要修改正式 `xiaoai` compose，也不要把測試拆成多個新服務。使用者偏好是：先用一個大測試容器驗證，覺得不好時可以刪一個容器/目錄清掉。

---

## 架構

測試容器本身提供瀏覽器 UI 與 API：

```text
Browser HTTPS page
  ↓ mic audio
xiaoai-voice-test :8095
  ├─ /asr          SenseVoice sherpa-onnx int8
  ├─ /chat         呼叫既有 agent-proxy
  ├─ /tts          CosyVoice-300M-SFT
  └─ /voice-chat   ASR → agent-proxy → CosyVoice TTS
```

外部依賴只使用既有正式後端，但不修改它們：

```text
ASR model mount：/home/hch/open-xiaoai-bridge/models:/models:ro
agent-proxy：    http://host.docker.internal:8082
CosyVoice cache：/home/hch/xiaoai-voice-test/cosy-models:/root/.cache/modelscope
```

目前已驗證：

- `/health` 正常
- ASR model available: true
- agent-proxy OK
- `/tts` 可回傳 CosyVoice WAV：`RIFF WAVE audio, PCM, 16 bit, mono 22050 Hz`

---

## 重要 URL / Port

HTTP 測試 URL：

```text
http://10.145.119.19:8095
```

但瀏覽器麥克風通常需要 HTTPS；HTTP IP 頁面可能無法使用 mic。

目前曾用 Cloudflare quick tunnel：

```text
cloudflared tunnel --url http://localhost:8095 --no-autoupdate
```

記錄位置：

```text
/home/hch/xiaoai-voice-test/cloudflared.log
/home/hch/xiaoai-voice-test/cloudflared.pid
```

quick tunnel URL 會變動；不要假設舊 URL 永久有效。若要查目前 URL：

```bash
ssh hch@10.145.119.19 'cat /home/hch/xiaoai-voice-test/cloudflared.log | grep -o "https://[-a-zA-Z0-9.]*trycloudflare.com" | tail -1'
```

---

## 目前關鍵檔案

```text
/home/hch/xiaoai-voice-test/Dockerfile.cosy
/home/hch/xiaoai-voice-test/docker-compose.cosy.yml
/home/hch/xiaoai-voice-test/requirements-cosy.txt
/home/hch/xiaoai-voice-test/cosy_app/main.py
/home/hch/xiaoai-voice-test/cosy_app/cosy_tts.py
/home/hch/xiaoai-voice-test/cosy-models/
```

Windows 暫存副本若存在只當參考，不是權威來源：

```text
C:\Users\HCH\AppData\Local\Temp\xiaoai-voice-test\...
```

實際修改以遠端 `/home/hch/xiaoai-voice-test` 為準。

---

## Compose 設定要點

`docker-compose.cosy.yml` 重點：

```yaml
services:
  xiaoai-voice-test:
    image: xiaoai-voice-test:cosy
    container_name: xiaoai-voice-test
    restart: "no"
    runtime: nvidia
    ports:
      - "8095:8095"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - ASR_MODEL_DIR=/models/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-int8-2024-07-17
      - AGENT_BASE_URL=http://host.docker.internal:8082
      - COSYVOICE_MODEL_DIR=iic/CosyVoice-300M-SFT
      - COSYVOICE_SPK_ID=中文女
      - COSYVOICE_FP16=1
      - NVIDIA_VISIBLE_DEVICES=1
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - /home/hch/open-xiaoai-bridge/models:/models:ro
      - /home/hch/xiaoai-voice-test/cosy-models:/root/.cache/modelscope
```

### 為什麼 `COSYVOICE_MODEL_DIR=iic/CosyVoice-300M-SFT`

一開始若設成：

```text
/models/CosyVoice-300M-SFT
```

CosyVoice/modelscope 會把它當成 ModelScope model id 去查，導致：

```text
The request model: /models/CosyVoice-300M-SFT does not exist!
```

目前用 ModelScope id：

```text
iic/CosyVoice-300M-SFT
```

模型會下載到容器內 `/root/.cache/modelscope/hub/iic/CosyVoice-300M-SFT`，而該 cache 掛到主機：

```text
/home/hch/xiaoai-voice-test/cosy-models
```

---

## 已踩過的 CosyVoice 依賴坑

`requirements-cosy.txt` 必須包含這兩個額外依賴：

```text
lightning==2.2.4
matplotlib==3.8.4
```

否則 `/tts` 會 502：

```text
No module named 'lightning'
No module named 'matplotlib'
```

`openai-whisper==20231117` 不建議直接放在 requirements 裡一起裝，曾遇到 build/dependency 問題。Dockerfile 目前採用：

```dockerfile
RUN pip install --no-cache-dir "setuptools<81" wheel
RUN pip install --no-cache-dir -r /app/requirements-cosy.txt
RUN pip install --no-cache-dir --no-build-isolation openai-whisper==20231117
```

若重建 image，確認 Dockerfile 仍保留這段。

---

## 常用操作

### 查狀態

```bash
ssh hch@10.145.119.19 'cd /home/hch/xiaoai-voice-test && docker compose -f docker-compose.cosy.yml ps && curl -s localhost:8095/health | python3 -m json.tool'
```

### 看 log

```bash
ssh hch@10.145.119.19 'docker logs --tail=120 xiaoai-voice-test 2>&1'
```

### 啟動 / 停止測試容器

```bash
ssh hch@10.145.119.19 'cd /home/hch/xiaoai-voice-test && docker compose -f docker-compose.cosy.yml up -d'
```

```bash
ssh hch@10.145.119.19 'cd /home/hch/xiaoai-voice-test && docker compose -f docker-compose.cosy.yml down'
```

### 重建測試 image

只會動測試容器：

```bash
ssh hch@10.145.119.19 'cd /home/hch/xiaoai-voice-test && docker compose -f docker-compose.cosy.yml build && docker compose -f docker-compose.cosy.yml up -d'
```

### 測 CosyVoice TTS

```bash
ssh hch@10.145.119.19 'curl -s --max-time 300 -X POST localhost:8095/tts -F "text=你好，我是 CosyVoice 小黑語音測試。" -o /tmp/xiaoai-cosy-check.wav -w "%{http_code} %{size_download}\n"; file /tmp/xiaoai-cosy-check.wav; ls -lh /tmp/xiaoai-cosy-check.wav'
```

成功應類似：

```text
200 111660
/tmp/xiaoai-cosy-check.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 22050 Hz
```

若看到 JSON text data，代表錯誤訊息，不是音檔：

```bash
ssh hch@10.145.119.19 'cat /tmp/xiaoai-cosy-check.wav'
```

### 測 agent-proxy 連線

```bash
ssh hch@10.145.119.19 'docker exec xiaoai-voice-test python - <<"PY"
import requests
print(requests.get("http://host.docker.internal:8082/health", timeout=5).text)
PY'
```

### 查 GPU/VRAM

```bash
ssh hch@10.145.119.19 'nvidia-smi --query-gpu=index,memory.used,memory.free --format=csv,noheader'
```

測試容器設定使用 GPU 1：

```text
NVIDIA_VISIBLE_DEVICES=1
```

如果 GPU 1 被 ComfyUI/MiniMax-H3 或其他工作佔用，先跟使用者確認再停服務或改 GPU。

---

## HTTPS quick tunnel

### 啟動 quick tunnel

若沒有正式 domain，只為了瀏覽器 mic 測試，可啟動 Cloudflare quick tunnel：

```bash
ssh hch@10.145.119.19 'cd /home/hch/xiaoai-voice-test; nohup cloudflared tunnel --url http://localhost:8095 --no-autoupdate > cloudflared.log 2>&1 & echo $! > cloudflared.pid; sleep 5; grep -o "https://[-a-zA-Z0-9.]*trycloudflare.com" cloudflared.log | tail -1'
```

### 停止 quick tunnel

```bash
ssh hch@10.145.119.19 'if [ -f /home/hch/xiaoai-voice-test/cloudflared.pid ]; then kill $(cat /home/hch/xiaoai-voice-test/cloudflared.pid) || true; rm -f /home/hch/xiaoai-voice-test/cloudflared.pid; fi'
```

---

## 首次下載模型會很久

第一次呼叫 `/tts` 會下載 CosyVoice 模型，可能花很久。已看過主要檔案包含：

```text
campplus.onnx                 27MB
flow.decoder.estimator.fp32.onnx 313MB
llm.pt                        約 1.16GB
speech_tokenizer_v1.onnx      約 498MB
spk2info.pt
```

主機 cache 可能超過 5GB：

```bash
ssh hch@10.145.119.19 'du -sh /home/hch/xiaoai-voice-test/cosy-models'
```

下載進行中不要重啟容器；否則可能中斷再續跑，浪費時間。

---

## 清理測試環境

這是破壞性操作，會刪掉測試容器/image/模型 cache。執行前先問使用者確認。

```bash
ssh hch@10.145.119.19 'cd /home/hch/xiaoai-voice-test && docker compose -f docker-compose.cosy.yml down; docker rmi xiaoai-voice-test:cosy || true'
```

若使用者要連模型 cache 與專案一起刪：

```bash
ssh hch@10.145.119.19 'rm -rf /home/hch/xiaoai-voice-test'
```

---

## 與正式 xiaoai skill 的分工

- 要操作正式小愛音箱、open-xiaoai-bridge、召喚小黑、`xiaoai up/down`：用 `xiaoai` skill。
- 要操作這個「瀏覽器語音測試頁 + CosyVoice」：用本 skill。
- 本測試目標是繞過小愛音箱 ASR/TTS；所以 `xiaoai` skill 的「小愛同學 → 召喚小黑 → 問題」鐵則只適用於正式音箱流程，不適用於本瀏覽器測試頁。

---

## 最後已知可用狀態

最後一次成功驗證：

```text
/tts HTTP 200
WAV: PCM 16-bit mono 22050 Hz
TTS: CosyVoice-300M-SFT
Speaker: 中文女
ASR: sherpa-onnx SenseVoice int8
Agent: http://host.docker.internal:8082
Port: 8095
```

---

## Conformance Addendum

## When to Use
在遠端 GPU 主機 10.145.119.19 的 `/home/hch/xiaoai-voice-test` 維護「不碰正式 xiaoai、單一測試容器」的瀏覽器語音助理實驗：SenseVoice/sherpa-onnx ASR + CosyVoice-300M-SFT TTS + 既有 agent-proxy/小黑後端。用在使用者要求「測試小黑語音網頁」「CosyVoice 小黑」「xiaoai voice test」「不要碰正式 xiaoai」「8095 語音測試」「SenseVoice + CosyVoice」時。此 skill 專門處理測試容器、HTTPS quick tunnel、CosyVoice 依賴/模型下載與驗證；正式小愛音箱/open-xiaoai-bridge 流程仍用 xiaoai skill。

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
