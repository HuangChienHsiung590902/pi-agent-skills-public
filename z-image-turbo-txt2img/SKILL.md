---
name: z-image-turbo-txt2img
description: >
  用遠端 ComfyUI http://10.145.119.19:8190 的 Z-Image Turbo 文生靜態圖。
  用在：使用者要 ComfyUI 8190 出圖、Z-Image Turbo、txt2img、生成照片／肖像／靜態圖、
  且由 Pi/agent 直接 POST /prompt 時。
  不要用於 Spark WebUI 生圖（用 spark-x25-docker-comfyui）、不要用於 MiniMax-H3 影片
  （用 minimax-h3-comfyui）、不要把 MCP comfyui 預設的 127.0.0.1:8188 當成這台機器。
---

# Z-Image Turbo 文生圖（ComfyUI 8190）

Agent **直接**對 `http://10.145.119.19:8190` 提交 Z-Image Turbo 工作流。本機 `comfyui-mcp` 預設連 `127.0.0.1:8188`，那台通常沒開，不要用它。

已驗證（2026-09-12）：`UNETLoaderDisTorch2MultiGPU` + `qwen_3_4b`（lumina2）+ `ae`，**cuda:1**，768×1024 肖像約 1–2 分鐘。

## When to Use

- 使用者指定 `10.145.119.19:8190` 或 `comfyui-h3` 生成**靜態圖**
- 提到 Z-Image Turbo、文生圖、出一張照片／肖像
- Pi / Claude / 本機 agent 自己出圖

不要用本 skill：

- 遠端 Spark Docker 自己 tool-call 生圖 → `spark-x25-docker-comfyui`
- MiniMax-H3 影片 → `minimax-h3-comfyui`
- 保險網站語意 icon → `insurance-planner-semantic-icons`

## Inputs and Outputs

**輸入**

- 畫面描述（中文可；送給腳本的 `--prompt` 用英文較穩）
- 可選：`--width` `--height` `--seed` `--prefix` `--out-dir`

**輸出**

- ComfyUI `prompt_id`
- PNG（ComfyUI output + 本機暫存）
- 預覽 URL：`http://10.145.119.19:8190/view?filename=...&type=output`

## Procedure

### 關鍵路徑

| 項目 | 值 |
|---|---|
| ComfyUI | `http://10.145.119.19:8190`（容器內聽 8188，對外 8190） |
| 模型 | `diffusion_models/z_image_turbo_bf16.safetensors` |
| CLIP | `text_encoders/qwen_3_4b.safetensors`，type=`lumina2` |
| VAE | `vae/ae.safetensors` |
| GPU | **cuda:1**（cuda:0 常被 Spark／其他負載佔用） |
| 取樣 | euler / simple，steps=9，cfg=1.0 |
| 腳本 | [`scripts/generate_txt2img.py`](scripts/generate_txt2img.py) |

### 1. 健康檢查（唯讀）

```powershell
curl -sS --max-time 15 http://10.145.119.19:8190/system_stats
curl -sS http://10.145.119.19:8190/models/diffusion_models
```

應看到 `z_image_turbo_bf16.safetensors`。

### 2. 生圖

先給使用者執行摘要與 **y/n**。確認後：

```powershell
python D:\OB\skills\z-image-turbo-txt2img\scripts\generate_txt2img.py --prompt "photorealistic portrait of an elderly man, natural light"
```

全身照用 `--width 768 --height 1280`。肖像預設 768×1024。

腳本會：POST `/prompt` → 輪詢 `/history/<id>` → 把 PNG 存到 `%TEMP%\pi-work\z-image-turbo\`（可用 `--out-dir`）。

### 3. 回報

必須含：`prompt_id`、seed、檔名、預覽 URL、本機暫存路徑。用 Read 工具把 PNG 給使用者看。

## Rules and Limitations

- 靜態圖固定 **cuda:1** 與 `*DisTorch2MultiGPU` loaders。不要用預設 `UNETLoader`（會打 GPU 0 然後 OOM）。
- 不要重啟 ComfyUI、不要 rebuild、不要為了出一張圖下載模型。
- 只生成明確成年角色。
- 測試／下載檔一律放暫存目錄，不要寫進 `D:\OB\skills` 或 CWD。
- MCP `comfyui_generate_image` 連錯 URL 時改走本腳本，不要改使用者機器上的 MCP 設定除非使用者要求。

## Pitfalls

- 對外 port 是 **8190**。容器內 8188 從 Windows 打不到。
- `comfyui-mcp` 的 `COMFYUI_URL` 常是 `127.0.0.1:8188`（ECONNREFUSED）。
- 方形 1024 不適合全身照。
- GPU 0 有負載時 Z-Image 12GB 級權重會 OOM；不要改 `compute_device` 到 cuda:0。

## Verification

1. 資料夾名與 frontmatter `name` 都是 `z-image-turbo-txt2img`。
2. `python .../generate_txt2img.py --help` 可執行。
3. `curl http://10.145.119.19:8190/system_stats` 可達。
4. 實際生圖後 `/history` 為 `completed` 且 PNG 可 Read。
5. `D:\OB\skills\SKILLS_INDEX.md` 含本 skill。
