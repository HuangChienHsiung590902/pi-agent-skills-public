---
name: llm-wiki-docker
description: >-
  在遠端 Linux 主機 10.145.119.19（hostname gigabyte）用 Docker 無頭跑 nashsu/llm_wiki
  （Tauri 桌面知識庫，官方沒有 image）。做法是 Linux .deb + Xvfb/openbox/noVNC/socat，
  API :19828、GUI :6080，並 bind-mount 現有 /home/hch/wiki/BASE。當使用者說
  「llm-wiki 裝進 docker」「遠端 docker llm-wiki」「掛 BASE 進容器」、要重建/啟動/檢查
  該主機的 llm-wiki 容器時使用。不要跟 xiaoai 整組、Windows 桌面版 tauri-wiki-app、
  connect-llm-wiki MCP、或 OMC 內建 wiki skill 搞混。
---

# llm-wiki-docker

把 **LLM Wiki**（GitHub `nashsu/llm_wiki`）跑在 `10.145.119.19` 的 Docker 裡。
官方沒有 server/Docker 版；這是用 release 的 Linux `.deb`，在 Debian 容器裡用
Xvfb 無頭開 GUI，再把 REST API 跟 noVNC 發佈出來。

## 不要跟這些搞混

| Skill | 是什麼 |
|---|---|
| `xiaoai` | 小愛整組（bridge + agent-proxy + llama.cpp + 舊版容器 wiki）。本 skill 只做 **單容器 llm-wiki**。 |
| `tauri-wiki-app` | Windows 桌面版 LLM Wiki 的 API 用法。 |
| `connect-llm-wiki` | 把桌面版接進 pi / opencode 當 MCP。 |
| `llm-wiki-novnc` | 同一容器的 noVNC 開關／驗證（`:6080/vnc.html`）。 |
| `wiki` | OMC 內建 `.omc/wiki/*.md`，跟這套無關。 |

## 現況（2026-09-12 已驗證）

| 項目 | 值 |
|---|---|
| 主機 | `hch@10.145.119.19`（與 `192.168.2.90` 同一台，hostname `gigabyte`） |
| 遠端目錄 | `/home/hch/llm-wiki/` |
| Image / 容器 | `llm-wiki:0.6.11` / `llm-wiki` |
| 套件版本 | `llm-wiki` 0.6.11（容器內 `.deb`） |
| REST API | `http://10.145.119.19:19828/health`、`/api/v1/...` |
| noVNC GUI | `http://10.145.119.19:6080/vnc.html`（無密碼） |
| 知識庫 | bind-mount `/home/hch/wiki/BASE` → 容器內同路徑 |
| 專案 id | `f974f7c9-d27f-435a-b8a3-3087bc015d8a`（名稱 `BASE`） |
| Token env | compose 的 `LLM_WIKI_API_TOKEN`（預設 `<KB_BEARER_TOKEN>`） |
| 授權 header | `Authorization: Bearer <token>` |
| restart | `"no"`（重開機不會自動起來） |

本 skill 目錄裡的 `Dockerfile`、`entrypoint.sh`、`docker-compose.yml` 就是遠端實際在用的檔。

## When to Use

- 使用者要在 `10.145.119.19` **用 Docker 跑 llm-wiki**
- 要重建、啟動、停止、看 log、查 API 是否活著
- 要把現有 `/home/hch/wiki/BASE` **掛進容器**（不要複製）
- 主機已用 `apt`/`dpkg` 裝過裸機版，但使用者明確說要 Docker 版

不要用在：查/改 wiki 頁內容（改 `tauri-wiki-app`）、接 pi MCP（改 `connect-llm-wiki`）、開小愛整組（改 `xiaoai`）。

## Procedure

相對路徑以本 skill 資料夾為準。

### 1. 部署（主機還沒有 `/home/hch/llm-wiki` 時）

1. SSH：`ssh -o BatchMode=yes hch@10.145.119.19`（免密碼金鑰）。
2. 把本 skill 的 `Dockerfile`、`entrypoint.sh`、`docker-compose.yml` 傳到 `/home/hch/llm-wiki/`。
   `entrypoint.sh` 必須是 LF、可執行。
3. `mkdir -p /home/hch/llm-wiki/data /home/hch/llm-wiki/docs`
4. `cd /home/hch/llm-wiki && docker compose build && docker compose up -d`
5. 建置會下載 Debian 套件 + GitHub `.deb`，第一次可能要 15–30 分鐘；中斷後再 `build` 會吃 cache。

`.deb` URL：

```text
https://github.com/nashsu/llm_wiki/releases/download/v0.6.11/LLM.Wiki_0.6.11_amd64.deb
```

### 2. 掛現有 BASE（不要複製）

