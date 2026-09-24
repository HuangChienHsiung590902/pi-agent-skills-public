---
name: unsloth-vl-training
description: 在遠端 GPU 主機 10.145.119.19（gigabyte，見 [[docker-remote-control]]）用 Docker 跑 unsloth，對 Qwen3-VL 系列視覺語言模型做 LoRA 微調或 teacher-student logit 蒸餾。當使用者要求「unsloth 微調」「用 unsloth 蒸餾」「微調 Qwen3-VL」「開 unsloth 容器」「連 unsloth studio/jupyter」時使用。容器叫 unsloth-studio，跟 GPU 0 上常駐的 vllm-server（見 [[vllm-qwythos-deploy]]）共用主機，訓練前後要注意互相搶 GPU。
---

# Unsloth 視覺語言模型訓練環境（10.145.119.19）

## 架構總覽

- 主機：`hch@10.145.119.19`（gigabyte），2x RTX 4060 Ti 16GB，54GB RAM
- 容器：`unsloth-studio`，image `unsloth/unsloth:latest`（官方維護，非 unsloth/unslothai 命名空間下那些都是模型 repo，別找錯）
- Workspace：主機 `/home/hch/unsloth-workspace` ↔ 容器 `/workspace`（bind mount，容器可以砍掉重建，這裡的東西不會丟）
- 對外服務：Jupyter Lab `:8888`、Unsloth Studio 網頁 GUI `:8000`

**GPU 共用注意**：GPU 0 平常固定給 `vllm-server`（qwythos 部署，見 [[vllm-qwythos-deploy]]）。要跑 8B 以上的 teacher 或需要雙卡時，得先 `docker stop vllm-server` 讓出 GPU 0，**訓練/蒸餾做完記得 `docker start vllm-server` 恢復**，不要放著不管。

## 建立容器

```bash
ssh hch@10.145.119.19 "docker run -d --name unsloth-studio \
  --gpus all --shm-size=8g \
  -p 8888:8888 -p 8000:8000 \
  -e JUPYTER_PASSWORD=<你的密碼> \
  -v /home/hch/unsloth-workspace:/workspace \
  unsloth/unsloth:latest"
```

**踩過的坑，务必注意：**

1. **一定要 `--gpus all`，不要用 `--gpus '\"device=N\"'` 限定單卡** —— 就算現在只想用一張卡，之後要做跨卡（teacher/student 分開放、大模型跨卡分層）時，容器完全看不到另一張卡（`nvidia-smi`/`torch.cuda.device_count()` 只會回報 1），bnb 4-bit 的 `device_map="sequential"` 也會直接噴 `Device 1 is not available`。改 GPU 可見度沒有熱更新這回事，只能整個容器砍掉重建（workspace 不受影響）。
2. 首次啟動會跑一次性設定（Studio 前端 build + 依 GPU 架構裝 flash-attn wheel），約 1.5~2 分鐘，看 `docker logs unsloth-studio` 出現 `success: studio entered RUNNING state` 才算好。
3. 密碼**務必在 `docker run` 時就用 `-e JUPYTER_PASSWORD=` 設好**。事後才想改密碼很麻煩：Jupyter 用密碼登入（不是 token），設定檔在容器內 `/home/unsloth/.jupyter/jupyter_lab_config.py`，且這個 image 的 `supervisord` 是用 `/etc/supervisor/conf.d/supervisord.conf` 啟動的，那份**沒有** `[rpcinterface:supervisor]` 區塊，所以 `supervisorctl` 完全用不了（會報 "did not recognize the supervisor namespace commands"）。事後改密碼的正確流程：
   ```python
   # 在容器內: /opt/venv/bin/python3 這段腳本
   from jupyter_server.auth import passwd
   password_hash = passwd("新密碼")
   # 讀取 config_path，把 c.ServerApp.password = '...' 那行整行取代成新 hash
   ```
   改完設定檔後，**不要**用 `pkill -f jupyter-lab`（模式字串會連 `docker exec sh -c` 這個外層指令自己都匹配到，把自己也殺了）。改用 `pgrep -f "jupyter-lab --no-browser"` 先精準拿到 PID 再 `kill -TERM`；supervisord 的 `autorestart=true` 會自動重新拉起 jupyter，新行程啟動時就會讀到新密碼。

## 模型能不能塞進顯存，先算過再下載

Qwen3-VL 官方尺寸只有 2B / 4B / 8B / 32B（dense）、30B-A3B / 235B-A22B（MoE），**沒有 27B、35B 這種中間規格**，聽到使用者講奇怪的數字先去 `unsloth` HF namespace 查一次實際存在的 repo 名稱再往下走。

