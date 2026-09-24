---
name: qwen3-embedding-jetson
description: 在 Jetson 主機 10.145.119.12（aarch64、JetPack 4.6.7、L4T R32.7.6、CUDA 10.2）上部署、驗證與維護 Qwen3-Embedding-0.6B 的本地 embedding API。當使用者提到「在 .12 跑 Qwen3 embedding」「Jetson embedding」「Qwen3-Embedding-0.6B」「本地向量模型」「/v1/embeddings」、要讓 Jetson GPU 做 embedding、或要排查 Jetson Docker NVIDIA runtime／tegrastats／llama.cpp CUDA 相容性時使用。不要把 Qwythos 或一般聊天模型當 embedding 模型；先確認 Jetson 舊 CUDA 與 llama.cpp 版本相容性，必要時安全地退回 CPU，不要擅自重建既有服務。
---

# Qwen3 Embedding on Jetson

## 目標

在 `hch@10.145.119.12` 上以 Docker／Compose 執行 Qwen3-Embedding-0.6B，優先提供 OpenAI-compatible embedding API：

```text
http://10.145.119.12:8191/v1/embeddings
```

模型：

```text
Qwen/Qwen3-Embedding-0.6B-GGUF
Qwen3-Embedding-0.6B-Q8_0.gguf
```

模型檔放在：

```text
/home/hch/models/qwen3-embedding/Qwen3-Embedding-0.6B-Q8_0.gguf
```

## 已確認的主機限制

部署前先確認現況，不要假設 Jetson 等於 RTX 主機：

- 主機：`10.145.119.12`，hostname `Jetson`
- 架構：`aarch64`
- JetPack/L4T：`R32.7.6` / JetPack 4.6.7
- CUDA：`10.2`
- GPU 監控：`tegrastats`，不是 `nvidia-smi`
- Docker：20.10.21
- NVIDIA runtime 已存在：`nvidia-container-runtime`
- 預設 runtime 是 `runc`；GPU 容器需明確指定 `runtime: nvidia`
- 容器 GPU 裝置驗證：`/dev/nvhost-gpu`
- Jetson GPU 使用率從 `tegrastats` 的 `GR3D_FREQ` 讀取
- Jetson 不適合 Spark-X2.5-4B：只有約 4 GB RAM、JetPack 4.6.7／CUDA 10.2；Spark Q4 GGUF 約 2.6 GB，還需 KV cache 與執行環境

Jetson 的 GPU 是整合式 GPU，共用系統記憶體；不要把它當成獨立 VRAM RTX 卡。

## 安全原則

1. 先檢查 `/home/hch`、Docker 容器、可用磁碟與記憶體；不要直接重建既有容器。
2. 新服務使用獨立名稱 `qwen3-embedding`、獨立目錄與 port `8191`。
3. 不要修改或停止 `.12` 上既有服務，除非使用者明確要求。
4. GPU 服務若在遠端主機上執行，不設定 `restart: always` 或 `restart: unless-stopped`，除非使用者明確要求；可建立 Compose，但預設不自動開機啟動。
5. API 先綁定主機 port 並限制網路暴露範圍；若需要公開給外部，先確認安全需求。
6. 如果 GPU 編譯失敗，不要反覆使用相同的新版 llama.cpp；改用相容版本、CPU fallback，或先請使用者確認升級 JetPack。
7. 不要在 Jetson 上把 Spark-X2.5-4B 當成一般 ARM64 模型部署；Spark 要使用 XHToken fork 與 `spark2_5` 架構，正確位置是 Windows Intel Arc 或 `.19` RTX GPU 主機。

## 建議流程

### 1. 查現況

```bash
ssh hch@10.145.119.12 'cat /etc/nv_tegra_release; docker info; tegrastats --interval 1000'
```

確認：

```bash
command -v tegrastats
command -v nvcc
ls -l /dev/nvhost-gpu
```

### 2. 取得模型

```bash
mkdir -p /home/hch/models/qwen3-embedding
wget -O /home/hch/models/qwen3-embedding/Qwen3-Embedding-0.6B-Q8_0.gguf \
  'https://huggingface.co/Qwen/Qwen3-Embedding-0.6B-GGUF/resolve/main/Qwen3-Embedding-0.6B-Q8_0.gguf?download=true'
```

Q8 模型約 610 MB；模型下載完成後先檢查檔案大小與 sha256，再啟動服務。

### 3. Docker GPU 驗證

使用 Jetson 相容 base image：

```bash
docker run --rm --runtime nvidia --gpus all \
  nvcr.io/nvidia/l4t-base:r32.7.1 \
  bash -lc 'ls -l /dev/nvhost-gpu; cat /etc/nv_tegra_release || true'
```

如果只需確認 runtime，看到 `/dev/nvhost-gpu` 即可；不要把 base image 內不存在 `/etc/nv_tegra_release` 誤判成 host 驅動不存在。

