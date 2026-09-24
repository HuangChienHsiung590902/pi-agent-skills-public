---
name: joyai-vl-interaction-docker
description: 在遠端雙 RTX 4060 Ti 主機 10.145.119.19 安裝、啟動、驗證與維護 JoyAI-VL-Interaction Docker 核心版；涵蓋 INT4 主模型、Qwen3-VL-4B 摘要模型、雙 GPU 隔離、WebRTC ICE/UDP、vLLM KV cache/context OOM，以及串流約 70 幀後出現 Adapter 502、摘要模型 Input length 16012 exceeds 8192 的修復。使用者提到 JoyAI-VL-Interaction、JoyAI Docker、8099 視覺互動服務，或要求重裝/除錯此部署時使用。
---

# JoyAI-VL-Interaction 雙 GPU Docker 部署

## When to Use

使用者要求以下任務時載入本 skill：

- 在 `hch@10.145.119.19` 安裝或重裝 `jd-opensource/JoyAI-VL-Interaction`。
- 啟動、停止或檢查 JoyAI Docker 容器。
- 排查 `7060`、`8065`、`8070`、`8099` 端點或 WebUI。
- 處理 vLLM `CUDA out of memory`、KV cache 不足、GPU 用錯卡、健康檢查一直 `unhealthy`。
- 修改這台雙 RTX 4060 Ti 16GB 主機上的 JoyAI context 或 GPU 配置。

本流程是目前硬體可穩定運作的**雙 GPU 核心版**：主模型、摘要模型、Streaming Adapter、WebUI。官方完整 ASR/TTS/background-agent 配置預期更多 GPU；未經重新規劃顯存，不要直接加入目前 compose。

## Current Deployment

- 遠端主機：`hch@10.145.119.19`（Ubuntu 24.04）
- 專案：`/home/hch/JoyAI-VL-Interaction`
- Compose：`/home/hch/JoyAI-VL-Interaction/container/docker-compose.2gpu.yml`
- 可重用模板：本 skill 的 `scripts/docker-compose.2gpu.yml`
- Docker images：
  - `vllm/vllm-openai:v0.22.0`
  - `joyai-vl-app:latest`
- 模型：
  - GPU 0：`jdopensource/JoyAI-VL-Interaction-INT4`
  - GPU 1：`Qwen/Qwen3-VL-4B-Instruct`
- 主模型 `max_model_len=24576`；摘要模型 `max_model_len=8192`
- WebUI：`https://10.145.119.19:8099`（自簽憑證）
- 容器：
  - `joyai-2gpu-vllm-main-1`
  - `joyai-2gpu-vllm-summary-1`
  - `joyai-2gpu-adapter-1`
  - `joyai-2gpu-webui-1`
- restart policy：`unless-stopped`

## Procedure

### 1. 先盤點，不要猜環境

```bash
ssh hch@10.145.119.19 '
  uname -a
  docker --version
  docker compose version
  nvidia-smi
  docker info | grep -i -E "Runtimes|Default Runtime"
  df -h /home
  docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
'
```

確認至少有：Docker、Compose、NVIDIA runtime、兩張約 16GB GPU，以及足夠磁碟空間。啟動前也要看其他 GPU 容器；JoyAI 穩定運行時兩張卡都會用約 13–14GB，不能與其他大型模型同時搶卡。

### 2. Clone repository

```bash
ssh hch@10.145.119.19 '
  cd /home/hch
  test -d JoyAI-VL-Interaction/.git || \
    git clone https://github.com/jd-opensource/JoyAI-VL-Interaction.git
'
```

若是維護既有部署，不要直接 `git reset --hard`，因為 `container/docker-compose.2gpu.yml` 是本機客製檔。先檢查 `git status`。

### 3. 放置雙 GPU Compose

把本 skill 附帶模板複製到遠端：

