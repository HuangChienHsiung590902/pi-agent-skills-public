---
name: xlsx
version: 1.0.0
description: "進階試算表工具包，支援內容提取、文件生成、資料操作和公式處理。適用於解析 Excel 資料與公式、建立專業試算表、處理複雜格式，或以程式化方式計算公式運算式。"
description_zh: "進階試算表工具包，支援內容提取、文件生成、資料操作和公式處理。適用於解析 Excel 資料與公式、建立專業試算表、處理複雜格式，或以程式化方式計算公式運算式。"
license: Proprietary. LICENSE.txt has complete terms
---

# 輸出標準

## 一般 Excel 規範

### 公式完整性
- 所有 Excel 交付物必須包含零個公式錯誤（#REF!、#DIV/0!、#VALUE!、#N/A、#NAME?）

### 範本保留（針對現有檔案）
- 編輯檔案時，仔細比對現有格式、樣式與慣例
- 絕不以標準化格式覆蓋已建立的模式
- 現有檔案慣例優先於本指南

## 財務試算表標準

### 色彩慣例
除非使用者或現有範本慣例另有規定

#### 標準色彩編碼
- **藍色文字（RGB：0,0,255）**：輸入值、情境參數
- **黑色文字（RGB：0,0,0）**：所有公式欄位與計算值
- **綠色文字（RGB：0,128,0）**：活頁簿內的跨工作表參照
- **紅色文字（RGB：255,0,0）**：外部檔案參照
- **黃色背景（RGB：255,255,0）**：關鍵假設或需更新的儲存格

### 數字格式

#### 格式指引
- **年份**：格式為文字（例如「2024」而非「2,024」）
- **貨幣**：套用 $#,##0 格式；在標題中標示單位（「Revenue ($mm)」）
- **零值**：所有零值（包括百分比）顯示為「-」（例如「$#,##0;($#,##0);-」）
- **百分比**：預設使用 0.0% 格式（一位小數）
- **倍數**：估值指標（EV/EBITDA、P/E）套用 0.0x 格式
- **負值**：使用括號（123）而非負號 -123

### 公式指引

#### 假設值組織
- 將所有假設（成長率、利潤率、倍數）放置於專屬假設儲存格
- 參照儲存格而非在公式中嵌入寫死的數值
- 範例：使用 =B5*(1+$B$6) 而非 =B5*1.05

#### 防錯措施
- 驗證所有儲存格參照
- 檢查範圍的差一錯誤
- 在預測期間維持一致的公式
- 以邊界情況測試（零值、負值、大數值）
- 避免非預期的循環參照

#### 寫死數值的文件說明
- 以格式新增備註或相鄰儲存格：「來源：[系統/文件]，[日期]，[參照]，[URL（若有）]」
- 範例：
  - 「Source: Company 10-K, FY2024, Page 45, Revenue Note, [SEC EDGAR URL]」
  - 「Source: Company 10-Q, Q2 2025, Exhibit 99.1, [SEC EDGAR URL]」
  - 「Source: Bloomberg Terminal, 8/15/2025, AAPL US Equity」
  - 「Source: FactSet, 8/20/2025, Consensus Estimates Screen」

# 試算表操作

## 概覽

使用者可能要求建立、修改或分析 .xlsx 檔案。不同的任務有不同的工具與方法可供選擇。

## 前置條件

**公式計算需要 LibreOffice**：使用 `scripts/formula_processor.py` 腳本計算公式值時，必須安裝 LibreOffice。腳本會在首次執行時自動處理 LibreOffice 設定。

## 資料分析

### 使用 pandas 進行分析
資料分析、視覺化與批次操作，請使用 **pandas**：

```python
import pandas as pd

# 載入 Excel
df = pd.read_excel('file.xlsx')  # 預設：第一個工作表
sheets_dict = pd.read_excel('file.xlsx', sheet_name=None)  # 所有工作表為字典

# 分析
df.head()      # 預覽列
df.info()      # 欄位詳細資訊
df.describe()  # 摘要統計

# 匯出 Excel
df.to_excel('output.xlsx', index=False)
```

