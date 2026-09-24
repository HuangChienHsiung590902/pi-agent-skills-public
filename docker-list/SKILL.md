---
name: docker-list
description: >-
  在 pi 用 /docker-list 或以表格列出 Docker containers。預設列出遠端 GPU 主機
  10.145.119.19（gigabyte）的容器、compose 專案與 GPU 佔用。當使用者說
  docker-list、/docker-list、列出容器、docker ps 表格、container 清單時使用。
---

# docker-list

用表格列出遠端 `10.145.119.19` 的 Docker containers。pi 斜線指令是 `/docker-list`。

不要用這個 skill 啟動、停止、刪除容器；那是 `docker-remote-control` 或其他服務專用 skill 的事。

## When to Use

- 使用者輸入 `/docker-list`
- 使用者要看遠端 Docker 容器清單、compose 專案、誰在佔 GPU

## Procedure

1. 執行本目錄腳本（路徑相對本 `SKILL.md`）：

```text
python scripts/docker-list.py
```

Windows 上若 `python` 不是 UTF-8，改用：

```text
python -X utf8 scripts/docker-list.py
```

2. 把腳本印出的 Markdown 表格原樣回給使用者，不要改寫欄位。
3. 若 ssh 失敗，回報錯誤，不要假裝本機 `docker ps` 就是遠端。

## 遠端規劃（2026-09-12）

| 角色 | compose project | 容器 | 預設 | GPU |
|---|---|---|---|---|
| 文字 LLM | `qwen38` | `llama-cpp-qwen38-27b-abliterated` | 開 | 雙卡滿載 |
| Spark 生圖 LLM | `spark-x25` | `llama-cpp-spark-x25-4b` | 關 | 與 Qwen 互斥 |
| ComfyUI | `comfyui` | `comfyui-h3` | 開（閒置 VRAM 小） | 出圖時跟 Qwen 搶 |
| SAM 3D | `sam-3d` | `sam3d-body`, `sam3d-objects` | 關 | 與 Qwen 互斥 |
| Wiki | `llm-wiki` | `llm-wiki` | 開 | 無 |
| Blender | `blender-docker` | `blender` | 開 | 無 |

Qwen / Spark 的 compose 在 `/home/hch/llama-cpp-docker/`，但 **project name 已拆開**，不要再用目錄預設的 `llama-cpp-docker` 一次 `compose down`。

## Pitfalls

- 本 skill 只列表，不 prune、不 `--remove-orphans`。
- `/docker-list` 靠 pi extension `docker-list-command.ts`；改完 `settings.json` 要重開 pi 才會出現斜線選單。
- 遠端 SSH 帳號是 `hch@10.145.119.19`。

## Verification

1. `python scripts/docker-list.py` 印出 Containers / Compose / GPU 三張表。
2. 專案欄應看到 `qwen38`，不應再把 Qwen 跟 Spark 都標成 `llama-cpp-docker`。

## Inputs and Outputs

### Inputs

- 遠端 Docker 主機 `hch@10.145.119.19` 的 SSH 可用性。
- `scripts/docker-list.py` 與遠端 Docker、Compose、GPU 查詢結果。

### Outputs

- Containers、Compose、GPU 三張 Markdown 表格。
- 若 SSH 或遠端查詢失敗，輸出實際錯誤，不以本機狀態代替。

## Rules and Limitations

- 本 Skill 僅做唯讀列出，不啟動、停止、刪除、prune 或修改容器。
- 遠端 SSH 失敗時不得把本機 `docker ps` 當成遠端結果。
- 回報欄位應保留腳本輸出的原始意義，不猜測不存在的容器或 GPU 狀態。