```bash
scp D:/OB/skills/joyai-vl-interaction-docker/scripts/docker-compose.2gpu.yml \
  hch@10.145.119.19:/home/hch/JoyAI-VL-Interaction/container/docker-compose.2gpu.yml

ssh hch@10.145.119.19 '
  cd /home/hch/JoyAI-VL-Interaction
  docker compose -f container/docker-compose.2gpu.yml config -q
'
```

GPU 隔離須使用：

```yaml
environment:
  NVIDIA_VISIBLE_DEVICES: "all"
  CUDA_VISIBLE_DEVICES: "0" # 主模型；摘要服務則為 "1"
```

### 4. 下載模型

建立獨立 venv，不污染系統 Python：

```bash
ssh hch@10.145.119.19 '
  set -e
  python3 -m venv /home/hch/.venvs/joyai-download
  /home/hch/.venvs/joyai-download/bin/pip install -U huggingface_hub
  mkdir -p /home/hch/JoyAI-VL-Interaction/models
  /home/hch/.venvs/joyai-download/bin/hf download \
    jdopensource/JoyAI-VL-Interaction-INT4 \
    --local-dir /home/hch/JoyAI-VL-Interaction/models/JoyAI-VL-Interaction-INT4
  /home/hch/.venvs/joyai-download/bin/hf download \
    Qwen/Qwen3-VL-4B-Instruct \
    --local-dir /home/hch/JoyAI-VL-Interaction/models/Qwen3-VL-4B-Instruct
'
```

下載可能需數十分鐘。長任務應在遠端用 `nohup`/`tmux` 並寫 log，避免 SSH 或 agent timeout 中止。完成後驗證：

```bash
ssh hch@10.145.119.19 '
  test -s /home/hch/JoyAI-VL-Interaction/models/JoyAI-VL-Interaction-INT4/model.safetensors
  test -s /home/hch/JoyAI-VL-Interaction/models/Qwen3-VL-4B-Instruct/model.safetensors.index.json
  du -sh /home/hch/JoyAI-VL-Interaction/models/*
'
```

預期約為主模型 `7.1G`、摘要模型 `8.3G`。

### 5. Pull vLLM 並 build app image

```bash
ssh hch@10.145.119.19 '
  set -e
  cd /home/hch/JoyAI-VL-Interaction
  docker pull python:3.12-slim-bookworm
  docker pull vllm/vllm-openai:v0.22.0
  docker build --network host \
    -f container/Dockerfile.app \
    -t joyai-vl-app:latest container
'
```

### 6. 分階段啟動

先啟動兩個 vLLM，確定健康後才啟動 Adapter/WebUI：

```bash
ssh hch@10.145.119.19 '
  set -e
  cd /home/hch/JoyAI-VL-Interaction
  docker compose -f container/docker-compose.2gpu.yml up -d vllm-main vllm-summary
  docker compose -f container/docker-compose.2gpu.yml ps -a
'
```

等待兩者 `healthy`，再執行：

```bash
ssh hch@10.145.119.19 '
  set -e
  cd /home/hch/JoyAI-VL-Interaction
  docker compose -f container/docker-compose.2gpu.yml up -d adapter webui
  docker compose -f container/docker-compose.2gpu.yml ps -a
'
```

### 7. 日常維護

```bash
ssh hch@10.145.119.19 'cd /home/hch/JoyAI-VL-Interaction && docker compose -f container/docker-compose.2gpu.yml ps -a'
ssh hch@10.145.119.19 'cd /home/hch/JoyAI-VL-Interaction && docker compose -f container/docker-compose.2gpu.yml logs -f --tail=200'
ssh hch@10.145.119.19 'cd /home/hch/JoyAI-VL-Interaction && docker compose -f container/docker-compose.2gpu.yml restart'
```

停止會中斷服務，操作前先確認使用者意圖：

```bash
ssh hch@10.145.119.19 'cd /home/hch/JoyAI-VL-Interaction && docker compose -f container/docker-compose.2gpu.yml down'
```

## Troubleshooting

### 主模型一啟動就 OOM，但 GPU 分配看起來很怪

