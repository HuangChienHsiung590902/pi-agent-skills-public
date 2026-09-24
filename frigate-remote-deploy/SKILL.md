---
name: frigate-remote-deploy
description: 在遠端 GPU 主機 10.145.119.19（hch@gigabyte）以 Docker Compose 安裝、設定、更新、除錯與驗證 Frigate NVR；適用於 Frigate、IP Camera、RTSP、錄影、物件偵測、NVIDIA 硬體解碼、H.264/H.265、Frigate Web UI、Home Assistant/MQTT 等任務。需要先檢查遠端既有 Docker 服務與 GPU 使用狀況，避免搶占正在執行的模型服務；除非使用者明確授權，不要停止或重啟其他容器、不要刪除 Docker 資料。
compatibility: Ubuntu/Debian Docker host；本機可用 SSH 連線 hch@10.145.119.19；Frigate 官方 amd64 Docker image；若使用 NVIDIA GPU，遠端需有 NVIDIA Container Toolkit。
---

# Frigate 遠端 Docker 部署

## When to Use

使用者要求在 `10.145.119.19` 安裝、啟動、更新或維護 Frigate，或提到以下任一項時載入本 Skill：

- Frigate / Frigate NVR
- IP Camera、RTSP、監視器錄影
- Frigate 的硬體解碼、NVIDIA GPU、物件偵測
- Frigate Web UI、錄影儲存、攝影機設定
- 將 Frigate 接到 Home Assistant、MQTT 或其他本地自動化系統

本 Skill 管理 Frigate 應用層；遠端 Docker context、SSH、跨主機備份與共用 GPU 資源管理仍參考 `docker-remote-control`。

## Inputs and Outputs

### 必要輸入

- 遠端主機：預設 `hch@10.145.119.19`
- 動作：安裝、啟動、更新、設定、檢查或移除
- 若要接攝影機：每支攝影機的 RTSP URL、編碼格式、解析度、FPS、用途（detect/record）

### 可選輸入

- 指定 NVIDIA GPU 編號
- 是否使用 NVIDIA 硬體解碼與 GPU 物件偵測
- 錄影保留天數與儲存位置
- MQTT broker 位址、帳號與密碼
- Home Assistant 整合需求
- 是否需要 H.264/H.265、WebRTC、RTSP restream

### 輸出

回報：

1. 實際修改的遠端檔案與路徑。
2. Compose project、容器名稱、image、port 與健康狀態。
3. Web UI URL、是否為自簽 HTTPS。
4. 攝影機與 detector 是否已實際驗證。
5. GPU／硬體解碼是否啟用，以及仍待使用者提供的資料。
6. 不要在回覆或 Skill 中保存密碼、token、RTSP 密碼或 API key。

## Known Deployment Facts

截至最近一次驗證，目標主機具備：

- hostname `gigabyte`
- Ubuntu 24.04 LTS、amd64
- Docker Engine 29.x、Docker Compose v2/v5 CLI
- `/home` 為大型獨立磁碟，適合作為 Frigate 媒體儲存位置
- 兩張 NVIDIA GeForce RTX 4060 Ti 16GB；GPU 可能被既有 llama.cpp／其他模型容器使用
- Frigate 專案預設位置：`/home/hch/frigate`
- Frigate 預設 Web UI：`https://10.145.119.19:8971/`
- 預設設定檔：`/home/hch/frigate/config/config.yml`
- 預設錄影資料：`/home/hch/frigate/media`

上述硬體與服務狀態可能變動；每次操作以現場檢查結果為準。

## Procedure

### 1. 查詢技能與官方文件

需要更新或大幅改配置時，先看官方文件：

- 安裝：https://docs.frigate.video/frigate/installation/
- 新安裝規劃：https://docs.frigate.video/frigate/planning_setup
- 物件偵測器：https://docs.frigate.video/configuration/object_detectors
- 影片硬體解碼：https://docs.frigate.video/configuration/hardware_acceleration_video
- 攝影機設定：https://docs.frigate.video/configuration/cameras

中文文件可作輔助，但官方英文文件與遠端現場狀態優先。

### 2. 先做遠端 preflight

以 SSH 檢查，不要先 `docker compose up`：

```bash
ssh hch@10.145.119.19 '\
  hostname; uname -a; cat /etc/os-release | head -8; \
  docker version --format "server={{.Server.Version}} client={{.Client.Version}}"; \
  docker compose version; \
  docker ps --format "table {{.Names}}\\t{{.Image}}\\t{{.Status}}\\t{{.Ports}}"; \
  df -h / /home; free -h; \
  nvidia-smi --query-gpu=index,name,memory.total,memory.used --format=csv,noheader 2>/dev/null || true; \
  ss -lntup | grep -E ":(8971|8554|8555)\\b" || true'
```

確認：

