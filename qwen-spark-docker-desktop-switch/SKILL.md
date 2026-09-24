---
name: qwen-spark-docker-desktop-switch
description: >-
  在遠端 GPU 主機 10.145.119.19（gigabyte）用 Docker Desktop 或 docker stop/start
  輪流開關 Qwen3.8-27B、Qwen3-30B-A3B、Spark-X2.5-4B。三個容器都共用宿主機 port 8080，
  不能同時啟動，但容器都要保留。涵蓋 Stop/Start 切換、誤用 compose down 後用
  up --no-start 恢復、GPU 釋放、同一個 OmniRoute/Pi/清箋 endpoint、健康檢查與
  8080 port 衝突。當使用者說「換模型」「Qwen 和 Spark 共用 8080」「兩個容器不能同時開」
  「27B 容器不見了」時使用。
---

# 8080 模型輪流切換（27B／30B-A3B／Spark）

## When to Use

- 遠端主機：`hch@10.145.119.19`，hostname `gigabyte`
- 用 Docker Desktop **Start／Stop** 或 `docker stop`／`docker start` 輪流切換
- 27B、30B-A3B、Spark 都要使用宿主機 `8080`
- Pi／OmniRoute／清箋共用：`http://10.145.119.19:8080/v1`
- 要確認目前 8080 實際是哪一顆模型
- 使用者明確說容器要留著、只是換來換去

## Current Architecture

| 項目 | Qwen3.8-27B | Qwen3-30B-A3B | Spark-X2.5-4B |
|---|---|---|---|
| Container | `llama-cpp-qwen38-27b-abliterated` | `llama-cpp-qwen3-30b-a3b` | `llama-cpp-spark-x25-4b` |
| Compose | `docker-compose.qwen38-27b-abliterated.yml` | `docker-compose.qwen3-30b-a3b.yml` | `docker-compose.spark-x25-4b.yml` |
| Host port | `8080` | `8080` | `8080` |
| GPU | GPU0 + GPU1 | GPU0 + GPU1 | GPU0 |
| Model | `/models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf` | `/models/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf` | `/models/Spark-X2.5-4B.gguf` |
| 清箋 model | 同上 | 同上 | 同上 |
| Pi/OmniRoute prefix | `wq` | 依 provider 實際列出 | `sp` |

compose 目錄：`/home/hch/llama-cpp-docker/`。

27B／30B 的 `ports` 是 `8080:8080`；Spark 使用 `network_mode: host`。三者都能存在於 Docker Desktop，但同一時間只能有一個容器綁定 `0.0.0.0:8080`。

2026-09-21 切到 30B 後實測：27B 應用 `docker compose ... up --no-start` 恢復成 Created；Spark 維持 Exited；30B Running healthy。

## Inputs and Outputs

### Inputs

- Docker Desktop 或 `docker ps -a` 的 Running／Created／Exited
- 遠端 `10.145.119.19:8080` 的 health、models API、GPU
- 目標模型，以及清箋／Pi／OmniRoute 目前 model ID

### Outputs

- 目標容器 Running，另外兩個不是 Running
- 8080 health、實際 `/v1/models`、GPU 釋放驗證
- 清箋 `[predict].model` 已對上當時 `/v1/models`

## Procedure

1. 先確認三個容器都已建立。沒有的用對應 compose `up --no-start` 建立，不要直接 `up -d` 去搶 8080。
2. **Stop** 目前 Running 的那一個；不要 `docker compose down`（會刪容器）。
3. 等待 GPU 記憶體釋放（兩張卡應掉到幾十 MiB 量級）。
4. **Start** 目標容器；等 `model loaded` 與 `/health` ok。
5. 看 `http://10.145.119.19:8080/v1/models` 的實際 id。
6. 清箋 `config.toml` 的 `[predict].model` 改成該 id；日誌應有 `雲聯想已接入 model=...`。
7. Pi／OmniRoute 選對應 model ID。選錯 ID 不會自動切 Docker。

### 切換口令

```text
Stop 目前 Running 的 llama-cpp-*
等 GPU 釋放
Start 目標 llama-cpp-*
核對 /v1/models
改清箋 model（若這次是給輸入法用）
```

## Verification

```bash
ssh hch@10.145.119.19 'docker ps -a --filter name=llama-cpp --format "{{.Names}} {{.Status}} {{.Ports}}"; nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv'
curl -sS http://10.145.119.19:8080/health
curl -sS http://10.145.119.19:8080/v1/models
```

- 只有一個 llama-cpp 容器 Running
- `/health` 為 ok
- `/v1/models` 的 id 是剛 Start 的那顆
- 不要用 8081／8083 判斷現行 endpoint；ComfyUI 仍是 8190

## Rules and Limitations

- 三個模型共用 8080，只能序列化切換。
- **保留容器**：Stop／Start，不要 `compose down`，除非使用者明確要刪掉。
- 不得加 `restart: always` 或 `unless-stopped`。
- 不得刪 `/home/hch/models`、ASR 檔或未經授權的容器。
- 未通過 health、models、port、GPU 驗證前，不得宣稱切換成功。

## Pitfalls

- **`compose down` 會 Removed 容器**：2026-09-21 切 30B 時對 27B 做了 down，使用者要求恢復；用 `up --no-start` 重建成 Created，30B 繼續佔 8080。
- **兩個／三個不能同時用 8080**。
- **Start 不會自動 Stop 另一個**。
- **8080 不是模型名稱**：一定看 `/v1/models`。
- **Pi 目前模型不是輸入法模型**：Pi 可能是 grok；清箋看 `%APPDATA%\Qingjian\config.toml` 與 `Local\Qingjian\logs\server.*.log`。
- **清箋 model 必須對 `/v1/models`**：寫成 `QW//models/...` 會連得上但選不到模型。
- **不要讓 compose 使用 `restart: unless-stopped`**。
- **修改 compose 後要重建容器**，不要刪模型目錄。

## Rollback

port 修改前備份：

```text
/home/hch/llama-cpp-docker/docker-compose.qwen38-27b-abliterated.yml.bak-port-20260914-114626
/home/hch/llama-cpp-docker/docker-compose.spark-x25-4b.yml.bak-port-20260914-114626
```

30B compose：`docker-compose.qwen3-30b-a3b.yml`。切回 27B 時 Stop 30B，Start 27B，清箋 model 改回 `/models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf`。

## Conformance

- **Input:** 三個 llama-cpp 容器狀態、8080 API、GPU、清箋／Pi model id。
- **Output:** 目前 active 模型、可逆切換、驗證結果。
- 不宣稱可同時跑兩顆；兩張 16GB GPU 必須序列化啟動。