## 試算表工作流程

## 關鍵原則：公式優先於寫死數值

**永遠優先使用 Excel 公式，而非 Python 計算後的寫死數值。** 這能維持試算表的動態性與可編輯性。

### 錯誤示範 - 寫死計算值
```python
# 避免：Python 計算後寫死結果
total = df['Sales'].sum()
sheet['B10'] = total  # 寫死為 5000

# 避免：在 Python 中計算成長率
growth = (df.iloc[-1]['Revenue'] - df.iloc[0]['Revenue']) / df.iloc[0]['Revenue']
sheet['C5'] = growth  # 寫死為 0.15

# 避免：Python 計算平均值
avg = sum(values) / len(values)
sheet['D20'] = avg  # 寫死為 42.5
```

### 正確示範 - Excel 公式
```python
# 建議：由 Excel 執行加總
sheet['B10'] = '=SUM(B2:B9)'

# 建議：在 Excel 中使用成長公式
sheet['C5'] = '=(C4-C2)/C2'

# 建議：使用 Excel 平均函數
sheet['D20'] = '=AVERAGE(D2:D19)'
```

此原則適用於所有計算——合計、百分比、比率、差異。試算表應在來源資料變更時自動重新計算。

## 標準工作流程
1. **選擇函式庫**：資料處理使用 pandas，公式/格式使用 openpyxl
2. **初始化**：建立新活頁簿或開啟現有檔案
3. **修改**：新增/更新資料、公式與格式
4. **儲存**：寫入檔案
5. **計算公式（使用公式時為必要步驟）**：執行 scripts/formula_processor.py 腳本
   ```bash
   python scripts/formula_processor.py output.xlsx
   ```
6. **檢查並修正錯誤**：
   - 腳本回傳含錯誤資訊的 JSON
   - 若 `status` 為 `errors_detected`，查看 `error_breakdown` 了解具體錯誤類型與位置
   - 修正已識別的錯誤並重新計算
   - 常見錯誤類型：
     - `#REF!`：無效儲存格參照
     - `#DIV/0!`：除以零
     - `#VALUE!`：公式中的型別不符
     - `#NAME?`：未知的公式名稱

### 建立試算表

```python
# 使用 openpyxl 處理公式與格式
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment

wb = Workbook()
sheet = wb.active

# 新增資料
sheet['A1'] = 'Hello'
sheet['B1'] = 'World'
sheet.append(['Row', 'of', 'data'])

# 新增公式
sheet['B2'] = '=SUM(A1:A10)'

# 格式設定
sheet['A1'].font = Font(bold=True, color='FF0000')
sheet['A1'].fill = PatternFill('solid', start_color='FFFF00')
sheet['A1'].alignment = Alignment(horizontal='center')

# 欄寬
sheet.column_dimensions['A'].width = 20

wb.save('output.xlsx')
```

### 修改試算表

```python
# 使用 openpyxl 保留公式與格式
from openpyxl import load_workbook

# 開啟現有檔案
wb = load_workbook('existing.xlsx')
sheet = wb.active  # 或 wb['SheetName'] 指定工作表

# 迭代工作表
for sheet_name in wb.sheetnames:
    sheet = wb[sheet_name]
    print(f"Sheet: {sheet_name}")

# 更新儲存格
sheet['A1'] = 'New Value'
sheet.insert_rows(2)  # 在第 2 列插入列
sheet.delete_cols(3)  # 刪除第 3 欄

# 新增工作表
new_sheet = wb.create_sheet('NewSheet')
new_sheet['A1'] = 'Data'

wb.save('modified.xlsx')
```

## 公式計算

由 openpyxl 建立或修改的 Excel 檔案，公式以文字儲存而非計算值。使用提供的 `scripts/formula_processor.py` 腳本來計算公式：

