---
name: spark-x25-llamacpp-local
description: >-
  在本機 D:\llama.cpp 或桌面獨立包 C:\Users\HCH\Desktop\spark-x2.5 用 XHToken/llama.cpp fork 跑 Spark-X2.5-4B GGUF（Intel Arc Vulkan 或 CPU），含 WebUI tools、stdio MCP（anytxt/es/llm-wiki/comfyui 生圖/comfyui-mcp 完整工具）、agents.md、網頁上傳圖片存檔後 AnyTXT OCR。用在：本機跑 Spark、桌面 spark-x2.5、Web UI 上傳圖片 OCR、Images require a vision-capable model、rebuild-spark-ocr-ui.bat、wiki 知識優先查、spark-mcp.json、start.bat、menu.ps1、spark2_5 不支援、讓本機 Spark 呼叫 ComfyUI 生圖、或不要覆蓋 ASR 舊 llama-server.exe 時。遠端 10.145.119.19 Docker Spark 生圖改用 spark-x25-docker-comfyui。
---

# 本機 Spark-X2.5-4B（D:\llama.cpp）

在這台 Windows 筆電用 llama.cpp 跑 Spark-X2.5-4B。正式推論路徑是 **官方 GGUF + XHToken fork + Vulkan（Intel Arc 140T）**。

同資料夾的 ASR 流程見 [`windows-asr-shandianshuo`](../windows-asr-shandianshuo/SKILL.md)。兩套共用 `D:\llama.cpp`，**執行檔與 port 必須分開**。

## When to Use

- 使用者要在本機 `D:\llama.cpp` 跑 Spark / Spark-X2.5 / Spark-X2.5-4B
- 給了 Hugging Face `XHToken/Spark-X2.5-4B-FP8`（或 Base / GGUF）並說用 llama.cpp
- 要編 Vulkan 給 Intel Arc、或選單啟動 Spark 失敗
- 要讓 Spark 用 llama.cpp 內建 tools 或接 stdio MCP（anytxt／es／llm-wiki／comfyui／comfyui-mcp／spark-mcp.json）
- 要讓本機 Spark 呼叫 ComfyUI 生圖，或本機模型說「沒有圖像生成能力」
- 桌面獨立包 `C:\Users\HCH\Desktop\spark-x2.5`、`start.bat`、`agents.md`、`spark-skills`
- 知識題要先查 llm-wiki、或以 wiki 為準
- 在 WebUI 問「能不能執行 MCP／有沒有 skills」模型說不行
- 網頁上傳圖片 OCR、出現 Images require a vision-capable model、要重編 llama-server UI
- 舊 `llama-server.exe` 報 unknown architecture / 不支援 `spark2_5`

不要用本 skill：遠端 10.145.119.19 的 Spark Docker 生圖（用 `spark-x25-docker-comfyui`）；遠端 Qwen／Qwythos（用對應 omniroute skill）；Jetson 10.145.119.12（RAM 約 4 GB、JetPack 4.6.7/CUDA 10.2，不適合 Spark-X2.5-4B）；本機閃電說 ASR（用 `windows-asr-shandianshuo`）。

## 關鍵事實

