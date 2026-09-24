---
name: qwencloud-token-plan-custom-provider
description: >-
  把 QwenCloud Token Plan（sk-sp- 專用金鑰）接到 WorkBuddy、Cherry Studio、Chatbox
  或其他 OpenAI 相容自訂提供商。用在：WorkBuddy「新增模型」不知填什麼、
  測試連線出現「API Key 無效或沒有許可權訪問該模型」、誤把 sk-sp- 配到
  dashscope-intl.aliyuncs.com、要選 qwen3.8-flash / qwen3.8-max、或要區分
  Token Plan 與按需付費（sk- / sk-ws-）兩套完全不能混用的 endpoint。
  不適用於本機自架 GGUF Qwen3.8-27B（見 qwen38-27b-llamacpp-omniroute）。
---

# QwenCloud Token Plan → OpenAI 相容自訂提供商

## When to Use

使用者出現以下任一情況時使用本 skill：

- WorkBuddy / Cherry Studio / Chatbox「新增自訂模型」，要接 QwenCloud
- 金鑰以 `sk-sp-` 開頭，或畫面是 QwenCloud「代幣計劃 / Token Plan」
- 連線失敗、401、`API Key 無效`、`Incorrect API key provided`、`invalid access token or token expired`
- 介面地址填了 `dashscope-intl.aliyuncs.com` 或 `dashscope.aliyuncs.com`

不要用本 skill：

- 本機 / 遠端自架 Qwen GGUF（`qwen38-27b-llamacpp-omniroute`）
- 按需付費 key（`sk-` / `sk-ws-`）走一般 DashScope（那是另一組 base URL）

## 先讀對照表

詳細 endpoint、模型 ID、401 對照見：

- [references/endpoints-and-models.md](references/endpoints-and-models.md)

核心規則（必記）：

| 金鑰前綴 | 計費 | OpenAI 相容 Base URL |
|---|---|---|
| `sk-sp-` | Token Plan 訂閱額度 | `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1` |
| `sk-` / `sk-ws-` | 按需付費 | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1` |

**兩套 key 與 URL 不可互換。** `sk-sp-` 打 dashscope 一定 401。

官方來源：

- https://docs.qwencloud.com/token-plan/personal/token-plan-personal-quickstart.md
- https://docs.qwencloud.com/api-reference/preparation/api-key.md
- https://docs.qwencloud.com/token-plan/personal/token-plan-personal-faq.md

## Procedure

1. **辨識金鑰類型**
   - `sk-sp-` → Token Plan，繼續本流程。
   - 其他前綴 → 不要套 Token Plan URL。
   - 金鑰只顯示一次；對話或截圖出現完整 key 時，先請使用者到 QwenCloud API 金鑰頁**重新產生**，再填新 key。不要把完整 key 寫進檔案、記憶或回覆。

2. **填第三方工具（WorkBuddy 實測）**

   | 欄位 | 值 |
   |---|---|
   | 提供商 | 自定義 |
   | 介面地址 | `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1/chat/completions` |
   | API Key | Token Plan 的 `sk-sp-`（只貼進該軟體） |
   | 模型名稱 | `qwen3.8-flash`（省額度）或 `qwen3.8-max`（較強） |
   | 工具呼叫 | 勾 |
   | 圖片輸入 | 視覺需求才勾；文字模型先不勾 |
   | 思考模式 | qwen3.8 系列可勾（會回 `reasoning_content`、多耗 Credits） |
   | 自定義協議 | 不勾 |
   | 輸入 | 128K |
   | 輸出 | 8K |

3. **測試連線**
   - 在工具內按「測試連線」。
   - 失敗且錯誤像 401 / API Key 無效：先確認介面地址**不是** dashscope。
   - 若工具會自己補 `/chat/completions`，改填 base：
     `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`
   - 也可用本 skill 腳本（金鑰只放環境變數）：

     ```powershell
     $env:QWENCLOUD_TOKEN_PLAN_KEY = '<貼在本機終端，不要貼到聊天>'
     python D:\OB\skills\qwencloud-token-plan-custom-provider\scripts\test_token_plan.py --model qwen3.8-flash
     ```

     成功條件：`GET /models` 與極小 `chat/completions` 都 HTTP 200，模型有回文字。

4. **模型名稱必須與允許清單逐字相符**
   - 個人版常用：`qwen3.8-flash`、`qwen3.8-max`、`qwen3.7-plus`、`qwen3.7-max`、`qwen3.6-flash`
   - 不要填 `qwen-plus`、`qwen3-coder-plus`（那是別套產品／按需付費 ID）
   - 完整清單以官方與 [references/endpoints-and-models.md](references/endpoints-and-models.md) 為準

5. **額度**
   - 個人 Standard：7 天視窗 10,000 Credits；用完會暫停，等到視窗結束或用重置卡。
   - 測連線用 `qwen3.8-flash` + 極小 `max_tokens`，不要拿來跑長對話或批次腳本。
   - Token Plan Individual 官方用途是個人互動式開發工具，不是生產自動化。

## Pitfalls

- 最常見錯：WorkBuddy 介面地址 placeholder 是 `https://api.example.com/v1/chat/completions`，被填成 `dashscope-intl.aliyuncs.com/...`。`sk-sp-` 必須用 `token-plan.ap-southeast-1.maas.aliyuncs.com`。
- 官方 FAQ：`Incorrect API key provided` = 用了 dashscope；`invalid access token or token expired` = 用了別種計費 Base URL。
- 有的客戶端要完整 `/v1/chat/completions`，有的只要 `/compatible-mode/v1`。先試完整路徑，失敗再改 base。
- qwen3.8 即使只回 `OK` 也可能產生 `reasoning_content`，Credits 會比表面 token 多。
- 金鑰出現在聊天、截圖、log 就視為外洩，立刻在 QwenCloud 重新產生。
- 本 skill 不存放 API key；腳本只讀 `QWENCLOUD_TOKEN_PLAN_KEY`。

## Verification

1. `D:\OB\skills\qwencloud-token-plan-custom-provider\SKILL.md` 存在，frontmatter `name` 與資料夾名相同。
2. 用 Token Plan URL 測通：HTTP 200 且模型有回覆。用 dashscope URL 應 401（證明沒混用）。
3. WorkBuddy「測試連線」成功後才能當設定完成。
4. 索引含本 skill：`SKILLS_INDEX.md` 與 `_consolidation/SKILLS_CATALOG.json`。

## Inputs and Outputs

- **Input:** 金鑰前綴、第三方工具欄位（介面地址／模型名）、連線錯誤原文。
- **Output:** 正確 Base URL 與模型 ID、WorkBuddy 欄位表、連線測試結果（不含完整 key）。

## Rules and Limitations

- 相對路徑以本 Skill 資料夾為準。
- endpoint / 模型清單可能改版；與官方文件衝突時以 `docs.qwencloud.com` 為準。
- 禁止把使用者 API key 寫進 skill、記憶、repo 或回覆全文。
