---
name: minimax-h3-comfyui
description: 在遠端伺服器 10.145.119.19（雙 RTX 4060 Ti 16GB）以唯一的 Docker Compose 雙 GPU ComfyUI 部署（僅 comfyui-h3，port 8190）管理並執行 MiniMax-H3 量化影片生成。涵蓋 /home/hch/ComfyUI 的 Dockerfile、docker-compose.yml、啟動/停止/重建/狀態檢查、NVIDIA GPU 驗證、ComfyUI-MultiGPU DisTorch2 雙卡權重分載，以及 /prompt API 生成帶音訊影片。使用者說「部署 ComfyUI 雙 GPU」「啟動 comfyui-h3」「ComfyUI 8190」「生成 H3 影片」時使用；此 Skill 是遠端 ComfyUI 的唯一 canonical 版本。
---

# MiniMax-H3 on dual-4060Ti ComfyUI（GGUF Q3 量化版，2026-08-29 重建）

## 環境現況

伺服器：`ssh hch@10.145.119.19`（雙 RTX 4060 Ti 16GB，54GB RAM，`/home` 6.6TB，driver 595.84）
API 位址：**固定 `http://10.145.119.19:8190`**（本機沒有 localhost 情境，看到 127.0.0.1/localhost 一律換成此 IP）

| 項目 | 值 |
|---|---|
| Compose 檔 | `/home/hch/ComfyUI/docker-compose.yml` |
| Image | `comfyui-h3:latest`（自建，`/home/hch/ComfyUI/Dockerfile`） |
| 容器 | `comfyui-h3`，`0.0.0.0:8190 -> 8188`，`gpus: all` |
| ComfyUI 版本 | 0.34.0（PyTorch 2.8.0+cu128 基底 image） |
| 啟動參數 | `--listen 0.0.0.0 --port 8188 --disable-smart-memory --reserve-vram 4.0` |
| custom_nodes（bake 在 image 內） | ComfyUI-GGUF、ComfyUI-MultiGPU、minimax-h3-audio-T8 |
| models（bind mount） | `./models`、`./input`、`./output-h3`、`./user-h3` |
| Compose 服務 | 僅 `comfyui-h3`；不使用 profile 切換 |

**⚠️ GPU 0 有常駐的 `VLLM::EngineCore` 服務佔用約 7.7 GB**（約剩 8 GB 可用），分配時 `cuda:0` 配額不要超過 4~5GB。GPU 1 通常全空。部署前仍須以 `nvidia-smi` 重新確認即時佔用。 

### Docker Compose 管理（唯一 canonical 部署）

Compose 檔位於 `/home/hch/ComfyUI/docker-compose.yml`，目前只定義 `comfyui-h3` 一個服務，不需要 profile 切換流程。

```bash
# 檢查 Compose 設定（唯讀）
ssh hch@10.145.119.19 "cd /home/hch/ComfyUI && docker compose config"

# 首次建立或 Dockerfile 有變更時重建
ssh hch@10.145.119.19 "cd /home/hch/ComfyUI && docker compose build --pull"

# 啟動/更新唯一服務
ssh hch@10.145.119.19 "cd /home/hch/ComfyUI && docker compose up -d comfyui-h3"

# 停止或重新啟動（保留容器與 bind mount 資料）
ssh hch@10.145.119.19 "cd /home/hch/ComfyUI && docker compose stop comfyui-h3"
ssh hch@10.145.119.19 "cd /home/hch/ComfyUI && docker compose restart comfyui-h3"
```

啟動前後都應查詢：

```bash
ssh hch@10.145.119.19 "docker ps -a --filter name=^/comfyui-h3$ --format 'table {{.Names}}\\t{{.Status}}\\t{{.Image}}\\t{{.Ports}}'"
ssh hch@10.145.119.19 "docker logs comfyui-h3 --tail 50"
ssh hch@10.145.119.19 "docker exec comfyui-h3 nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv,noheader"
curl http://10.145.119.19:8190/system_stats
```