| 項目 | 值 |
|---|---|
| 可用模型 | `D:\llama.cpp\models\Spark-X2.5-4B-Q4_K_M.gguf`（官方 GGUF，約 2.42 GiB） |
| GGUF 來源 | https://huggingface.co/XHToken/Spark-X2.5-4B-GGUF |
| **不能直接跑** | https://huggingface.co/XHToken/Spark-X2.5-4B-FP8（safetensors FP8，不是 GGUF） |
| Spark 原始碼 | `D:\llama.cpp\spark-src`（[XHToken/llama.cpp](https://github.com/XHToken/llama.cpp)，架構 `spark2_5`） |
| Vulkan 執行檔 | `D:\llama.cpp\spark-src\build-vulkan\bin\`（含 `ggml-vulkan.dll`） |
| CPU 後備 | `D:\llama.cpp\spark-src\build\bin\` |
| **禁止覆蓋** | `D:\llama.cpp\llama-server.exe` / `llama-cli.exe`（2026-07-14，b9994，給 ASR，無 `spark2_5`） |
| 選單 | `D:\llama.cpp\menu.bat` → `menu.ps1`（`Spark-X2.5*` 自動走 Spark 執行檔，並組 `spark-ui-config.json`） |
| 獨立包 | `C:\Users\HCH\Desktop\spark-x2.5\`（`start.bat`，約 2.5 GB，只含 Spark） |
| System prompt | 獨立包／選單啟動時讀 `agents.md` + `spark-skills\**\SKILL.md` → `spark-ui-config.json`（`--ui-config-file`） |
| MCP json | `spark-mcp.json`：anytxt、es、llm-wiki、`comfyui`（1 工具 `generate_image`）、`comfyui-mcp`（完整工具，約 41 個）。僅 stdio；HTTP／URL MCP 無效 |
| Spark 短 skills | `spark-skills\generate-image`、`spark-skills\use-local-tools`、`spark-skills\search-files`（給 4B 看，保持短） |
| ComfyUI | 遠端 `http://10.145.119.19:8190`；簡單生圖用 `comfyui_generate_image`；進階用 `comfyui-mcp_*` |
| Spark port | **8080** |
| ASR port | 8090 / 8091，不要占用 |
| GPU | Intel Arc 140T（內顯，Vulkan）。這台 **沒有 NVIDIA CUDA** |
| 編譯器 | VS 2022 **Build Tools**（`Program Files (x86)\...\BuildTools`），不是 Community |
| Vulkan SDK | `C:\VulkanSDK\1.4.357.0`（winget `KhronosGroup.VulkanSDK`） |
| OCR 上傳目錄 | `%TEMP%\spark-uploads\img-*.png` |
| 重編腳本 | `D:\llama.cpp\rebuild-spark-ocr-ui.bat` |
| RAM | 16 GB 焊死；context 預設 32768（實測可到 65536），不要開官方 1M |
| Jetson 限制 | `10.145.119.12` 約 4 GB RAM、JetPack 4.6.7/CUDA 10.2；Spark Q4 約 2.6 GB，且需要 XHToken fork，**不要部署到 Jetson** |

上游 `ggml-org/llama.cpp` **沒有** `LLM_ARCH_SPARK2_5`。必須用 XHToken fork。

## Procedure

### 1. 日常啟動（已裝好時）

1. 雙擊 `D:\llama.cpp\menu.bat`
2. 選 `Spark-X2.5-4B-Q4_K_M.gguf`（標籤 `Spark (需 spark-src)`）
3. 參數直接 Enter：Vulkan 時 `-c 32768 -ngl 99 --jinja --tools all --mcp-servers-config D:\llama.cpp\spark-mcp.json --host 127.0.0.1 --port 8080`
4. 瀏覽器開 `http://127.0.0.1:8080`
5. `GET http://127.0.0.1:8080/tools` 應看到內建工具 + `anytxt_*` + `es_es_search` + `llm-wiki_*` + `comfyui_generate_image` + 一批 `comfyui-mcp_*`

CLI（工作目錄必須是 `bin`，否則找不到 DLL）：

```powershell
cd D:\llama.cpp\spark-src\build-vulkan\bin
.\llama-server.exe `
  -m "D:\llama.cpp\models\Spark-X2.5-4B-Q4_K_M.gguf" `
  -ngl 99 -t 8 -c 32768 --jinja --tools all `
  --mcp-servers-config "D:\llama.cpp\spark-mcp.json" `
  --host 127.0.0.1 --port 8080
```

MCP 設定：`spark-mcp.json`（Cursor stdio 格式）。目前：anytxt、es、llm-wiki、comfyui、comfyui-mcp。URL／HTTP MCP（Context7 網址、unreal、esp32）llama.cpp **不支援**，必須改成 `command`+`args`。工具名會加前綴，例如 `anytxt_anytxt_search`、`llm-wiki_wiki_search`、`comfyui_generate_image`、`comfyui-mcp_generate_image`。

簡單生圖（水果盤、一張圖）走 `comfyui_generate_image`：stdio MCP `D:\MCP\spark-comfyui-mcp\server.py`，Z-Image Turbo POST 到 `http://10.145.119.19:8190/prompt`，PNG 存 `%TEMP%\spark-comfyui\`。完整 ComfyUI 能力走 `comfyui-mcp_*`（`timeout_ms` 180000，`COMFYUI_URL=http://10.145.119.19:8190`）。4B 容易被 40+ 工具淹沒，生圖 skill 規定第一個輸出必須是 `comfyui_generate_image`。

單次測試：

```powershell
cd D:\llama.cpp\spark-src\build-vulkan\bin
.\llama-completion.exe `
  -m "D:\llama.cpp\models\Spark-X2.5-4B-Q4_K_M.gguf" `
  -ngl 99 -t 8 -c 2048 -n 256 `
  -cnv -st --jinja --simple-io `
  -p "你好"
```

檢查 GPU：`.\llama-completion.exe --list-devices` 應看到 `Vulkan0: Intel(R) Arc(TM) 140T GPU`。

### 2. 下載 GGUF（還沒有模型時）

不要下 FP8 safetensors。下官方 Q4_K_M（16GB 機器首選）：

```text
https://huggingface.co/XHToken/Spark-X2.5-4B-GGUF/resolve/main/Spark-X2.5-4B-Q4_K_M.gguf
→ D:\llama.cpp\models\Spark-X2.5-4B-Q4_K_M.gguf
```

Q8_0（~4.4 GB）／BF16（~8.2 GB）這台很緊，不要當第一次。

### 3. 編譯 / 重編 Vulkan

腳本：[`scripts/build-vulkan.bat`](scripts/build-vulkan.bat)

前置：VS 2022 Community + CMake/Ninja、Vulkan SDK、`D:\llama.cpp\spark-src` 已 clone。

```bat
git clone --depth 1 https://github.com/XHToken/llama.cpp.git D:\llama.cpp\spark-src
```

CMake 必須：

- `-DGGML_VULKAN=ON -DGGML_CUDA=OFF`
- `-DLLAMA_BUILD_TESTS=OFF -DLLAMA_BUILD_EXAMPLES=OFF`（`test-unicode` 在 MSVC 會 LNK2019）
- 輸出到 **`build-vulkan`**，不要蓋 `build\`（CPU）或根目錄 ASR exe

CPU-only 後備：同一套源碼、`-DGGML_VULKAN=OFF`、輸出 `build\`。

### 4. 桌面獨立包（只要 Spark、不要整份 llama.cpp）

`C:\Users\HCH\Desktop\spark-x2.5\`：Vulkan `llama-server.exe`+DLL、模型、`spark-mcp.json`、`spark-skills`、`agents.md`、`start.bat`。

1. 雙擊 `start.bat`（預設 `127.0.0.1:8080`，占用則問 port；`-c 65536`）
2. `start.ps1` 會把 `agents.md` + `spark-skills` 寫進 `spark-ui-config.json`（含 `agenticMaxTurns: 6`）並加 `--ui-config-file`、`--tools all`、`--mcp-servers-config`
3. **Web UI 的 systemMessage／agenticMaxTurns 只在瀏覽器第一次造訪套用。** 已用過 localhost:8080 時必須 Settings → **Reset to Default**，再開**新對話**。舊對話不會帶新設定。

`agents.md` 是 Spark 當下規則（MCP 名單、wiki 優先查）；長流程寫本 skill，不要把操作手冊塞進 `spark-skills`（4B context）。

### 5. 上下文滿載與壓縮判斷

桌面獨立包 `start.ps1` 目前以 `-c 65536` 啟動。`llama-server.exe --help` 顯示支援：

- `--context-shift`／`--no-context-shift`：上下文滿載時是否滑動；滑動是丟棄較舊 token，不是語意摘要。
- `--keep N`：context shift 時保留初始 prompt 的 token 數。
- `--cache-type-k TYPE`／`--cache-type-v TYPE`：設定 K/V KV cache 資料型別，可用 `q8_0` 或 `q4_0` 降低記憶體，但不會減少對話 token。

目前獨立包的實際命令只有 `-c 65536`，沒有明確指定 `--cache-type-k/v` 或 `--keep`；因此不能把它描述成已啟用 KV cache 量化。沒有發現 `/compact`、`--compress-context` 或自動語意摘要命令。若要真正壓縮舊對話，必須由 WebUI／外部 client 先呼叫模型摘要，再用摘要重建 prompt。

`Q4_K_M` 只代表 GGUF 模型權重量化，與 KV cache 或對話語意壓縮是不同層次。

### 6. 知識庫（llm-wiki）

- llm-wiki 是使用者知識庫，預設專案 BASE：`D:/WIKI/BASE`（MCP 專案 ID `725f0a49-f68b-43a5-9b5d-a53b2bf62b11`）。
- `agents.md` HARD RULE：知識題（誰／什麼／為什麼、人物、最帥等）**第一個輸出必須是** `llm-wiki_wiki_search`，不准先用記憶回答；wiki 為準；沒有就說 wiki 沒有這筆資料。
- 搜尋是關鍵字／token，不是腦補。來源沒寫「最帥」，問最帥就找不到人物頁。要對上，來源裡要有對應詞。
- 測 wiki：`POST /tools` `llm-wiki_wiki_search`；改 `agents.md` 後必須重開 server。

### 6. 網頁上傳圖片 → AnyTXT OCR

Spark-X2.5 **沒有 mmproj**，模型看不見圖。Web UI 預設會擋上傳（*Images require a vision-capable model*）。

已改 `spark-src`：
- UI：無 vision 也可選 Images（`modality-file-validation.ts`、`attachment-menu.constants.ts`）
- Server：`server-common.cpp` 在 `!allow_image` 時把 `image_url` 存成 `%TEMP%\spark-uploads\img-<ms>.png`，改成文字：「圖片已存成 <path>。請立刻呼叫 anytxt_anytxt_ocr」
- 必須 **LLAMA_USE_PREBUILT_UI=OFF** 重編，否則 UI 仍是 Hugging Face 預建包

重編（先關 Spark）：
1. `cd D:\llama.cpp\spark-src\tools\ui && npm install && npm run build`（改 UI 時）
2. 雙擊 `D:\llama.cpp\rebuild-spark-ocr-ui.bat`（Build Tools + Vulkan SDK，複製 exe/dll 到桌面獨立包）
3. `start.bat`，瀏覽器 **Ctrl+F5**，上傳圖片。ATGUI 要開著。

AnyTXT MCP（`D:\MCP\anytxt-mcp` 1.2.0）另有 `filterName`（檔名過濾）、`searchType` 1精確/2進階/4正規（4 無真正 regexp）。

### 7. 驗證

跑 [`scripts/verify.ps1`](scripts/verify.ps1)。通過條件見下方 Verification。

## Pitfalls

- **FP8 權重不是 GGUF**。llama.cpp 載不了 `XHToken/Spark-X2.5-4B-FP8`。
- **根目錄舊 binary 不能跑 Spark**。`menu.ps1` 已對 `Spark-X2.5*` 改走 `spark-src`；其他模型仍用舊 exe。
- **Start-Process 的 WorkingDirectory 必須是 Spark `bin`**，否則缺 `llama.dll` / `ggml-vulkan.dll`。
- Spark chat 需要 **`--jinja`**。
- 預設會先「思考」；`-n` 太短會只看到思考、沒有最終答案。加長 `-n`（例如 256）或接受思考輸出。
- Arc 是內顯、跟系統共用 RAM。實測 Q4_K_M 可 37/37 層 offload，生成大約 20+ tok/s，prompt 不一定比 CPU 快。
- Spark 用 **8080** 且綁 **127.0.0.1**（開了 `--tools` / MCP，有 exec_shell）。ASR 用 **8090/8091**。不要停 ASR、不要覆蓋 ASR exe。
- 問模型「你能執行 MCP 嗎／有沒有 skills」不可靠；以 `GET /tools` 與實際 tool_calls 為準。相對路徑落在 server 工作目錄（獨立包是桌面目錄，選單是 `build-vulkan\bin`），請用絕對路徑。
- 官方 1M context 這台跑不動。桌面獨立包 `start.ps1` 已用 `-c 65536`；選單 `menu.ps1` 預設仍可能是 32768。
- Web UI `request (33126 tokens) exceeds the available context size (32768)` 是**這則對話 prompt 塞爆**，不是 Spark 掛了。先看 `/health` 與 `llama-server` process；要過關：重開 `start.bat`（65536）+ **開新對話**。舊長對話一樣會爆。
- 4B 工具鬼打牆：`agents.md` NO LOOP（同一則最多 2 次 tool call）；`agenticMaxTurns: 6`。Reset to Default 才吃得到。
- context shift 是捨棄舊 token，不要誤稱為「摘要壓縮」；KV cache `q8_0`／`q4_0` 只降低 cache 記憶體，也不會產生語意摘要。
- `--cache-type-k/v` 與 `--keep` 沒有寫入啟動命令時，不要假設它們已啟用；以實際 process command line 或 server log 為準。
- `--ui-config-file` 的 `systemMessage` **只套用第一次造訪**；之後 localStorage 蓋掉。改 `agents.md` 後：重開 server + Reset to Default + 新對話。
- 4B 會忽略埋太深的規則。wiki 優先查必須放 `agents.md` 最前面，並要求「先 tool call、不要先寫答案」。規定「第一行必須是有 agents.md」會污染無關問題。
- llama.cpp MCP 只有 stdio。Web UI 加 `https://mcp.context7.com/mcp` 不會出現在 `GET /tools`。
- ComfyUI 生圖連 `http://10.145.119.19:8190`，不是本機 `127.0.0.1:8188`。`comfyui-mcp` 必須帶 `COMFYUI_URL`／`--comfyui-url`，且 `timeout_ms` 180000（預設 30 秒不夠）。
- 4B 加上完整 `comfyui-mcp_*` 可能選錯工具；簡單生圖必須走 `comfyui_generate_image`。改 MCP 後要重開 Spark，WebUI 要 Reset to Default + 新對話。
- wiki 搜「全世界最帥的人」找不到只寫了姓名／地址的 `raw/sources/測試的資料.txt`；搜「黃建雄」才會中。
- 網頁上傳圖 **不是** 磁碟路徑；舊 UI 會直接拒絕。沒重編／沒 Ctrl+F5 仍會看到 vision 錯誤。
- 重編必須用 **Build Tools** 的 `cl.exe`。舊 `build-vulkan` cache 若指向已刪的 Community `cl.exe`／`link.exe`，刪 `CMakeCache.txt` 再 configure。
- `searchType=4` 與 Advanced 筆數相同；HTTP JSON-RPC 沒有 regexp 開關。

## Verification

1. `D:\OB\skills\spark-x25-llamacpp-local\SKILL.md` 的 frontmatter `name` 等於資料夾名。
2. `powershell -NoProfile -File D:\OB\skills\spark-x25-llamacpp-local\scripts\verify.ps1` 結束碼 0。
3. `--list-devices` 有 `Vulkan0` Arc 140T。
4. 選單或 `llama-completion.exe` 能載入 `Spark-X2.5-4B-Q4_K_M.gguf`；verbose 可見 `offloaded 37/37 layers to GPU`。
5. `D:\llama.cpp\llama-server.exe` 時間戳仍是 ASR 舊檔（不應被 Spark 編譯覆蓋）。
6. `GET /tools` 含 built-in + anytxt + es + llm-wiki + `comfyui_generate_image` + 一批 `comfyui-mcp_*`（無 Context7 除非 stdio 寫進 json）。
7. API 帶 system prompt 問 MCP／skills：應列出 anytxt、es、llm-wiki、comfyui 與 generate-image、use-local-tools、search-files。問生圖時 round1 應 `tool_calls` 且 name=`comfyui_generate_image`，目標 ComfyUI 為 `10.145.119.19:8190`。
8. 檢查啟動命令時，若要宣稱 KV cache 已量化，必須實際看見 `--cache-type-k` 與 `--cache-type-v`；若要宣稱有摘要壓縮，必須找到 WebUI／client 的摘要與 prompt 重建邏輯。
9. 問「全世界最帥的人」round1 `finish_reason=tool_calls` 且 name=`llm-wiki_wiki_search`；wiki 無對應詞則回答 wiki 沒有這筆資料，而不是先用記憶長篇回答。
10. 無 vision 的 Web UI 可上傳圖片；server log 有 `saved uploaded image for OCR`；檔在 `%TEMP%\spark-uploads`；模型應呼叫 `anytxt_anytxt_ocr`。
11. 獨立包啟動命令含 `-c 65536`；超長舊對話在 32768 的 server 上會 400，health 仍可能 ok。

## Inputs and Outputs

### Inputs

- 本機 `D:/llama.cpp`、Spark GGUF、XHToken llama.cpp fork、Vulkan/CPU 執行環境與 MCP 設定。
- Web UI、`/tools`、OCR 上傳與 ComfyUI/LLM Wiki 整合的實際狀態。

### Outputs

- Spark server 啟動、工具清單、圖片 OCR、ComfyUI 生圖與 context 設定的可驗證結果。
- 編譯或設定變更的備份、命令、port 與驗證輸出；不覆蓋 ASR 的既有執行檔。

## Rules and Limitations

- 不得覆蓋 `D:/llama.cpp/llama-server.exe`、`llama-cli.exe` 等 ASR 執行檔，Spark 使用獨立 build 輸出。
- FP8 safetensors 不是 GGUF；不得宣稱可以直接由 llama.cpp 載入。
- 只有實際出現在 `/tools` 且完成 tool call 的能力才能算可用；不要依模型自述或 HTTP health 猜測。
- 不把 context shift、KV cache 量化誤稱為語意摘要；任何模型、MCP、port 或 OCR 改動都要重啟並重新驗證。