實測結論：
- **32B（dense）就算 4-bit 量化、跨兩張 16GB GPU（合計 ~31GB 可用）也裝不下。** 不是設定錯誤——語言模型主體 4-bit 後 ~16-18GB，但 vision encoder、embedding、lm_head 這些模組 bnb 不量化、维持 fp16，加起來會超過 31GB，`device_map="sequential"`/`"auto"` 會把裝不下的模組丟去 CPU，然後 bnb 4-bit quantizer 直接報錯拒絕（要開 `llm_int8_enable_fp32_cpu_offload=True` 才能硬做，但每步都要 CPU↔GPU 搬資料，訓練會明顯變慢）。
- **8B（4-bit）單卡 16GB 綽綽有餘**（實測 peak 7.67GB），這個量級拿來當 teacher 或直接微調都很穩，是目前驗證過能跑的上限。
- 2B（4-bit）+ LoRA 顯存需求很低（SFT 實測 peak 3.9GB，蒸餾裡當 student 實測 peak 3.4GB），CPU/GPU 資源緊張時的預設選擇。

## 已驗證可行的兩套流程

### 1. 一般 SFT LoRA 微調（單模型）

用官方 notebook 改模型 ID 即可，流程來自 `unslothai/notebooks` repo 的 `Qwen3_VL_(8B)-Vision.ipynb`（把模型換成 2B 版）：

```python
from unsloth import FastVisionModel
model, tokenizer = FastVisionModel.from_pretrained(
    "unsloth/Qwen3-VL-2B-Instruct-unsloth-bnb-4bit",
    load_in_4bit=True,
    use_gradient_checkpointing="unsloth",
)
model = FastVisionModel.get_peft_model(model, r=16, lora_alpha=16, ...)
# dataset: unsloth/LaTeX_OCR，convert_to_conversation 包成 messages 格式
# UnslothVisionDataCollator + trl.SFTTrainer，SFTConfig 裡要加：
#   remove_unused_columns=False, dataset_text_field="", dataset_kwargs={"skip_prepare_dataset": True}
```

存放位置：`/workspace/Qwen3_VL_2B_Vision.ipynb`（乾淨版）、`Qwen3_VL_2B_Vision.executed.ipynb`（跑過含完整輸出，30 步約 1.3 分鐘，peak 3.9GB）。

執行整份 notebook（非互動、驗證用）：
```bash
docker exec unsloth-studio /opt/venv/bin/jupyter nbconvert --to notebook --execute \
  --ExecutePreprocessor.timeout=1800 \
  --output <輸出檔名>.ipynb /workspace/<筆記本>.ipynb
```

### 2. Teacher-Student Logit 蒸餾

架構：Teacher 8B 固定 `device_map={"":0}`（凍結、`requires_grad=False`），Student 2B+LoRA 固定 `device_map={"":1}`。兩邊用同一個 processor/tokenizer 準備 batch（Qwen3-VL 全系列共用同一份 tokenizer，vocab size 151669 對得起來，不用擔心跨尺寸不相容）。

**兩個非常隱蔽的坑，會讓 loss 完全失真或直接 crash：**

1. **`.logits` 預設是空的。** unsloth 2024.11 後訓練/推論預設不回傳完整 logits（省顯存），存取 `.logits` 會拿到一個 `raise_logits_error` 佔位函式，一 subscript 就 `TypeError: 'function' object is not subscriptable`。要看 dense logits（蒸餾一定要）得設 `os.environ["UNSLOTH_RETURN_LOGITS"] = "1"`。**关键：`FastVisionModel.for_training(model)` 呼叫時內部會把這個環境變數強制蓋回 `"0"`**（`for_inference()` 則會設回 `"1"`），所以順序一定是：先呼叫完所有 `for_inference`/`for_training`，最後再手動 `os.environ["UNSLOTH_RETURN_LOGITS"] = "1"`，不要以为在腳本開頭 import 前設一次就夠。
2. **`F.kl_div(reduction="batchmean")` 對 3D tensor（batch, seq, vocab）的坑**：`batchmean` 只除以 `input.size(0)`（也就是 batch size），不會連 `seq_len` 一起除，導致 loss 被序列長度放大好幾百倍（實測 seq_len=186 時 loss 從正常的 ~2 灌水到 ~850）。正確做法：先用 `labels != -100` 篩掉 padding/image token 位置，再把 `(batch, seq, vocab)` 攤平成 `(N, vocab)` 兩維，才餵給 `kl_div`。

驗證通過的 loss 公式：
```python
mask = labels.view(-1) != -100
student_flat = student_logits.view(-1, vocab_size)[mask]
teacher_flat = teacher_logits.view(-1, vocab_size)[mask]
kl = F.kl_div(
    F.log_softmax(student_flat / T, dim=-1),
    F.softmax(teacher_flat / T, dim=-1),
    reduction="batchmean",
) * (T ** 2)
loss = alpha * kl + (1 - alpha) * student_out.loss  # student_out.loss 是內建的 CE loss
```

