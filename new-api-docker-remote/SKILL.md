---
name: new-api-docker-remote
description: >-
  在遠端 GPU 主機 10.145.119.19（gigabyte，hch@10.145.119.19）以 Docker Compose 部署、更新、驗證與維護 QuantumNous New API。當使用者提到 New API、new-api、newapi、docs.newapi.ai 的 Docker Compose 安裝、要把 New API 裝到 GPU server、檢查 New API :3000、PostgreSQL/Redis 依賴、或要求重啟／更新／除錯這個服務時，務必使用本 Skill。此服務本身不需要 GPU；不要為了 New API 停止既有的 Qwen、ComfyUI、Kimodo 或其他 GPU 容器。
compatibility: 需要可用的 SSH 金鑰登入 hch@10.145.119.19、遠端 Docker Engine 及 Docker Compose plugin；遠端專案固定放在 /home/hch/new-api。
---

# New API 遠端 Docker Compose

在共用的 GPU 主機上部署 New API 的可重複流程。New API 是 OpenAI／Claude／Gemini 相容的 API gateway；它不需要 GPU，但要避免影響同主機既有 GPU 模型服務。

## When to Use

- 使用者說「安裝 New API」「new-api」「newapi」「New API Docker」或提供 New API 官方 Docker Compose 文件。
- 使用者要求把 New API 部署到 GPU server、`10.145.119.19`、`gigabyte` 或 `hch@10.145.119.19`。
- 使用者要求檢查、更新、重啟、查看 log 或修復遠端 New API、PostgreSQL、Redis。
- 使用者要確認 `http://10.145.119.19:3000` 或 `/api/status` 是否正常。

不要用於一般 Docker 跨主機備份、清理、Portainer 或其他應用的業務邏輯；這些使用 `docker-remote-control` 或對應的應用 Skill。

## Inputs and Outputs

### Inputs

- 目標主機：預設 `hch@10.145.119.19`，但執行前仍應用 SSH 和 hostname 驗證。
- New API 官方文件：
  `https://docs.newapi.ai/zh/docs/installation/deployment-methods/docker-compose-installation`
- 官方 Compose 設定說明：
  `https://docs.newapi.ai/zh/docs/installation/config-maintenance/docker-compose-yml`
- 可選的公開 port；預設為 `3000`。

### Outputs

- 遠端 `/home/hch/new-api/docker-compose.yml`、`.env`、`data/`、`logs/`。
- Compose project `new-api`，服務為 `new-api`、`new-api-postgres`、`new-api-redis`。
- 可開啟的 URL、版本、容器狀態、API health check 結果，以及任何剩餘限制。
- 絕不在回覆、Skill、shell history 或 log 中顯示 `.env` 內的密碼／session secret。

## Known Deployment State

最近一次已驗證的主機狀態如下；這些資訊可能隨時間改變，執行時以實際查詢為準：

- 主機：`gigabyte`，`10.145.119.19`
- 專案：`/home/hch/new-api`
- New API image：`calciumion/new-api:latest`
- 最近驗證版本：`v1.0.0-rc.40`
- URL：`http://10.145.119.19:3000`
- PostgreSQL：`postgres:15`
- Redis：`redis:7-alpine`

## Procedure

### 1. 先檢查遠端和衝突

```bash
ssh -o ConnectTimeout=10 -o BatchMode=yes hch@10.145.119.19 \
  'printf "HOST="; hostname; printf "\\nDOCKER="; docker version --format "{{.Server.Version}}"; \
   printf "\\nCOMPOSE="; docker compose version; \
   printf "\\nPROJECT="; docker compose ls -a; \
   printf "\\nPORT_3000="; ss -ltn | grep -E ":3000[[:space:]]" || true; \
   printf "\\nGPU="; nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv,noheader || true'
```

確認：

