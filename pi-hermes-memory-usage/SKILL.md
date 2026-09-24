---
name: pi-hermes-memory-usage
description: 說明 pi-hermes-memory 擴充功能如何自動運行、哪些機制是自動的、哪些需要手動觸發，以及安裝檢查與疑難排解方式。當使用者詢問記憶擴充功能運作機制、memory_add/memory_search/skill_manage/session_search 等工具背後的自動化邏輯時使用。
---

# pi-hermes-memory 記憶擴充功能使用說明

## When to Use
當使用者詢問 pi-hermes-memory 記憶擴充功能是否會自動運行、記憶何時被儲存、如何安裝/檢查/更新此擴充功能，或想了解 memory_add/memory_search/skill_manage/session_search 等工具背後的自動化機制時使用。

## Procedure
1. 安裝：執行 `pi install npm:pi-hermes-memory`，套件會寫入 `~/.pi/agent/settings.json` 的 packages 清單，本體位於 `~/.pi/agent/npm/node_modules/pi-hermes-memory`（此環境中 `~/.pi` 是指向 `D:\.system\.pi` 的 symlink）
2. 檢查安裝是否正常：1) `pi list` 確認套件列出 2) 檢查 better-sqlite3 原生模組是否可用：`node -e "require('better-sqlite3')"`（若報 NODE_MODULE_VERSION 不符，需在該套件目錄跑 `npm rebuild better-sqlite3`）3) 檢查 sessions.db 完整性：用 better-sqlite3 開啟該檔案並執行 `db.pragma('integrity_check')` 應回傳 ok 4) 用 `pi -p "..." --no-session` 開一個新 session，請它列出以 memory/skill_manage/session_search 開頭的工具名稱，確認擴充功能真的被載入
3. 了解自動運行機制（預設值全部啟用，除非使用者自訂 `~/.pi/agent/hermes-memory-config.json`）：背景審查每 10 回合或每 15 次工具呼叫觸發一次，自動判斷值得記住的內容並寫入；使用者糾正 Agent 時（如「不對，用 pnpm 不是 npm」）會立即觸發 correction 記憶，不等背景審查；session 結束前（flushOnShutdown）或即將被 compact 壓縮前（flushOnCompact）會自動 flush 記憶；每次 session 結束會自動把對話索引進 sessions.db 供 session_search 使用；MEMORY.md/USER.md 等寫滿 5000 字元上限時會自動觸發 consolidation 整併，不會直接報錯；單一回合內用了 8 次以上工具且涉及 2 種以上不同工具時，會主動詢問使用者要不要存成技能（這步是問使用者，不是全自動寫入）
4. 了解需要手動觸發的部分：`/memory-index-sessions` 一次性補索引安裝前的舊對話到搜尋資料庫（新對話會自動索引）；`/memory-sync-markdown` 把舊 Markdown 記憶回填進 SQLite 搜尋庫；`/memory-consolidate` 手動觸發整併（一般不需要，容量滿了會自動做）；`/memory-interview`、`/memory-pin`、`/memory-skills` 等是使用者主動要調整/查看設定時才用的指令
5. 更新擴充功能：`pi update --extension npm:pi-hermes-memory` 或 `pi update --extensions` 更新所有已裝套件；這些斜線指令（/memory-index-sessions 等）以及套件更新後的生效，都需要在互動式 Pi TUI session 中執行/重啟，無法從既有 session 內部代為執行

## Pitfalls
- 在同一個既有的 Pi session 內無法自己重啟 Pi 或執行 /memory-index-sessions 這類斜線指令，必須告知使用者去終端機手動開新 session 執行
- 用 `node -e` 直接 require 絕對路徑字串會報 MODULE_NOT_FOUND，要先 cd 進套件目錄或用正確的 require.resolve 路徑，不是模組真的遺失
- 此環境中 `~/.pi` 是 symlink 指向 `D:\.system\.pi`，檢查設定檔時要注意實際落地路徑在 D 槽，符合使用者「主要設定檔在 D:\」的規則
- background review、correction save、consolidation 這些 LLM 驅動的操作在 reviewTransport=direct（預設）下走 in-process completeSimple()，失敗才 fallback 到 subprocess `pi -p`；不要誤以為每次都會開子行程
- MEMORY.md/USER.md 有 5000 字元硬上限，若 memoryOverflowStrategy 設為 reject 則寫滿時會直接報錯而非自動整併，需先確認設定檔內容再下結論

## Verification
1. `pi list` 顯示 npm:pi-hermes-memory 且路徑存在
2. 新開 session 用 `pi -p` 讓 Agent 列出 memory_add/memory_replace/memory_remove/memory_search/skill_manage/session_search 工具名稱，全部都要出現
3. 測試寫入：呼叫 memory_add 寫一筆測試資料，檢查 `~/.pi/agent/pi-hermes-memory/MEMORY.md` 是否出現該內容，再用 memory_remove 清除確認可正常刪除
4. sessions.db 用 better-sqlite3 開啟執行 integrity_check 回傳 ok

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
