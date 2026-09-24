---
name: llm-wiki-novnc
description: >-
  在遠端 GPU 主機 gigabyte（10.145.119.19 / 192.168.2.90）啟動、停止、驗證 LLM Wiki
  Docker 的 noVNC 網頁 GUI。當使用者提到 http://10.145.119.19:6080/vnc.html、
  安裝 VNC/noVNC、x11vnc、websockify、llm-wiki GUI、容器版 wiki 桌面沒開、
  或 6080 連不上時使用。這是容器內 Xvfb 虛擬桌面，不是主機 GDM/X0 實體螢幕。
  不是 pi/opencode MCP 接線（用 connect-llm-wiki），也不是小愛整組開關（用 xiaoai）。
---

# llm-wiki-novnc — 遠端 LLM Wiki 的 noVNC GUI

在 `hch@10.145.119.19`（hostname `gigabyte`，內網 `192.168.2.90`）用 Docker 跑
LLM Wiki 桌面程式：Xvfb + openbox + x11vnc + websockify/noVNC。瀏覽器開
`http://10.145.119.19:6080/vnc.html` 就是這個 GUI。

正式 compose 在遠端 `/home/hch/llm-wiki/`。`xiaoai` skill 的副本
`D:/OB/skills/xiaoai/llm-wiki/` 仍可能是舊版 `0.6.8`；**以遠端檔與實際 image tag 為準**。

## When to Use

- 使用者貼 `http://10.145.119.19:6080/vnc.html` 或說要在這台伺服器裝 VNC
- llm-wiki 網頁 GUI / noVNC 打不開、6080 沒在聽
- 要啟動或確認容器 `llm-wiki` 的虛擬桌面

不要用本 skill：

- 把 LLM Wiki 接進 pi / opencode MCP → `connect-llm-wiki`
- Windows 桌面版 LLM Wiki → `tauri-wiki-app`
- 小愛音箱整組 up/down → `xiaoai`
- 主機實體 Ubuntu 桌面（GDM/`X0`）的 VNC → 另案，且**不要佔 6080**
- Blender 容器 GUI → `/home/hch/blender-docker`，noVNC 在 **6081**

## 現況事實（2026-09-12 實測）

| 項目 | 值 |
|---|---|
| SSH | `hch@10.145.119.19`（與 `192.168.2.90` 同一台） |
| 遠端目錄 | `/home/hch/llm-wiki/` |
| image / 容器 | `llm-wiki:0.6.11` / `llm-wiki` |
| noVNC | `http://10.145.119.19:6080/vnc.html`（無密碼） |
| REST API | `http://10.145.119.19:19828`（compose 發佈 `19828:19829`） |
| health | `GET /health` → `"ok": true`、`"version": "0.6.11"` |
| token | compose env `LLM_WIKI_API_TOKEN=<KB_BEARER_TOKEN>` |
| restart | `"no"`（主機重開**不會**自動起來） |
| 虛擬顯示 | 容器內 `Xvfb :99` 1280x800，不是 host `X0` |

容器內應看到：`entrypoint.sh`、`Xvfb`、`openbox`、`x11vnc :5900`、
`websockify :6080`、`llm-wiki --no-sandbox`、`socat :19829 → :19828`。

主機另有 Debian 套件 `llm-wiki` 0.6.11（裸機）。**不要卸載**；也不要同時
跑裸機與容器，會搶 `19828`。

## Procedure

1. **先唯讀查狀態**，不要先 build。
   ```bash
   python scripts/llm_wiki_novnc.py status
   ```
   腳本路徑以本 skill 目錄為準：`D:/OB/skills/llm-wiki-novnc/scripts/llm_wiki_novnc.py`。

2. **容器已在跑且 `:6080/vnc.html` 回 200** → 工作已完成，不要重建。

3. **image 在、容器沒跑** → `start`（`docker compose up -d`）。
   ```bash
   python scripts/llm_wiki_novnc.py start
   python scripts/llm_wiki_novnc.py verify
   ```