`docker-compose.yml` 必須有：

```yaml
volumes:
  - ./data:/data
  - ./docs:/docs
  - /home/hch/wiki/BASE:/home/hch/wiki/BASE
```

`app-state.json` 實際路徑是 **XDG_DATA_HOME**，不是 config：

```text
/home/hch/llm-wiki/data/share/com.llmwiki.app/app-state.json
```

裡面的 `lastProject` / `recentProjects` / `projectRegistry` 要指向：

- id：`f974f7c9-d27f-435a-b8a3-3087bc015d8a`（來自 `BASE/.llm-wiki/project.json`）
- name：`BASE`
- path：`/home/hch/wiki/BASE`

改 compose 後要 `docker compose up -d` 重建容器，volume 才會生效。
改 `app-state.json` 後也建議重建或至少確認 `/api/v1/projects` 已切到 BASE。

容器裡若曾開過 `/home/appuser/base/base` 那種空專案，那是容器可寫層，重建就會消失；真正知識庫在 host 的 BASE。

### 3. 日常開關

```bash
ssh hch@10.145.119.19 'cd /home/hch/llm-wiki && docker compose up -d'
ssh hch@10.145.119.19 'cd /home/hch/llm-wiki && docker compose down'
ssh hch@10.145.119.19 'docker logs --tail 50 llm-wiki'
```

主機重開後不會自動啟動，要再 `up -d`。

### 4. 不要做的事

- 不要卸載主機套件 `llm-wiki` 0.6.11，除非使用者明確要求。
- 不要同時跑主機裸機 `/usr/bin/llm-wiki` 與容器（會搶 `:19828`，也會同時寫 BASE）。
- 不要把 BASE 複製進 `./data`；一律 bind-mount。
- 不要順手把 xiaoai 整組 `up`。

## Pitfalls

1. **app-state 在 `XDG_DATA_HOME`**：`/data/share/com.llmwiki.app/app-state.json`。
   寫到 `/data/config/...` 無效。本 skill 的 `entrypoint.sh` 已改對。
2. **API 預設只綁 127.0.0.1**：entrypoint 用 socat `0.0.0.0:19829 → 127.0.0.1:19828`，
   compose 發佈 `19828:19829`。即使 seed 了 `allowLanAccess: true` 也不要拿掉 socat。
3. **`shm_size: 512m`**：WebKit/Tauri 預設 64MB shm 會炸。
4. **`user: "1000:1000"`**：對應主機 `hch`，BASE 才寫得進去。
5. **沒有建專案 API**：新專案只能在 noVNC 點。現有 BASE 用掛載 + 改 app-state。
6. **健康檢查**：`GET /health` 免授權。`/chat` 仍可能要 Bearer token。
7. **Xvfb 非 root 警告** `_XSERVTransmkdir: euid != 0` 可忽略，不影響 API。
8. **dbus/appindicator 警告** 可忽略，API 仍會起來。

## Verification

遠端：

```bash
ssh hch@10.145.119.19 'docker ps --filter name=llm-wiki --format "{{.Names}} {{.Image}} {{.Status}} {{.Ports}}"'
ssh hch@10.145.119.19 'docker inspect llm-wiki --format "{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}"'
ssh hch@10.145.119.19 'curl -sS http://127.0.0.1:19828/health'
ssh hch@10.145.119.19 'curl -sS http://127.0.0.1:19828/api/v1/projects'
```

通過條件：

- 容器 `Up`，image `llm-wiki:0.6.11`
- mounts 含 `/home/hch/wiki/BASE -> /home/hch/wiki/BASE`
- `/health` 回 `ok: true`、`version: 0.6.11`
- `/api/v1/projects` 的 current 是 `BASE`、id `f974f7c9-d27f-435a-b8a3-3087bc015d8a`、path `/home/hch/wiki/BASE`
- `GET /api/v1/projects/current/files?root=wiki` 看得到 `wiki/entities` 等目錄
- `http://10.145.119.19:6080/vnc.html` 回 200

本機也可：

```bash
curl -sS http://10.145.119.19:19828/health
```

## Inputs and Outputs

- **Input:** SSH 可連的 `hch@10.145.119.19`、本 skill 的三個 compose 檔、現有 `/home/hch/wiki/BASE`
- **Output:** 正在跑的 `llm-wiki` 容器、API `:19828`、noVNC `:6080`、與 host BASE 共用的知識庫

## Rules and Limitations

- 相對路徑以本 `SKILL.md` 所在目錄為準。
- 版本、埠、專案 id 以遠端現場為準；本文件當 2026-09-12 已驗證快照。
- 不要把 token 打進對話；需要時讀 compose env 或 app-state，回報時遮掉。
