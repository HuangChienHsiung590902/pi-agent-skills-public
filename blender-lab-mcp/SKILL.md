---
name: blender-lab-mcp
description: >-
  安裝、設定、驗證與排除本機 Windows Blender 5.2 的官方 Blender Lab MCP（官方頁面
  https://www.blender.org/lab/mcp-server/）。當使用者要求「Blender MCP」「官方 Blender MCP」、
  想讓 Pi/Claude 連接 Blender、安裝 MCP 1.0.3、檢查 MCP 工具、或遇到 MCP 連線／背景 CLI
  逾時時使用。不要把第三方 MCP for Blender 與官方 Blender Lab MCP 混用。
---

# Blender Lab MCP（Windows 本機官方版）

## When to Use

- 使用者要安裝或使用 Blender 官方 Lab MCP。
- 使用者要讓 Pi 透過 MCP 讀取、分析或操作目前開啟的 Blender 場景。
- 使用者要確認 Blender MCP 外掛、TCP bridge、MCP server 或所有工具是否正常。
- 使用者遇到背景 `.blend` 查詢逾時、AnimChar Tools 與 MCP 衝突、或 Pi MCP 設定問題。

## Canonical Environment

- Blender：`C:\Program Files\Blender Foundation\Blender 5.2\blender.exe`
- Blender 版本：目前實測 `5.2.1 LTS`
- 官方來源：`https://www.blender.org/lab/mcp-server/`
- 官方 release：Blender Lab `v1.0.3`
- 官方 addon ZIP：`C:\Users\HCH\Downloads\Blender-MCP-official-addon-v1.0.3.zip`
- 官方 MCPB server：`C:\Users\HCH\Downloads\Blender-MCP-official-server-v1.0.3.mcpb`
- 已安裝 addon：
  `C:\Users\HCH\AppData\Roaming\Blender Foundation\Blender\5.2\extensions\user_default\mcp`
