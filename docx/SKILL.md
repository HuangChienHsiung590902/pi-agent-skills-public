---
name: docx
version: 1.0.0
description: "Advanced Word document toolkit for content extraction, document generation, page manipulation, and interactive form processing. Use when you need to parse DOCX text and tables, create professional documents, combine or split files, or complete fillable forms programmatically."
description_zh: "進階 Word 文件工具集，支援內容提取、文件生成、頁面操作和互動式表單處理。用於解析 DOCX 文字和表格、建立專業文件、合併或拆分檔案，或以程式化方式填寫表單。"
license: Proprietary. LICENSE.txt has complete terms
---

# Word 文件處理工具集

## 概述

使用者可能會要求您生成、修改或提取 .docx 檔案中的內容。.docx 檔案本質上是一個包含 XML 檔案和其他資源的 ZIP 壓縮包，您可以讀取或編輯它。針對不同任務，您有不同的工具和工作流程可供使用。

## 工作流程選擇指南

### 內容提取與分析
使用下方的「文字提取」或「原始 XML 存取」章節

### 文件生成
使用「生成新 Word 文件」工作流程

### 文件修改
- **自己的文件 + 簡單變更**
  使用「基本 OpenXML 修改」工作流程

- **第三方文件**
  使用**「修訂追蹤工作流程」**（建議預設）

- **法律、學術、商業或政府文件**
  使用**「修訂追蹤工作流程」**（必要）

## 內容提取與分析

### 文字提取
若您只需讀取文件的文字內容，應使用 pandoc 將文件轉換為 markdown。Pandoc 對於保留文件結構提供了優秀的支援，並可顯示追蹤的變更：

```bash
# Convert document to markdown with tracked changes
pandoc --track-changes=all path-to-file.docx -o output.md
# Options: --track-changes=accept/reject/all
```

### 原始 XML 存取
以下情況需要原始 XML 存取：註解、複雜格式、文件結構、嵌入媒體和中繼資料。對於這些功能，您需要解壓縮文件並讀取其原始 XML 內容。

#### 解壓縮檔案
`python openxml/scripts/extract.py <office_file> <output_directory>`

#### 關鍵檔案結構
* `word/document.xml` - 文件主要內容
* `word/comments.xml` - document.xml 中參照的註解
* `word/media/` - 嵌入的圖片和媒體檔案
* 追蹤變更使用 `<w:ins>`（插入）和 `<w:del>`（刪除）標籤

## 生成新 Word 文件

從頭生成新 Word 文件時，使用 **docx-js**，它允許您使用 JavaScript/TypeScript 建立 Word 文件。

### 工作流程
1. **必要步驟 - 讀取完整檔案**：從頭到尾完整讀取 [`word-generator.md`](word-generator.md)（約 500 行）。**讀取此檔案時絕對不要設定任何範圍限制。** 在進行文件建立之前，讀取完整檔案內容以了解詳細語法、關鍵格式規則和最佳實踐。
2. 使用 Document、Paragraph、TextRun 元件建立 JavaScript/TypeScript 檔案（可假設所有相依套件已安裝，若未安裝，請參閱下方相依性章節）
3. 使用 Packer.toBuffer() 匯出為 .docx

## 修改現有 Word 文件

修改現有 Word 文件時，使用 **WordFile 函式庫**（一個用於 OpenXML 操作的 Python 函式庫）。此函式庫自動處理基礎架構設置，並提供文件操作的方法。對於複雜情境，您可以透過函式庫直接存取底層 DOM。

### 工作流程
1. **必要步驟 - 讀取完整檔案**：從頭到尾完整讀取 [`office-xml-spec.md`](office-xml-spec.md)（約 600 行）。**讀取此檔案時絕對不要設定任何範圍限制。** 讀取完整檔案內容以了解 WordFile 函式庫 API 和直接編輯文件檔案的 XML 模式。
2. 解壓縮文件：`python openxml/scripts/extract.py <office_file> <output_directory>`
3. 使用 WordFile 函式庫建立並執行 Python 腳本（請參閱 office-xml-spec.md 中的「WordFile 函式庫」章節）
4. 打包最終文件：`python openxml/scripts/assemble.py <input_directory> <office_file>`

WordFile 函式庫提供常見操作的高階方法以及複雜情境的直接 DOM 存取。

## 文件審查的修訂追蹤工作流程

此工作流程允許您在用 markdown 規劃全面的追蹤變更後，再於 OpenXML 中實作。**重要**：若要完整追蹤變更，您必須系統性地實作所有變更。

**批次策略**：將相關變更分組為 3-10 個變更的批次。這使除錯更易管理，同時保持效率。在進入下一批次之前，先測試每個批次。

**原則：最小化、精確的編輯**
實作追蹤變更時，只標記實際發生變更的文字。重複未變更的文字會使編輯難以審查，且顯得不專業。將替換分解為：[未變更文字] + [刪除] + [插入] + [未變更文字]。透過從原始 `<w:r>` 元素提取並重複使用它，為未變更的文字保留原始 run 的 RSID。

範例 - 將句子中的「30 days」改為「60 days」：
```python
# BAD - Replaces entire sentence
'<w:del><w:r><w:delText>The term is 30 days.</w:delText></w:r></w:del><w:ins><w:r><w:t>The term is 60 days.</w:t></w:r></w:ins>'

# GOOD - Only marks what changed, preserves original <w:r> for unchanged text
'<w:r w:rsidR="00AB12CD"><w:t>The term is </w:t></w:r><w:del><w:r><w:delText>30</w:delText></w:r></w:del><w:ins><w:r><w:t>60</w:t></w:r></w:ins><w:r w:rsidR="00AB12CD"><w:t> days.</w:t></w:r>'
```

