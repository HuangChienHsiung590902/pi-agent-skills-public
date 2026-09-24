---
name: spark-x25-docker-comfyui
description: >
  在遠端 GPU 主機 10.145.119.19 讓 Docker Spark-X2.5（llama-cpp-spark-x25-4b，
  WebUI http://10.145.119.19:8083）用 MCP comfyui_generate_image 呼叫 Docker ComfyUI
  （comfyui-h3，8190）以 Z-Image Turbo 生靜態圖。用在：遠端 Spark 生圖、8083 找不到 ComfyUI、
  幫 119.19 設 Spark MCP、8083 WebUI 的 ComfyUI URL 該填什麼、llama-cpp-spark-x25-4b、
  模型說沒有圖像生成能力、或要從 WebUI 直接叫 Spark 出圖時。設定在 119.19 的 spark-mcp.json
  （容器內 URL http://127.0.0.1:8190），不是本機 Pi 的 mcp.json。不要用於本機 Windows Spark
  （用 spark-x25-llamacpp-local），也不要用來跑 MiniMax-H3 影片（用 minimax-h3-comfyui）。
---

# 遠端 Spark-X2.5 Docker → ComfyUI 生圖

在 `10.145.119.19` 上，**由 Spark LLM 呼叫 ComfyUI**，不是 agent 自己 POST `/prompt`。

已驗證路徑：Spark MCP `comfyui_generate_image` → ComfyUI `/prompt`（`http://127.0.0.1:8190`），Z-Image Turbo 在 **cuda:1**。舊的 `exec_shell_command` + `generate_image.py` 仍留著當後備，WebUI 生圖不要再用它。

## When to Use

- 使用者要在 **10.145.119.19 的 Spark Docker** 生圖
- 提到 `llama-cpp-spark-x25-4b`、`http://10.145.119.19:8083`、遠端 Spark 呼叫 ComfyUI
- 8083 找不到 ComfyUI、一直探 `8188`、或問「8083 裡的 ComfyUI 該設成什麼」
- 要幫 **119.19** 設 Spark MCP（`spark-mcp.json`），給 8083 網頁用
- WebUI 直接問 Spark「幫我生成圖片」，且應走這台機器上的 ComfyUI
- 模型回「我沒有圖像生成能力」

不要用本 skill：

- Jetson `10.145.119.12`：RAM 約 4 GB、JetPack 4.6.7/CUDA 10.2，不適合 Spark-X2.5-4B
- 本機 Windows Spark（`spark-x25-llamacpp-local`，port 8080 + MCP `comfyui_generate_image`）
- 本機 Pi 的 ComfyUI MCP（`D:\.system\.pi\agent\mcp.json`，那是 Pi 用，不是 8083）
- MiniMax-H3 影片（`minimax-h3-comfyui`）
- 遠端 Qwen／Qwythos

## Inputs and Outputs

**輸入**

- 畫面描述（中文可；送給腳本的 `--prompt` 用英文較穩）
- 可選：寬高、seed、檔名前綴

**輸出**

- ComfyUI `prompt_id`
- PNG：`/home/hch/ComfyUI/output-h3/spark_x25/`
- 預覽：`http://10.145.119.19:8190/view?filename=...&subfolder=spark_x25&type=output`

## Procedure

### 關鍵路徑

| 項目 | 值 |
|---|---|
| Spark | `llama-cpp-spark-x25-4b`，`http://10.145.119.19:8083`，host network，**GPU 0** |
| Jetson | `10.145.119.12` 不部署 Spark-X2.5-4B；改用 `.19` RTX GPU 或 Windows Intel Arc |
| ComfyUI | `comfyui-h3`，`http://10.145.119.19:8190` |
| 生圖模型 | `z_image_turbo_bf16` + `qwen_3_4b`（lumina2）+ `ae`，**cuda:1** |
| Spark 生圖工具 | MCP `comfyui_generate_image`（`spark-mcp.json` 的 `comfyui`） |
| MCP 腳本 | `/home/hch/llama-cpp-docker/spark-x25-config/scripts/comfyui_mcp_server.py` |
| ComfyUI URL（容器內） | `http://127.0.0.1:8190`（禁止 8188） |
| 後備 CLI | `python3 /spark-config/scripts/generate_image.py --prompt "<English>"` |
| Spark 短 skill | `/home/hch/llama-cpp-docker/spark-x25-config/spark-skills/generate-image/SKILL.md` |
| llama.cpp `/tools` | body 用 `{"tool":"comfyui_generate_image","params":{"prompt":"..."}}`（不是 `arguments`） |

