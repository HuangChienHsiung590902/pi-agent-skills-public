---
name: blender-unreal-linux-docker
description: >-
  在遠端 GPU 主機 gigabyte（hch@10.145.119.19）用 Docker 安裝、啟動、停止、驗證
  Linux Blender GUI（官方 5.2.1、noVNC 6081）與 Unreal Engine Linux Editor。
  當使用者說遠端 Docker 裝 Blender／Unreal、blender.org/download、UE Linux
  Development Quickstart、blender-docker、http://10.145.119.19:6081/vnc.html、
  或要把 Unreal 裝進 19 的容器時使用。不是本機 Windows Blender 5.2／UE 5.8
  （用 sam3d-body-rigify-unreal-pipeline），也不是 llm-wiki 的 6080 noVNC。
---

# blender-unreal-linux-docker

在 `hch@10.145.119.19`（hostname `gigabyte`）用 Docker 跑官方 Linux Blender，
以及（僅在有 Epic GitHub 授權時）規劃中的 Unreal Engine Linux Editor。

## When to Use

- 使用者要在 `10.145.119.19` 的 Docker 安裝 Blender 或 Unreal Engine
- 貼 `https://www.blender.org/download/` 或 Unreal Linux Development Quickstart
- 要開／修 `http://10.145.119.19:6081/vnc.html`、容器 `blender`、目錄 `/home/hch/blender-docker`
- 問遠端 Linux Unreal 怎麼裝、缺不缺 GitHub token

不要用本 skill：

- 本機 Windows Blender 5.2／UE 5.8、Rigify、FBX、IK Retargeter → `sam3d-body-rigify-unreal-pipeline`
- ComfyUI Hunyuan3D 角色 mesh／Paint → `comfyui-blender-character-model`
- llm-wiki noVNC `:6080` → `llm-wiki-novnc`
- 本機 UE 編輯器繁中 → `ue-editor-zh-hant-localization`
- 本機正在跑的 UE Editor MCP／CmdLink → `ue-cmdlink-mcp`

## Inputs and Outputs

**Input**

- SSH：`hch@10.145.119.19`（BatchMode 公鑰）
- Blender：已存在的遠端 compose `/home/hch/blender-docker/`
- Unreal：Epic 已綁定 GitHub 的 PAT（Classic 至少 `repo`），能開 `EpicGames/UnrealEngine`；沒有 token 就停止

**Output**

- Blender：容器 `blender` Up，`blender --version` 為 5.2.1 LTS，`http://10.145.119.19:6081/vnc.html` HTTP 200
- Unreal：僅在 token 齊全且使用者 `y` 後才 clone／編譯；成功時 GUI 預留 `http://10.145.119.19:6082/vnc.html`

## 現況事實（2026-09-12 實測）

| 項目 | 值 |
|---|---|
| SSH | `hch@10.145.119.19` |
| Blender 目錄 | `/home/hch/blender-docker/` |
| image / 容器 | `blender:5.2.1` / `blender` |
| 版本 | 官方 Blender 5.2.1 LTS Linux x64 |
| 壓縮包 | `blender-5.2.1-linux-x64.tar.xz`（約 366MB） |
| SHA256 | `a31f524fa99a527d3d52b7f5aaa68c34e1a19d5a1c9473f79c5cc610fd5b10e9` |
| noVNC | `http://10.145.119.19:6081/vnc.html`（無密碼；容器內 6080） |
| restart | `"no"`（主機重開不會自動起來） |
| GPU | **未掛**。`LIBGL_ALWAYS_SOFTWARE=1`，Cycles GPU 不可用 |
| Unreal | **尚未安裝**。遠端無 `gh`、無 git credentials |

埠分配：`6080` = llm-wiki，`6081` = Blender，Unreal GUI 預留 **6082**。

系統碟 `/var/lib/docker` 只剩約 260G；`/home` 約 6.7T。大檔與 UE 原始碼必須放 `/home` bind mount，不可把 UE 寫進映像層。

## Procedure

相對本 skill 目錄的腳本：

```text
scripts/blender_unreal_docker.py
```

```bash
python scripts/blender_unreal_docker.py status
python scripts/blender_unreal_docker.py start
python scripts/blender_unreal_docker.py stop
python scripts/blender_unreal_docker.py verify
```

預設 SSH `hch@10.145.119.19`。需要時加 `--host user@ip`。

### Blender：日常啟停

1. 先 `status`，不要先 build。
2. 容器已在跑且 `:6081/vnc.html` 回 200 → 完成，不要重建。
3. image 在、容器沒跑 → `start`（`docker compose up -d`），再 `verify`。
4. 要停用 `stop`（`docker compose stop`），不要 `down -v`。