正常 Compose 設定必須包含 `NVIDIA_VISIBLE_DEVICES=0,1`、`gpus: all`、`8190:8188` 與 `restart: unless-stopped`。`docker compose up -d` 之後仍要確認容器內兩張 GPU 都可見；「容器可見雙卡」不代表每次推理會自動平行分配，跨卡權重分載仍需使用下方的 MultiGPU loader。

**部署限制：** Docker build 會下載基底映像、ComfyUI 與 custom nodes；模型權重不放入 image，而是放在 `/home/hch/ComfyUI/models/` bind mount。首次 build 或長時間下載時，應以 `ps`、`docker images`、`docker ps` 重新確認實際進度，不要因 SSH 命令輸出中斷就重複啟動另一個 build。

## 模型檔案（Abiray/MiniMax-H3-GGUF 量化版，已就緒）

```
models/diffusion_models/MiniMax-H3-FL2VA-Q3_K_S.gguf                15.6 GB (UNet, 3-bit GGUF；另有 Q5_K_M 23GB)
models/diffusion_models/MiniMax-H3-Ref2VA-Q3_K_S.gguf               15.6 GB (R2V / 影片生影片，2026-09-12 已下)
models/text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors   27.1 GB (Qwen3VL-32B NVFP4/AWQ)
models/vae/minimax_h3_video_vae_fp16.safetensors                     5.2 GB
models/vae/minimax_h3_audio_vae_fp32.safetensors                     0.6 GB
```

**FL2VA** = T2V / I2V / 首尾幀（節點 `MiniMaxH3ImageToVideo`）。**Ref2VA** = R2V / 影片生影片（節點 `MiniMaxH3ReferenceToVideo`，接 `ref_videos`）。兩者都已在 `diffusion_models/`。若 Ref2VA 遺失，從 `Abiray/MiniMax-H3-GGUF` 的 `unet/MiniMax-H3-Ref2VA-Q3_K_S.gguf` 用下方 aria2c 重下（15567048992 bytes）。

Q3_K_S 是省空間版，畫質低於 INT8；若品質不足可改用同 repo 的 Q4_K_S/Q5_K_M（更大）。

### 下載模型：用 aria2c 多連線（不要用單執行緒的 `hf download`）

這台機器對 HF 單連線只有 ~0.3 MB/s，aria2c 8 連線可達 ~10 MB/s。範例：

```bash
aria2c --continue=true --allow-overwrite=true --max-tries=8 --retry-wait=5 \
  --timeout=60 --connect-timeout=30 --split=8 --max-connection-per-server=8 \
  --min-split-size=20M --file-allocation=none --summary-interval=30 \
  "https://huggingface.co/Abiray/MiniMax-H3-GGUF/resolve/main/<repo_file>" \
  -d /home/hch/ComfyUI/models/<子目錄> -o <檔名>
```

注意：`hf download` 新版 CLI 沒有 `--local-dir-use-symlinks` 參數；且用 `docker run` 跑 `hf download` 產生的檔案是 root 所有，搬移需 `sudo mv`。

## 已驗證的最佳配置（雙卡權重分載）

**鐵則：DisTorch2 是「顯存分片」不是「運算並行」。** 取樣計算只發生在 `compute_device` 那張卡，另一張卡是放權重 + 減少 CPU offload。短片（≤2 秒）用純 CPU offload 反而更快；**大影片用雙卡分載才有加速效果**（實測 8 秒 480×864 = 519 秒，舊 INT8 配置同級任務要 13~16 分鐘）。

### Loader 設定（驗證可行，2026-08-29）

