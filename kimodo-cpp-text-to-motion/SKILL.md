---
name: kimodo-cpp-text-to-motion
description: 在遠端 Docker 主機 10.145.119.19（hch@gigabyte）管理並使用 LocalAI kimodo.cpp 文字轉動作服務；涵蓋 Docker Compose 建置、模型下載、Vulkan/CPU fallback 修復、瀏覽器 UI 生成動作、取得動畫 GLB 與服務驗證。當使用者提到 kimodo.cpp、Kimodo 文字轉動作、有人下跪/走路/揮手等動作生成、8094，或要在 10.145.119.19 生成骨架動畫時使用本 Skill。
compatibility: Windows pi、SSH key-based access、Docker Compose、Playwright MCP；遠端主機需有 x86_64 Linux、Docker、NVIDIA container runtime（GPU 目前僅保留相容性，不代表 Kimodo 實際使用 GPU）。
---

# Kimodo.cpp 遠端文字轉動作

## When to Use

使用者要求以下任一事項時載入本 Skill：

- 在 `10.145.119.19` 安裝、重建或檢查 `localai-org/kimodo.cpp`。
- 開啟或測試 Kimodo web demo：`http://10.145.119.19:8094`。
- 將中文或英文動作描述生成 SOMA/G1 等骨架動作，例如「有人下跪」、「一個人走路並揮手」。
- 取得生成後的 `animation.glb`，或確認生成結果是否能在 web UI 播放。
- 排查 `native worker stopped: EOF`、`vk::createInstance: ErrorIncompatibleDriver`、模型檔案不存在或生成佇列卡住。

本 Skill 只處理目前這套 Kimodo 部署，不要把它誤當成一般 LLM 聊天服務或 ComfyUI 影片生成服務。

## Inputs and Outputs

### 輸入

- 動作描述：建議使用明確、單一的人體動作句子，例如 `有人下跪`。
- 可選幀數：目前 UI 接受 60–150 frames；60 frames 適合快速測試。
- 遠端 SSH：`hch@10.145.119.19`。優先使用現成 key-based SSH，不要在 Skill 內保存密碼。

### 輸出

- Web UI：`http://10.145.119.19:8094`
- 生成後 GLB：`http://10.145.119.19:8094/api/animations/<animation-id>/animation.glb`
- 遠端原始檔案：`/home/hch/kimodo.cpp/demo-output/<animation-id>/animation.glb`
- 生成 metadata：`/home/hch/kimodo.cpp/demo-output/<animation-id>.json`

回覆時要說明實際狀態（`ready`、`running`、`failed`）、frames、使用的 motion model/文字量化，以及是否實際取得 GLB；不要只因頁面能開就宣稱生成成功。

## Current Deployment

遠端專案固定在：

```text
/home/hch/kimodo.cpp
```

重要檔案：

```text
/home/hch/kimodo.cpp/docker-compose.yml
/home/hch/kimodo.cpp/Dockerfile
/home/hch/kimodo.cpp/docker-entrypoint.sh
/home/hch/kimodo.cpp/download-models.sh
/home/hch/kimodo.cpp/models/
/home/hch/kimodo.cpp/demo-output/
```

服務設定：

- Compose service/container：`kimodo`
- Image：`kimodo.cpp:latest`
- Port：`8094:8094`
- motion model：SOMA RP v1.1
- text encoder：`Llama-3-Kimodo-Q4_K_M.gguf`
- runtime model volume：`./models:/data/models:ro`
- output volume：`./demo-output:/data/demo-output`
- 服務重啟策略：`unless-stopped`
- 生成 worker 是序列化的，一次只能可靠處理一個生成工作。

目前實際可用模型檔案：

```text
models/kimodo-soma-rp-v1.1-f32.gguf
models/Llama-3-Kimodo-Q4_K_M.gguf
models/tokenizer.gguf
```

官方原始碼與模型來源：

```text
https://github.com/localai-org/kimodo.cpp
https://huggingface.co/LocalAI-io/Kimodo-SOMA-RP-v1.1-GGML
https://huggingface.co/LocalAI-io/Llama-3-Kimodo-GGML
```

## Procedure

### 1. 先確認遠端與服務

```bash
ssh hch@10.145.119.19 'hostname && docker version --format "{{.Server.Version}}"'
ssh hch@10.145.119.19 'docker ps --filter name=kimodo --format "{{.Names}}\t{{.Status}}\t{{.Ports}}"'
ssh hch@10.145.119.19 'curl -fsS -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:8094/'
```

預期：hostname 為 `gigabyte`、container 為 `Up`、HTTP 為 `200`。

