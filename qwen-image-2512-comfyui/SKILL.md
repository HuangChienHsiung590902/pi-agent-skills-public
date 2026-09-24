---
name: qwen-image-2512-comfyui
description: >
  在遠端 ComfyUI http://10.145.119.19:8190（容器 comfyui-h3）安裝、補檔，並用
  Qwen-Image-2512（fp8 unet + Qwen2.5-VL CLIP + VAE + Lightning 4-step LoRA）文生靜態圖。
  用在：使用者說「Qwen-Image-2512」「安裝 Qwen-Image」「用 Qwen 出圖」、
  ComfyUI 缺 Lightning LoRA、subgraph「Text to Image (Qwen-Image 2512)」報缺少模型、
  或要 Pi/agent 對 8190 POST /prompt 生圖時。
  不要用於 Z-Image Turbo（z-image-turbo-txt2img）、不要用於 MiniMax-H3 影片
  （minimax-h3-comfyui）、不要裝雲端 Qwen／RunComfy API skill。
---

# Qwen-Image-2512（ComfyUI 8190）

遠端 GPU 主機 `hch@10.145.119.19` 的 Docker ComfyUI（`comfyui-h3`，對外 **8190**）跑 **Qwen-Image-2512** 文生圖。權重在 bind mount `/home/hch/ComfyUI/models/`，不進 image。

已驗證（2026-09-12）：

- 四個權重檔大小與 Hugging Face 一致
- `/prompt` API：`UNETLoaderDisTorch2MultiGPU` + Lightning 4-step LoRA + `ModelSamplingAuraFlow` shift 3.1，**cuda:1**，768×1024，約 **193 秒**，產出正確紅蘋果照片

參考過外部 skill `https://www.skills.sh/artokun/comfyui-mcp/qwen-txt2img`（通用工作流），未採用：那份假設別的節點／路徑，不是這台 `comfyui-h3`。

## When to Use

- 使用者要在 `10.145.119.19:8190` / `comfyui-h3` 裝、補或用 **Qwen-Image-2512** 出圖
- UI 出現 `Text to Image (Qwen-Image 2512)`，錯誤「缺少的模型」或 `lora_name` 無效
- 缺 `Qwen-Image-2512-Lightning-4steps-V1.0-fp32.safetensors`
- Pi / agent 要對 8190 送 Qwen 文生圖

不要用本 skill：

- Z-Image Turbo 靜態圖 → `z-image-turbo-txt2img`
- MiniMax-H3 影片（T2V／I2V／V2V）→ `minimax-h3-comfyui`
- 舊版 `qwen_image_fp8_e4m3fn`（非 2512）除非使用者明確要舊 unet
- 雲端通義萬相／RunComfy API

## Inputs and Outputs

**輸入**

- 安裝／核對／補 Lightning LoRA
- 生圖：畫面描述（`--prompt`）；可選 `--width` `--height` `--seed` `--prefix` `--out-dir`

**輸出**

- 遠端四個權重檔就位（路徑與 byte 大小見下表）
- 生圖：`prompt_id`、PNG（ComfyUI output + 本機暫存）、預覽 URL `http://10.145.119.19:8190/view?filename=...&type=output`

## Procedure

### 關鍵路徑

| 角色 | 檔名 | 目錄 | 期望大小 | 來源 |
|---|---|---|---|---|
| UNet | `qwen_image_2512_fp8_e4m3fn.safetensors` | `models/diffusion_models/` | 20430679144 | `Comfy-Org/Qwen-Image_ComfyUI` `split_files/diffusion_models/` |
| CLIP | `qwen_2.5_vl_7b_fp8_scaled.safetensors` | `models/text_encoders/` | 9384670680 | 同上 `split_files/text_encoders/` |
| VAE | `qwen_image_vae.safetensors` | `models/vae/` | 253806246 | 同上 `split_files/vae/` |
| LoRA | `Qwen-Image-2512-Lightning-4steps-V1.0-fp32.safetensors` | `models/loras/` | 1698951104 | `lightx2v/Qwen-Image-2512-Lightning` |

| 項目 | 已驗證值 |
|---|---|
| CLIPLoader `type` | `qwen_image` |
| UNet `weight_dtype` | `fp8_e4m3fn` |
| GPU | **cuda:1**（DisTorch2，`virtual_vram_gb` unet 8 / clip 6） |
| 取樣 | euler / simple，**steps=4**，cfg=1.0，Lightning LoRA strength 1.0 |
| `ModelSamplingAuraFlow` | shift **3.1** |
| latent | `EmptySD3LatentImage` 768×1024（先不要用 UI 預設 1328） |
| 腳本 | [`scripts/generate_txt2img.py`](scripts/generate_txt2img.py) |

容器已有原生節點，不必為 2512 重建 image。

### 1. 健康檢查（唯讀）