```json
"6":  {"class_type":"UnetLoaderGGUFDisTorch2MultiGPU","inputs":{
        "unet_name":"MiniMax-H3-FL2VA-Q3_K_S.gguf","compute_device":"cuda:1",
        "virtual_vram_gb":4.0,"donor_device":"cpu",
        "expert_mode_allocations":"cuda:0,4gb;cuda:1,9gb;cpu,*","eject_models":true}},
"13": {"class_type":"CLIPLoaderDisTorch2MultiGPU","inputs":{
        "clip_name":"qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors","type":"minimax",
        "device":"cuda:1","virtual_vram_gb":4.0,"donor_device":"cpu",
        "expert_mode_allocations":"cuda:0,2gb;cuda:1,5gb;cpu,*","eject_models":true}}
```

- UNet 是 GGUF → 必須用 `UnetLoaderGGUFDisTorch2MultiGPU`（不是 `UNETLoaderDisTorch2MultiGPU`，後者只吃 safetensors）。
- Text encoder 是 safetensors → 用 `CLIPLoaderDisTorch2MultiGPU`。
- VAE 用 `VAELoaderDisTorch2MultiGPU` 放 cuda:0（video VAE decode 時會吃滿該卡）。
- 若 GPU 0 的 vLLM 已停、兩卡都空，可調成 `cuda:0,7gb;cuda:1,7gb;cpu,*` 更平衡。

### 已實測的解析度/時長組合（4 steps, res_multistep, simple scheduler）

| 時長 | length | 解析度 | 耗時 | 結果 |
|---|---|---|---|---|
| ~1.6s | 39 | 384×512 | ~190-335 秒 | ✅ |
| ~8s | 192 | 480×864 | **519 秒** | ✅ 推薦甜蜜點 |

length 必須符合 `17k+5` 網格：`base=max(5,round(sec*24)); length=base+(5-(base%17))%17`（5s→124、8s→192、15s→362）。

720p（1280×720）在雙 4060 Ti 上跑不動，安全上限約 480×864 / 960×544。

## 完整可用的 API prompt 範本（T2V，已驗證）

```python
import json, urllib.request
api="http://127.0.0.1:8190"
graph={
 "6":{"class_type":"UnetLoaderGGUFDisTorch2MultiGPU","inputs":{"unet_name":"MiniMax-H3-FL2VA-Q3_K_S.gguf","compute_device":"cuda:1","virtual_vram_gb":4.0,"donor_device":"cpu","expert_mode_allocations":"cuda:0,4gb;cuda:1,9gb;cpu,*","eject_models":True}},
 "13":{"class_type":"CLIPLoaderDisTorch2MultiGPU","inputs":{"clip_name":"qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors","type":"minimax","device":"cuda:1","virtual_vram_gb":4.0,"donor_device":"cpu","expert_mode_allocations":"cuda:0,2gb;cuda:1,5gb;cpu,*","eject_models":True}},
 "11":{"class_type":"VAELoaderDisTorch2MultiGPU","inputs":{"vae_name":"minimax_h3_video_vae_fp16.safetensors","compute_device":"cuda:0","virtual_vram_gb":0.0,"donor_device":"cpu","expert_mode_allocations":"","eject_models":True}},
 "24":{"class_type":"VAELoaderDisTorch2MultiGPU","inputs":{"vae_name":"minimax_h3_audio_vae_fp32.safetensors","compute_device":"cuda:0","virtual_vram_gb":0.0,"donor_device":"cpu","expert_mode_allocations":"","eject_models":True}},
 "104":{"class_type":"MiniMaxH3ImageToVideo","inputs":{"clip":["13",0],"vae":["11",0],
        "prompt":"<場景與動作描述>。Audio: <音訊描述>。",
        "width":480,"height":864,"length":192}},   # I2V 時加 "first_frame":["114",0] 並用 LoadImage 節點
 "15":{"class_type":"RandomNoise","inputs":{"noise_seed":20260829}},
 "9":{"class_type":"BasicScheduler","inputs":{"model":["6",0],"scheduler":"simple","steps":4,"denoise":1.0}},
 "17":{"class_type":"KSamplerSelect","inputs":{"sampler_name":"res_multistep"}},
 "16":{"class_type":"BasicGuider","inputs":{"model":["6",0],"conditioning":["104",0]}},
 "14":{"class_type":"SamplerCustomAdvanced","inputs":{"noise":["15",0],"guider":["16",0],"sampler":["17",0],"sigmas":["9",0],"latent_image":["104",1]}},
 "10":{"class_type":"VAEDecode","inputs":{"samples":["14",0],"vae":["11",0]}},
 "23":{"class_type":"VAEDecodeAudio","inputs":{"samples":["14",0],"vae":["24",0]}},
 "91":{"class_type":"CreateVideo","inputs":{"images":["10",0],"fps":24,"audio":["23",0]}},
 "92":{"class_type":"SaveVideo","inputs":{"video":["91",0],"filename_prefix":"video/MiniMax_H3/<name>","format":"auto","codec":"auto"}}
}
body=json.dumps({"prompt":graph,"client_id":"agent"}).encode()
print(urllib.request.urlopen(urllib.request.Request(api+"/prompt",data=body,headers={"Content-Type":"application/json"}),timeout=30).read().decode())
# 回傳 {"prompt_id": "...", "node_errors": {}} 才算成功；node_errors 非空要先修
```