- Docker 正常。
- `/home` 有足夠錄影空間。
- `8971`、`8554`、`8555` 沒有被其他服務占用。
- 既有容器和 GPU 使用者不可被無意間停止。
- 若 Frigate 已存在，先備份或讀取現有 Compose/config，不要覆蓋使用者設定。

### 3. 建立最小可啟動專案

首次安裝時使用 `/home/hch/frigate`，目錄內容至少包含：

```text
/home/hch/frigate/
├── docker-compose.yml
├── config/config.yml
└── media/
```

初始 Compose 可採以下原則：

```yaml
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    restart: unless-stopped
    stop_grace_period: 30s
    shm_size: "512mb"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /home/hch/frigate/config:/config
      - /home/hch/frigate/media:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "8971:8971"
```

`shm_size` 要按 detect stream 解析度與攝影機數量調整。高解析度或多路攝影機不可盲目使用 128MB；若出現 `Bus error`，先重新計算 SHM，再檢查 PID/file limits。

初次啟動可用空攝影機設定：

```yaml
mqtt:
  enabled: false

detectors:
  cpu:
    type: cpu

cameras: {}
```

這只用於先驗證容器與 Web UI，不代表正式部署；Frigate log 會警告 CPU detector 不適合正式使用。

### 4. 拉取 image 與啟動

先驗證 Compose，再拉取與啟動：

```bash
cd /home/hch/frigate
docker compose config
docker pull --platform linux/amd64 ghcr.io/blakeblackshear/frigate:stable
docker compose up -d
docker compose ps
```

遠端 registry 慢或 SSH 容易中斷時，可在遠端使用 `nohup` 背景執行，寫入 `/home/hch/frigate/install.log`，之後讀 log 驗證；不要重複啟動多個 pull。若 pull 長時間只有 `Pulling fs layer`，先檢查單一 pull process、Docker daemon log 與 GHCR token/網路，再決定是否停止該次 pull。

### 5. 驗證 Web UI 與初始帳號

Frigate 新版 Web UI 走 HTTPS。從操作端驗證：

```bash
curl -k -I --max-time 10 https://10.145.119.19:8971/
ssh hch@10.145.119.19 'docker compose -f /home/hch/frigate/docker-compose.yml ps; docker compose -f /home/hch/frigate/docker-compose.yml logs --tail=100'
```

預期：

- container 為 `healthy`。
- `https://10.145.119.19:8971/` 回 HTTP 200 或可載入頁面。
- HTTP 直連 8971 可能得到 `plain HTTP request was sent to HTTPS port`，這不是 Frigate 未啟動；改用 HTTPS。
- 初次啟動的 admin 密碼只從當次 container log 取得並安全交給使用者；不要把它寫進 Skill、Git、shell history 或長期文件。使用者登入後應立即改密碼。

### 6. 加入攝影機前先確認串流

每支攝影機優先分成：

- high-resolution main stream：錄影。
- low-resolution sub stream：detect。

加入設定前驗證：

```bash
ffprobe -hide_banner -rtsp_transport tcp '<RTSP_URL>'
```

不要把含帳密的 RTSP URL 寫入回覆、log、Skill 或 Git。若 URL 必須在遠端檔案中保存，設定檔權限至少限制為服務帳號可讀，並避免把該檔案提交到版本庫。

### 7. 設定硬體解碼與物件偵測

NVIDIA 主機的 Frigate Compose 需要 NVIDIA Container Toolkit，並依官方方式加入 GPU reservation；不要因為主機有 GPU 就自動搶其中一張卡。

概念配置：

```yaml
services:
  frigate:
    image: ghcr.io/blakeblackshear/frigate:stable-tensorrt
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ["<未被占用的GPU編號>"]
              count: 1
              capabilities: [gpu]
```

設定 NVIDIA 影片解碼：

```yaml
ffmpeg:
  hwaccel_args: preset-nvidia
```

物件 detector 應依官方支援矩陣與實際負載選擇。不要預設把同一張 GPU 同時分給既有 LLM、Frigate detector、轉碼與其他高負載服務；先檢查 `nvidia-smi`，需要調度 GPU 時先告知使用者影響。

驗證硬體解碼：

```bash
ssh hch@10.145.119.19 'docker logs frigate 2>&1 | grep -Ei "hwaccel|ffmpeg|decode|error|detector" | tail -100; nvidia-smi'
```

只看到 GPU 有顯存使用不代表影片解碼成功；需同時確認 Frigate log 無解碼錯誤，並在有攝影機串流時看到 FFmpeg／NVDEC 相關活動。

### 8. 設定儲存與保留策略

- 設定與 SQLite database 放 `/home/hch/frigate/config`。
- 錄影與 snapshots 放 `/home/hch/frigate/media`。
- 長期 24/7 錄影前先估算硬碟容量。
- 監控資料最好使用獨立硬碟或陣列，不要和私人資料共用高寫入陣列。
- NAS/NFS/SMB 只有在網路與 NAS 能承受持續寫入時使用；本地儲存優先。
- 不要用 `docker volume prune` 或 `docker system prune --volumes` 來清理 Frigate 資料。