### 4. llama.cpp 相容性

Qwen3 Embedding 官方 llama.cpp 啟動概念：

```bash
llama-server -m Qwen3-Embedding-0.6B-Q8_0.gguf \
  --embedding --pooling last \
  --host 0.0.0.0 --port 8191
```

但 JetPack 4 的 CUDA 10.2 可能無法編譯最新版 llama.cpp。常見錯誤：

- `cuda_bf16.h: No such file or directory`
- `CUDA17 ... compiler does not support this`
- `CUDA Toolkit not found`
- `filesystem: No such file or directory`（GCC 7／C++17 不完整）

處理順序：

1. 先在 host 使用 `/usr/local/cuda/bin/nvcc` 與 CMake 3.28+ 嘗試相容編譯。
2. 對 CUDA 10.2 使用 `CMAKE_CUDA_STANDARD=14` 與 `CMAKE_CXX_STANDARD=14`；不要把 C++17 錯誤一直重試。
3. 如果新版 llama.cpp 仍依賴 CUDA 10.2 沒有的 BF16 header，改用經測試的舊版或 CPU build。
4. 若只需要 API 可用，先用 CPU fallback；不要自行 patch BF16 kernel 後宣稱 GPU 已啟用。
5. 成功 GPU build 後才加入 `runtime: nvidia` 啟動；驗證 `tegrastats` 的 `GR3D_FREQ` 必須在 embedding 請求期間上升。

### 5. Compose 基本模板

```yaml
version: '3.8'
services:
  qwen3-embedding:
    image: local/qwen3-embedding-jetson:r32.7.6
    container_name: qwen3-embedding
    runtime: nvidia
    environment:
      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: compute,utility
      MODEL_PATH: /models/Qwen3-Embedding-0.6B-Q8_0.gguf
    ports:
      - '8191:8191'
    volumes:
      - /home/hch/models/qwen3-embedding:/models:ro
```

Jetson 舊版 compose 可能不接受 `gpus: all`，優先使用 `runtime: nvidia`。

GPU 服務不要預設加 `restart: unless-stopped`；啟動命令要由使用者明確執行：

```bash
docker-compose -f /home/hch/qwen3-embedding/docker-compose.yml up -d
```

### 6. API 驗證

```bash
curl -s http://10.145.119.12:8191/v1/models
curl -s http://10.145.119.12:8191/v1/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-embedding-0.6b","input":"測試向量"}'
```

成功標準：

- `/v1/models` 回傳 model id
- `/v1/embeddings` 回傳非空的 `embedding` array
- 回傳向量維度符合模型設定（預設最多 1024）
- 請求期間 `tegrastats` 的 `GR3D_FREQ` 有變化（GPU 版）
- `docker inspect qwen3-embedding --format '{{.HostConfig.Runtime}}'` 回傳 `nvidia`（GPU 版）

## 不適合的模型：Spark-X2.5-4B

Spark-X2.5-4B 官方 Q4 GGUF 約 2.6 GB。`.12` 只有約 4 GB RAM，且目前已有 Docker 服務；即使模型檔能下載，也不代表可以穩定推理。Spark 另外依賴 XHToken/llama.cpp fork、`spark2_5` 架構，不能直接套用上游 llama.cpp。

如要跑 Spark，改用：

- Windows 本機 `spark-x25-llamacpp-local`（Intel Arc Vulkan／CPU）
- 遠端 `10.145.119.19` `spark-x25-docker-comfyui`（RTX GPU）

不要為了在 `.12` 試 Spark 而停止或重建既有服務。

## CPU fallback

如果 CUDA 10.2 相容性阻塞，但使用者要先讓 embedding API 可用：

- 編譯／執行 CPU 版 llama.cpp
- 保留同一模型與 port `8191`
- 明確回報「目前是 CPU，不是 GPU」
- 不要用 GPU 服務的驗證標準宣稱成功

CPU fallback 啟動前，先確認使用者接受較低吞吐量與較高延遲。

## 回報格式

完成或失敗都要報告：

1. 主機、JetPack/CUDA、模型檔路徑與版本
2. 使用 Docker/Compose 設定
3. 是否使用 GPU；GPU 證據（runtime、`/dev/nvhost-gpu`、`tegrastats`）
4. API URL、model id、實際 curl 驗證結果
5. 是否設定開機啟動（GPU 版本預設否）
6. 未完成項目與下一步，不要把「模型已下載」寫成「服務已部署」

## Verification

- `D:\OB\skills\skill-creator\scripts\quick_validate.py D:\OB\skills\qwen3-embedding-jetson`
- `node`／Docker／Compose 設定語法檢查（依實際產物）
- 遠端 `docker ps`、`docker inspect`、API `/v1/models`、`/v1/embeddings`
- GPU 版本另外收集 `tegrastats` 請求前後證據