完整腳本：`/workspace/train_distill.py`（T=2.0, alpha=0.5）。已從「20 步煙霧測試」升級成「正式訓練」版本，差異：

- 資料集用完整 `unsloth/LaTeX_OCR`（68,686 筆，`shuffle(seed=3407)`），用 index 取樣（`get_batch(start_idx, batch_size)`）現場轉換，不要學舊版把整個資料集先 eager 轉成 python list——6.8 萬張圖一次全部解出來太吃主機記憶體
- `BATCH_SIZE=4`（沒有做 gradient accumulation，這個自寫迴圈裡單純每步一個 batch）、`MAX_STEPS` 依需求調（可調參數都在檔案最上面）
- `torch.optim.lr_scheduler.LinearLR` 從 1.0 線性衰減到 0.1
- 每 `CHECKPOINT_EVERY`（預設 200）步存一次 LoRA 到 `OUTPUT_DIR`（`/workspace/distilled_qwen3vl2b_lora`，覆蓋式，非累加版本），長跑中斷也不會全部白費
- 每步印 `elapsed`/`eta`（用累計平均步時反推，訓練前幾步較慢會導致 eta 偏保守，跑到 ~50 步後才會準）

**實測吞吐量**（8B teacher GPU0 + 2B LoRA student GPU1，batch=4，LaTeX_OCR）：穩定後約 **0.4~0.45 秒/步**，1000 步全程約 18~19 分鐘（比最初粗估的「每步 2.6~5 秒」快很多——原本的估計是從 SFT-only 的 30 步煙霧測試線性外推，沒算到教師是 eval+no_grad 模式，重疊 H2D 跟 compute 之後遠比預期快）。抓時間預算可以用「steps × 0.45 秒」當基準，正式開跑前先看前 50 步的 `eta` 數字收斂再決定要不要調整步數。

Loss 走勢參考：α=0.5、T=2 下，KL 項從 ~1.6 開始，數百步內收斂到 ~0.3~0.5 附近（batch 小、噪音大是正常的，看多步移動平均而非單步數字）。

跑完別忘記：`ssh hch@10.145.119.19 "docker start vllm-server"` 把 GPU 0 還給 qwythos。

**⚠️ 窄領域全層 LoRA 蒸餾會犧牲通用對話能力，這是預期行為不是 bug：** 用 `finetune_vision_layers/language_layers/attention_modules/mlp_modules` 全開的 LoRA，在單一窄任務資料集（例如全部都是「圖片→LaTeX」這種格式）上跑到 1000 步，模型會嚴重過擬合這個任務型態，**通用聊天能力被覆蓋掉**。實測現象：拿訓練時同型態的輸入（圖片 + 轉錄指令）測試，輸出跟 ground truth 幾乎一致、正常結束；但丟一句純文字聊天（沒有圖片，訓練資料裡沒出現過的輸入型態）進去，模型會答非所問，且在 Studio 的取樣參數下容易陷入重複生成迴圈吐不出 EOS（看起來像「卡住」，其實是 GPU 仍在跑、只是生成失控——用 `nvidia-smi` 看 GPU 是否仍有 utilization 可以分辨是真卡住還是失控生成）。

**判斷模型是不是真的訓練壞掉，還是只是「窄化」了**：一定要用訓練資料同型態的輸入測試（例如這裡要附圖片再問轉錄指令），跟訓練時沒見過的輸入型態（純文字閒聊）分開測，不要只用一句「你是誰」就下結論模型壞了。

如果目的真的是需要模型**同時保留通用能力**（不是只做單一窄任務的專家模型），下次訓練要做的調整：
- 資料集混入一部分通用指令/對話資料，不要 100% 都是單一任務格式
- 縮小 LoRA 覆蓋範圍（例如只開 `finetune_language_layers`，關掉 vision/attention/mlp 部分模組），降低對整體行為的擾動
- 降 `MAX_STEPS` 或降 learning rate，避免對窄任務過擬合太深

## Studio 網頁 GUI 拿來跟自己訓練的模型對話

**卡住的生成怎麼強制中止**：Studio 的 `/api/inference/cancel`（body: `cancel_id`/`session_id`/`completion_id`）很難從後端直接命中前端內部生成的 ID，實測傳 log 裡看到的 `request_id` 也對不上（回傳 `{"cancelled":0}`）。真正有效的方法是直接呼叫 `/api/inference/unload`（body: `{"model_path": "<模型路徑>"}`），會強制把模型從推論後端卸載，GPU 顯存立刻釋放，卡住的生成請求也會跟著終止。判斷是不是真的卡住 vs 介面問題：`nvidia-smi` 看該 GPU 的 `utilization.gpu` 是不是持續非 0——是的話代表還在真的生成（多半是失控重複生成），不是連線/前端問題。

