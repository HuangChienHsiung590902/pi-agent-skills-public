---
name: windows-ocr-toolkit
description: Use when Windows built-in OCR is needed for screenshots, images, scanned PDFs, or desktop UI text. Calls Windows.Media.Ocr without Tesseract/EasyOCR, returns recognized text with per-word bounding boxes, can find text coordinates for windows-mcp clicks, reconstruct table rows, and OCR PDF pages into Markdown/JSON. Trigger for Windows OCR, screenshot text recognition, image/PDF OCR, finding a button by visible text, or OCR-assisted desktop automation.
---

# Windows OCR Toolkit

呼叫 Windows 10/11 內建的 `Windows.Media.Ocr` 引擎辨識圖片文字，並用 `Windows.Data.Pdf` 將 PDF 頁面轉成圖片後辨識，**完全免安裝**
（不需要 Tesseract / EasyOCR / PaddleOCR），並回傳每個字詞的座標框，可用於
還原表格排版、或找出文字在螢幕上的可點擊座標（例如驅動 `adb shell input tap`
自動化操作透過 scrcpy 鏡像的 Android 畫面）。

**本技能所有腳本已隨附在技能目錄，直接複製使用，不需要重寫。**

## When to Use

## Inputs and Outputs

- **輸入：** PNG/JPG/BMP/螢幕截圖，或 PDF；可選 OCR 語言與頁碼範圍。
- **輸出：** 完整文字、逐行文字、每個 word 的邊界座標；PDF 另輸出 Markdown 與 JSON。

## Procedure

1. 先確認輸入是圖片、桌面截圖、文字型 PDF 或掃描 PDF。
2. 文字型 PDF 優先直接提取文字；掃描 PDF 使用 `pdf_ocr.ps1`。
3. 圖片使用 `ocr_demo.py` 或 `ocr_core.ps1`；需要定位按鈕時使用 `ocr_find_and_click.py`。
4. Windows 桌面座標交給 `windows-mcp_Click` 前，先處理 Screenshot scale、DPI、多螢幕與負座標。
5. 點擊或轉換後重新截圖／OCR 驗證，不要只依單次執行結果宣稱成功。

## Rules and Limitations

- 只有在使用者授權的畫面、文件與桌面上執行 OCR 或自動點擊。
- 不要把密碼、驗證碼或個人資料寫入輸出檔或回覆；必要時遮蔽或刪除暫存結果。
- OCR 座標是影像座標，不是天然的 Windows 螢幕座標。

## 何時使用

- 目前對話的模型**看不到圖片**（`Read` 圖片回傳 "Cannot read image" 之類錯誤），
  但又需要知道截圖/照片裡的文字內容
- 需要在螢幕截圖裡定位某段文字的精確像素座標，接著模擬點擊
  （典型場景：App 用 WebView 渲染、`uiautomator dump` 抓不到文字元素）
- 需要把截圖裡排版混亂的表格文字，還原成有列有欄的結構化資料
- 需要 OCR 掃描 PDF；或先將 PDF 頁面轉成圖片再逐頁辨識
- 需要把 OCR 找到的文字座標交給 `windows-mcp_Click` 操作 Windows 桌面

## 為什麼要拆成 PowerShell + Python 兩層

`Windows.Media.Ocr` 是 WinRT（UWP）API，Python 無法直接呼叫，必須靠 PowerShell
橋接。所以固定分兩層：

```
scripts/ocr_core.ps1              <- 唯一真正呼叫 Windows OCR API 的地方，輸出 JSON
     ↑ (subprocess 呼叫)
scripts/ocr_demo.py                <- 提供 run_ocr()，解析 JSON
     ↑ (import run_ocr)
     ├── scripts/ocr_table_reconstruct.py   <- 座標應用1：還原表格
     ├── scripts/ocr_find_and_click.py      <- 座標應用2：找文字算點擊座標
     ├── scripts/pdf_page_to_png.ps1        <- Windows.Data.Pdf PDF 分頁轉 PNG
     └── scripts/pdf_ocr.ps1                <- PDF 逐頁轉圖 + OCR，輸出 Markdown/JSON
```

`scripts/ocr_core.ps1` 是地基，**絕對不能刪**；其他三支 Python 都靠它才能運作。

## 關鍵限制（踩過的坑）

- **必須用 `powershell.exe`（Windows PowerShell 5.1）**，`scripts/ocr_demo.py` 內部已經
  寫死呼叫 `powershell.exe`。PowerShell 7+（`pwsh`）無法載入
  `[Windows.Media.Ocr.OcrEngine, ...]` 這類 WinRT 型別，會報
  `Unable to find type [Windows.Media.Ocr.OcrEngine,...]`。
