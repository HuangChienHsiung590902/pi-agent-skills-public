---
name: qwen38-comfyui-mcp-video-pipeline
description: 透過遠端 Qwen3.8-27B Abliterated 產生影片劇本與 prompt，再經 ComfyUI MCP 呼叫 MiniMax-H3 產生影片。當使用者要求「Qwen 寫劇本並讓 ComfyUI 生成影片」、「完整跑一次 Qwen→MCP→ComfyUI」或要重複執行這條流程時使用。
---

# Qwen3.8 → ComfyUI MCP 影片流程

## When to Use

用於遠端 `10.145.119.19` 的完整影片生成流程：

```text
Qwen3.8-27B Abliterated llama.cpp :8081
        ↓ 產生劇本／英文 video prompt
ComfyUI HTTP MCP :9100/mcp
        ↓ 呼叫 allow-listed tool
ComfyUI MiniMax-H3 Docker :8190
        ↓ GPU0 生成影片
MP4：/home/hch/ComfyUI/output-h3/video/mcp/
```

這個 Skill 適用於單段影片示範與可重複的單段生成，不負責多段影片剪輯、字幕合成或 Remotion 後製。

## Inputs and Outputs

### Inputs

- 影片主題／prompt、時長（5–15 秒）與寬高（`384` 或 `512`）。
- 遠端已在跑的 Qwen llama.cpp、ComfyUI MCP 與 MiniMax-H3。

### Outputs

- Qwen 劇本／英文 `video_prompt`、MCP `prompt_id` 與 ComfyUI history 狀態。
- 遠端 MP4：`/home/hch/ComfyUI/output-h3/video/mcp/`。

## 固定環境

| 項目 | 值 |
|---|---|
| 遠端主機 | `hch@10.145.119.19` |
| Qwen API | `http://10.145.119.19:8081/v1` |
| ComfyUI API | `http://10.145.119.19:8190` |
| ComfyUI MCP | `http://10.145.119.19:9100/mcp` |
| Qwen 模型 | `Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf` |
| ComfyUI 模型 | MiniMax-H3 FL2VA Q3_K_S |
| GPU 分配 | GPU0 = ComfyUI；GPU1 = Qwen3.8 |
| llama.cpp offload | GPU1 + 遠端主機 RAM，context `16384` |

## 執行方式

使用本 Skill 附帶的確定性腳本；不要每次手動重組 JSON-RPC 或直接把 MCP `wait:true` 請求長時間掛住：

```powershell
node D:\OB\skills\qwen38-comfyui-mcp-video-pipeline\scripts\run_pipeline.cjs `
  --topic "東亞女性、棕色高馬尾、黑色帶橘色點綴的飄逸舞裙，在攝影棚中優雅跳舞" `
  --duration 10 `
  --width 512 `
  --height 512
```

可選參數：

```text
--title <英文或數字標題>
--duration <5-15 秒，預設 10>
--width <384 或 512，預設 512>
--height <384 或 512，預設 512>
--max-wait <最長等待秒數，預設 1800>
```

腳本會把所有中間 JSON 放在 Windows `%TEMP%\pi-work\` 的專用目錄，正常結束或失敗時清理；不會把測試資料、log 或報告寫進 Skill 目錄或目前工作目錄。

## Procedure

1. 檢查三個遠端 Docker container 是否存在且為 `Up`：
   - `llama-cpp-qwen38-27b-abliterated`
   - `qwen-comfyui-http-mcp`
   - `comfyui-h3`
2. 檢查 Qwen `/health`、模型清單、ComfyUI `/system_stats`、MCP `/health`。
3. 檢查 ComfyUI queue；已有任務執行或排隊時停止，不重複送任務。
4. 呼叫 Qwen3.8 `/v1/chat/completions`，要求產生單場景劇本與英文 `video_prompt`。
5. 移除 Qwen 回覆中的 `<think>...</think>`，解析 JSON；若模型沒有遵守 JSON，使用清理後的文字作為 fallback prompt，並在結果中標記。
6. 對 MCP 執行 `initialize`、`tools/list`，確認有 `comfyui_status` 與 `comfyui_generate_video`。
7. 先呼叫 `comfyui_status`，確認 ComfyUI GPU0、版本與 queue。
8. 呼叫 `comfyui_generate_video`，固定使用 `wait:false`，取得 `prompt_id` 後由腳本直接輪詢 ComfyUI `/history/<prompt_id>`。這樣不會因影片生成超過 HTTP client 的 header timeout 而誤判失敗。
9. 等待 `status_str=success` 或 `status_str=error`；只有成功且輸出檔案存在時才宣稱生成完成。
10. 回報 Qwen 產生的劇本／video prompt、MCP prompt ID、ComfyUI 狀態與遠端 MP4 路徑。

## 規則與限制

- 一次只送一個 ComfyUI 任務。
- 單段影片 5–15 秒。
- 寬高只使用 `384` 或 `512`。
- 目前工作流使用 GPU0 與 CPU RAM 的 Q3 fallback route；不要自行改成 INT8 雙 GPU handoff。
- 不要停止或重建 Qwen、ComfyUI、MCP container。
- 不要讓模型工具操作任意 shell、任意 workflow JSON 或任意檔案路徑。
- 影片工具回傳 `queued` 不代表完成；必須輪詢 history 並確認輸出檔案。

## 常見陷阱

- **本機 Windows 找不到 `/usr/bin/python3`**：不要在本機補這個路徑，也不要使用舊 GPU handoff；目前 HTTP bridge 使用固定 GPU 配置，已跳過 handoff。
- **瀏覽器 `Failed to fetch`**：MCP HTTP bridge 的 CORS 必須允許 `MCP-Protocol-Version`；目前使用 `qwen-comfyui-http-mcp` image。
- **Qwen 回傳 `<think>`**：送入 ComfyUI 前先剝除；`/no_think` 與 `chat_template_kwargs.enable_thinking=false` 仍需保留清理邏輯。
- **MCP `wait:true` 長請求逾時**：使用 `wait:false`，由 ComfyUI history 輪詢；不要重送同一 prompt。
- **GPU 記憶體**：GPU0 給 ComfyUI，GPU1 給 Qwen；生成期間兩者同時存在，禁止跨 GPU handoff。

## Verification

至少確認：

```text
llama.cpp health              → {"status":"ok"}
ComfyUI system_stats          → HTTP 200
MCP health                    → {"ok":true}
MCP initialize                → HTTP 200 + session id
MCP tools/list                → 包含 comfyui_generate_video
MCP status tool               → ok=true、version=0.34.0、queue 為空
ComfyUI history               → status_str=success、completed=true
MP4                           → 遠端 output-h3/video/mcp/ 下存在且 size > 0
```

生成完成後，影片通常可從：

```text
http://10.145.119.19:8190/view?filename=<filename>&subfolder=video%2Fmcp&type=output
```

存取；正式檔案位於遠端：

```text
/home/hch/ComfyUI/output-h3/video/mcp/<filename>
```
