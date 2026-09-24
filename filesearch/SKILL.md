---
name: filesearch
description: 統一檔案搜尋與比較工具。自動判斷使用 es（找檔名）、ripgrep（找字串）或 WinMerge（比較差異）。
---

# FileSearch — 檔案搜尋與比較

你有三個 MCP 工具可以使用，根據使用者的意圖自動選擇：

## 工具路由規則

### 1. 找檔案（名稱/路徑）→ es
**觸發情境**：使用者想找「某個檔案在哪」、「有哪些 .json 檔」、「找 config 相關的檔案」

工具：`es_search`、`es_count`、`es_total_size`

```
使用者說：「哪裡有 config.yaml？」
使用者說：「D:\Dev 裡有哪些 .log 檔？」
使用者說：「找所有名稱含 backup 的檔案」
```

**常用參數**：
- `query`：搜尋文字，支援 Everything 語法（`ext:pdf`、`size:>1mb`、`dm:今天`）
- `search_path`：限制搜尋目錄
- `files_only` / `folders_only`：只要檔案或資料夾
- `max_results`：限制筆數（預設 50）
- `show_size`、`show_date_modified`：顯示額外欄位
- `sort`：依 name / size / date-modified 等排序

---

### 2. 找字串（檔案內容）→ ripgrep
**觸發情境**：使用者想找「某段程式碼」、「哪些檔案含有某個字串」、「找 TODO 註解」

工具：`rg_search`（顯示內容）、`rg_files`（只列檔案）、`rg_count`（統計次數）

```
使用者說：「哪些檔案有 API_KEY？」
使用者說：「在 D:\Dev 找所有 console.log」
使用者說：「找 Python 檔裡的 import pandas」
```

**常用參數**：
- `pattern`：搜尋文字或正規表達式（支援 PCRE2）
- `path`：搜尋目錄
- `fixed_strings: true`：把 pattern 當純文字（不用正規式）
- `case_insensitive`：忽略大小寫
- `file_type`：`js`、`ts`、`py`、`json`、`xml` 等
- `glob`：`*.ts`、`*.{js,ts}` 等
- `context`：顯示匹配行前後幾行
- `files_only`：只列有匹配的檔案名
- `max_results`：預設 100

---

### 3. 比較差異 → WinMerge
**觸發情境**：使用者想比較「兩個檔案哪裡不同」、「新舊版本的差異」、「兩個資料夾有沒有差別」

工具：
- `winmerge_open`：開啟 GUI（互動式比較）
- `winmerge_check`：靜默比較，回傳相同/不同（不開視窗）
- `winmerge_report`：產生 HTML 差異報告

```
使用者說：「比較 a.json 和 b.json」
使用者說：「Dev 和 Prod 資料夾有差異嗎？」
使用者說：「幫我產生兩個設定檔的差異報告」
```

**常用參數**：
- `left` / `right`：比較的兩個路徑（必填）
- `middle`：第三個路徑（三方比較）
- `recursive`：遞迴比較子資料夾
- `filter`：只比較特定檔案類型，如 `*.json`
- `ignore_whitespace` / `ignore_case` / `ignore_eol`：忽略選項
- `report_path`：HTML 報告輸出路徑（`winmerge_report` 專用）

---

## 組合使用流程

當需求複雜時，串接工具：

```
1. 先用 es_search 找到目標檔案
2. 再用 rg_search 確認檔案內容
3. 再用 winmerge_open 與另一版本比對
```

## 執行原則

1. **先確認意圖**：是找檔名、找字串、還是比較差異？
2. **預設值**：es max_results=50、rg max_results=100，避免輸出過多
3. **路徑格式**：Windows 路徑一律用正斜線（`D:/Dev/`）傳給工具
4. **無結果時**：調整 pattern、放寬過濾條件、或詢問使用者確認路徑
5. **WinMerge 開 GUI**：呼叫 `winmerge_open` 後告知使用者視窗已開啟，不需等待

## 附錄：Everything 查詢語法完整參考

`es_search` 的 `query` 參數背後就是 Everything 的查詢語法，MCP 工具沒有涵蓋到的細節可以直接寫在 `query` 裡：

| 語法 | 說明 |
|--------|-------------|
| `text` | 包含文字 |
| `text1 text2` | AND（同時包含兩者） |
| `text1 \| text2` | OR（包含任一） |
| `ext:pdf` 或 `ext:py;js;ts` | 副檔名（可用 `;` 列多個） |
| `size:>1mb` / `size:<100kb` | 檔案大小篩選 |
| `dm:2026-01-01..2026-03-01` | 修改日期範圍（`dm:今天` 也可以） |
| `path:Documents` | 路徑包含「Documents」 |
| `!text` | NOT（不包含） |
| `-r "pattern"` | 正規表達式搜尋（例：`-r "^doc.*2025"`） |

常用路徑捷徑：`C:\Users\HCH\Desktop`、`Downloads`、`Documents`、`Pictures`、`Videos`。

若 MCP 工具（`es_search` 等）不可用，可退回直接呼叫底層 CLI 二進位檔：
```
C:\Users\HCH\.config\opencode\bin\es.exe <query> [-path "..."] [-n 20] [-sort-date-modified-descending]
```
指令列旗標（`-path`、`-n`、`-sort-*`）與 MCP 工具的參數（`search_path`、`max_results`、`sort`）是同一件事的兩種介面，效果相同。

---

## Conformance Addendum

## When to Use
統一檔案搜尋與比較工具。自動判斷使用 es（找檔名）、ripgrep（找字串）或 WinMerge（比較差異）。

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