Compose 操作：

```bash
ssh hch@10.145.119.19 'cd /home/hch/kimodo.cpp && docker compose ps'
ssh hch@10.145.119.19 'cd /home/hch/kimodo.cpp && docker compose logs --tail 100'
ssh hch@10.145.119.19 'cd /home/hch/kimodo.cpp && docker compose restart'
```

### 2. 需要模型時下載並驗證

不要重複下載已驗證的檔案。既有腳本會以 SHA-256 驗證，且使用 `.part` 暫存檔，避免中斷後留下假完成檔：

```bash
ssh hch@10.145.119.19 'nohup /home/hch/kimodo.cpp/download-models.sh > /home/hch/kimodo.cpp/download.log 2>&1 < /dev/null & echo $!'
ssh hch@10.145.119.19 'tail -30 /home/hch/kimodo.cpp/download.log'
ssh hch@10.145.119.19 'ls -lh /home/hch/kimodo.cpp/models'
```

目前已驗證的 SHA-256：

```text
kimodo-soma-rp-v1.1-f32.gguf  3bf1229f4c1eff1d28f5196a854113da2df9a11a5e21c60694630903bf948ee4
Llama-3-Kimodo-Q4_K_M.gguf    c06f2cb6a7615e949bcbb5b582ad35a191338b68b5a51abe17d259ebe574c005
tokenizer.gguf                 81614aca62a98846c02b72cc2e5378e5bdce8b4d507b88f96eb1faa90dae607e
```

### 3. 建置或重建容器

```bash
ssh hch@10.145.119.19 'cd /home/hch/kimodo.cpp && docker compose build'
ssh hch@10.145.119.19 'cd /home/hch/kimodo.cpp && docker compose up -d --force-recreate'
ssh hch@10.145.119.19 'docker exec kimodo /opt/kimodo/build/release/kmd-inspect /data/models/kimodo-soma-rp-v1.1-f32.gguf'
```

`kmd-inspect` 應回報：

```text
Kimodo motion GGUF: valid (414 F32 tensors)
```

### 4. 透過 web UI 生成動作

使用 Playwright MCP 操作一般內網 web UI：

1. 導航到 `http://10.145.119.19:8094`。
2. 重新取得 accessibility snapshot；不要沿用過期的 element ref。
3. 確認 Motion model 選為 `SOMA RP v1.1`。
4. 確認 Text encoder quantization 選為目前唯一可用的 `Q4_K mixed`。
5. 將動作描述填入 `Describe a motion`。
6. 測試時優先用 60 frames；需要較長動作才用 150 frames。
7. 點擊 `Generate motion`，等待狀態變成 `Generation complete` 或 gallery item 變成 `ready`。
8. 點擊最新的 ready gallery item，確認畫面顯示 `frame 1 / N` 或其他 frame 訊息。
9. 從頁面上的 `Download animated GLB` 連結，或 metadata 取得 animation id。

典型輸入：

```text
有人下跪
```

成功時會出現類似：

```text
ready · 60 frames · 100 steps · q4_k_m
```

### 5. 用檔案與 HTTP 做第二層驗證

只看瀏覽器狀態不夠，生成後執行：

```bash
ssh hch@10.145.119.19 'find /home/hch/kimodo.cpp/demo-output -maxdepth 2 -type f -printf "%T@ %s %p\n" | sort -n | tail -10'
ssh hch@10.145.119.19 'cat /home/hch/kimodo.cpp/demo-output/<animation-id>.json'
ssh hch@10.145.119.19 'curl -fsSI http://127.0.0.1:8094/api/animations/<animation-id>/animation.glb | grep -iE "HTTP/|content-length|content-type"'
```

`metadata.status` 必須是 `ready`，並且 output 目錄應有：

```text
root_positions.f32
local_rotations_xyzw.f32
animation.glb
```

### 6. 進度與時間預期

CPU fallback 下，模型載入與 100 diffusion steps 可能需要數分鐘：

- 60 frames：通常要等待數分鐘。
- 150 frames：可能需要更久。
- 不要因 30 或 60 秒沒有結果就重複送出請求。
- 不要平行送出多個生成；後續工作會排隊，還會增加 RAM/CPU 壓力。
- 若頁面顯示 `running`，每 30–60 秒檢查一次即可。

可從遠端確認 worker：

```bash
ssh hch@10.145.119.19 'docker exec kimodo sh -lc "ps -eo pid,etime,pcpu,pmem,args | grep -E \"kmd-generate\" | grep -v grep || true"'
```

