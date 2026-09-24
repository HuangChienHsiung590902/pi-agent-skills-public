---
name: pandoc
description: 在不同格式之間轉換文件（Markdown、HTML、DOCX、PDF、LaTeX、ODT、EPUB 等）。當使用者想要轉換、變換或匯出文件；將 Markdown 轉為 HTML/PDF/DOCX；或在 Word、LibreOffice 及其他文件格式之間互轉時使用。
---

# Pandoc 文件轉換器

Pandoc 3.x — 通用文件轉換器，支援 40 種以上格式。

## 支援格式

| 類別 | 格式 |
|------|------|
| 標記語言 | `markdown`, `gfm`（GitHub Flavored）, `commonmark` |
| 文件 | `docx`, `odt`, `rtf` |
| 網頁 | `html`, `html5` |
| 列印 | `latex`, `pdf` |
| 電子書 | `epub`, `epub3` |
| 大綱 | `docbook`, `opendocument` |
| 其他 | `texinfo`, `asciidoc`, `reStructuredText`, `txt`（純文字） |

## 核心語法

```bash
pandoc [OPTIONS] input_file -o output_file
```

## 常用轉換

### Markdown → HTML
```bash
pandoc input.md -o output.html
```

### Markdown → DOCX（Word）
```bash
pandoc input.md -o output.docx
```

### Markdown → PDF（需要 LaTeX）
```bash
pandoc input.md -o output.pdf
```

### DOCX → Markdown
```bash
pandoc input.docx -o output.md
```

### HTML → Markdown
```bash
pandoc input.html -o output.md
```

### DOCX → HTML
```bash
pandoc input.docx -o output.html
```

## 基本選項

| 旗標 | 用途 |
|------|------|
| `-o FILE` | 輸出檔案 |
| `-f FORMAT` | 輸入格式（通常自動偵測） |
| `-t FORMAT` | 輸出格式（通常依副檔名自動偵測） |
| `-s, --standalone` | 產生包含頁首/頁尾的獨立文件 |
| `--wrap=auto\|none\|preserve` | 換行方式（預設：auto） |
| `--dpi=NUMBER` | 設定 PDF/PNG 輸出的 DPI |
| `-M KEY=VAL` | 傳遞中繼資料變數 |

### 獨立輸出（含頁首）
```bash
pandoc input.md -s -o output.html   # 完整 HTML（含 <head>）
pandoc input.md -s -o output.tex    # 完整 LaTeX 文件
```

### 明確指定格式
```bash
pandoc -f markdown -t html input.txt -o output.html
```

## 中繼資料與變數

### 設定文件標題 / 作者 / 日期
```bash
pandoc input.md -o output.pdf \
  -M title="My Document" \
  -M author="John Doe" \
  -M date="2025-01-01"
```

### 使用自訂樣式的參考 DOCX
```bash
pandoc input.md -o output.docx \
  --reference-doc=template.docx
```

## 表格轉換

Pandoc 自動偵測 Markdown 表格：

```markdown
| Column 1 | Column 2 | Column 3 |
|----------|----------|----------|
| Data     | Data     | Data     |
```

**轉換 HTML/DOCX 中的表格時，請務必使用管道表格語法。**

## Markdown 擴充功能

| 擴充功能 | 啟用功能 |
|----------|----------|
| `+tex_math_dollars` | `$...$` 行內 LaTeX 數學式 |
| `+raw_tex` | 原始 LaTeX 直通 |
| `+pipe_table_tables` | 管道表格語法 |
| `+autolink_bare_uris` | 自動連結裸 URL |

### 啟用擴充功能
```bash
pandoc -f markdown+tex_math_dollars+raw_tex input.md -o output.html
```

## Lua 過濾器（進階）

Pandoc 支援 Lua 過濾器以進行自訂轉換：

```bash
pandoc input.md --filter=myfilter.lua -o output.pdf
```

常見用途：章節編號、自訂引用、條件式內容。

## PDF 輸出選項

輸出 PDF 需要安裝 LaTeX 發行版。

### 最佳 PDF 做法
```bash
pandoc input.md -o output.pdf
```

### 使用 ConTeXt
```bash
pandoc input.md --pdf-engine=contexttex -o output.pdf
```

### 使用 XeLaTeX（支援 Unicode）
```bash
pandoc input.md --pdf-engine=xelatex -o output.pdf
```

## EPUB 轉換

```bash
pandoc input.md -o output.epub
```

### 附帶中繼資料檔案
```bash
pandoc input.md \
  --epub-metadata=meta.xml \
  -o output.epub
```

## 批次轉換

將目錄中所有 `.md` 檔案轉換為 HTML：

```powershell
Get-ChildItem *.md | ForEach-Object {
    pandoc $_.FullName -s -o "$($_.BaseName).html"
}
```

或使用 bash：
```bash
for f in *.md; do pandoc "$f" -s -o "${f%.md}.html"; done
```

## 常用工作流程

### Markdown → Word（清晰格式）
```bash
pandoc input.md -s --reference-doc=blank.docx -o output.docx
```

### Markdown → Reveal.js 簡報（HTML）
```bash
pandoc -t revealjs input.md -s -o output.html
```

### Markdown → Beamer（LaTeX 簡報）
```bash
pandoc -t beamer input.md -o output.pdf
```

### Markdown → Org-mode
```bash
pandoc -t org input.md -o output.org
```

## 疑難排解

- **PDF 失敗**：安裝 MiKTeX 或 TeX Live（`choco install miktex`）
- **DOCX 格式遺失**：使用 `--reference-doc` 搭配有樣式的範本
- **LaTeX 數學式未在 HTML 中渲染**：加上 `-s` standalone + `--mathjax` 或 `--katex`
  ```bash
  pandoc input.md -s --mathjax -o output.html
  ```
- **中文/Unicode 問題**：PDF 請使用 `--pdf-engine=xelatex`

## 快速參考

```bash
# Markdown → HTML
pandoc README.md -o README.html

# Markdown → PDF
pandoc README.md -o README.pdf

# Markdown → DOCX
pandoc README.md -o README.docx

# DOCX → Markdown（盡可能保留格式）
pandoc report.docx -o report.md

# HTML → Markdown
pandoc page.html -o page.md

# 含 LaTeX 數學式的 Markdown → HTML
pandoc -s --mathjax input.md -o output.html
```

---

## Conformance Addendum

## When to Use
在不同格式之間轉換文件（Markdown、HTML、DOCX、PDF、LaTeX、ODT、EPUB 等）。當使用者想要轉換、變換或匯出文件；將 Markdown 轉為 HTML/PDF/DOCX；或在 Word、LibreOffice 及其他文件格式之間互轉時使用。

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