### 0. 設定 8083 網頁用的 ComfyUI MCP（在 119.19，不是本機）

`http://10.145.119.19:8083/#/` 的 MCP 由遠端 llama-server 啟動參數載入：

`--mcp-servers-config /spark-config/spark-mcp.json`

對應主機檔：`/home/hch/llama-cpp-docker/spark-x25-config/spark-mcp.json`（掛進容器 `/spark-config`，**:ro**）。

**不要改** `D:\.system\.pi\agent\mcp.json`。那是 Windows Pi agent 用的；改了 8083 也不會看到。
8083 網頁 Settings 也不用填 ComfyUI URL。若有欄位：不要填 `8188`、不要填 `8083`。

正確 MCP 區塊（canonical：[`references/spark-mcp-comfyui.json`](references/spark-mcp-comfyui.json)）：

```json
"comfyui": {
  "command": "python3",
  "args": ["-u", "/spark-config/scripts/comfyui_mcp_server.py"],
  "env": { "COMFYUI_URL": "http://127.0.0.1:8190" },
  "timeout_ms": 60000
}
```

- 合併進既有 `mcpServers`，**保留** `anytxt`、`llm-wiki`，不要整檔覆寫。
- 只要這一個 `generate_image` 工具（對外名稱 `comfyui_generate_image`）。不要塞 41 個 `comfyui-mcp_*`。
- Spark 與 ComfyUI 都在 119.19、Spark 用 host network，所以容器內打 `http://127.0.0.1:8190`。外面瀏覽器用 `http://10.145.119.19:8190`。
- 主機 `8188` 是空的；寫進 prompt 或設定欄，4B 就會去探然後說找不到。

改完後：

```bash
python3 /home/hch/llama-cpp-docker/spark-x25-config/scripts/rebuild_ui_config.py
docker restart llama-cpp-spark-x25-4b
curl -s http://10.145.119.19:8083/health
curl -s http://10.145.119.19:8083/tools   # 必須出現 comfyui_generate_image
```

然後在 8083：**Settings → Reset to Default → 開新對話**。舊對話還會亂找 ComfyUI。

已驗證（2026-09-12）：`GET /tools` 23 個工具，含 `comfyui_generate_image`；`8190` 活著；`8188` 連不上。

### 1. 使用者在 Spark WebUI 生圖

1. 確認容器：`docker ps` 有 `llama-cpp-spark-x25-4b`、`comfyui-h3`
2. 開 http://10.145.119.19:8083
3. 若剛改過 `agents.md`：Settings → **Reset to Default** → **開新對話**
4. 直接說「生成一張……圖」
5. Spark 第一個輸出必須是 `comfyui_generate_image`
6. 回 `prompt_id` 後約 1–2 分鐘，到 `output-h3/spark_x25/` 取 PNG

全身照用較高畫幅，例如 `--width 768 --height 1280`。

### 2. Agent 代為呼叫 Spark（不要自己打 ComfyUI）

```powershell
python D:\OB\skills\spark-x25-docker-comfyui\scripts\spark_comfyui_generate.py --prompt "full body photograph of ..."
```