1. SSH 真的連到預期的 `gigabyte`，不要把本機 Docker 狀態當成遠端狀態。
2. Docker daemon 與 Compose plugin 可用。
3. `3000` 沒有被其他服務使用；若有衝突，先詢問使用者要改成哪個 host port，不要覆蓋既有服務。
4. 查看既有容器與 GPU 使用量，但不要停止、重啟或刪除其他服務。
5. 若已有 `/home/hch/new-api`，先讀取其 `docker-compose.yml` 與 `docker compose ps`；不要無條件刪除目錄或 volume。

### 2. 建立專案與安全設定

官方推薦 PostgreSQL + Redis。Compose 使用 `${POSTGRES_PASSWORD}`、`${REDIS_PASSWORD}`、`${SESSION_SECRET}`，並將秘密只放在遠端 `.env`：

```bash
ssh -o ConnectTimeout=10 -o BatchMode=yes hch@10.145.119.19 '\
  install -d -m 0750 /home/hch/new-api && \
  cd /home/hch/new-api && \
  if [ ! -f .env ]; then \
    printf "POSTGRES_PASSWORD=%s\\nREDIS_PASSWORD=%s\\nSESSION_SECRET=%s\\n" \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" "$(openssl rand -hex 32)" > .env && \
    chmod 600 .env; \
  fi'
```

不要把官方文件的 `123456` 預設密碼直接用於正式服務，也不要把實際 `.env` 內容貼回聊天。若既有 `.env` 已存在，保留它；改密碼前必須先確認資料庫／Redis 資料與停機風險。

Compose 應包含以下要點：

- `calciumion/new-api:latest`，container name `new-api`，host port `3000:3000`。
- `./data:/data` 與 `./logs:/app/logs`。
- PostgreSQL DSN 指向 `postgres`，Redis DSN 指向 `redis`。
- `postgres:15` 使用 named volume `pg_data`。
- Redis 啟用 password，且與 `REDIS_PASSWORD` 一致。
- 所有服務 `restart: always`。
- 不要加 GPU reservation；New API 不需要 GPU。
- 多節點部署才設定跨節點一致的 `SESSION_SECRET`／`CRYPTO_SECRET`；單節點也建議固定 `SESSION_SECRET`。

建立或修改 Compose 後，先執行：

```bash
cd /home/hch/new-api
docker compose config
```

若是首次安裝，可以從官方 repository 取得原始檔案，但應先檢查內容，並以本機／遠端產生的安全 `.env` 為準：

```bash
git clone --depth 1 https://github.com/QuantumNous/new-api.git /home/hch/new-api
```

不要因為更新而刪除 `data/`、`logs/` 或 Docker volume。

### 3. 拉取與啟動

```bash
cd /home/hch/new-api
docker compose config
docker compose pull
docker compose up -d
```

首次啟動 PostgreSQL 後，New API 可能在資料庫尚未 ready 時短暫重啟或 health check 失敗。等待約 20–40 秒後再檢查，不要立即判定安裝失敗：

```bash
sleep 20
docker compose ps
docker compose logs --tail=100 new-api postgres redis
```

### 4. 驗證服務

依序執行：

```bash
cd /home/hch/new-api
docker compose config >/dev/null
docker compose ps
curl -fsS --max-time 15 http://127.0.0.1:3000/api/status
curl -fsS --max-time 15 http://10.145.119.19:3000/api/status
```

成功條件：

- `new-api` 顯示 `Up ... (healthy)`。
- `new-api-postgres` 與 `new-api-redis` 顯示 running。
- `/api/status` 回 HTTP 200 且 JSON 有 `"success":true`。
- 回報實際版本（可從 response header `X-New-Api-Version` 或 log 取得）。
- 從操作者端實際連線確認 `http://10.145.119.19:3000`，不可只說「容器啟動所以成功」。

若首次 curl 出現 connection reset，先等資料庫 migration 完成，再重試一次；若仍失敗，查看 `docker compose logs --tail=100 new-api postgres redis`。

### 5. 更新 New API

