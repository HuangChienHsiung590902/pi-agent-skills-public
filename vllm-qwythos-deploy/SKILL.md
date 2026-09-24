---
name: vllm-qwythos-deploy
description: 在 10.145.119.19（gigabyte）上用 vLLM 部署/重新部署 qwythos-9b-v2-awq（empero-ai/Qwythos-9B-v2 的 AWQ 量化版），目前線上是 vllm-server 雙卡 tensor parallel（port 8182，max_model_len 98304）；換模型/換參數、調整 --max-model-len、GPU KV cache 容量規劃、以及 pi/OmniRoute 端 context/maxTokens 快取同步都在這裡。當使用者要求「部署 vllm」「重啟 vllm」「換 vllm 模型」「vllm 上跑 qwythos」「qwythos 工具呼叫壞了/連不上」「vllm OCR/看圖壞了」「調 vllm context/context length」時使用。
---

# vLLM qwythos-9b 部署（10.145.119.19）

管一顆用 vLLM 跑在 `10.145.119.19:8182` 的模型。跟同一台機器上的
audiocpp（TTS/ASR）共用雙卡 RTX 4060 Ti（各 16GB）——2026-07-22 起
llama.cpp/ollama/whisper 已徹底移除，GPU 只剩 vllm/audio 兩個服務用
`gpu-switch` 切換（見 `omc-learned` 的 `llama-cpp-server-setup.md`，或
memory `reference_gigabyte_gpu_switch`）。

**⚠️ 這台機器有每天自動排程開關機**（近期實測：約 06:30 開機、約
18:00-18:25 自動關機），排程外的時段機器是關的，SSH/服務都連不上——遇到
連線突然逾時或 reset，先用 `ssh ... "uptime; last reboot"` 確認是不是撞到
這個排程或機器被手動關/開機，不要一開始就懷疑是自己剛做的操作把它弄壞了。

## ⚠️ 動手前：先確認模型身份是真的

**2026-07-20 的教訓**：一開始以為「qwythos-9b」可以隨便挑一顆同尺寸的
Qwen 官方模型頂替，選了 `Qwen/Qwen3-VL-8B-Instruct`，整套裝完測完才被
使用者拿一篇文章戳破——真正的 Qwythos-9B 是 Empero AI 用 Claude
Mythos/Fable 對話軌跡微調 Qwen3.5-9B 做出來的**特定命名模型**（正牌 repo：
`empero-ai/Qwythos-9B-Claude-Mythos-5-1M`），架構、tool-call 格式、
reasoning 行為都跟猜測的替代品不一樣，等於白工重來一輪。

如果使用者要求換一顆「具體命名」的模型（不是「隨便挑一顆差不多大的」），
**先用 WebSearch 查證那個確切名字有沒有對應的真實 HF repo**，不要用同系列
官方模型猜測頂替。詳見 memory `feedback_verify_named_model_identity`。

## 現況（已驗證可用的設定，2026-08-15 更新）

- 模型：AWQ 量化版 `/models/Qwythos-9B-v2-AWQ`
  （原始模型 `empero-ai/Qwythos-9B-v2` 的量化版，checkpoint 約 11GiB），
  served-model-name 是 `qwythos-9b-v2-awq`。v2 是 Empero AI 針對 v1
  （`empero-ai/Qwythos-9B-Claude-Mythos-5-1M`）的改版：保留深度
  chain-of-thought reasoning，但用 FTPO 消除了 v1 會出現的
  looping/degeneration（重複輸出）問題。舊的 v1 checkpoint
  （`~/models/Qwythos-9B-Claude-Mythos-5-1M-AWQ`）還留在磁碟上沒刪。
- **目前線上容器**：`~/vllm-service/docker-compose.yml` 裡的
  `vllm-server`，不是舊的手建 `vllm-qwythos` container。判斷現在到底是哪個
  container 在跑，直接 `ssh hch@10.145.119.19 docker ps` 看 port `8182`
  對應的 container 名稱。
- **目前線上參數（雙卡）**：
  - GPU：`count: all`（兩張 RTX 4060 Ti 16GB 都給 vLLM）
  - `--tensor-parallel-size 2`
  - `--max-model-len 98304`
  - `--gpu-memory-utilization 0.95`
  - `--tool-call-parser qwen3_xml`
  - `--reasoning-parser qwen3`
  - **不要加 `--language-model-only`**；這會讓 vLLM 直接拒收圖片並回
    `At most 0 image(s) may be provided`。