I2V（圖生影片）：加 `"114":{"class_type":"LoadImage","inputs":{"image":"xxx.png"}}`，圖片先 `scp` 到 `/home/hch/ComfyUI/input/`，並在 104 加 `"first_frame":["114",0]`。

節點參數有疑慮時先查：`curl http://10.145.119.19:8190/object_info/<NodeType>`。

## 提交、監控、取檔

```bash
# 監控
ssh hch@10.145.119.19 "curl -s http://127.0.0.1:8190/queue"               # queue_running/queue_pending
ssh hch@10.145.119.19 "curl -s http://127.0.0.1:8190/history/<prompt_id>"  # status.status_str=success/error
ssh hch@10.145.119.19 "docker logs comfyui-h3 --tail 50"                   # 看 tqdm / Prompt executed in X seconds

# 輸出與取檔
ssh hch@10.145.119.19 "ls -lh /home/hch/ComfyUI/output-h3/video/MiniMax_H3/"
scp hch@10.145.119.19:/home/hch/ComfyUI/output-h3/video/MiniMax_H3/<file>.mp4 "C:\Users\HCH\Videos\"
```

**鐵則：生成影片一律最終放到 `C:\Users\HCH\Videos`。**

判斷進度看 ComfyUI log 的 tqdm 與 `Prompt executed in ... seconds`；`nvidia-smi` util 低不代表卡住（層遷移/PCIe 可能是瓶頸）。

## 硬規則（防主機被打掛）

1. **一次只送一個任務**，等 queue 空、檔案出現、容器仍 alive 再送下一個。
2. 長片（>15 秒或多段批次）需使用者明確同意「可能卡住主機」。
3. 每段前後查 `free -h`（available < 8GB 就停）、`nvidia-smi`、`curl /queue`。
4. OOM（`Exited (137)` / `torch.OutOfMemoryError`）後先 `docker restart comfyui-h3`，不要同 process 連續重試。
5. 解析度與時長不要同時往上加；720p 以上不要碰。

## 重建 SOP（環境壞掉時照做）

```bash
# /home/hch/ComfyUI/Dockerfile 基底：
#   FROM pytorch/pytorch:2.8.0-cuda12.8-cudnn9-runtime
#   git clone comfyanonymous/ComfyUI（master，需 ≥0.30.0 才有 H3 節點）→ pip install -r requirements.txt
#   git clone city96/ComfyUI-GGUF + pollockjj/ComfyUI-MultiGPU + T8mars/comfyui-minimax-h3-audio-T8 進 custom_nodes/ 並裝各自 requirements
#   CMD python main.py --listen 0.0.0.0 --port 8188 --disable-smart-memory --reserve-vram 4.0
ssh hch@10.145.119.19 "cd /home/hch/ComfyUI && docker compose build && docker compose up -d"
# 模型檔若遺失：用上方 aria2c 指令重下（約 48GB，~10MB/s 約 1.5 小時）
```