曾出現摘要模型實際佔 GPU 0、主模型也使用 GPU 0，造成主模型載入時 OOM。原因是僅用 `NVIDIA_VISIBLE_DEVICES=0/1` 搭配舊式 `runtime: nvidia`/`privileged: true` 在此環境未可靠隔離。

修法是兩個 vLLM 容器都設：

```yaml
NVIDIA_VISIBLE_DEVICES: "all"
CUDA_VISIBLE_DEVICES: "0" # 或 "1"
```

重建容器後用 `nvidia-smi` 確認主模型與摘要模型分居兩張卡。

### `max seq len 32768` 需要的 KV cache 超過可用顯存

此硬體實測錯誤：

```text
To serve at least one request with the model's max seq len (32768),
4.5 GiB KV cache is needed ... available ... 3.93 GiB.
estimated maximum model length is 28640.
```

不要一直提高 `gpu-memory-utilization` 硬撐。已驗證穩定值是：

```text
--max-model-len 24576
--gpu-memory-utilization 0.92
```

修改後要 `--force-recreate vllm-main`，單純 `restart` 不一定帶入 compose 變更：

```bash
docker compose -f container/docker-compose.2gpu.yml up -d --force-recreate vllm-main
```

### API 已可用，但容器一直 `unhealthy`

若 health log 顯示 Python URL 引號消失，例如：

```text
urllib.request.urlopen(http://127.0.0.1:8065/v1/models ...)
SyntaxError: invalid syntax
```

這是 Compose `CMD-SHELL` 巢狀 quoting 問題，不是 vLLM 故障。改用 exec-form curl：

```yaml
healthcheck:
  test: ["CMD", "curl", "-fsS", "http://127.0.0.1:8065/v1/models"]
```

WebUI 自簽憑證則使用：

```yaml
test: ["CMD", "curl", "-kfsS", "https://127.0.0.1:8099/"]
```

### 串流約 70 秒後出現 `Input length ... exceeds ... 8192`

若外層是 Adapter 502、內層是摘要模型 400，例如：

```text
Input length (16012) exceeds model's maximum context length (8192)
```

這不是主模型 `24576` context 超限，而是 `Qwen3-VL-4B-Instruct` 摘要模型在第 1 個 `CHUNK=70` 邊界收到過多視覺 tokens。16GB profile 的摘要模型只能開 `8192`，不能只靠提高 context；要縮小摘要輸入與輸出：

```yaml
SUMMARIZER_KEY_FRAMES: "3"
SUMMARIZER_MAX_PIXELS: "131072"
MID_TERM_MAX_TOKENS: "1500"
MID_TERM_TARGET_TOKEN_COUNT: "1000"
LONG_TERM_MAX_TOKENS: "1200"
LONG_TERM_TARGET_TOKEN_COUNT: "800"
```

只重建 Adapter，避免不必要地重啟兩個 vLLM：

```bash
docker compose -f container/docker-compose.2gpu.yml \
  up -d --no-deps --force-recreate adapter
```

舊 WebUI session 已卡住失敗的 async summary job；修正後要停止並重新開始串流，或呼叫 `/v1/streaming/reset` 清掉該 session。驗證不能只做一張圖，必須送至少 71 幀跨過 `CHUNK=70`，確認摘要 API 回 200，Adapter 沒有 `BadRequestError`。目前合成 71 幀實測：第 70/71 幀成功，摘要約 6.3 秒，無 context 400。

### 透過 ZeroTier 開啟 WebUI 後，Webcam 顯示 `ICE disconnected`

如果 HTTPS、WebSocket、`POST /offer` 都正常，但瀏覽器 console 顯示：

```text
ICE connection state: checking
Received remote track
ICE connection state: disconnected
```

而伺服器端出現 `RTCIceTransport is closed`，先查遠端 iptables。此主機的 INPUT chain 最後會 DROP 未允許的 UDP；WebRTC/aiortc 使用動態 UDP port，TCP 8099 可通不代表媒體路徑可通。