### 追蹤變更工作流程

1. **取得 markdown 表示**：將文件轉換為保留追蹤變更的 markdown：
   ```bash
   pandoc --track-changes=all path-to-file.docx -o current.md
   ```

2. **識別並分組變更**：審查文件並識別所有需要的變更，將其組織為邏輯批次：

   **位置方法**（用於在 XML 中尋找變更）：
   - 章節/標題編號（例如：「第 3.2 節」、「第 IV 條」）
   - 若已編號，則使用段落識別符
   - 使用唯一周圍文字的 Grep 模式
   - 文件結構（例如：「第一段」、「簽名區塊」）
   - **不要使用 markdown 行號** - 它們不對應 XML 結構

   **批次組織**（每批次分組 3-10 個相關變更）：
   - 按章節：「批次 1：第 2 節修訂」、「批次 2：第 5 節更新」
   - 按類型：「批次 1：日期更正」、「批次 2：當事人名稱變更」
   - 按複雜度：從簡單文字替換開始，然後處理複雜結構性變更
   - 按順序：「批次 1：第 1-3 頁」、「批次 2：第 4-6 頁」

3. **讀取文件並解壓縮**：
   - **必要步驟 - 讀取完整檔案**：從頭到尾完整讀取 [`office-xml-spec.md`](office-xml-spec.md)（約 600 行）。**讀取此檔案時絕對不要設定任何範圍限制。** 特別注意「WordFile 函式庫」和「追蹤變更模式」章節。
   - **解壓縮文件**：`python openxml/scripts/extract.py <file.docx> <dir>`
   - **記錄建議的 RSID**：解壓縮腳本會建議一個用於追蹤變更的 RSID。複製此 RSID 供步驟 4b 使用。

4. **分批次實作變更**：邏輯性地分組變更（按章節、按類型或按鄰近性），並在單一腳本中一起實作。此方法：
   - 使除錯更容易（批次越小 = 越容易隔離錯誤）
   - 允許漸進式進度
   - 保持效率（3-10 個變更的批次大小效果良好）

   **建議的批次分組：**
   - 按文件章節（例如：「第 3 節變更」、「定義」、「終止條款」）
   - 按變更類型（例如：「日期變更」、「當事人名稱更新」、「法律術語替換」）
   - 按鄰近性（例如：「第 1-3 頁的變更」、「文件前半部分的變更」）

   對於每批次相關變更：

   **a. 將文字對應至 XML**：在 `word/document.xml` 中 Grep 文字，以驗證文字如何跨 `<w:r>` 元素分割。

   **b. 建立並執行腳本**：使用 `locate_element` 尋找節點，實作變更，然後執行 `doc.persist()`。請參閱 office-xml-spec.md 中的**「WordFile 函式庫」**章節了解模式。

   **注意**：在撰寫腳本之前，始終立即 grep `word/document.xml` 以取得當前行號並驗證文字內容。每次腳本執行後行號都會改變。

5. **打包文件**：所有批次完成後，將解壓縮目錄轉換回 .docx：
   ```bash
   python openxml/scripts/assemble.py unpacked reviewed-document.docx
   ```

6. **最終驗證**：對完整文件進行全面檢查：
   - 將最終文件轉換為 markdown：
     ```bash
     pandoc --track-changes=all reviewed-document.docx -o verification.md
     ```
   - 驗證所有變更均已正確套用：
     ```bash
     grep "original phrase" verification.md  # Should NOT find it
     grep "replacement phrase" verification.md  # Should find it
     ```
   - 確認沒有引入非預期的變更


## 將文件轉換為圖片

若要以視覺方式分析 Word 文件，請使用兩步驟流程將其轉換為圖片：

1. **將 DOCX 轉換為 PDF**：
   ```bash
   soffice --headless --convert-to pdf document.docx
   ```

2. **將 PDF 頁面轉換為 JPEG 圖片**：
   ```bash
   pdftoppm -jpeg -r 150 document.pdf page
   ```
   這會建立 `page-1.jpg`、`page-2.jpg` 等檔案。

選項：
- `-r 150`：將解析度設為 150 DPI（調整以平衡品質/檔案大小）
- `-jpeg`：輸出 JPEG 格式（若偏好 PNG 則使用 `-png`）
- `-f N`：要轉換的第一頁（例如：`-f 2` 從第 2 頁開始）
- `-l N`：要轉換的最後一頁（例如：`-l 5` 在第 5 頁停止）
- `page`：輸出檔案的前綴

特定範圍的範例：
```bash
pdftoppm -jpeg -r 150 -f 2 -l 5 document.pdf page  # Converts only pages 2-5
```

## 程式碼風格指南
**重要**：為 DOCX 操作生成程式碼時：
- 撰寫簡潔的程式碼
- 避免冗長的變數名稱和重複操作
- 避免不必要的 print 陳述式

## 相依性

必要的相依套件（若未安裝請先安裝）：

- **pandoc**：`sudo apt-get install pandoc`（用於文字提取）
- **docx**：`npm install -g docx`（用於建立新文件）
- **LibreOffice**：`sudo apt-get install libreoffice`（用於 PDF 轉換）
- **Poppler**：`sudo apt-get install poppler-utils`（用於 pdftoppm 將 PDF 轉換為圖片）
- **defusedxml**：`pip install defusedxml`（用於安全的 XML 解析）

---

## Conformance Addendum

## When to Use
Advanced Word document toolkit for content extraction, document generation, page manipulation, and interactive form processing. Use when you need to parse DOCX text and tables, create professional documents, combine or split files, or complete fillable forms programmatically.

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
