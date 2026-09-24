---
name: skills-openjev-routing
description: >
  決定要不要用 OpenJEV 來挑選 D:\OB\skills 的 SKILL.md。預設答案是否定的：
  主路由永遠是 obsidian-mcp-skill-router（catalog / rg / 讀完整 SKILL.md），
  不要把 OpenJEV 當 skill 選擇器或取代現有索引。當使用者說「用 jev 選 skill」、
  「openjev 路由 skills」、「skills 改接 jev」、「jev 會不會比 skills 好」、
  「不要換 jev 寫成 skill」、「比較兩個選 skill 的效果」、「對照 jev 跟 skills 索引」、
  或想把問題丟給 10.145.119.19:8090 來挑 SKILL.md 時使用。含 2026-09-19 實測對照。
---

# Skills 主路由不要換成 OpenJEV

本機正式 skill library 在 `D:\OB\skills`。選哪個 `SKILL.md` 的預設方法是
`obsidian-mcp-skill-router`：從問題抽關鍵字，搜 `SKILLS_INDEX.md` /
`_consolidation/SKILLS_CATALOG.json` / 各份 `SKILL.md`，挑 1–3 個候選，
**讀完整檔再執行**。

OpenJEV（`http://10.145.119.19:8090`，skill `openjev`）是 NLI cross-encoder，
只會判 entailment / contradiction / neutral，**不會讀流程、不會生成、
也沒有 `/v1/chat/completions`**。2026-09-19 已決定：**不換成 JEV 當主路由**。

## When to Use

- 有人提議用 OpenJEV、NLI、`/rerank`、`/predict` 來決定載入哪份 skill。
- 比較「jev 選 skill」和「現有 D:\OB\skills 索引」誰比較好、要看實測數字。
- 想把使用者問題接到 `openjev-svc` 當 skill 路由器。
- 要把這項決策或對照結果寫進 library，避免之後的 agent 再做一次。

不適用：

- 一般任務選 skill：走 `obsidian-mcp-skill-router`，不要先打 OpenJEV。
- 真正的 NLI 用途（有參考答案評分、兩段文字是否矛盾）：走 `openjev`。
- 用生成式 LLM 整理 wiki：那是 LLM Wiki + Qwen 等 chat 模型，不是 JEV。

## Inputs and Outputs

### Inputs

- 使用者問題或「要不要改用 JEV 選 skill」的提案。
- 本機索引：`D:\OB\skills\SKILLS_INDEX.md`、
  `D:\OB\skills\_consolidation\SKILLS_CATALOG.json`（約 220 份頂層 skill）。
- 若有人堅持當第二段裁判：遠端 `http://10.145.119.19:8090` 的健康狀態。

### Outputs

- 預設：用 catalog / `rg` 得到的 1–3 個候選，並讀對應 `SKILL.md`。
- 若被問「JEV 會不會比較好」或「比較兩個效果」：明確回答「當主路由不會」，
  並引用下方 2026-09-19 實測（索引 top-1 5/7、hit@3 6/7、約 2ms；
  JEV 每 pair 8–17s，全庫 rerank 會 timeout）。
- 只有使用者明確要求第二段打分時，才對**已經縮小的候選**呼叫 OpenJEV，
  分數只當參考，執行前仍須讀檔。

## Procedure

1. **主路由不變。** 先載入並遵循 `obsidian-mcp-skill-router`：
   - 抽工具名、路徑、錯誤字、中英同義詞。
   - 搜 `D:\OB\skills` 檔案索引，不要為了選 skill 去打 `:8090`。
   - 挑 1–3 個候選，讀完整 `SKILL.md`。
2. **有人要把 JEV 當預設選擇器時直接拒絕**，改解釋：
   - 這個庫的 skill 名稱幾乎就是專案名，關鍵字命中率高、延遲接近 0。
   - OpenJEV `/rerank` 無參考答案時，知識密集題 ≈ 隨機（見 `openjev` skill）。
   - CPU 約 7 秒/次；全庫 200+ 份會又慢又不準。
   - `max_len = 4096`，塞不進完整 `SKILL.md`。
   - 有 contradiction bias，無關 skill 也可能被判成矛盾。
3. **真正容易錯的是相近 skill**（例如 `llm-wiki-docker` /
   `connect-llm-wiki` / `tauri-wiki-app`）。要改進就改各 skill 的
   `description` / `When to Use` 寫互斥，不要接 JEV。
4. **可選第二段（預設關閉）。** 只有同時滿足才可呼叫 OpenJEV：
   - 使用者這次明確要求用 JEV 當參考；或
   - 關鍵字已經縮到 3–5 個描述高度重疊的候選，且名稱對不上。
   - 作法：對每個候選用 `/predict`，premise 放「id + 截斷 description」，
     hypothesis 放「應該使用這個 skill 來處理：{使用者問題}」。
   - entailment 明顯高於其他候選才當加分。實測正確項可低於 0.6
     （MCP 題 `connect-llm-wiki` 只有 0.57），**不要死守 0.6 當唯一門檻**。
   - 兩個分數接近就**兩份都讀**。禁止只信第一名、禁止用分數取代讀檔。
   - 不要對整份 catalog 做 `/rerank`。正確項若沒被第一段索引撈到，JEV 也沒機會。
