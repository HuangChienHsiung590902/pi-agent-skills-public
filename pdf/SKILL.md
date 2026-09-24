---
name: pdf
version: 1.0.0
description: 進階 PDF 文件工具包，支援內容提取、文件生成、頁面操作和互動式表單處理。適用於解析 PDF 文字與表格、建立專業文件、合併或拆分檔案，或以程式化方式填寫可填寫表單。
description_zh: 進階 PDF 文件工具包，支援內容提取、文件生成、頁面操作和互動式表單處理。適用於解析 PDF 文字與表格、建立專業文件、合併或拆分檔案，或以程式化方式填寫可填寫表單。
license: Proprietary. LICENSE.txt has complete terms
---

# PDF 文件工具包

## 簡介

本工具包使用 Python 函式庫與 shell 工具，提供完整的 PDF 文件操作功能。進階用法、JavaScript API 及詳細程式碼範例，請參閱 advanced-guide.md。填寫 PDF 表單請參閱 form-handler.md 並遵循其工作流程。

## 重要：完成後驗證

**產生或修改 PDF 檔案後，務必驗證輸出是否有 CJK 文字渲染問題：**

1. **開啟產生的 PDF**，目視檢查所有文字內容
2. **檢查亂碼字元** — 留意以下情形：
   - 出現黑色方塊（■）或矩形，而非 CJK 字元
   - 出現問號（?）或替代字元（）
   - CJK 內容應出現之處文字缺失
   - 字元渲染不正確或重疊
3. **若發現問題**，請參閱下方「CJK（中日韓）文字支援」章節的字型設定解決方案

當 PDF 包含中文、日文或韓文文字時，此驗證步驟至關重要。

## 快速開始

```python
from pypdf import PdfReader, PdfWriter

# 開啟 PDF 文件
doc = PdfReader("sample.pdf")
print(f"Total pages: {len(doc.pages)}")

# 擷取文字內容
content = ""
for pg in doc.pages:
    content += pg.extract_text()
```

## Python 函式庫

### pypdf - 核心操作

#### 合併多個 PDF
```python
from pypdf import PdfWriter, PdfReader

output = PdfWriter()
for pdf in ["first.pdf", "second.pdf", "third.pdf"]:
    doc = PdfReader(pdf)
    for pg in doc.pages:
        output.add_page(pg)

with open("combined.pdf", "wb") as out_file:
    output.write(out_file)
```

#### 拆分 PDF 頁面
```python
doc = PdfReader("source.pdf")
for idx, pg in enumerate(doc.pages):
    output = PdfWriter()
    output.add_page(pg)
    with open(f"part_{idx+1}.pdf", "wb") as out_file:
        output.write(out_file)
```

#### 讀取文件屬性
```python
doc = PdfReader("sample.pdf")
props = doc.metadata
print(f"Title: {props.title}")
print(f"Author: {props.author}")
print(f"Subject: {props.subject}")
print(f"Creator: {props.creator}")
```

#### 旋轉文件頁面
```python
doc = PdfReader("source.pdf")
output = PdfWriter()

pg = doc.pages[0]
pg.rotate(90)  # 順時針旋轉 90 度
output.add_page(pg)

with open("turned.pdf", "wb") as out_file:
    output.write(out_file)
```

### pdfplumber - 內容提取

#### 含版面配置的文字提取
```python
import pdfplumber

with pdfplumber.open("sample.pdf") as doc:
    for pg in doc.pages:
        content = pg.extract_text()
        print(content)
```

#### 提取表格資料
```python
with pdfplumber.open("sample.pdf") as doc:
    for pg_num, pg in enumerate(doc.pages):
        data_tables = pg.extract_tables()
        for tbl_num, tbl in enumerate(data_tables):
            print(f"Table {tbl_num+1} on page {pg_num+1}:")
            for row in tbl:
                print(row)
```