```bash
python scripts/formula_processor.py <excel_file> [timeout_seconds]
```

範例：
```bash
python scripts/formula_processor.py output.xlsx 30
```

此腳本：
- 首次執行時自動設定 LibreOffice 巨集
- 計算所有工作表中的所有公式
- 掃描所有儲存格的 Excel 錯誤（#REF!、#DIV/0! 等）
- 回傳含詳細錯誤位置與計數的 JSON
- 相容 Linux 與 macOS

## 公式驗證清單

快速確認公式運作正確的檢查項目：

### 必要檢查
- [ ] **驗證樣本參照**：建立完整模型前，先確認 2-3 個樣本參照是否取得正確值
- [ ] **欄位對應**：確認 Excel 欄位正確（例如第 64 欄 = BL，而非 BK）
- [ ] **列偏移量**：記住 Excel 列為 1 索引（DataFrame 第 5 列 = Excel 第 6 列）

### 常見問題
- [ ] **NaN 處理**：使用 `pd.notna()` 檢查空值
- [ ] **最右欄**：財年資料通常在第 50+ 欄
- [ ] **多個符合項**：搜尋所有出現位置，而非僅第一個
- [ ] **除以零**：在公式中使用 `/` 前先檢查分母（#DIV/0!）
- [ ] **錯誤參照**：確認所有儲存格參照指向預期儲存格（#REF!）
- [ ] **跨工作表參照**：連結工作表時使用正確格式（Sheet1!A1）

### 公式測試方法
- [ ] **從小開始**：在廣泛套用前，先對 2-3 個儲存格測試公式
- [ ] **驗證相依性**：確認公式參照的所有儲存格均存在
- [ ] **測試邊界情況**：包含零、負值和非常大的數值

### 了解 scripts/formula_processor.py 的輸出
腳本回傳含錯誤詳情的 JSON：
```json
{
  "status": "success",           // 或 "errors_detected"
  "error_count": 0,              // 錯誤總數
  "formula_count": 42,           // 檔案中的公式數量
  "error_breakdown": {           // 僅在發現錯誤時出現
    "#REF!": {
      "count": 2,
      "cells": ["Sheet1!B5", "Sheet1!C10"]
    }
  }
}
```

## 最佳實踐

### 函式庫選擇
- **pandas**：最適合資料分析、批次操作、簡單資料匯出
- **openpyxl**：最適合複雜格式、公式、Excel 特定功能

### openpyxl 使用指引
- 儲存格索引為 1 起始（row=1, column=1 表示儲存格 A1）
- 使用 `data_only=True` 讀取計算值：`load_workbook('file.xlsx', data_only=True)`
- **警告**：以 `data_only=True` 儲存會永久以數值取代公式
- 大型檔案：讀取使用 `read_only=True`，寫入使用 `write_only=True`
- 公式會被保留但不計算——使用 scripts/formula_processor.py 更新數值

### pandas 使用指引
- 指定資料型別以避免推斷問題：`pd.read_excel('file.xlsx', dtype={'id': str})`
- 大型檔案可讀取特定欄位：`pd.read_excel('file.xlsx', usecols=['A', 'C', 'E'])`
- 正確處理日期：`pd.read_excel('file.xlsx', parse_dates=['date_column'])`

## 程式碼風格指引
**重要**：產生 Excel 操作的 Python 程式碼時：
- 撰寫簡潔的 Python 程式碼，不加不必要的注釋
- 避免冗長的變數名稱與多餘的操作
- 避免不必要的 print 陳述式

**對於 Excel 檔案本身**：
- 為複雜公式或重要假設的儲存格新增注釋
- 為寫死數值記錄資料來源
- 為關鍵計算與模型區段加上說明

---

## Conformance Addendum

## When to Use
進階試算表工具包，支援內容提取、文件生成、資料操作和公式處理。適用於解析 Excel 資料與公式、建立專業試算表、處理複雜格式，或以程式化方式計算公式運算式。

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