腳本會：請 Spark tool-call → 輪詢 `/history/<prompt_id>` → 把 PNG 存到 `%TEMP%\pi-work\spark-comfyui-gen\`。

### 3. 改 Spark 生圖規則：先改本 skill 的 reference，再拷到遠端

遠端根目錄：`/home/hch/llama-cpp-docker/spark-x25-config/`（掛進容器為 `/spark-config`，**:ro**）。

| 本 skill canonical | 拷到遠端 |
|---|---|
| [`scripts/generate_image.py`](scripts/generate_image.py) | `scripts/generate_image.py` |
| [`scripts/comfyui_mcp_server.py`](scripts/comfyui_mcp_server.py) | `scripts/comfyui_mcp_server.py` |
| [`scripts/rebuild_ui_config.py`](scripts/rebuild_ui_config.py) | `scripts/rebuild_ui_config.py` |
| [`references/agents.md`](references/agents.md) | `agents.md` |
| [`references/spark-generate-image.md`](references/spark-generate-image.md) | `spark-skills/generate-image/SKILL.md` |
| [`references/use-local-tools.md`](references/use-local-tools.md) | `spark-skills/use-local-tools/SKILL.md` |
| [`references/spark-mcp-comfyui.json`](references/spark-mcp-comfyui.json) | 合併進 `spark-mcp.json` 的 `mcpServers.comfyui`（不要覆寫 anytxt／llm-wiki） |

拷完後在主機執行：

```bash
python3 /home/hch/llama-cpp-docker/spark-x25-config/scripts/rebuild_ui_config.py
docker restart llama-cpp-spark-x25-4b
```

`rebuild_ui_config.py` 會把 `agents.md` + `spark-skills/**/SKILL.md` 寫進 `spark-ui-config.json`。

**Spark 面向的這三份文字禁止出現 `8188`。** 只寫 `http://127.0.0.1:8190`。改完後 WebUI 必須 Settings → Reset to Default → 開新對話。

Agent 代跑（不要自己 POST `/prompt`）：[`scripts/spark_comfyui_generate.py`](scripts/spark_comfyui_generate.py)。

## Rules and Limitations

- 生圖必須經 Spark tool-call；agent 不得直接 `POST /prompt` 然後假裝是 Spark 做的。
- GPU 0 給 Spark；靜態圖固定 **cuda:1**。不要把 12GB 級 Z-Image 塞進 GPU 0。
- 不要 `docker compose up --build`、不要 rebuild ComfyUI、不要為了生圖卸載 Spark。
- 不要把 41 個 `comfyui-mcp` 工具塞給 4B。只要單工具 `comfyui` MCP。
- `comfyui_generate_image` 只提交 `/prompt`，不在 MCP 行程裡等到出圖；`timeout_ms` 60000。
- 同一則 Spark 對話最多 2 次 tool call；提交後用文字回報並停止。
- MiniMax-H3 影片不走這條路。
- 只生成明確成年角色；不要未成年人形象。

## Pitfalls

- llama.cpp `POST /tools` 要 **`params`**。用 `arguments` 會 `key 'command' not found`。
- 長 `python3 -c '...'` 一定會被引號弄壞；必須跑 `generate_image.py`。
- `--ui-config-file` 的 `systemMessage` 只在瀏覽器第一次造訪套用。改 prompt 後要重啟 Spark + Reset to Default + 新對話。
- `spark-x25-config` 掛進容器是 **:ro**；改檔在主機 `/home/hch/llama-cpp-docker/spark-x25-config/`。
- 預設 `UNETLoader` 會打 GPU 0 然後 OOM；必須用 `*DisTorch2MultiGPU` 且 device=`cuda:1`。
- 4B 容易說「我不能生圖」。規則必須放在 `agents.md` 前面，短 skill 只留一行命令。
- **Spark 的 system prompt 禁止寫 `8188`。** 容器內 8188 是空的（ECONNREFUSED）；只有 8190。寫「8190 -> 8188」4B 就會去探 8188。
- 生圖必須走 `comfyui_generate_image`。`use-local-tools` 不可再把生圖指去 `exec_shell_command`。
- 方形 1024 不適合全身照，會像半身特寫；全身用 768×1280。
- 8083 的 MCP 在 119.19 的 `spark-mcp.json`，不是本機 Pi `mcp.json`。改錯檔會以為「設好了」但網頁還是找不到 ComfyUI。

## Verification

1. `D:\OB\skills\spark-x25-docker-comfyui\SKILL.md` 的 `name` 等於資料夾名。
2. 遠端存在 `/spark-config/scripts/generate_image.py` 與 `spark-skills/generate-image/SKILL.md`。
3. `spark-ui-config.json` 含 `generate-image` 與 `comfyui_generate_image`，且 **不含** `8188`。
4. 遠端 `spark-mcp.json` 的 `mcpServers` 含 `anytxt`、`llm-wiki`、`comfyui`；`comfyui.env.COMFYUI_URL` 為 `http://127.0.0.1:8190`。
5. `curl http://10.145.119.19:8083/health` 為 ok；`GET /tools` 有 `comfyui_generate_image`。
6. WebUI 新對話問生圖時，第一個輸出是 `comfyui_generate_image`，不是拒絕、也不是探測 port。