- **驗證標準**：
  - `curl http://10.145.119.19:8182/v1/models` 應回
    `"max_model_len": 98304`。
  - `nvidia-smi` 應看到 GPU0/GPU1 都各用約 14GB VRAM。
  - pi/OmniRoute 發 request 時不應再送 `max_tokens: 16384` 給這顆模型。
- **pi/OmniRoute 同步重點（不要再漏）**：改 vLLM context 後，還要檢查本機
  pi 快取：
  - `D:\.system\.pi\agent\settings.json` 要有 `"reserveTokens": 8192`。
  - `D:\.system\.pi\agent\models.json` 裡兩個 model 都要同步：
    - `qwythos/qwythos-9b-v2-awq`
    - `openai-compatible-chat-dfdac666-5d49-4585-a3d4-210e5f8613e0/qwythos-9b-v2-awq`
    - 欄位：`"contextWindow": 98304`、`"maxTokens": 8192`。
  - 否則 pi 會沿用預設/快取的 `16384` output tokens，出現
    `input 81921 + output 16384 = 98305 > 98304` 這種只差 1 token 的 400。
- **算 context / KV cache 的方法**：先用目前 `--gpu-memory-utilization` 跑一次，
  啟動 log 會印 `Available KV cache memory: X GiB` 和
  `GPU KV cache size: Y tokens`，算每 token 成本 = X GiB / Y tokens。要拉高
  `--max-model-len` 到 Z，先估 KV cache 需求，再回推
  `gpu-memory-utilization`，保留安全餘裕。模型原生支援 YaRN 1M context，但 1M
  需要大量 KV cache，這台雙 16GB 不要硬試。
- **2026-07-20 遷移到 Docker**：不再用 `~/venvs/vllm` + systemd 跑，改成
  `~/vllm-service/docker-compose.yml`，image `vllm/vllm-openai:v0.25.1`，掛載
  `/home/hch/models:/models`。容器需要 `ipc: host`，雙卡 tensor parallel 的
  NCCL/Gloo 通訊才不會被預設 64MB `/dev/shm` 卡死。
- port `8182`（host/container 都是 8182）。

## 重新部署 / 換參數 / 換模型

**不要每次手動 ssh 進去逐條踩坑重新推導**——所有已知坑（huggingface-cli
棄用、KV cache 超額、殘留 VRAM race condition、CUDA graph OOM、
FlashInfer 版本不合、tool-choice 沒開、tool-call-parser/reasoning-parser
選錯、容器 `/dev/shm` 太小）都已經封裝進腳本（原本 systemd venv 時代的
「PATH 找不到 ninja」問題隨遷移到官方 Docker image 一起消失，image 內建
編譯好的 kernel，不用 JIT）：

```bash
cd D:/.system/.claude/skills/vllm-qwythos-deploy/scripts

# 換一顆新模型（先用 WebSearch 確認過 repo 是對的再下載）
./scripts/download-model.sh <hf-repo-id> [本機資料夾名，預設用 repo basename]

# 用 scripts/deploy-vllm.sh 重現目前線上這組雙卡設定
# vision/OCR 可用；不要加 --language-model-only；不要加 --gpu-device
./scripts/deploy-vllm.sh --model-dir Qwythos-9B-v2-AWQ --served-name qwythos-9b-v2-awq \
  --tensor-parallel-size 2 --max-model-len 98304 --gpu-mem-util 0.95 \
  --tool-call-parser qwen3_xml --reasoning-parser qwen3

# 舊的手動 docker run 方式（context 只有 32768）留著參考，目前不是線上這份
ssh hch@10.145.119.19 "docker rm -f vllm-qwythos; docker run -d --name vllm-qwythos --gpus all -p 8182:8000 -v /home/hch/models:/models --ipc=host vllm/vllm-openai:v0.25.1 /models/Qwythos-9B-v2-AWQ --served-model-name qwythos-9b-v2-awq --tensor-parallel-size 1 --max-model-len 32768 --max-num-seqs 2 --gpu-memory-utilization 0.90 --host 0.0.0.0 --port 8000"

# 換模型/換參數重新部署（預設雙卡；只有明確要單卡才加 --gpu-device）
./scripts/deploy-vllm.sh --model-dir <資料夾名> --served-name <served-model-name> \
  --tensor-parallel-size 2 --tool-call-parser <parser> \
  --reasoning-parser <parser或空字串>
```