5. 呼叫細節與端點限制以 `openjev` skill 為準（timeout 拉長、CPU、無 API key）。
6. 索引選錯且正確項根本沒進前 5 時，優先改該 skill 的 `description` / `When to Use`
   （實測：`docker-remote-control` 的備份還原題），不要改接 JEV。

## 2026-09-19 實測對照

條件：catalog 224 份；OpenJEV `http://10.145.119.19:8090`，容器 `openjev-svc`，
**CPU**（`cuda: false`）。索引在本機對 8 題一次算完；JEV 在遠端 loopback 對
「正確項 vs 干擾項」各打一組 `/predict`（每題 3 候選從本機打會整批 timeout）。

### 延遲

| 方法 | 延遲 |
|---|---|
| 本機 catalog 關鍵字（224 份） | 約 **2 ms**（8 題一次） |
| OpenJEV `/predict` 一組 pair | **8–17 s**，偶發 timeout |
| 每題 3 個候選再打 JEV（從 Windows 打） | 常超過 2–3 分鐘或整批 timeout |

### 有標準答案的 7 題 + 1 個負例

| 題 | 正確 | 索引 top-1 | top-1 | hit@3 | JEV 正確項 e | JEV 干擾項 e |
|---|---|---|---|---|---|---|
| 把 llm-wiki 接到 pi 當 MCP | `connect-llm-wiki` | `connect-llm-wiki` | 是 | 是 | 0.57 | `llm-wiki-docker` 0.02 |
| 查 LLM Wiki 桌面頁並搜尋 | `tauri-wiki-app` | `wiki` | 否 | 是（第 2） | 0.75 | `wiki` 0.08 |
| 遠端 docker 無頭 llm-wiki 掛 BASE | `llm-wiki-docker` | `llm-wiki-docker` | 是 | 是 | （未單打） | — |
| jev 會不會比 skills 好／不要換 | `skills-openjev-routing` | `skills-openjev-routing` | 是 | 是 | （未單打） | — |
| 列出遠端 docker 容器表 | `docker-list` | `docker-list` | 是 | 是 | （未單打） | — |
| 遠端 docker 備份還原 compose | `docker-remote-control` | `docker-list` | 否 | **否**（沒進前 5） | 0.62 | `docker-list` 0.02 |
| openjev 怎麼呼叫 /predict | `openjev` | `openjev` | 是 | 是 | （未單打） | — |
| 明天台北熱晚餐吃什麼 | （不該選） | 無命中 | 正確拒絕 | — | vs `openjev` 0.02 | 偏低，當沒匹配 |

索引：**top-1 5/7（71%）**，**hit@3 6/7（86%）**，負例沒亂抓 skill。

JEV 在「已經餵對正確項與干擾項」時能分開分數，桌面 wiki 與備份題能糾正索引 top-1。
但它**不能自己從 224 份裡找出正確項**；備份題若第一段沒撈到 `docker-remote-control`，
第二段也沒用。MCP 正確項 entailment 只有 0.57，死守 0.6 會漏接。

### 怎麼讀這組數字

- 名稱對得上時，索引 2ms 內 top-1 正確；JEV 再等十幾秒沒有比較好。
- JEV 的局部價值只在相近 skill（`wiki` vs `tauri-wiki-app`）。
- 索引漏撈（`docker-remote-control`）要改 description，不要改主路由。
- 因此：**預設只用 skills 索引；JEV 不是第二套主路由。**

## Rules and Limitations

- 正式來源永遠是 `D:\OB\skills\<id>\SKILL.md`，不是 OpenJEV 的選項字串。
- 禁止把 OpenJEV 註冊成 skill provider、MCP skill 來源、或 LLM Wiki 的
  `llmConfig`。
- 禁止用 `/rerank` 對 8 個以上、或未先用關鍵字篩過的 skill 打分。
- OpenJEV 分數不能當「要不要讀 SKILL.md」的門檻；找不到匹配就說沒有，
  改走一般流程。
- 這份 skill 記錄的是路由政策，不取代 `openjev` 的 API 說明，也不取代
  router 的搜尋步驟。

## Pitfalls

- 「接上 JEV 比較聰明」是錯的：它不會讀 Procedure，只會看兩句像不像。
- 把完整 `SKILL.md` 丟進 `/predict` 會被截斷，分數沒意義。
- 第一次推論在 CPU 上可能超過 20 秒；拿它當每個問題的前置步驟會拖垮對話。
- 本機對每題 3 個候選打 `/predict` 曾整批 timeout；遠端 loopback 單 pair 約 8–17s。
- 服務健康（`GET /health` 200）不代表它適合選 skill。
- 不要為了用 JEV 去複製一份平行的 skills 索引。
- `entailment >= 0.6` 當硬門檻會漏掉實測中的正確 MCP 題（0.57）。

## Verification

1. `D:\OB\skills\skills-openjev-routing\SKILL.md` 存在，資料夾名與
   frontmatter `name` 都是 `skills-openjev-routing`。
2. `obsidian-mcp-skill-router` 仍是選 skill 的預設入口；本 skill 只在有人
   要改用 JEV 時出現。
3. `openjev` skill 的不適用表含「選 D:\OB\skills 的主路由」。
4. 問「jev 會不會比現在的 skills 方式好」或「比較兩個效果」時，答案應為
   「當主路由不會」，並引用本檔實測表與 `obsidian-mcp-skill-router`。