### 9. 更新與維護

更新前先檢查設定與備份：

```bash
cd /home/hch/frigate
docker compose config
tar -C /home/hch/frigate -czf /home/hch/frigate-config-$(date +%Y%m%d-%H%M%S).tar.gz config docker-compose.yml
# 確認備份完成後再執行：
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

備份檔若含 RTSP 密碼或其他敏感資料，需限制權限並避免放進公開位置。

## Rules and Limitations

- 只對使用者授權的 `10.145.119.19` 執行操作。
- 不要記錄或回報密碼、token、RTSP credentials、MQTT secrets。
- 不要停止現有 `llama-cpp`、ComfyUI、AI Learning Studio 或其他容器來替 Frigate 讓路，除非使用者明確授權。
- 不要預設 Frigate 可以使用任一張 GPU；先查看 GPU process、顯存與既有服務。
- 不要把 LongCat-2.0 或一般文字 LLM 當作 Frigate 的即時物件 detector。
- 不要把 Frigate 的 CPU detector 視為正式生產配置。
- 不要把 HTTP 8971 當成有效驗證方式；新版 UI 應用 HTTPS。
- 不要直接覆蓋已存在的 `/home/hch/frigate/config/config.yml`。
- 不要在未收到攝影機 URL、格式與保留需求前，自行編造 camera 設定。
- 不要把 Frigate 暴露到公網；若需要遠端存取，使用 VPN、受保護的反向代理與強密碼。
- 監控與 AI 辨識結果可能誤報、漏報，不可當成唯一門禁、保全或法律判定依據。

## Pitfalls

- `curl http://127.0.0.1:8971` 回 `plain HTTP request was sent to HTTPS port`：改用 `curl -k https://127.0.0.1:8971/`。
- container `healthy` 但沒有攝影機：這只代表 Frigate 服務正常，尚未完成 camera 設定。
- CPU 使用率很高：檢查是否沒有硬體解碼、detect stream 太高、FPS 太高，或 detector 配置不當。
- `Bus error`：檢查 `shm_size`、detect resolution、攝影機數量與 PID limit。
- FFmpeg decode error：確認攝影機編碼、GPU driver、Frigate image variant 與 `hwaccel_args` 相符；不要期待硬體失敗會自動安全 fallback。
- GPU OOM 或既有 LLM 變慢：檢查 GPU 使用者與顯存，不要只重啟 Frigate；先撤回 GPU reservation 或重新分配負載。
- RTSP 不通：先從遠端主機用 `ffprobe` 測試，因為攝影機可能只允許同網段存取。
- 影像能播放但 detector 沒結果：確認 detect stream、`detect.width/height/fps`、model/detector 與 logs。
- GHCR pull 卡住：確認只有一個 pull process，檢查 Docker journal、registry token、DNS、磁碟與網路；不要同時開多個 pull。
- Frigate log 出現自動產生的 admin 密碼：只在本次回覆安全提供給使用者，登入後立刻修改，不要寫回 Skill。

## Verification

完成安裝或修改後至少執行：

```bash
ssh hch@10.145.119.19 '\
  docker compose -f /home/hch/frigate/docker-compose.yml config >/dev/null && \
  docker compose -f /home/hch/frigate/docker-compose.yml ps && \
  docker inspect --format "{{.State.Status}} health={{if .State.Health}}{{.State.Health.Status}}{{end}}" frigate && \
  curl -k -fsS --max-time 10 https://127.0.0.1:8971/ >/dev/null && \
  echo FRIGATE_WEB_OK'
```

驗證結果要分層回報：

1. **部署層**：image、container、Compose、health、port。
2. **Web 層**：HTTPS UI 能否載入、是否需要登入。
3. **串流層**：每支 RTSP 是否能從遠端主機讀取。
4. **解碼層**：硬體解碼 preset 是否成功且 logs 無錯誤。
5. **偵測層**：detector 是否能處理影格、是否產生正確物件事件。
6. **儲存層**：錄影與 snapshots 是否確實寫入 `/home/hch/frigate/media`。
7. **整合層**：MQTT、Home Assistant 或通知是否實際收到事件。

不能只因容器 `Up` 就宣稱 Frigate 完整可用；若沒有攝影機與事件測試，應明確標示為「基礎服務已部署，攝影機與 AI 流程尚未驗證」。

## References

- Frigate GitHub：https://github.com/blakeblackshear/frigate
- 官方安裝：https://docs.frigate.video/frigate/installation/
- 新安裝規劃：https://docs.frigate.video/frigate/planning_setup
- 物件偵測器：https://docs.frigate.video/configuration/object_detectors
- 影片硬體解碼：https://docs.frigate.video/configuration/hardware_acceleration_video
- 遠端 Docker 基礎設施：`D:\OB\skills\docker-remote-control\SKILL.md`