Studio（`:8000`）**有獨立的帳號系統，跟 Jupyter 密碼完全無關**，用 JWT 登入。首次啟動會在 SQLite 裡建一個帳號：

- 使用者名稱固定 `unsloth`
- 密碼是隨機 diceware 密語，寫在 `<studio_root>/auth/.bootstrap_password`（一次性，正常走 UI 改密碼後會被清掉）
- `studio_root` 預設 `~/.unsloth/studio`，但這個 image 的 supervisord 把 `UNSLOTH_STUDIO_HOME` 設成 `/workspace/studio`（在 bind mount 裡，容器砍掉重建也留著）——**用 `docker exec` 手動查 auth db 路徑時，一定要自己補上 `-e UNSLOTH_STUDIO_HOME=/workspace/studio`，不然會查到 `~/.unsloth/studio` 這個從沒被用過的預設路徑，誤判成「auth.db 不存在」**

如果 bootstrap 密碼檔已經不見（例如容器重建過，DB 留下來但密碼沒人記錄），直接改資料庫重設，不用走忘記密碼流程：
```bash
docker exec -e UNSLOTH_STUDIO_HOME=/workspace/studio unsloth-studio /opt/venv/bin/python3 -c "
import sys; sys.path.insert(0, '/opt/venv/lib/python3.12/site-packages/studio/backend')
from auth import storage
storage.update_password('unsloth', '新密碼')
"
```

**API 可以完全跳過瀏覽器操作**（這台 Windows 的工作目錄常在 `C:\Program Files\...` 之類沒有寫入權限的地方，Playwright MCP 開瀏覽器會因為建不了 profile 資料夾直接失敗——遇到這狀況別硬試，改走 curl/API）：

```bash
# 登入拿 token（access_token 有效期看 main.py 設定，過期重登就好）
TOKEN=$(curl -s -X POST http://127.0.0.1:8000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"unsloth","password":"<密碼>"}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')

# 把 /workspace 加成自訂掃描資料夾（不會自動出現在 recommended-folders，那個只認 LM Studio/Ollama 的固定路徑）
curl -s -X POST http://127.0.0.1:8000/api/models/scan-folders \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"path":"/workspace"}'

# 確認掃描結果（認 adapter_config.json 辨識 LoRA，會自動關聯回 base model）
curl -s http://127.0.0.1:8000/api/models/local -H "Authorization: Bearer $TOKEN"
```
以上都要在容器內執行（`docker exec unsloth-studio sh -c "..."`），因為是打 `127.0.0.1:8000`。路由前綴：認證 `/api/auth`，模型管理 `/api/models`。

加完資料夾後，`distilled_qwen3vl2b_lora`、`qwen_lora`、`outputs/checkpoint-*` 這些訓練產物就會出現在 Studio 的模型清單裡，可以直接在內建 Chat 對話測試效果，或匯出成 GGUF/16-bit safetensors 給 llama.cpp/vLLM/Ollama 用。

## 常用操作

跑腳本（改完先 scp 到 workspace，因為容器內用 unsloth 使用者、host 上是 hch，兩邊 UID 不同，workspace 目錄權限有時要 `chmod o+w` 才能讓容器寫入）：
```bash
scp <本機腳本> hch@10.145.119.19:/home/hch/unsloth-workspace/<檔名>
ssh hch@10.145.119.19 "docker exec unsloth-studio /opt/venv/bin/python3 /workspace/<檔名>"
```

GPU / 容器狀態：
```bash
ssh hch@10.145.119.19 "nvidia-smi --query-gpu=index,memory.used,memory.total,utilization.gpu --format=csv"
ssh hch@10.145.119.19 "docker ps --filter name=unsloth-studio; docker logs unsloth-studio --tail 40"
```

---

## Conformance Addendum

## When to Use
在遠端 GPU 主機 10.145.119.19（gigabyte，見 [[docker-remote-control]]）用 Docker 跑 unsloth，對 Qwen3-VL 系列視覺語言模型做 LoRA 微調或 teacher-student logit 蒸餾。當使用者要求「unsloth 微調」「用 unsloth 蒸餾」「微調 Qwen3-VL」「開 unsloth 容器」「連 unsloth studio/jupyter」時使用。容器叫 unsloth-studio，跟 GPU 0 上常駐的 vllm-server（見 [[vllm-qwythos-deploy]]）共用主機，訓練前後要注意互相搶 GPU。

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