- 系統需安裝對應語言的語言包（繁中 `zh-Hant-TW` 通常內建）。
- Python 3.10+（`scripts/ocr_demo.py` 用到 `str | None` 型別語法）。
- 中文字之間辨識結果偶爾會自動插入多餘空格，這是 Windows OCR 本身行為，非 bug。
- 這台機器的實測使用 `powershell.exe` 執行可正常載入 WinRT；PowerShell 腳本已使用 UTF-8 輸出，讓 Python 可保留中文文字。

## Step 1 — 取得截圖

若目標是 Android 裝置（例如透過 scrcpy 鏡像操作）：

```powershell
adb shell screencap -p /sdcard/screen.png
adb pull /sdcard/screen.png "C:\path\to\screen.png"
```

若目標是 Windows 畫面本身，用任何截圖工具存成 png/jpg 即可。

## Step 2 — 複製腳本到工作目錄（或直接用技能目錄路徑呼叫）

```powershell
# $SKILL_DIR = 這個技能的目錄
Copy-Item "$SKILL_DIR\scripts/ocr_core.ps1" "你的工作目錄\"
Copy-Item "$SKILL_DIR\scripts/ocr_demo.py" "你的工作目錄\"
Copy-Item "$SKILL_DIR\scripts/ocr_table_reconstruct.py" "你的工作目錄\"   # 需要還原表格時
Copy-Item "$SKILL_DIR\scripts/ocr_find_and_click.py" "你的工作目錄\"      # 需要找點擊座標時
```

也可以不複製，直接用完整路徑呼叫技能目錄裡的腳本。

## PDF OCR

先判斷 PDF 是否有文字層；若可直接選取文字，優先使用 PDF 文字抽取工具，避免不必要的 OCR。掃描 PDF 或需要統一視覺辨識時，使用內建的 PDF 分頁轉圖與逐頁 OCR：

```powershell
powershell.exe -ExecutionPolicy Bypass -File scripts/pdf_page_to_png.ps1 `
  -PdfPath "C:\input.pdf" -OutputPath "C:\temp\page-1.png" -Page 1

powershell.exe -ExecutionPolicy Bypass -File scripts/pdf_ocr.ps1 `
  -PdfPath "C:\input.pdf" -OutputDir "C:\ocr-result" -Language zh-Hant-TW
```

`pdf_ocr.ps1` 會輸出 `<原檔名>.ocr.md` 與 `<原檔名>.ocr.json`；JSON 的每一頁都包含 `text` 與 `lines[].words[]`，每個 word 有 `x1,y1,x2,y2` 座標。可用 `-FromPage`、`-ToPage` 只處理指定頁數。

### 大型 PDF 實測與驗證紀錄

- 已實測使用 Windows 內建 `Windows.Data.Pdf` + `Windows.Media.Ocr`，以 `zh-Hant-TW` 完成 990 頁掃描 PDF 的逐頁 OCR。
- 建議先用 `pdf_page_to_png.ps1` 測試第 1 頁，確認 PDF 頁數、PDF 分頁轉圖與繁體中文 OCR 語言包都可用，再執行完整 `pdf_ocr.ps1`。
- 完成後不要只看 PowerShell 結束碼；應確認輸出的 `.ocr.md`、`.ocr.json` 都存在，並核對 JSON 的 `pages`/`pageCount`、Markdown 行數與檔案大小是否合理。
- JSON 會保留每個 word 的座標，因此大型 PDF 的 JSON 可能遠大於 Markdown；只需要文字時優先使用 `.ocr.md`，需要座標、表格重建或後續定位時才保留 `.ocr.json`。
- 從 Git Bash 呼叫 Windows PowerShell 時，含中文的 Windows 路徑可能遭 shell/code page 轉碼；遇到路徑或參數變形時，改用原生 `powershell.exe`，或用 UTF-16LE Base64 `-EncodedCommand` 傳遞整段指令。

## Step 3 — 依需求選一支腳本執行

### 3a. 基本辨識（拿完整文字 + 逐字座標）

```powershell
python scripts/ocr_demo.py "C:\path\to\screen.png" zh-Hant-TW
```

輸出：完整辨識文字（單行）+ 逐行逐字座標明細
`「文字」座標：(x1,y1) - (x2,y2)`。

### 3b. 還原表格排版

適合截圖本身是表格/清單，純文字辨識結果會把欄位全部黏在一起、失去對應關係時：

```powershell
python scripts/ocr_table_reconstruct.py "C:\path\to\screen.png" zh-Hant-TW
```

原理：用 y 座標把字詞分組成「列」（容許 15px 誤差），列內再依 x 座標排序，
還原出跟原圖排版一致的逐列文字。

### 3c. 找文字並算點擊座標（自動化操作用）

```powershell
python scripts/ocr_find_and_click.py "C:\path\to\screen.png" "要找的關鍵字" zh-Hant-TW
```