手動等價：

```bash
ssh hch@10.145.119.19 'cd /home/hch/blender-docker && docker compose up -d'
ssh hch@10.145.119.19 'docker exec blender blender --version'
ssh hch@10.145.119.19 'docker stop blender'
```

瀏覽器：`http://10.145.119.19:6081/vnc.html?autoconnect=1&resize=scale`

### Blender：重建映像（須先出執行摘要等 y）

1. 主機用 `aria2c` 把官方 tar.xz 抓到 `/home/hch/blender-docker/`，核對 SHA256。
2. Dockerfile **COPY** 壓縮包再 `tar`，不要在 `docker build` 裡 wget（實測極慢會 timeout）。
3. compose：`container_name: blender`、`restart: "no"`、`6081:6080`、不設 `gpus` / `runtime: nvidia`。
4. 不停 `comfyui-h3`、`llama-cpp-spark-x25-4b`、`llm-wiki`。
5. 兩張 GPU 都被佔時維持無 GPU；要掛 GPU1 必須再問一次 y（可能把 ComfyUI OOM）。

### Unreal Engine Linux（尚未安裝）

沒有 Epic GitHub PAT **禁止開始**。禁止拉 Docker Hub 非官方 UE 映像。

授權且使用者 `y` 之後才：

1. 資料全部放 `/home/hch/unreal-engine/` bind mount。
2. 容器名預設 `unreal-editor`，`restart: no`，GPU1（先確認 nvidia-smi），noVNC **6082**。
3. Token 只寫權限 `600` 的檔，做一次 `git clone`，不寫進 Dockerfile／compose／本 skill。
4. 官方流程：`Setup.sh` → `GenerateProjectFiles.sh` → `make UnrealEditor`（限制 `-j`，避免打滿 54GB RAM）。
5. 預期 6–12 小時、80–150GB+；16GB VRAM 跑 Editor 很緊。clone 失敗就停。

Windows 的 `C:\Program Files\Epic Games\UE_5.8` 是 Win64，不能搬去 Linux。

## Rules and Limitations

- GPU 容器預設 `restart: no`，不可改成隨開機自動跑。
- 不停止現有 GPU 服務，除非使用者明確授權。
- Blender 官方來源：`https://download.blender.org/release/Blender5.2/`。
- Unreal 唯一合法來源：已授權的 `https://github.com/EpicGames/UnrealEngine`。
- Token、密碼不寫進 skill、git、compose、Dockerfile。
- 不佔 `6080`（llm-wiki）。Blender 固定 `6081`，Unreal 預留 `6082`。
- 不 prune、不刪其他映像或 volume。
- 本 skill 不管本機 Blender MCP／UE MCP。

## Pitfalls

- Docker build 內 wget blender.org 可能只有 100–300KB/s，SSH 900 秒會 timeout；改主機 `aria2c` + COPY。
- `${VAR}` 經雙引號 SSH 傳 Dockerfile 會被本機／遠端 shell 吃掉；版本字串寫死或用 stdin heredoc。
- `comfyui-h3` 常設 `NVIDIA_VISIBLE_DEVICES=0,1`。GPU1「看起來空」可能幾分鐘後被佔滿。
- 系統碟 Docker 根目錄裝不下 UE；只把小映像放 overlay，大資料放 `/home`。
- noVNC 無密碼，只走 ZeroTier／內網。
- `restart: no`：重開機後要再 `start`。
- Openbox 缺 debian-menu.xml 的訊息可忽略。
- 關閉 Blender 視窗時 entrypoint 會再拉起；要停服務用 `docker stop blender`。

## Verification

Blender：

1. `docker ps` 看到 `blender` Up，埠 `6081->6080`，`restart=no`，無 DeviceRequests。
2. `docker exec blender blender --version` 含 `Blender 5.2.1 LTS`。
3. 本機與遠端 `curl` `http://10.145.119.19:6081/vnc.html` → HTTP 200，body 含 noVNC。
4. `docker exec blender ps aux` 含 `Xvfb`、`openbox`、`x11vnc`、`websockify`、`blender`。
5. `nvidia-smi` 沒有 blender 行程（目前無 GPU）。

`python scripts/blender_unreal_docker.py verify` 做 1–3。

Unreal：容器不存在、`/home/hch/unreal-engine` 未建立，視為尚未安裝，不是驗證失敗。

## 相關 skill

| 主題 | skill |
|---|---|
| llm-wiki `:6080` noVNC | `llm-wiki-novnc` |
| 本機 Blender／UE 5.8 角色管線 | `sam3d-body-rigify-unreal-pipeline` |
| 遠端 ComfyUI 雙 GPU | `minimax-h3-comfyui` |