```bash
ssh hch@10.145.119.19 "docker ps --filter name=^/comfyui-h3$ --format '{{.Names}} {{.Status}} {{.Ports}}'"
curl -sS --max-time 15 http://10.145.119.19:8190/system_stats
curl -sS http://10.145.119.19:8190/models/diffusion_models
curl -sS http://10.145.119.19:8190/models/loras
```

四檔都在且無 `.aria2` → 不要重下。生圖前看 `nvidia-smi`：GPU 1 應大致空閒；GPU 0 常被 vLLM 佔用。

### 2. 缺檔才下載

先給執行摘要與 **y/n**。確認後用 **aria2c 8 連線**（`hf download` 在這台約 0.3 MB/s）。已有其他 aria2c 時要排隊。

```bash
aria2c --continue=true --allow-overwrite=true --max-tries=8 --retry-wait=5 \
  --timeout=60 --connect-timeout=30 --split=8 --max-connection-per-server=8 \
  --min-split-size=20M --file-allocation=none --summary-interval=30 \
  "https://huggingface.co/<repo>/resolve/main/<path>" \
  -d /home/hch/ComfyUI/models/<子目錄> -o <檔名>
```

- `https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI/resolve/main/split_files/diffusion_models/qwen_image_2512_fp8_e4m3fn.safetensors`
- `https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI/resolve/main/split_files/text_encoders/qwen_2.5_vl_7b_fp8_scaled.safetensors`
- `https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI/resolve/main/split_files/vae/qwen_image_vae.safetensors`
- `https://huggingface.co/lightx2v/Qwen-Image-2512-Lightning/resolve/main/Qwen-Image-2512-Lightning-4steps-V1.0-fp32.safetensors`  
  （`mkdir -p /home/hch/ComfyUI/models/loras`）

`stat -c%s` 必須等於上表；有 `.aria2` 就是沒下完。長下載用遠端 `nohup`。

UI 模板 `Text to Image (Qwen-Image 2512)` 即使 `enable_turbo_mode=false` 仍要 Lightning 檔名，否則同時報「缺少的模型」+ `lora_name` 無效。補檔後 F5／重新整理模型列表。

### 3. 生圖

先給執行摘要與 **y/n**。確認後：

```powershell
python D:\OB\skills\qwen-image-2512-comfyui\scripts\generate_txt2img.py --prompt "a red apple on a wooden table, photorealistic, natural light"
```

全身照可用 `--width 768 --height 1280`。預設 768×1024。timeout 預設 600 秒（實測約 3 分鐘）。

腳本會：POST `/prompt` → 輪詢 `/history/<id>` → PNG 存到 `%TEMP%\pi-work\qwen-image-2512\`。

回報必須含：`prompt_id`、seed、檔名、預覽 URL、本機暫存路徑。用 Read 把 PNG 給使用者看。

## Rules and Limitations

- 只要 **2512 fp8**。不要自動下 bf16（約 40.8GB）、NVFP4、8-step LoRA、fused 20GB Lightning unet。
- 不要重建 `comfyui-h3` image、不要為了這模型改 compose。
- 生圖固定 **cuda:1** 與 `*DisTorch2MultiGPU`。不要用預設 `UNETLoader`（會打 GPU 0）。
- 解析度先 768×1024；1328×1328 未驗證，可能 OOM。
- 一次只送一個任務。測試檔放 `%TEMP%\pi-work\qwen-image-2512\`，不要寫進 `D:\OB\skills` 或 CWD。

## Pitfalls

- 對外 port 是 **8190**，容器內 8188。本機 `127.0.0.1:8188` 通常沒開。
- 「Qwen-Image」可能是 2512 或舊 `qwen_image_fp8_e4m3fn`；檔名不同。
- UI 缺的常是 Lightning LoRA，不是 unet。
- `models/loras/` 可能不存在，aria2c `-d` 前要 mkdir。
- Lightning 必須 **4 steps + cfg 1.0 + shift 3.1**。當成 20-step 一般取樣會又慢又怪。
- CLIP 用 `type=qwen_image`，不要用 Z-Image 的 `lumina2` / `qwen_3_4b`。
- 與 MiniMax／Z-Image 共用同一 ComfyUI；大檔下載與生圖不要並行。

## Verification

1. 資料夾名與 frontmatter `name` 都是 `qwen-image-2512-comfyui`。
2. 四個遠端檔 `stat` 大小符合上表，無 `.aria2`。
3. `python D:\OB\skills\qwen-image-2512-comfyui\scripts\generate_txt2img.py --help` 可執行。
4. `curl http://10.145.119.19:8190/system_stats` 回 200。
5. 生圖 `/history` 為 `success` 且 PNG 可 Read。
6. `D:\OB\skills\SKILLS_INDEX.md` 含本 skill。