4. **正在 `docker compose build`**（`buildx bake` 且 cwd/allow 指向 `/home/hch/llm-wiki`）
   → **等它結束，禁止再開第二個 build**。完成後再 `start`。

5. **沒有 image 才考慮 build**。先出執行摘要等 `y`：
   - 只在 `/home/hch/llm-wiki` 建 `llm-wiki:0.6.11`
   - 不卸載主機套件、不開整組 xiaoai、不改 ComfyUI
   - WebKit + `fonts-noto-cjk` 下載慢，可能 10–20+ 分鐘；頻寬被其他下載佔用時更久

6. 要停時用 `stop`（`docker compose stop`），不要 `down -v` 除非使用者明確要刪資料。

## 腳本

相對本 skill 目錄：

```text
scripts/llm_wiki_novnc.py
```

```bash
python scripts/llm_wiki_novnc.py status
python scripts/llm_wiki_novnc.py start
python scripts/llm_wiki_novnc.py stop
python scripts/llm_wiki_novnc.py verify
```

預設 SSH `hch@10.145.119.19`。需要時加 `--host user@ip`。

## Pitfalls

- **6080 是 llm-wiki noVNC**，不是整台 Ubuntu 桌面。GDM/`X0` 另有實體座。
- **blender-docker 用 `6081:6080`**。不要把 blender 改去搶 6080。
- **禁止並行 build**。先前已有一個 `compose build` 在跑時再開第二個，只會互搶、拖死 apt。
- 本機 skill 副本 `../xiaoai/llm-wiki/docker-compose.yml` 可能仍寫 `0.6.8`；遠端已是 `0.6.11`。改遠端或改副本前先對一下實際 `docker images`。
- `restart: "no"`：重開機後要再 `start`。不要擅自改成 `unless-stopped`。
- 日誌裡 `libayatana-appindicator` / `LIBDBUSMENU` / `_XSERVTransmkdir` 警告**不影響** noVNC。
- noVNC **無密碼**，只應走 ZeroTier / 內網，不要對公網曝光。
- 建專案沒有 API，只能進 noVNC 點。REST 主要是唯讀；細節見 `xiaoai`。

## Verification

1. `docker ps` 看到 `llm-wiki` Up，埠 `6080->6080`、`19828->19829`。
2. 本機與遠端都 `curl -sS -m 5 -o /dev/null -w "%{http_code}" http://10.145.119.19:6080/vnc.html` → `200`，body 含 `noVNC`。
3. `curl -sS -m 5 http://10.145.119.19:19828/health` → `"ok": true`。
4. `docker exec llm-wiki sh -c 'ps aux'` 含 `Xvfb`、`x11vnc`、`websockify`、`llm-wiki`。

`python scripts/llm_wiki_novnc.py verify` 會做 1–3。

## 相關 skill

| 主題 | skill |
|---|---|
| 小愛全棧（含 llm-wiki 知識庫用法） | `xiaoai` |
| 遠端 llm-wiki **Docker 建置／掛 BASE**（同一容器） | `llm-wiki-docker` |
| pi / opencode 接 LLM Wiki MCP | `connect-llm-wiki` |
| Windows 桌面版 | `tauri-wiki-app` |
| 遠端 Docker 通用管理 | `docker-remote-control` |

## Inputs and Outputs

### Inputs

- 遠端主機 `hch@10.145.119.19` 的 SSH、Docker Compose、埠 `6080` 與 REST API `19828` 狀態。
- `scripts/llm_wiki_novnc.py` 的 `status`、`start`、`stop`、`verify` 操作。

### Outputs

- LLM Wiki noVNC 容器狀態、HTTP/API 驗證結果與必要的修復摘要。
- 若不能修復，指出是 image、容器、虛擬桌面、埠或 API 哪一層失敗。

## Rules and Limitations

- 不要卸載主機版 LLM Wiki，不要讓裸機與容器同時搶 `19828`。
- 不要把 noVNC 無密碼服務暴露到公網；只允許內網或 ZeroTier 使用。
- 不要在已有 build 時並行啟動第二個 build；不要用 `down -v` 取代一般 stop。
- Token、compose env 與遠端設定不得寫入公開回報。