更新前先記錄現況與資料位置：

```bash
cd /home/hch/new-api
docker compose ps
docker volume ls | grep new-api || true
git -C /home/hch/new-api log -1 --format='%H %cs %s' || true
```

一般 image 更新流程：

```bash
cd /home/hch/new-api
docker compose pull new-api
docker compose up -d new-api
docker compose ps
docker compose logs --tail=100 new-api
curl -fsS --max-time 15 http://127.0.0.1:3000/api/status
```

除非使用者明確要求，不要 `docker compose down -v`、不要刪除 `pg_data`、不要 `docker system prune`，也不要更新 PostgreSQL major version。

### 6. 常用維護

```bash
cd /home/hch/new-api

docker compose ps
docker compose logs --tail=100 new-api
docker compose logs --tail=100 postgres
docker compose logs --tail=100 redis

docker compose restart new-api
```

重啟前確認是 New API 專案；不要用全域 `docker restart $(docker ps -q)`。停機操作只限本專案：

```bash
cd /home/hch/new-api
docker compose stop
```

## Rules and Limitations

- 遠端主機是共用 GPU 環境；先檢查既有服務，不能擅自停止其他容器、清理 Docker 或釋放 GPU。
- 不得把密碼、token、`.env` 內容、SSH 私鑰或 session secret 寫入 Skill、回覆或 command output。
- 不使用官方預設 `123456` 作為正式密碼；用遠端 `openssl rand` 產生秘密。
- 不因為 New API 不用 GPU 就修改或重啟現有 GPU 模型容器。
- 不要把 PostgreSQL 或 Redis port 暴露到主機，除非使用者明確要求且已確認防火牆範圍；New API 只需透過 Compose network 連線。
- 若 `3000` 已被使用，停止並詢問 port，不要搶占或改動既有服務。
- 任何 destructive 操作（刪 volume、刪 data、`down -v`、全域 prune）都要先列出影響並取得使用者明確確認。
- 官方 `latest` image 可能變動；更新後一定重新查版本、health 與 log，不要只依 pull exit code 宣稱成功。
- New API 第一次啟動通常會顯示尚未初始化 root user；這是預期狀態，告知使用者從 Web UI 完成第一次管理員註冊。

## Pitfalls

- **把本機 Docker 當成 GPU server**：所有遠端查詢都用 SSH 並帶 hostname／預期 IP 驗證。
- **資料庫 race condition**：`depends_on` 只保證容器啟動順序，不代表 PostgreSQL 已可連線；等待後重新 curl，必要時查 log。
- **Compose YAML regex quoting**：healthcheck 儘量使用簡單的 `grep -q success`，避免 YAML double quote 吞掉 `\s` 等 escape。
- **密碼不一致**：`POSTGRES_PASSWORD` 必須與 `SQL_DSN` 相同；`REDIS_PASSWORD` 必須同時存在於 Redis command 和 `REDIS_CONN_STRING`。
- **誤刪資料**：`docker compose down -v` 會刪除資料庫 volume；維護 New API 不應使用它。
- **port 衝突**：GPU server 已有多個服務；先查 `ss -ltn`，不要假設 `3000` 永遠空著。
- **把 GPU 當成必要條件**：New API、PostgreSQL、Redis 不需要 NVIDIA runtime 或 GPU reservation。
- **只看 `docker compose ps`**：容器 running 不等於 API ready，必須實測 `/api/status`。

## Verification

部署或修改完成後，回報以下證據：

1. SSH 目標 hostname 與 Docker／Compose 版本。
2. 實際 Compose 目錄與 project name。
3. `docker compose config` 成功。
4. 三個服務的實際狀態與 health。
5. `/api/status` 的 HTTP 200／`success:true`。
6. 實際版本與 Web URL。
7. 是否遇到資料庫初始化延遲、port 衝突、驗證失敗或其他限制。

不要回報秘密值；只回報「已產生並以 `.env` 權限 600 保存」。