已驗證修法是只允許 ZeroTier 網段進入 WebRTC 動態 UDP 範圍，並持久化：

```bash
ssh hch@10.145.119.19 '
  set -e
  sudo iptables -C INPUT -i ztcdchae37 -s 10.145.119.0/24 \
    -p udp --dport 32768:60999 -j ACCEPT 2>/dev/null || \
  sudo iptables -I INPUT 2 -i ztcdchae37 -s 10.145.119.0/24 \
    -p udp --dport 32768:60999 \
    -m comment --comment joyai-webrtc-zerotier -j ACCEPT
  sudo netfilter-persistent save
'
```

不要對所有來源開放整段 UDP；限制 `ztcdchae37` 與 `10.145.119.0/24`。驗證計數器有增加：

```bash
sudo iptables -L INPUT -n -v --line-numbers | head
sudo grep joyai-webrtc /etc/iptables/rules.v4
```

修正後用 CDP 實測應看到 `Streaming`、相機畫面、Latency/Count 持續更新。若 TTS/background-agent 未部署，Settings 裡也要關閉 `Speak VLM output` 和 `Enable delegation solver`，避免每次回覆額外出現 TTS/delegation 錯誤。

### WebUI 未啟動

WebUI 依賴 Adapter，而 Adapter 又等待兩個 vLLM `healthy`。依序檢查：

```bash
docker compose -f container/docker-compose.2gpu.yml ps -a
docker logs joyai-2gpu-vllm-main-1 --tail 200
docker logs joyai-2gpu-vllm-summary-1 --tail 200
docker logs joyai-2gpu-adapter-1 --tail 200
docker logs joyai-2gpu-webui-1 --tail 200
```

先修最上游的 vLLM，不要反覆重啟 WebUI。

## Pitfalls

- 官方完整 compose 以一張主模型 GPU 加三張 API GPU 為目標；這台只有兩張 16GB 卡，不能照 `--with-all`/完整四 GPU 配置直接開。
- 目前核心版未啟用 ASR、TTS、background-agent；WebUI 的對應 URL 留空。若要增加功能，需重新做顯存與服務隔離規劃。
- 不要把 BF16 Full 主模型放進單張 16GB；使用 `JoyAI-VL-Interaction-INT4`。
- 不要假設模型權重下載完成；先 `test -s` 檢查實體檔案。
- 不要以 `docker compose up -d` 回傳成功當作部署完成；必須等 health、測端點並看 GPU。
- 不要任意 prune Docker；主機有其他專案與大型 image。清理是破壞性操作，需先取得使用者確認。
- `8099` 是 HTTPS 自簽憑證，測試需 `curl -k`；瀏覽器第一次需接受安全例外。
- 若同機其他 GPU 服務已啟動，先協調停用，避免 OOM；不要未確認就停止使用者的其他服務。

## Verification

部署完成必須同時通過：

```bash
ssh hch@10.145.119.19 '
  set -e
  cd /home/hch/JoyAI-VL-Interaction
  docker compose -f container/docker-compose.2gpu.yml ps -a
  curl -fsS http://127.0.0.1:7060/v1/models
  curl -fsS http://127.0.0.1:8065/v1/models
  curl -fsS http://127.0.0.1:8070/health
  curl -kfsS -o /dev/null -w "webui_http=%{http_code}\n" https://127.0.0.1:8099/
  nvidia-smi --query-gpu=index,name,memory.used,memory.total,utilization.gpu --format=csv,noheader
'
```

預期：

1. 四個容器均為 `healthy`。
2. `7060/v1/models` 回傳 `joyai-vl-interaction`，`max_model_len` 為 `24576`。
3. `8065/v1/models` 回傳 Qwen3-VL 摘要模型。
4. `8070/health` 回傳 `ok: true` 且 `summarizer_enabled: true`。
5. WebUI HTTP status 為 `200`。
6. GPU 0/1 各約使用 13–14GB，不是兩個模型擠在同一張卡。
7. 外部瀏覽器可開啟 `https://10.145.119.19:8099`。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