- 官方 source checkout：`D:\MCP\blender-mcp\`
- Python 環境：`D:\MCP\blender-mcp\.venv\`
- server executable：`D:\MCP\blender-mcp\.venv\Scripts\blender-mcp.exe`
- Blender addon bridge：`127.0.0.1:9876`
- Pi/adapter MCP 設定：`D:\.system\.config\mcp\mcp.json`
- Pi MCP adapter：`npm:pi-mcp-adapter@2.32.1`

## Components

官方 MCP 由三個部分組成：

1. **Blender Lab addon `MCP`**：安裝在 Blender 5.2 user extension，負責 TCP bridge。
2. **MCP server `blender-mcp`**：在本機 venv 中執行，透過 socket 將 MCP 工具轉給 Blender。
3. **LLM client**：Pi 使用 `pi-mcp-adapter` 讀取標準 MCP 設定。

官方頁面要求三個外部部分都安裝並執行；Blender 本身沒有內建 LLM 連接功能。

## Installation / Repair

### 官方 addon

若尚未安裝，下載官方 release `v1.0.3` 的 `mcp-1.0.3.zip`，在 Blender 5.2：

`Edit → Preferences → Extensions → Install from Disk`

選擇 ZIP，安裝後啟用名稱為 **MCP**、維護者為 **Blender Lab** 的 extension。可用 Blender CLI 以 `bpy.ops.extensions.package_install_files(filepath=..., repo="user_default", enable_on_install=True, overwrite=True)` 安裝；不要手動覆蓋官方 extension 目錄。

### MCP server

目前已存在並通過 help 檢查：

```text
D:\MCP\blender-mcp\.venv\Scripts\blender-mcp.exe --help
```

server 預設使用 stdio，並透過環境變數連接 addon：

```text
BLENDER_MCP_HOST=127.0.0.1
BLENDER_MCP_PORT=9876
```

### Pi MCP adapter

Pi core 不直接讀 MCP；需安裝：

```bash
pi install npm:pi-mcp-adapter@2.32.1
```

使用共用設定檔 `D:\.system\.config\mcp\mcp.json`。目前官方 server entry 應為：

```json
{
  "mcpServers": {
    "blender-official": {
      "command": "D:/MCP/blender-mcp/.venv/Scripts/blender-mcp.exe",
      "args": [],
      "env": {
        "BLENDER_MCP_HOST": "127.0.0.1",
        "BLENDER_MCP_PORT": "9876",
        "BLENDER_PATH": "C:/Program Files/Blender Foundation/Blender 5.2/blender.exe"
      },
      "lifecycle": "lazy"
    }
  }
}
```

不要把 `BLENDER_HOST`／`BLENDER_PORT` 當成官方 source 使用的變數；官方 source 實際讀的是 `BLENDER_MCP_HOST`／`BLENDER_MCP_PORT`。修改設定後重啟 Pi 或執行 `/reload`。

## Procedure

1. Verify the official addon manifest, Blender executable, server executable, and MCP JSON before changing anything.
2. Ensure Blender's **MCP** addon is enabled and its panel reports **Server is running** on `127.0.0.1:9876`.
3. Ensure Pi uses `pi-mcp-adapter` and the standard `D:\.system\.config\mcp\mcp.json` entry shown below.
4. After changing server code or MCP JSON, restart Pi/adapter; do not assume an already-loaded session has refreshed its tools.
5. Run a read-only initialize/tools-list check, then call `get_objects_summary` before any scene modification.
6. For background CLI tools, use a temporary fixture `.blend`, isolated `BLENDER_USER_RESOURCES`, factory startup, disabled auto-exec, and clean up the fixture afterward.
7. Save the user's `.blend` before any destructive MCP operation and report actual verification results.

## Normal Use

1. 開啟 Blender 5.2。
2. 在 Blender 的 3D View 按 `N`。
3. 開啟 **MCP** 分頁。
4. 確認 Host 為 `localhost`、Port 為 `9876`。
5. 啟用 **Auto Start**，或手動按 **Start MCP Server**。
6. 必須看到 **Server is running**。
7. 在 Pi 中連接 lazy MCP server，先讀取場景摘要，再執行操作。
8. 修改場景前先保存 `.blend`；官方 MCP 可執行 LLM 產生的 Blender Python，具有破壞性。

## 建立人形模型的適用方式

Blender MCP 可以直接透過 Python 建立簡單的人形幾何模型，也可以協助操作已匯入的人體模型；但 MCP 本身不是高品質人體生成模型。要依目標選擇流程：

| 目標 | 建議流程 |
|---|---|
| 快速測試 MCP | MCP Python 建立頭、軀幹、手臂、腿的簡單人形 |
| 從人物照片建立人體 | SAM 3D Body → OBJ/GLB → Blender → Rigify |
| 建立包含服裝、髮型、配件的角色 | Hunyuan3D 或其他角色生成器 → Blender 整理 |
| 做姿勢、舞蹈、手語動畫 | T-pose/A-pose → Rigify → Automatic Weights → AnimChar |
| 匯出 Unreal | T-pose → Rigify → FBX → Unreal Engine |
| 手動高品質角色 | Mirror/建模 → Sculpt → Retopology → UV/材質 → Rigify |

使用 MCP 建立測試人形時，可先要求：

```text
請在目前 Blender 場景建立一個簡單的人形模型，包含頭部、軀幹、兩隻手臂與兩條腿，使用不同材質區分各部位，讓角色站在 T-pose，並回報建立的物件名稱。
```

這種 MCP 生成的是幾何體原型，不會自動提供高品質人體拓撲、臉部、頭髮、服裝或可直接用於遊戲的權重。任何建立或修改場景前先保存 `.blend`，並先要求 MCP 讀取現有場景，避免覆蓋使用者物件。

## Available Tools

官方 server 目前列出 26 個工具，包含：

- 即時 `execute_blender_code`
- 背景 CLI `execute_blender_code_for_cli`
- Blend data-block、遺失檔案、linked libraries、path、usage summary（即時與 CLI）
- `get_object_detail_summary`、`get_objects_summary`
- Python API 與 Blender manual 搜尋
- 視窗／區域截圖與視窗 layout JSON
- 切換 workspace、聚焦物件與物件 data
- thumbnail 與 viewport render

## Verification

### 檢查 Blender bridge

```bash
python - <<'PY'
import socket
s=socket.create_connection(("127.0.0.1", 9876), timeout=5)
print("TCP_OPEN")
s.close()
PY
```

### 檢查即時工具

使用 MCP initialize、tools/list，再呼叫 `get_objects_summary`。成功時應回傳目前場景、workspace、active object 與 collections。

可用安全測試 code：

```python
import bpy
result = {
    "ok": True,
    "blender_version": bpy.app.version_string,
    "object_count": len(bpy.data.objects),
}
```

### 檢查背景 CLI 工具

使用臨時 `.blend`，不要直接覆蓋使用者正式檔案。背景命令必須使用：

- `--factory-startup`
- `--disable-autoexec`
- `BLENDER_USER_RESOURCES` 指向獨立暫存目錄
- `stdin=subprocess.DEVNULL`

目前 `D:\MCP\blender-mcp\mcp\blmcp\tools_helpers\blender_cli.py` 已加入這些隔離設定。已驗證 6 個 CLI 工具全部成功，每項約 1.5 秒。

### 已驗證結果

- MCP initialize：成功
- MCP tools/list：26 個工具
- 即時工具測試：20 / 20 通過
- 背景 CLI 工具修正後：6 / 6 通過
- 合計：26 / 26 通過
- Blender：`5.2.1 LTS`
- 測試場景預設有 `Camera`、`Cube`、`Light`

## Pitfalls

- **第三方與官方不要混用**：`MCP for Blender`（第三方 `blender_mcp.py`）與官方 Blender Lab `MCP`（`bl_ext.user_default.mcp`）是不同專案。建議使用官方版時停用第三方版，避免兩者搶 bridge port。
- 官方 MCP addon 的名稱是 **MCP**，維護者是 **Blender Lab**，版本目前為 `1.0.3`。
- Blender 必須顯示 **Server is running**；只有 addon 檔案存在不代表 bridge 已啟動。
- 即時 bridge 讀取的 socket port 是 `9876`，不是官方 MCP server 的 HTTP port `8000`。
- 官方 MCP server 的 HTTP transport 只在需要 HTTP client 時使用；Pi stdio 設定不要把 `--transport http --port 8000` 混入一般設定。
- 背景 CLI 若沿用使用者 `BLENDER_USER_RESOURCES`，AnimChar Tools 可能啟動 bridge／timer，造成背景程序逾時。必須使用隔離 user resources 與 factory startup。
- CLI tool-code 使用 `__BLMCP_PARAMS__` placeholder；測試 tool-code 時要先以 `toolcode_format_call(..., None)` 替換，不要直接執行原始模板。
- MCP 會執行任意 Blender Python；官方頁面明確警告可能刪除資料或將資料送到遠端。敏感 `.blend` 應在 VM 或隔離環境操作。
- 截圖工具需要 Blender 有可用的圖形視窗；純 background Blender 不適合做 window／area screenshot。
- 不要用 `--factory-startup` 啟動使用者的正式 GUI Blender，僅限一次性的 background CLI 查詢。

## Related Skills

- `sam3d-body-rigify-unreal-pipeline`：SAM 3D Body、Rigify、FBX、UE 流程，必要時使用本 skill 的官方 MCP 連線。
- `pi-coding-agent-setup`：Pi 本體與套件管理。
- `obsidian-skills-library`：維護 `D:\OB\skills` 技能庫。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** local Blender 5.2 state, official Blender Lab MCP addon/server files, Pi MCP configuration, and current bridge evidence.
- **Output:** a reproducible official Blender MCP setup with verified real-time and background tool paths.

## Rules and Limitations
- Treat MCP server execution as potentially destructive; save user files first.
- Do not expose or print API keys, tokens, or credentials.
- Do not silently install a different third-party Blender MCP over the official addon.
- Prefer temporary fixture files for CLI tests and remove them afterward.

## Verification
1. Official addon manifest exists under Blender 5.2 user extensions and reports version 1.0.3.
2. Official server executable exists and `--help` succeeds.
3. Pi adapter package is installed and standard MCP JSON parses successfully.
4. Blender TCP bridge at `127.0.0.1:9876` accepts connections.
5. MCP initialize and tools/list succeed.
6. Real-time tool tests and background CLI tests complete without modifying the user's formal `.blend` file.
