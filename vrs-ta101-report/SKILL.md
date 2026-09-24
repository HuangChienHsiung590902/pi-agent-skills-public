---
name: vrs-ta101-report
description: 當使用者要處理「座席狀態明細表」TA101.xlsx 報表（例如把報表最上面標題改成座席代碼、把嵌在時間欄裡的「分機(帳號)」拆成獨立的 id 欄），或要拿 VRS 服務紀錄列表跟 TA101 的「未就緒原因」（如電話進線未就緒）互相比對配對時使用此 skill。
version: 1.0.0
---

# TA101 座席狀態明細表 清理與比對

## 這個 Skill 的作用

處理 aipower/ECP 匯出的「座席狀態明細表」(TA101) 原始報表，並可進一步跟 VRS 服務紀錄列表互相比對，找出服務與未就緒事件的對應關係。

> VRS 系統其他需求（登入、抓資料、通話記錄）見 `vrs-service` / `vrs-calllog` 總覽。

## 何時做什麼

| 使用者說 | 該做什麼 |
|---------|---------|
| 「TA101 標題改成座席代碼」、「標題取括號內的值」 | 修改 A1 為公式 `=MID(...)` 或（清理後）`=A5`，取「分機(帳號)」括號內值 |
| 「加一個 id 欄」、「把座席代碼獨立出來」、「原始報表要整理」 | 跑 `scripts\scripts/clean_ta101.py` |
| 「服務記錄對應 TA101 未就緒」、「電話進線未就緒有沒有對應到服務」 | 跑 `scripts\scripts/match_service_to_ta101.py`，**先跟使用者確認方向與時間門檻**（見下方陷阱） |

## 原始 TA101 匯出格式（未清理前）

固定 3 欄 A:C：
- row1：title，merged A1:C1（例：「座席狀態明細表」）
- row2：A2:B2「報表編號：TA101」，C2「列印時間：...」
- row3：A3:C3「資料期間：...」
- row4：欄位標題 A4=時間 B4=座席狀態 C4=未就緒原因
- row5+：資料列與「分隔列」交錯 — 分隔列是整列合併儲存格，內容為 `分機(帳號)`（例：`5107(CS0007)`），代表接下來的資料列都屬於這個座席，直到下一個分隔列出現

`scripts/clean_ta101.py` 會把分隔列的括號內容（如 `CS0007`）拆成獨立的 `id` 欄放在最前面，並移除分隔列本身，輸出 4 欄 A:D（id/時間/座席狀態/未就緒原因），A1 標題改成公式 `=A5` 自動顯示第一筆資料的 id。**執行前務必詢問使用者是否要保留原分隔列**——曾有使用者選擇移除。

## ⚠️ 比對陷阱：時間方向跟直覺相反

用 2026-06~07 的真實資料分析過，「未就緒原因=電話進線未就緒」事件跟服務紀錄的時間關係，**不是**直覺以為的「電話先響 → 座席變未就緒 → 才開始服務」。

實際關係是「服務先開始，未就緒事件發生在其後」（可能是服務進行中/剛結束又有新電話進線）：
- 用「服務開始時間**之後**最近一筆」比對，5 分鐘內比對率約 65~71%
- 若誤用「之前最近一筆」，5 分鐘內比對率只有 ~2%（方向抓反的訊號）

**換一批新資料時**，不要predict 直接假設方向一致，先用 `--direction nearest` 搭配較寬的 window（例如 1800 秒）跑一次，看 after/before 的分佈再決定，或至少把統計結果攤開給使用者確認（例如 median 時間差、各 window 的比對率），因為使用者原本就是憑直覺猜錯方向。

## 重要路徑

| 用途 | 路徑 |
|------|------|
| 清理原始 TA101，拆 id 欄 | `scripts\scripts/clean_ta101.py` |
| TA101 ↔ 服務紀錄比對 | `scripts\scripts/match_service_to_ta101.py` |

兩支都是 CLI（argparse），用 `--help` 看參數。都是直接用 openpyxl 讀寫 xlsx，不需要 LibreOffice。

---

## Conformance Addendum

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