輸出每個符合關鍵字的整行文字、外接框，以及可直接用的：
```
adb shell input tap <x> <y>
```

## Step 4 — 將 OCR 座標交給控制工具

OCR 回傳的是截圖／圖片座標，不一定等於實際桌面座標。若 Screenshot 有縮放，先依工具回報的 scale 還原；多螢幕、負座標、DPI 縮放也要納入換算。對 Windows 桌面，優先用 `windows-mcp_Snapshot` 找 UI 元素；找不到時才用 OCR，再把中心座標交給 `windows-mcp_Click`，點擊後重新 Screenshot/OCR 驗證。

Android 鏡像場景則可使用：

```powershell
adb shell input tap <click_x> <click_y>
```

點擊後畫面可能需要載入時間，建議 `Start-Sleep -Seconds 2~3` 再重新截圖確認。

## 也可以單獨呼叫 PowerShell 核心（不透過 Python）

```powershell
powershell.exe -ExecutionPolicy Bypass -File scripts/ocr_core.ps1 -ImagePath "C:\test.png" -Language "zh-Hant-TW"
```

會直接印出 JSON（`fullText` + `lines[].words[]`，每個 word 帶 `x1,y1,x2,y2`）。

## Success criteria

- [ ] `python scripts/ocr_demo.py <圖片> zh-Hant-TW` 能印出完整辨識文字，且沒有跳出
      `Unable to find type` 之類 WinRT 錯誤
- [ ] 需要還原表格排版時，`scripts/ocr_table_reconstruct.py` 印出的列數與原圖列數大致相符
- [ ] 需要自動化點擊時，`scripts/ocr_find_and_click.py` 找到的座標點擊後畫面有正確反應
- [ ] `pdf_page_to_png.ps1` 能將 PDF 指定頁輸出 PNG
- [ ] `pdf_ocr.ps1` 能輸出每頁 OCR Markdown/JSON
- [ ] OCR 座標交給控制工具前已處理圖片縮放與多螢幕座標
- [ ] 全程沒有另外安裝 Tesseract / EasyOCR 等套件

## 已知限制

| 限制 | 說明 |
|---|---|
| 必須用 `powershell.exe` | `pwsh`（PowerShell 7+）無法載入 WinRT 型別 |
| 中文字間偶有多餘空格 | Windows OCR 本身行為，非 bug |
| 手寫字辨識效果較差 | 印刷體/螢幕截圖效果佳 |
| 表格還原僅切「列」 | 目前只用 y 座標分組成列；若表格欄位邊界不固定，
  x 座標分欄需依實際圖片手動調整 `scripts/ocr_table_reconstruct.py` 的分欄邏輯 |
| 語言包缺失 | 指定 `-Language` 但系統未安裝該語言包時，
  `TryCreateFromLanguage` 回傳 null，腳本會報錯並提示去
  `設定 > 時間與語言 > 語言與地區` 安裝 |
| PDF 文字層 | 有文字層的 PDF 應優先直接提取文字；本 Skill 的 PDF OCR 會把頁面渲染成圖片，因此可能讀到頁首頁尾或瀏覽器列印資訊 |
| PDF 輸出 | PDF OCR 目前輸出 Markdown/JSON；若要把 OCR 文字嵌回成可搜尋 PDF，需另做文字層封裝 |

## 實際應用案例

這套工具原本是為了在「模型無法讀取圖片」的限制下，操作透過 scrcpy 鏡像的
Android 手機畫面而寫的：截圖 → OCR 辨識畫面文字 → 用座標定位目標按鈕/商品 →
`adb shell input tap` 模擬點擊 → 重新截圖確認，藉此繞過某些 App（用 WebView
渲染、`uiautomator dump` 抓不到文字節點）的自動化操作限制。

---

## Pitfalls

- 不要把 PDF 文字層和掃描 PDF 混為一談；先直接提取，失敗才 OCR。
- 中文 OCR 常以單字切分並插入空格；搜尋關鍵字時要正規化空白。
- UI Tree 已找到原生按鈕時，優先使用 Snapshot label，不要用 OCR 或固定座標硬點。
- 不要忽略 Screenshot 回報的縮放比例；錯誤換算可能點到其他螢幕或其他視窗。
- 不要因 OCR 有部分錯字就宣稱整頁完全正確；需回報辨識品質與限制。

## Verification

1. 確認 `powershell.exe` 5.1 可以載入 `Windows.Media.Ocr`。
2. 用實際圖片確認輸出包含文字與 `x1,y1,x2,y2` 座標。
3. 用測試 PDF 確認 `pdf_page_to_png.ps1` 可輸出 PNG，`pdf_ocr.ps1` 可輸出 Markdown/JSON。
4. 若用於桌面點擊，點擊後重新 Screenshot 或 Snapshot 確認畫面狀態。

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
