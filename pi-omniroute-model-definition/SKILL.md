---
name: pi-omniroute-model-definition
description: >-
  在 pi 的 D:/.system/.pi/agent/models.json 裡為 OmniRoute（omni provider）動態回傳、
  但本機沒有明確定義的模型補上完整條目（maxTokens、input 含 image 等），
  修兩種典型故障：(1) 400 "max_tokens (512000) exceeds model's maximum output tokens"
  ——pi 對未定義模型用 contextWindow/2 當 max_tokens 送出，超過上游實際上限；
  (2) 圖片傳不出去 / 模型收不到 vision——本機條目缺 input:["text","image"]，
  pi 不允許對純 text 模型送圖片。涵蓋：models.json 結構、用 scripts/add_model.py
  安全新增/更新條目（自動備份）、用 scripts/omni_chat_test.py 直接打
  http://localhost:20128/v1/chat/completions 驗證（含 SSE 串流解析、圖片 base64、
  Windows cp950 終端機亂碼的規避）。當使用者回報 pi 打 omni 模型出現
  max_tokens 400 錯誤、要求「讓某模型支援看圖/vision/OCR」、或要新增/調整
  pi 裡的 omni 模型設定時使用。搭配 omniroute-native（OmniRoute 本體）
  與 pi-omni-model-test（用 pi 跑其他 LLM）兩個 skill 一起看。
---

# pi × OmniRoute：為模型補本機定義（maxTokens / vision）

## 背景：為什麼會出問題

pi 的模型能力（contextWindow、maxTokens、是否支援 image 輸入）以
`D:/.system/.pi/agent/models.json` → `providers.omni.models[]` 為準。
OmniRoute（`http://localhost:20128/v1`）會動態回傳一大堆模型
（`oc/*`、`ollama-cloud/*`、`QW/*`、`auto/*` 前綴等），但 models.json
只手工維護了其中一部分。對**沒有本機條目**的模型：

1. **maxTokens 預設陷阱**：pi 用 `contextWindow / 2` 當 max_tokens。
   contextWindow 1M 的模型 → 送出 512000 → 上游拒收：
   `400: max_tokens (512000) exceeds model's maximum output tokens (131072)`。
2. **vision 預設陷阱**：無條目或條目 `input:["text"]` 的模型，pi 不送圖片。

修法都是同一個：**在 models.json 的 omni provider 加入/更新該模型的明確定義**。

## 模型條目格式

```json
{
  "id": "ollama-cloud/minimax-m3",   // 與 pi 模型選擇器/OmniRoute 的 model id 完全一致
  "name": "MiniMax M3",
  "api": "omni-prompt-tools",        // omni provider 固定值
  "input": ["text", "image"],        // 有 vision 加 "image"；純 text 只留 "text"
  "contextWindow": 1048576,
  "maxTokens": 131072,               // 必須 ≤ 上游實際上限，寧可保守
  "reasoning": true                  // 有內建思考（會吃 maxTokens 額度）設 true
}
```

已知上限（2026-08 驗證）：
- `ollama-cloud/minimax-m3` → 131072（上游 400 訊息直接寫明）
- `QW/qwen3.8-27b`（Jetson 自架 VL）→ 8192，有 vision、有 reasoning_content

上游上限不確定時：先發一個小 max_tokens（如 64）的請求，若回 400 會直接
告訴你該模型的上限值。

## 標準流程

1. **查現況**（先確認條目是否存在、長什麼樣）：
   ```powershell
   python -c "import json;d=json.load(open(r'D:/.system/.pi/agent/models.json',encoding='utf-8'));[print(p,json.dumps(m,ensure_ascii=False)) for p,pp in d['providers'].items() for m in pp.get('models',[]) if '<關鍵字>' in m['id'].lower()]"
   ```

2. **新增/更新條目**（自動備份、id 重複時更新而非重複新增）：
   ```powershell
   python D:\OB\skills\pi-omniroute-model-definition\scripts\add_model.py `
     --model-id "QW/qwen3.8-27b" --name "Qwen 3.8 27B (Vision)" `
     --context-window 32768 --max-tokens 8192 --vision --no-reasoning
   ```
   參數說明：
   - `--vision` 加 `"image"` 到 input；不加則 `["text"]`
   - `--reasoning` / `--no-reasoning`（預設 --reasoning）
   - `--provider omni`（預設）；`--api omni-prompt-tools`（預設）
   - 每次執行先複製 `models.json.bak-<timestamp>`，寫完做 JSON 可解析驗證