鐵則：**`.dockerignore` 必須在第一次 build 前就位**（排除 `models/`、`output*/`、`.git/`、`*.safetensors`、`*.gguf` 等），否則幾十 GB 會被打包進 build context。

## Prompt 撰寫要點

- 結構：整體場景/角色描述 → `SHOT 1 (0s-Xs): ...` 分鏡 → 結尾 `Audio: ...`
- 卡通素材要開頭加風格約束：`Cartoon illustration animation, flat cel-shaded 2D style, simple plain white background, no camera movement, character stays centered in frame`
- 情緒轉折動作至少 3 秒（73 幀）才鋪陳得開

## LINE 動態貼圖 / 素材前處理（簡要）

- 裁掉播放器 UI、格子編號後再送模型；小圖先 LANCZOS 放大到 ~512
- APNG：`ffmpeg -i in.mp4 -vf fps=N frame_%02d.png` 抽格 → 挑 5-20 張 → 首格換成能單獨傳達情緒的畫面 → `ffmpeg -framerate <fps> -i frame_%02d.png -plays 0 out.apng`

## 歷史與延伸閱讀

- 舊版 INT8 環境的 OOM 血淚史、長片策略、舊基準數據 → [`references/legacy-int8-environment.md`](references/legacy-int8-environment.md)
- MiniMax-H3 模型背景（三模組系統、授權、輸出規格、量化生態）→ 同上述 legacy 文件的來源研究，或查 HF `MiniMaxAI/MiniMax-H3`
- Docker 跨主機管理 → `docker-remote-control` skill；GPU 監控桌面小工具 → `rainmeter-remote-gpu-monitor` skill

## Conformance Addendum

## When to Use
在遠端伺服器 10.145.119.19（雙 RTX 4060 Ti 16GB）用 ComfyUI 0.34.0 Docker（comfyui-h3，port 8190）跑 MiniMax-H3 量化版（GGUF Q3_K_S）生成帶音訊影片。使用者說「生成 H3 影片」「跑 minimax」「ComfyUI 8190」時使用。也涵蓋此環境的重建、模型下載、OOM 除錯。

## Inputs and Outputs
- **Input:** 生成需求（文字/圖片、時長、解析度）、或重建/除錯請求。
- **Output:** 生成的影片（最終放 `C:\Users\HCH\Videos`）或環境變更紀錄 + 驗證證據。

## Procedure
1. 先確認容器與 GPU 狀態：`docker ps --filter name=comfyui`、`nvidia-smi`、`curl http://10.145.119.19:8190/system_stats`。
2. 依「已驗證的最佳配置」組 prompt JSON，一次只送一個任務。
3. 監控 queue / log 直到完成，確認 `status=success` 與輸出檔案存在。
4. 下載成品到 `C:\Users\HCH\Videos` 並回報實際耗時與檔案。

## Rules and Limitations
- 相對路徑以本 skill 資料夾為基準。
- 記錄的版本/路徑可能過時，以遠端實際狀態為準。
- 一次一個任務；長片需使用者同意；不做未授權的破壞性操作。
- GPU 0 有 vLLM 常駐佔 ~7.7GB，cuda:0 配額 ≤4~5GB。

## Pitfalls
- GGUF 模型要用 `UnetLoaderGGUFDisTorch2MultiGPU`，用錯 loader 會找不到模型。
- `virtual_vram_gb` 不要亂調大（詳見 legacy 文件第 2 點）；要精確控制用 `expert_mode_allocations`。
- 不要期待兩張卡「並行運算」——DisTorch2 只是顯存分片，運算在 compute_device 一張卡。
- `hf download` 單連線極慢，下載模型一律用 aria2c 多連線。

## Verification
1. `curl http://10.145.119.19:8190/system_stats` 回 200 且 devices 有兩張 4060 Ti。
2. 生成任務 `history/<id>` 的 `status_str=success` 且輸出檔存在。
3. 回報變更內容、實測耗時與剩餘限制。