`scripts/deploy-vllm.sh` 會：用新參數重寫 `~/vllm-service/docker-compose.yml` →
透過 `gpu-switch vllm`（內部先 `docker compose down` 停舊容器、等 GPU
記憶體釋放乾淨，才 `docker compose up -d`——不要手動
`docker compose restart`，會撞殘留 VRAM OOM，也不會套用新設定，因為
compose 對容器定義沒變的既有容器是 no-op）重啟 → 監看 `docker logs` 等
啟動完成或報錯 → 印狀態 + API smoke test。

換 vLLM 版本用 `--image-tag <tag>`（換之前先用 WebSearch/Docker Hub 確認
該 tag 真的存在，不要用同系列版本推測猜一個）。

**⚠️ 這個腳本不會自動同步 client 端快取**：改了 `--max-model-len` 之後，
要同步檢查 pi / OmniRoute / opencode 的模型 metadata。pi 目前最容易漏：
`D:\.system\.pi\agent\settings.json` 的 `reserveTokens`，以及
`D:\.system\.pi\agent\models.json` 內 qwythos 兩個 id 的
`contextWindow` / `maxTokens`。opencode 若使用 `vllm-remote` provider，
也要把 `models.qwythos-9b-v2-awq.limit.context` 改成一樣的值（見
[[opencode-config]] skill）。

## 換模型時，tool-call-parser / reasoning-parser 怎麼選

**不要照模型系列名字直覺猜，也不要照抄上一顆模型驗證過的結果**——不同
模型（甚至同是 Qwen 家族不同代/不同微調）原生輸出的 tool-call 標籤格式
可能完全不同，選錯 parser **不會報錯，只會悄悄解析失敗**（vLLM 照樣回
200 OK，但 `tool_calls` 是空的、原始標籤文字混進 `content`，opencode 顯示
出來是空白，很難第一時間定位）。

換模型後用這個腳本檢查：

```bash
./scripts/test-tool-call-parser.sh
```

它會送一個帶 `tools` 的請求，判斷 `finish_reason` 是不是 `tool_calls`，
不是的話會印出 `content` 裡混著的原始標籤格式，照格式判斷該用哪個
parser（目前見過的兩種）：

| 模型 | tool_call 格式 | `--tool-call-parser` |
|------|----------------|----------------------|
| Qwythos-9B v2（Empero AI，目前在用） | `<tool_call><function=name><parameter=k>v</parameter></function></tool_call>` | `qwen3_xml` |
| Qwythos-9B v1（Empero AI，已不用） | 同上 | `qwen3_xml` |
| Qwen3-VL-8B-Instruct（官方，已不用） | `<tool_call>{"name":...,"arguments":...}</tool_call>` | `hermes` |

vLLM 支援的完整 parser 清單：`vllm serve --help=all \| grep -A2 tool-call-parser`
（要先 `source ~/venvs/vllm/bin/activate`）。

`--reasoning-parser` 只有模型本身會做 chain-of-thought（輸出
`<think>...</think>`）才需要開；不開的話推理內容會直接混進 `content`。
Qwythos-9B（v1、v2 皆是）是 reasoning 模型，用 `qwen3`（Qwen3 系列共通的
`<think>` 格式）。

## 已知殘留問題（尚未解決，不要照抄成「已驗證正常」）

用有開工具的 opencode agent 叫它做完整任務（例如「列出檔案並摘要」）時，
工具呼叫本身正確執行，但收尾的自然語言摘要偶爾出現亂碼字元或重複列表
內容——目前判斷是模型生成品質問題，不是這套 vLLM/opencode plumbing 壞掉
（純文字問答、單純工具呼叫都個別驗證正常），根因還沒細查。

---

## Conformance Addendum

## When to Use
在 10.145.119.19（gigabyte）上用 vLLM 部署/重新部署 qwythos-9b-v2-awq（empero-ai/Qwythos-9B-v2 的 AWQ 量化版），目前線上是 vllm-server 雙卡 tensor parallel（port 8182，max_model_len 98304）；換模型/換參數、調整 --max-model-len、GPU KV cache 容量規劃、以及 pi/OmniRoute 端 context/maxTokens 快取同步都在這裡。當使用者要求「部署 vllm」「重啟 vllm」「換 vllm 模型」「vllm 上跑 qwythos」「qwythos 工具呼叫壞了/連不上」「vllm OCR/看圖壞了」「調 vllm context/context length」時使用。

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Procedure
1. Confirm the current environment and read the task-specific instructions already documented in this Skill.
2. Apply the smallest safe change that satisfies the request; confirm before destructive operations.
3. Run the checks in the Verification section and report actual results.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.

## Pitfalls
- Do not guess configuration paths or claim success without checking the resulting state.
- Do not execute copied commands or scripts before reviewing their targets and side effects.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