3. **驗證上游 API**（不重啟 pi 就能測，直接打 OmniRoute）：
   ```powershell
   # 先在目前 process 安全提供 key；不要把 raw key 寫進 Skill 或 script
   $env:OMNIROUTE_API_KEY = Read-Host "OmniRoute API key"
   # 純 text
   python D:\OB\skills\pi-omniroute-model-definition\scripts\omni_chat_test.py --model "QW/qwen3.8-27b" --prompt "say ok"
   # vision（自動產生 3 色幾何圖 + OCR 文字圖測試）
   python D:\OB\skills\pi-omniroute-model-definition\scripts\omni_chat_test.py --model "QW/qwen3.8-27b" --prompt "列出圖中所有形狀與顏色，並說出圖中文字。簡短回答。" --image D:\OB\skills\pi-omniroute-model-definition\assets\vision_test.png
   ```
   腳本輸出 HTTP status、finish_reason、reasoning 字數、最終答案，
   並把完整 SSE 原始回應存到 `D:\omni_chat_test_<model-slug>.json` 備查。

4. **告知使用者重啟 pi / 新開 session** 才會讀到新 models.json。

## 已知坑

- **SSE 串流**：OmniRoute 對這些模型常回 `data: {...}` 串流（即使沒開 stream
  參數），直接 `json.loads(response)` 會爆。解析方式見 omni_chat_test.py；
  注意某些 chunk 的 `choices` 是空 list，要先 guard。
- **reasoning 吃額度**：有思考的模型（qwen3.8-27b 即使 pi 設 `reasoning:false`
  仍會思考），小 max_tokens 會 `finish_reason: length` 且 content 為空——
  不是 vision 壞掉，是思考把額度吃光。測試時給 2000~4000。
- **cp950 終端機亂碼**：Python 在 Windows 預設 stdout 是 cp950，print 繁中會
  UnicodeEncodeError 或亂碼。腳本一律先把結果寫 UTF-8 檔案；互動 print 時
  用 `sys.stdout.reconfigure(encoding='utf-8')` 或只打 ASCII。
- **max_tokens 別照抄 contextWindow/2**：那正是故障根源。上游 400 訊息裡的
  數字才是真的上限。
- **備份檔會堆**：models.json 旁已有多個 .bak-*；新備份用 timestamp 命名，
  不要覆蓋既有 .bak。
- **PIL 產生測試圖**：`assets/vision_test.png`（640×360：紅方塊/藍圓/綠三角 +
  "Hello Vision 2026" 文字）已預建；若缺可用 Pillow 重畫，或改用任何真實截圖。

## 驗證完成標準

1. `add_model.py` 輸出 "added" 或 "updated"，且 JSON 可解析。
2. `omni_chat_test.py` 純 text 回 200 + 合理答案。
3. 有 vision 的模型：圖片請求 200，答案正確辨識圖中元素/文字
   （2026-08-28 實測 QW/qwen3.8-27b 全數辨識正確）。
4. 已提醒使用者重啟 pi 讓 models.json 生效。

## When to Use

當 OmniRoute 動態提供的 model 沒有 Pi 明確 metadata，造成 `max_tokens` 超過上游上限、vision 圖片被拒絕，或需要為新 OmniRoute model 建立安全定義時使用。

## Inputs and Outputs

### Inputs

- OmniRoute model id、provider、上游實際 output token 上限及 vision 能力。
- `D:/.system/.pi/agent/models.json` 與必要的測試圖片。

### Outputs

- 明確的 model metadata（`api`、`input`、`contextWindow`、`maxTokens`、`reasoning`）。
- models.json 的 timestamped backup、更新結果與 text/vision 驗證結果。

## Procedure

1. 先讀取目前 `models.json`，確認 provider 與 model id 是否已存在。
2. 以 OmniRoute 實際回應或上游錯誤訊息確認 `maxTokens`，不要用 `contextWindow / 2` 代替。
3. 使用 Skill 內的 `scripts/add_model.py` 新增或更新條目，讓腳本建立備份並維持 JSON 格式。
4. 使用 `scripts/omni_chat_test.py` 先測 text，再依 model 宣告的能力測 vision。
5. 重啟 Pi 後重新確認 model picker 與實際請求。

## Rules and Limitations

- `maxTokens` 不得高於上游實際 output token 上限；有 reasoning 的 model 要預留思考 token。
- 只有實際支援圖片的 model 才能設定 `input: ["text", "image"]`，不能因 model 名稱猜測 vision。
- 不要直接覆蓋 models.json，也不要刪除既有 model 或 backup。
- 測試時遮蔽 credential；圖片測試只使用 Skill 內的測試資產或使用者授權的檔案。

## Pitfalls

- 未定義 model 可能讓 Pi 自動把 `contextWindow / 2` 當成 `max_tokens`，導致 400。
- 把 `image` 加到純文字 model 會讓請求在 Pi 端或上游失敗。
- 有 reasoning_content 的 model 若 max token 給太小，可能在產生答案前耗盡額度。
- OmniRoute `/v1/models` 的動態清單不等於每個 model 都有相同的上限或圖片能力。