#### 將表格匯出至 Excel
```python
import pandas as pd

with pdfplumber.open("sample.pdf") as doc:
    collected_tables = []
    for pg in doc.pages:
        data_tables = pg.extract_tables()
        for tbl in data_tables:
            if tbl:  # 確認表格非空
                df = pd.DataFrame(tbl[1:], columns=tbl[0])
                collected_tables.append(df)

# 合併所有表格
if collected_tables:
    merged_df = pd.concat(collected_tables, ignore_index=True)
    merged_df.to_excel("tables_export.xlsx", index=False)
```

### reportlab - 文件生成

#### 建立簡單 PDF
```python
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

c = canvas.Canvas("greeting.pdf", pagesize=letter)
width, height = letter

# 插入文字
c.drawString(100, height - 100, "Welcome!")
c.drawString(100, height - 120, "Generated using reportlab library")

# 繪製分隔線
c.line(100, height - 140, 400, height - 140)

# 儲存文件
c.save()
```

#### 產生多頁文件
```python
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph, Spacer, PageBreak
from reportlab.lib.styles import getSampleStyleSheet

doc = SimpleDocTemplate("document.pdf", pagesize=letter)
styles = getSampleStyleSheet()
elements = []

# 新增內容
heading = Paragraph("Document Title", styles['Title'])
elements.append(heading)
elements.append(Spacer(1, 12))

body_text = Paragraph("This is the main content section. " * 20, styles['Normal'])
elements.append(body_text)
elements.append(PageBreak())

# 第二頁
elements.append(Paragraph("Section 2", styles['Heading1']))
elements.append(Paragraph("Content for the second section", styles['Normal']))

# 產生 PDF
doc.build(elements)
```

## Shell 工具

### pdftotext (poppler-utils)
```bash
# 轉換為文字
pdftotext source.pdf result.txt

# 保留版面配置格式
pdftotext -layout source.pdf result.txt

# 轉換指定頁面範圍
pdftotext -f 1 -l 5 source.pdf result.txt  # 第 1-5 頁
```

### qpdf
```bash
# 合併文件
qpdf --empty --pages doc1.pdf doc2.pdf -- result.pdf

# 提取頁面範圍
qpdf source.pdf --pages . 1-5 -- subset1-5.pdf
qpdf source.pdf --pages . 6-10 -- subset6-10.pdf

# 旋轉指定頁面
qpdf source.pdf result.pdf --rotate=+90:1  # 將第 1 頁旋轉 90 度

# 解密受保護的 PDF
qpdf --password=secret --decrypt protected.pdf unlocked.pdf
```

### pdftk（若可用）
```bash
# 合併文件
pdftk doc1.pdf doc2.pdf cat output result.pdf

# 拆分為單頁
pdftk source.pdf burst

# 旋轉頁面
pdftk source.pdf rotate 1east output turned.pdf
```

## 常見操作

### 掃描文件 OCR
```python
# 需安裝：pip install pytesseract pdf2image
import pytesseract
from pdf2image import convert_from_path

# 將 PDF 頁面轉換為圖片
pages = convert_from_path('scanned.pdf')

# 對每頁執行 OCR
content = ""
for idx, img in enumerate(pages):
    content += f"Page {idx+1}:\n"
    content += pytesseract.image_to_string(img)
    content += "\n\n"

print(content)
```

### 套用浮水印
```python
from pypdf import PdfReader, PdfWriter

# 載入浮水印（或自行建立）
watermark_page = PdfReader("stamp.pdf").pages[0]

# 套用至所有頁面
doc = PdfReader("sample.pdf")
output = PdfWriter()

for pg in doc.pages:
    pg.merge_page(watermark_page)
    output.add_page(pg)

with open("stamped.pdf", "wb") as out_file:
    output.write(out_file)
```

### 匯出嵌入圖片
```bash
# 使用 pdfimages (poppler-utils)
pdfimages -j source.pdf img_prefix

# 輸出：img_prefix-000.jpg、img_prefix-001.jpg 等
```

