---
name: videoseek-remote-docker
description: 在遠端 GPU 主機 hch@10.145.119.19（gigabyte）用 Docker 部署、啟動、更新、除錯與驗證 6v17/VideoSeek。當使用者提到 VideoSeek、6v17/VideoSeek、遠端影片語意搜尋、noVNC VideoSeek、VideoSeek 亂碼／中文字變方框、OpenAI CLIP、8765 Agent API，或要求把 VideoSeek 裝進 Docker 時使用。固定使用 /home/hch/videoseek、容器 videoseek、GPU Docker、noVNC :6080 與 Agent API :8765；不要把這個桌面 Qt 應用誤當成純 FastAPI 服務，也不要停止同主機既有的 llama.cpp、Frigate 或其他 GPU 容器。 #docker #frontend-ui #gpu #media #remote #setup-install #windows
compatibility: 需要可免密碼 SSH 登入 hch@10.145.119.19、遠端 Docker Compose、NVIDIA Container Toolkit/GPU runtime；本機只透過 SSH 操作，不需要本機 Docker Desktop。
---

# VideoSeek Remote Docker

## 已驗證部署基線

| 項目 | 值 |
|---|---|
| Repository | `https://github.com/6v17/VideoSeek` |
| 遠端主機 | `hch@10.145.119.19`，hostname `gigabyte` |
| 專案目錄 | `/home/hch/videoseek` |
| Compose service / container | `videoseek` |
| 目前執行模式 | Ubuntu 24.04 + NVIDIA CUDA runtime + Qt/PySide6 + Xvfb + x11vnc + noVNC |
| 目前驗證版本 | VideoSeek `v1.0.90`，commit 以遠端實際 checkout 為準 |
| noVNC | `http://10.145.119.19:6080/vnc.html` |
| Agent API | `http://10.145.119.19:8765/api/v1/health` |
| 影片掛載 | `/home/hch/videoseek/videos` → `/data/videos` |
| 索引資料 | `/home/hch/videoseek/data` → `/data/app` |
| 模型資料 | `/home/hch/videoseek/models` → `/data/models` |

這是 Linux 容器內的**桌面 GUI**，不是官方提供的 headless VideoSeek image。使用 noVNC 看 Qt 畫面；內建 Agent API 由 Qt 應用啟動，不能只啟動一個 `uvicorn` 就取代整個應用。

## When to Use

- 使用者要求在 `hch@10.145.119.19` 安裝或重部署 VideoSeek。
- VideoSeek noVNC 開得出來但中文顯示方框／亂碼。
- 要下載 OpenAI CLIP 模型、放影片、啟動索引或檢查 GPU。
- 要檢查 `videoseek` 容器、8765 API、6080 noVNC、Docker build 或啟動日誌。
- 要更新 GitHub source 後重新 build；更新前先保留 `data/`、`models/`、`videos/`、`home/`、`work/`。

## 不要做的事

- 不要停止或重啟 `llama-cpp-qwen38-27b-abliterated`、`frigate` 或其他既有 GPU 容器；先查看 GPU 與 port。
- 不要刪除 `/home/hch/videoseek/data`、`models`、`videos`、`home`，除非使用者明確要求清除資料。
- 不要把模型下載到 image layer；模型必須放在 host 的 `models/` bind mount。
- 不要只設定 `LANG` 就宣稱修好了中文；Qt stylesheet 若仍指定 `Segoe UI`／`Microsoft YaHei UI`，Linux 仍可能顯示 tofu 方框。
- 不要把 `127.0.0.1:8765` 當成遠端可連線地址；容器內要用 `VIDEOSEEK_AGENT_API_HOST=0.0.0.0`，並由 Compose 發佈 8765。
- 不要向回覆輸出 SSH 密碼、API key 或 cookie。

## 部署流程

### 1. 先檢查現場

```bash
ssh -o BatchMode=yes hch@10.145.119.19 \
  "hostname; docker version --format '{{.Server.Version}}'; \
   docker ps --format '{{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}'; \
   nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv,noheader; \
   df -h / /home"
```

確認：

- Docker server 可用、Linux x86_64。
- NVIDIA runtime 存在：`docker run --rm --gpus all nvidia/cuda:12.8.1-base-ubuntu24.04 nvidia-smi`。
- 6080、8765 沒被其他服務使用。
- `/home` 有足夠空間；OpenAI CLIP zip 約 376 MB，GPU image 與 Python wheels 另需數 GB。

### 2. 準備專案

目前採 source checkout + 自訂 Docker wrapper。更新 source 時保留使用者資料目錄：

```bash
ssh hch@10.145.119.19
mkdir -p /home/hch/videoseek
cd /home/hch/videoseek
mkdir -p data models videos home work downloads
```

把下列檔案放在專案根目錄：

- `Dockerfile`
- `docker-compose.yml`
- `entrypoint.sh`
- `VideoSeek/`：GitHub source checkout

若從本機傳送，使用 tar pipe，避免把 Windows 路徑寫進 Linux Compose：