## CPU / Vulkan 已知限制

目前這台遠端的 NVIDIA container runtime 可提供 CUDA/NVIDIA device，但容器內 Vulkan ICD 不相容；實測曾出現：

```text
vk::createInstance: ErrorIncompatibleDriver
native worker stopped: EOF
```

因此 `Dockerfile` 目前在 builder 階段把 demo 的子程序設定由 `KIMODO_BACKEND=vulkan` 改成 `KIMODO_BACKEND=cpu`，以 CPU 完成可靠生成。Compose 中保留 `gpus: all` 與 `NVIDIA_DRIVER_CAPABILITIES: all` 只是為了保留未來 GPU/Vulkan 修復空間，**不能宣稱目前推理使用 GPU**。

確認實際策略：

```bash
ssh hch@10.145.119.19 'docker logs --tail 50 kimodo 2>&1'
ssh hch@10.145.119.19 'docker exec kimodo sh -lc "ps -eo args | grep kmd-generate | grep -v grep"'
```

若未來要恢復 Vulkan，必須先在容器內以 `vulkaninfo --summary` 確認真的看得到 NVIDIA Vulkan device，再修改 Dockerfile、重建 image、重建容器，並重新做一次實際 prompt 生成驗證；不要只改 UI 標題或 Compose 環境變數。

## Rules and Limitations

- 只操作使用者管理的 `10.145.119.19` 與其 Kimodo 專案；不要任意修改其他 compose project。
- 不要刪除 `/home/hch/kimodo.cpp/demo-output`、models 或 Docker image；清理前須取得明確確認。
- 不要在 Skill 或回覆內保存 SSH 密碼、token 或私人憑證。
- 不要把 `SOMA RP v1.1` 說成 SMPL-X；目前 demo 實際輸出是 SOMA compact 30-joint control skeleton。
- 不要把「頁面 HTTP 200」當成「模型生成成功」；必須看 metadata、raw streams 與 GLB。
- 不要平行大量開頁、快速重送 prompt 或同時排入多個生成。
- 避免用粗暴的 `docker system prune`；這台主機還有其他共用服務。
- 生成內容仍受模型能力與量化影響；「有人下跪」成功代表產生了模型動作資料，不保證每一幀都符合人類主觀判斷。

## Pitfalls

- **`native worker stopped: EOF`**：先查 `docker logs`；若同時有 `vk::createInstance: ErrorIncompatibleDriver`，通常是 Vulkan ICD 問題。使用 CPU fallback 重建，不要無限重送 prompt。
- **Vulkan 編譯失敗**：Debian 12 的 Vulkan-Hpp 可能缺 `VK_EXT_layer_settings` 型別；現有 Dockerfile 會移除 GGML 的 optional validation layer-settings 建構段，並安裝 `spirv-headers`、`glslang-tools`。
- **`libgomp.so.1` 缺失**：runtime 必須安裝 `libgomp1`，否則 `kmd-inspect` 與部分 native binary 無法啟動。
- **模型只下載 `.part`**：代表仍在下載或未通過 checksum；不可把它掛成正式模型使用。
- **瀏覽器 ref 過期**：每次重新導航或 UI 狀態更新後先做 snapshot，再點擊新 ref；必要時用 DOM evaluate 找 `#generate` 或 ready gallery item。
- **生成看似卡住**：查看 `kmd-generate` CPU 使用率與 output 檔案時間；CPU 推理可能數分鐘，避免重啟容器中斷工作。
- **GLB 404**：metadata 還在 `running` 或生成失敗時，`/api/animations/<id>/animation.glb` 尚不存在；等 `ready` 後再查。

## Verification

1. `docker compose ps` 顯示 `kimodo` 為 `Up` 且 port `8094` 已發布。
2. `curl http://127.0.0.1:8094/` 回 `HTTP 200`。
3. `/api/models` 顯示 `soma-rp-v1.1` 為 `available: true`。
4. `/api/text-quantizations` 顯示 `q4_k_m` 為 `available: true`。
5. `kmd-inspect` 回報 motion GGUF valid。
6. 實際生成後 metadata 的 `status` 為 `ready`。
7. output 目錄同時存在 `root_positions.f32`、`local_rotations_xyzw.f32` 與 `animation.glb`。
8. GLB URL 回 `HTTP 200`，且有非零 `Content-Length`。
9. 瀏覽器 gallery 顯示 `ready · N frames`，選取後顯示 `frame 1 / N` 或目前 frame。
10. 回報時列出實際 animation id、GLB URL、生成狀態與目前 CPU/Vulkan 限制。
