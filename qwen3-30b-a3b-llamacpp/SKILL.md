---
name: qwen3-30b-a3b-llamacpp
description: >-
  在遠端 GPU 主機 10.145.119.19 用 llama.cpp Docker 跑 Qwen3-30B-A3B-Instruct-2507
  Q4_K_M GGUF（MoE，約 18G），並接清箋／OmniRoute 的 OpenAI-compatible 8080。用在：下載或啟動
  30B-A3B、從 Qwen3.8-27B 換成這顆、清箋 [predict] model id、aria2c 下載卡住、或
  `llama-cpp-qwen3-30b-a3b` 容器狀態。與 27B、Spark 共用 8080，只能輪流開；切換見
  qwen-spark-docker-desktop-switch。
---

# Qwen3-30B-A3B Instruct on llama.cpp

## When to Use

- 遠端 `hch@10.145.119.19`（gigabyte，2× RTX 4060 Ti 16GB）要跑 Qwen3-30B-A3B
- 清箋雲聯想要改用這顆，或確認輸入法現在打的是哪顆 GGUF
- 權重還沒下完、compose 要新建、或 `compose down` 誤刪了 27B 容器要恢復
- 不要用本 skill 去改 OmniRoute 的 8 路並發；那是 `omniroute-native` 的 admission gate，與 llama.cpp 無關

## 現況（2026-09-21 實測）

```text
權重：/home/hch/models/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf
大小：18556686752 bytes（約 18G）
來源：unsloth/Qwen3-30B-A3B-Instruct-2507-GGUF
compose：/home/hch/llama-cpp-docker/docker-compose.qwen3-30b-a3b.yml
容器：llama-cpp-qwen3-30b-a3b
宿主 port：8080
image：ghcr.io/ggml-org/llama.cpp:server-cuda
-c 8192  -ngl 99  flash-attn on  KV q8_0  --jinja
n_slots=4  kv_unified=true
載入後 VRAM：GPU0 ~9500 MiB，GPU1 ~9050 MiB
生成速度實測：約 86–88 tok/s
清箋 model：/models/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf
```

遠端沒有「Qwen3.8 MoE」權重；要換較小／較省的 MoE 時用這顆 30B-A3B，不要假裝已有 Qwen3.8 MoE。

## Inputs and Outputs

### Inputs

- SSH `hch@10.145.119.19`
- 目前 8080 上哪個容器 Running
- 清箋 `%APPDATA%\Qingjian\config.toml` 的 `[predict]`

### Outputs

- 30B 容器 Running、27B／Spark 為 Created 或 Exited
- `/health` ok、`/v1/models` 的 id 是 30B 路徑
- 清箋日誌出現 `雲聯想已接入 model=/models/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf`

## Procedure

### 1. 下載權重

HuggingFace 直連 `huggingface_hub` 在這台常因沒有 `HF_TOKEN` 卡住。用 aria2c：

```bash
ssh hch@10.145.119.19 'aria2c -x 16 -s 16 -k 1M --file-allocation=none \
  -d /home/hch/models \
  -o Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf \
  https://huggingface.co/unsloth/Qwen3-30B-A3B-Instruct-2507-GGUF/resolve/main/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf'
```

完成條件：log 出現 `Status Legend: (OK):download completed.`，檔案約 18G。下載中 `ls` 看到 17G 不代表完成。

### 2. 切換上線（保留其他容器）

27B、Spark、30B **都要留著**，使用者只是輪流開。禁止對還要保留的 compose 做 `docker compose down`（會刪容器）。

正確切到 30B：

```bash
ssh hch@10.145.119.19 'docker stop llama-cpp-qwen38-27b-abliterated llama-cpp-spark-x25-4b 2>/dev/null || true'
# 等 GPU 釋放（兩張卡 memory.used 應掉到幾十 MiB）
ssh hch@10.145.119.19 'cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwen3-30b-a3b.yml up -d'
```

若 27B 已被 `down` 刪掉，用 **不啟動** 的方式重建，避免搶 8080：

```bash
ssh hch@10.145.119.19 'cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwen38-27b-abliterated.yml up --no-start'
```

### 3. 清箋

`%APPDATA%\Qingjian\config.toml`：

```toml
[predict]
enabled = true
base_url = "http://10.145.119.19:8080/v1"
model = "/models/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf"
timeout_ms = 20000
reasoning_effort = "none"
```

改完看 `AppData\Local\Qingjian\logs\server.*.log` 是否出現 `雲聯想已接入 model=/models/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf`。實際模型以遠端 `/v1/models` 為準，不要用 Pi 目前的 `PI_MODEL` 判斷輸入法。

### 4. compose 重點

```yaml
command:
  - --host
  - 0.0.0.0
  - --port
  - "8080"
  - -m
  - /models/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf
  - -ngl
  - "99"
  - -c
  - "8192"
  - --flash-attn
  - "on"
  - --cache-type-k
  - q8_0
  - --cache-type-v
  - q8_0
  - --jinja
ports:
  - "8080:8080"
restart: "no"
volumes:
  - /home/hch/models:/models:ro
```

`-c 8192` 是這次部署值。27B 曾用很大 context；30B 權重約 18G，兩張 16GB 卡載入後約各 9GB，KV 不要一開始就拉到 27B 那種 131072。

## Verification

```bash
ssh hch@10.145.119.19 'docker ps -a --filter name=llama-cpp --format "{{.Names}} {{.Status}} {{.Ports}}"'
curl -sS http://10.145.119.19:8080/health
curl -sS http://10.145.119.19:8080/v1/models
```

成功：

- 30B 容器 healthy，27B 為 Created／Exited，Spark 為 Exited
- health `{"status":"ok"}`
- model id 含 `Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf`
- log 有 `llama_server: model loaded` 與 `listening on http://0.0.0.0:8080`

## Pitfalls

- **`compose down` 會刪容器**：使用者要換來換去時只 `docker stop`／Docker Desktop Stop。誤刪後用 `up --no-start` 恢復。
- **huggingface_hub 無 token 會 stall**：改 aria2c；中途不要把未完成檔當成品。
- **8080 不是模型名**：同一 port 可能是 27B、30B 或 Spark；一定看 `/v1/models`。
- **Pi 模型 ≠ 輸入法模型**：Pi session 可能是 `xai/grok-4.6`；清箋看 `config.toml` 與 server log。
- **OmniRoute 8 路並發不是 llama.cpp `-np`**：`OMNIROUTE_CHAT_MAX_HEAVY_IN_FLIGHT=8` 只擋 gateway；這顆 llama.cpp 仍是 `n_slots=4`。
- **不要把 API Key 寫進本 skill**。
- curl 測 chat 時 Windows 終端 UTF-8 可能讓 llama.cpp 回 JSON parse error；用 ASCII 或檔案 body。

## Rules and Limitations

- 與 27B、Spark 共用 8080，不能同時 Running。
- 不得加 `restart: always`／`unless-stopped`。
- 不得刪 `%APPDATA%\Qingjian` 學習資料或遠端 `/home/hch/models`。
- 切換模型後清箋 `model` 必須對上當時 `/v1/models`。