```bash
tar -C /path/to/stage -czf - . | \
  ssh hch@10.145.119.19 'tar -xzf - -C /home/hch/videoseek'
```

### 3. Docker 必備設定

Dockerfile 應具備：

- base image：`nvidia/cuda:12.8.1-runtime-ubuntu24.04`
- Python 3.12、PySide6、VideoSeek `requirements.txt`
- Linux 使用 `onnxruntime-gpu==1.23.0`，不要留 CPU-only `onnxruntime`
- `ffmpeg`、Xvfb、openbox、x11vnc、novnc、websockify
- 中文字型：`fonts-noto-cjk`、`fonts-noto-cjk-extra`、`fonts-wqy-zenhei`、`locales`
- Qt XCB dependencies：`libxcb-cursor0`、`libxkbcommon-x11-0` 等
- 建立 UID 1000 的 `appuser`；NVIDIA CUDA image 可能已經有 UID 1000 的 `ubuntu`，先 `userdel ubuntu`／`groupdel ubuntu` 再建立。
- 用 `fc-cache -f -v` 更新字型。

不要在只安裝 `ubuntu` image 的版本中直接 `useradd --uid 1000`，會遇到 `UID 1000 is not unique`。

Compose 應包含：

```yaml
services:
  videoseek:
    build: .
    image: videoseek:latest
    container_name: videoseek
    restart: unless-stopped
    shm_size: 1g
    environment:
      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: compute,utility,video
      QT_QPA_PLATFORM: xcb
    ports:
      - "6080:6080"
      - "8765:8765"
    devices:
      - /dev/dri:/dev/dri
    gpus: all
    volumes:
      - ./data:/data/app
      - ./models:/data/models
      - ./videos:/data/videos
      - ./home:/data/home
      - ./work:/data/work
```

### 4. entrypoint 的關鍵順序

`entrypoint.sh` 必須：

1. 設定 `HOME=/data/home`、`XDG_RUNTIME_DIR=/tmp/runtime-app`。
2. 設定 `LANG=zh_TW.UTF-8`、`LANGUAGE=zh_TW:zh`、`LC_ALL=zh_TW.UTF-8`。
3. 寫入 `/data/home/.videoseek/config.json`，至少設定：
   - `data_root=/data/app`
   - `model_dir=/data/models`
   - `agent_api_enabled=true`
   - `prefer_gpu=true`
4. 啟動 `Xvfb :99`。
5. **等待 `/tmp/.X11-unix/X99` 出現後**才啟動 openbox、x11vnc 與 VideoSeek；否則 appuser 啟動太快會得到 `could not connect to display :99`，容器以 exit 139／restart loop 結束。
6. `websockify --web=/usr/share/novnc 6080 localhost:5900`。
7. export `VIDEOSEEK_AGENT_API_HOST=0.0.0.0`。
8. 最後用 `exec python3 /opt/videoseek/main.py`。

等待 X socket 的簡單模式：

```bash
Xvfb :99 -screen 0 1600x1000x24 -ac +extension GLX +render -noreset >/tmp/xvfb.log 2>&1 &
for _ in $(seq 1 100); do
  [ -S /tmp/.X11-unix/X99 ] && break
  sleep 0.1
done
[ -S /tmp/.X11-unix/X99 ] || { cat /tmp/xvfb.log >&2; exit 1; }
```

### 5. 修正 Linux 中文字型

VideoSeek 原始碼在 Windows 偏好 `Microsoft YaHei UI`，stylesheet 預設 `Segoe UI`。Linux 容器即使已安裝 Noto CJK，若不改 Qt font family，仍可能顯示方框。

`main.py` 建立 QApplication 後應按平台設定：

```python
font = app.font()
if sys.platform == "win32":
    font.setFamily("Microsoft YaHei UI")
else:
    font.setFamily("Noto Sans CJK TC")
app.setFont(font)
```

`ui/widgets/styles.py` 的全域 stylesheet 應把第一順位改為：

```css
font-family: "Noto Sans CJK TC", "Noto Sans CJK SC", "WenQuanYi Zen Hei", "Segoe UI", "Microsoft YaHei UI", sans-serif;
```

若發生亂碼，優先依序檢查：

```bash
docker exec videoseek locale
docker exec videoseek fc-match "Noto Sans CJK TC"
docker logs --tail 100 videoseek
```

不要只看 `fc-list`；要確認 Qt app 實際使用的 global font/stylesheet。

### 6. Build、啟動、模型

把長 build log 留在遠端檔案，不要把數千行安裝輸出塞回使用者對話：

```bash
cd /home/hch/videoseek
docker compose config >/dev/null
docker compose build >/tmp/videoseek-build.log 2>&1
docker compose up -d >/tmp/videoseek-up.log 2>&1
```

第一次 build 會安裝 PySide6、OpenCV、LanceDB、ONNX Runtime GPU 等，實際時間受網路影響；開始前告知使用者預估需要幾分鐘，完成後只回報摘要與錯誤尾端。

下載預設 OpenAI CLIP：