### 新增文件密碼
```python
from pypdf import PdfReader, PdfWriter

doc = PdfReader("source.pdf")
output = PdfWriter()

for pg in doc.pages:
    output.add_page(pg)

# 設定密碼
output.encrypt("user_pwd", "admin_pwd")

with open("secured.pdf", "wb") as out_file:
    output.write(out_file)
```

## 快速參考表

| 操作 | 建議工具 | 範例 |
|------|----------|------|
| 合併文件 | pypdf | `output.add_page(pg)` |
| 拆分文件 | pypdf | 每頁各輸出一個檔案 |
| 提取文字 | pdfplumber | `pg.extract_text()` |
| 提取表格 | pdfplumber | `pg.extract_tables()` |
| 建立文件 | reportlab | Canvas 或 Platypus |
| Shell 合併 | qpdf | `qpdf --empty --pages ...` |
| 掃描 OCR | pytesseract | 先轉換為圖片 |
| 填寫 PDF 表單 | pdf-lib 或 pypdf（參見 form-handler.md） | 參見 form-handler.md |

## CJK（中日韓）文字支援

**重要**：標準 PDF 字型（Arial、Helvetica 等）不支援 CJK 字元。若在未指定適當 CJK 字型的情況下使用 CJK 文字，字元將顯示為黑色方塊（■）。

### 自動字型偵測

`apply_text_overlays.py` 工具會自動：
1. 偵測文字內容中的 CJK 字元
2. 搜尋系統上可用的 CJK 字型
3. **若偵測到 CJK 字元但找不到 CJK 字型，則以錯誤訊息退出**

### 支援的系統字型

| 作業系統 | 字型路徑 |
|----------|----------|
| macOS | `/System/Library/Fonts/PingFang.ttc`、`/System/Library/Fonts/STHeiti Light.ttc` |
| Windows | `C:/Windows/Fonts/msyh.ttc`（微軟正黑體）、`C:/Windows/Fonts/simsun.ttc` |
| Linux | `/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc`、`/usr/share/fonts/truetype/wqy/wqy-zenhei.ttc` |

### 若出現「找不到 CJK 字型」錯誤

請為您的作業系統安裝 CJK 字型：

```bash
# Ubuntu/Debian
sudo apt-get install fonts-noto-cjk

# Fedora/RHEL
sudo dnf install google-noto-sans-cjk-fonts

# macOS - 已預裝 PingFang
# Windows - 已預裝微軟正黑體
```

### 手動字型註冊（用於 reportlab）

直接使用 reportlab 時，請在繪製文字前先註冊 CJK 字型：

```python
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont

# 註冊 CJK 字型（以 macOS 為例）
# 注意：TTC（TrueType Collection）檔案需指定 subfontIndex 參數
pdfmetrics.registerFont(TTFont('PingFang', '/System/Library/Fonts/PingFang.ttc', subfontIndex=0))

# 使用字型繪製 CJK 文字
c.setFont('PingFang', 14)
c.drawString(100, 700, '你好世界')      # 中文
c.drawString(100, 680, 'こんにちは')    # 日文
c.drawString(100, 660, '안녕하세요')    # 韓文
```

**TTC 檔案常見 subfontIndex 值：**
- PingFang.ttc：0（Regular）、1（Medium）、2（Semibold）等
- msyh.ttc：0（Regular）、1（Bold）
- NotoSansCJK-Regular.ttc：依語言變體而異

詳細 CJK 字型設定請參閱 form-handler.md。

## 其他資源

- pypdfium2 進階用法，請參閱 advanced-guide.md
- JavaScript 函式庫（pdf-lib），請參閱 advanced-guide.md
- 填寫 PDF 表單，請遵循 form-handler.md 中的說明
- 疑難排解提示，請參閱 advanced-guide.md

---

## Conformance Addendum

## When to Use
進階 PDF 文件工具包，支援內容提取、文件生成、頁面操作和互動式表單處理。適用於解析 PDF 文字與表格、建立專業文件、合併或拆分檔案，或以程式化方式填寫可填寫表單。

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
