---
name: pi-hermes-memory-session-backfill
description: >
  修 pi-hermes-memory 啟動警告 Session backfill complete: … (1 file error)。
  用在：JSONL 缺少 type=session header、截斷的 session 檔每次啟動都報 file error、
  parseSessionFile 回 null、或 pi update 後該警告又出現要重補 parser。
---

# pi-hermes-memory session backfill file error

## When to Use

啟動 Pi 看到：

```text
Warning: 🧠 Session backfill complete: 0 indexed, N skipped, 0 messages (1 file error)
```

或 `session_search` 漏掉某次對話、某個 `.jsonl` 沒有第一行 `{"type":"session",...}`。

一般機制與安裝檢查改走 [`pi-hermes-memory-usage`](../pi-hermes-memory-usage/SKILL.md)。auto-review / `omni-prompt-tools` 改走 [`pi-hermes-memory-config-tune`](../pi-hermes-memory-config-tune/SKILL.md)。

## Inputs and Outputs

- **Input:** 啟動警告原文、sessions 目錄、`pi-hermes-memory` 套件路徑。
- **Output:** 可解析的 JSONL（必要時）、可選的 parser 修補、索引 `errors: []` 的驗證結果。

## Procedure

1. **確認不是 OmniRoute 壞掉。** 警告來自 `pi-hermes-memory` 的 `formatBackfillResult()`，`file error` = `parseSessionFile()` 回 `null` 或讀檔例外。
2. **掃描缺 header 的 JSONL。** 相對路徑 `scripts/scan_session_headers.py`：
   ```powershell
   python D:\OB\skills\pi-hermes-memory-session-backfill\scripts\scan_session_headers.py
   ```
   預設掃 `D:\.system\.pi\agent\sessions`。`desktop.ini` 可忽略（不是 `.jsonl`）。
3. **修檔（最小修復）。** 對缺 `type=session` 的檔，用檔名還原 header 後 prepend。先備份到 `%TEMP%\pi-work\pi-hermes-memory-session-backfill\`，不要寫進 CWD。
   ```powershell
   python D:\OB\skills\pi-hermes-memory-session-backfill\scripts\repair_session_header.py --apply
   ```
   檔名格式：`2026-09-05T05-03-47-469Z_<uuid>.jsonl`  
   目錄編碼：`--C--Users-HCH--` → `C:\Users\HCH`  
   前半段對話若已截斷，補 header **不能復原遺失內容**；`sessions.db` 裡若早已有該 session 的 messages，搜尋可能仍找得到舊內容。
4. **耐久修補（避免下次再警告）。** 上游 `parseSessionFile()` 沒 header 就當 error，且不會寫入 `session_files` metadata，於是**每次啟動都重試**。把推斷邏輯補進已安裝套件：
   - `D:\.system\.pi\agent\npm\node_modules\pi-hermes-memory\src\store\session-parser.ts`
   - `D:\.system\.pi\agent\npm\node_modules\pi-hermes-memory\src\store\session-indexer.ts`
   - 對照 [`references/parser-patch.md`](references/parser-patch.md)
   - 先把原檔備份到暫存目錄。
5. **驗證。** 用 bun 載入修補後的套件跑 `parseSessionFile` + `indexChangedSessions`，確認 `errors: []`。本 session **無法**執行 `/memory-index-sessions` 或重啟 Pi。
6. **告知使用者開新 Pi session。** 目前行程仍是舊 parser。新 session 不應再出現 `(1 file error)`。

## Rules and Limitations

- 相對路徑以本 Skill 資料夾為準。
- `~/.pi` 是 `D:\.system\.pi` 的 symlink；改設定與套件一律用 D 槽真實路徑。
- 不要重建整份 `sessions.db`、不要刪歷史 session 列。
- `session_files.session_id` 有 FK 指向 `sessions(id)`，無法為「完全無法識別的檔」寫 dummy metadata；此類檔改 skip、不當 error。
- `pi update npm:pi-hermes-memory` **會蓋掉** node_modules 修補。重開解決不了蓋檔；更新後要再套 `references/parser-patch.md`。
- 不要在正在寫入的目前 session JSONL 上 prepend（先對照檔名 uuid）。

## Pitfalls

- `0 indexed, N skipped` 裡的 skipped 多半是 metadata 已相符，不是失敗。
- 同一個 Pi session 不能熱重載套件，也不能自己重開。
- `C:\Users\HCH\.pi\...` 與 `D:\.system\.pi\...` 是同一批檔；`session_files` 可能同時有兩種 path，不要當成兩份資料去刪。
- 空 `.jsonl` 但檔名合法：修補後的 parser 應推斷出 id/timestamp/cwd、0 則訊息，不當 error。
- bun 可直接 import 套件的 `.ts`；系統 Node 的 type-strip 不會把 `.js` specifier 對到 `.ts`。

## Verification

1. `scan_session_headers.py` 對現有 `.jsonl` 不再列出 missing header（或只剩正在寫的空檔且可推斷）。
2. bun 跑 `indexChangedSessions` 得到 `errors: []`。
3. `sessions.db` `PRAGMA integrity_check` 為 `ok`。
4. **新開** Pi session 啟動訊息沒有 `(1 file error)`。