```bash
cd /home/hch/videoseek/downloads
curl -L --fail --retry 2 -o openai-clip.zip \
  https://github.com/6v17/VideoSeek/releases/download/models/openai-clip.zip
rm -rf /tmp/vs-model-extract
mkdir -p /tmp/vs-model-extract
unzip -q -o openai-clip.zip -d /tmp/vs-model-extract
rm -rf /home/hch/videoseek/models/openai-clip
cp -a /tmp/vs-model-extract/openai-clip /home/hch/videoseek/models/
```

預期模型目錄：

```text
models/openai-clip/vit-base-patch32/
  clip_visual.onnx
  clip_text.onnx
  bpe_simple_vocab_16e6.txt.gz
  model_manifest.json
```

其他模型不可擅自平行下載；需使用者指定模型與確認磁碟／GPU 預算。

## 驗證

完成後執行：

```bash
cd /home/hch/videoseek
docker ps --filter name=videoseek \
  --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
curl -fsS http://127.0.0.1:8765/api/v1/health?mode=ping
curl -fsSI http://127.0.0.1:6080/vnc.html | head

docker exec videoseek /opt/venv/bin/python -c \
  'import onnxruntime as o; print(o.__version__); print(o.get_available_providers())'
docker exec videoseek nvidia-smi --query-gpu=name --format=csv,noheader
```

通過條件：

- container `videoseek` 是 `Up`，不是 `Restarting`。
- 8765 health 回 `ok: true`、`service: videoseek-agent-api`。
- 6080 `/vnc.html` 回 HTTP 200。
- ONNX providers 至少有 `CUDAExecutionProvider`、`CPUExecutionProvider`；若有 TensorRT 也正常。
- `nvidia-smi` 在容器內看得到 GPU。
- noVNC 畫面中文不是 tofu 方框。
- `/data/models/openai-clip/vit-base-patch32` 的四個檔案存在。

完整 health 可看索引是否準備好：

```bash
curl -fsS http://127.0.0.1:8765/api/v1/health
```

新安裝 `index_ready=false`、`video_count=0` 是正常的；要在 GUI 新增 `/data/videos` 為影片庫並同步後，才會變成可搜尋。

## 日常操作

```bash
ssh hch@10.145.119.19 'cd /home/hch/videoseek && docker compose up -d'
ssh hch@10.145.119.19 'cd /home/hch/videoseek && docker compose restart'
ssh hch@10.145.119.19 'docker logs --tail 100 videoseek'
ssh hch@10.145.119.19 'cd /home/hch/videoseek && docker compose down'
```

更新 source 時只重建 `videoseek`，不要刪 bind mounts：

```bash
cd /home/hch/videoseek
docker compose build
docker compose up -d --force-recreate
```

## 常見故障

### 中文變成方框／亂碼

症狀：畫面文字全是空方框，但英文正常。

處理：確認 image 有 Noto CJK + locales、`main.py` 有 Linux global font、`styles.py` 沒把 Noto 排在 Windows 字型後面；重建 image 並 `docker compose up -d`。瀏覽器端 noVNC 舊畫面再做 `Ctrl+F5`。

### `could not connect to display :99`

Xvfb 與 Qt 啟動競速。加入 X socket wait loop；不要用無限 restart 掩蓋問題。查看：

```bash
docker logs --tail 120 videoseek
```

### `UID 1000 is not unique`

CUDA Ubuntu image 已有 `ubuntu` UID 1000。刪除該 image user/group 後再建 `appuser`，或改用現有 UID 1000 user；不要讓 bind mount 寫入權限失控。

### API 有回應但 GPU 沒使用

確認三層：

1. Compose 有 `gpus: all`。
2. 容器內 `nvidia-smi` 可用。
3. Python 是 `onnxruntime-gpu` 且 `CUDAExecutionProvider` 存在。

不要只看主機 `nvidia-smi`，那不能證明 VideoSeek 的 ONNX inference 使用 GPU。

### 容器反覆重啟／退出 139

先停用 restart 方便診斷：

```bash
docker update --restart=no videoseek
docker stop videoseek || true
docker compose up
```

檢查 Xvfb socket、Qt XCB libraries、shm size；修好後再恢復 `restart: unless-stopped`。

## Inputs and Outputs

- Input：GitHub source、可用的遠端 SSH、Docker/NVIDIA runtime、影片目錄。
- Output：`videoseek` 容器、noVNC GUI :6080、Agent API :8765、持久化 data/models/videos/home/work bind mounts。
- 回報必須包含：實際容器狀態、驗證 URL、GPU/ONNX provider 結果、模型是否已就緒、是否仍需使用者同步影片庫。

## Rules and Limitations

- 版本、模型檔案、現場 port 與 GPU 使用率以遠端即時檢查為準；本文件的 v1.0.90 是已驗證基線，不是永遠固定的最新版本。
- 不要把任何密碼、token、cookie 寫進 skill；使用 SSH key 或當次 session 的既有認證。
- 不要聲稱「已完成搜尋」；部署完成只代表服務可用，影片仍需建立索引。
- 若 build 失敗，回報 `/tmp/videoseek-build.log` 最後錯誤段落，不要假裝完成。
