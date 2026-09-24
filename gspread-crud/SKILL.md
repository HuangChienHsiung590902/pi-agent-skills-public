---
name: gspread-crud
description: Python CRUD operations on Google Spreadsheets using gspread. Use when reading, writing, updating, or deleting data in Google Sheets, or when working with Google Spreadsheet APIs.
---

# gspread CRUD

使用 `gspread` + `pandas` 對 Google Spreadsheet 進行 CRUD 操作。

## 安裝

```bash
pip install gspread pandas google-auth google-auth-oauthlib
```

## 認證設定

### 所需檔案

| 檔案 | 來源 |
|------|------|
| `client_secret_*.json` | Google Cloud Console > OAuth 2.0 用戶端ID |
| `token.json` | 首次授權後自動產生 |

### 認證流程

```python
import os
from google_auth_oauthlib.flow import InstalledAppFlow
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request

CREDS_PATH = r'D:\path\to\client_secret_*.json'
TOKEN_PATH = r'D:\path\to\token.json'
SCOPES = ['https://www.googleapis.com/auth/spreadsheets']

def get_authenticated_client():
    """取得已認證的 gspread client"""
    if os.path.exists(TOKEN_PATH):
        creds = Credentials.from_authorized_user_file(TOKEN_PATH, SCOPES)
        if creds.expired and creds.refresh_token:
            creds.refresh(Request())
            with open(TOKEN_PATH, 'w') as f:
                f.write(creds.to_json())
    else:
        flow = InstalledAppFlow.from_client_secrets_file(CREDS_PATH, SCOPES)
        creds = flow.run_local_server(port=0)
        with open(TOKEN_PATH, 'w') as f:
            f.write(creds.to_json())
    
    import gspread
    return gspread.authorize(creds)
```

### 取得 client_secret

1. 前往 [Google Cloud Console](https://console.cloud.google.com/)
2. 建立 OAuth 2.0 用戶端 ID（類型：桌面應用程式）
3. 下載 JSON 檔案

## CRUD 操作

```python
client = get_authenticated_client()
sheet = client.open_by_key('SPREADSHEET_ID')  # 試算表 ID
ws = sheet.sheet1  # 第一個工作表
```

### Read 讀取

```python
# 讀取所有記錄（自動以第一列為 header）
records = ws.get_all_records()
df = pd.DataFrame(records)

# 讀取特定範圍
data = ws.get('A1:F10')

# 讀取單一儲存格
value = ws.acell('B2').value
```

### Create 新增

```python
# 新增一列（append to bottom）
new_row = ['ORD005', '2026/4/25', '陳大明', '充電線', 2, 200]
ws.append_row(new_row)

# 在指定列插入
ws.insert_row(['header1', 'header2'], index=1)
```

### Update 更新

```python
# 更新單一儲存格
ws.update_cell(row, col, value)
ws.update_cell(2, 5, 10)  # 第2列第5欄改為 10

# 更新多個儲存格
ws.update('A1:F5', [['a','b','c','d','e','f']])

# 找到特定值的儲存格並更新
cell = ws.find('ORD003')
ws.update_cell(cell.row, 5, 999)  # 更新該列第5欄
```

### Delete 刪除

```python
# 刪除一列
cell = ws.find('ORD003')
ws.delete_rows(cell.row)

# 刪除多列
ws.delete_rows(2, 3)  # 刪除第2-4列

# 刪除所有資料（保留格式）
ws.clear()
```

## 常用範例

### 完整 CRUD 流程

```python
import gspread
from google_auth_oauthlib.flow import InstalledAppFlow
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request
import pandas as pd
import os

CREDS_PATH = r'D:\GMail\client_secret_186023343884-rso6hbfda132n3d79j6a5ai2pr06ss22.apps.googleusercontent.com.json'
TOKEN_PATH = r'D:\GMail\Apps Script\token.json'
SCOPES = ['https://www.googleapis.com/auth/spreadsheets']
SPREADSHEET_ID = '1jNZ9_6o4IhdTE34IMM2chq5FNq-thl62B73lsRolqco'

# 認證
if os.path.exists(TOKEN_PATH):
    creds = Credentials.from_authorized_user_file(TOKEN_PATH, SCOPES)
    if creds.expired and creds.refresh_token:
        creds.refresh(Request())
        with open(TOKEN_PATH, 'w') as f:
            f.write(creds.to_json())
else:
    flow = InstalledAppFlow.from_client_secrets_file(CREDS_PATH, SCOPES)
    creds = flow.run_local_server(port=0)
    with open(TOKEN_PATH, 'w') as f:
        f.write(creds.to_json())

client = gspread.authorize(creds)
sheet = client.open_by_key(SPREADSHEET_ID)
ws = sheet.sheet1

# READ
df = pd.DataFrame(ws.get_all_records())
print(df)

# CREATE
ws.append_row(['ORD006', '2026/4/25', '李小龍', '拳套', 1, 800])

# UPDATE
cell = ws.find('ORD003')
ws.update_cell(cell.row, 5, 10)

# DELETE
cell = ws.find('ORD006')
ws.delete_rows(cell.row)
```

## 工作表操作

```python
# 取得所有工作表名稱
sheetNames = sheet.title

# 切換工作表
ws = sheet.worksheet('工作表2')

# 建立新工作表
sheet.add_worksheet('新工作表', rows=100, cols=20)

# 複製工作表
sheet.duplicate_sheet(ws.id, new_title='複製的 工作表1')
```

## 提示

- **試算表 ID**：從 Google Sheets URL 中取得
  `https://docs.google.com/spreadsheets/d/{ID}/edit`
- **欄位編號**：從 1 開始（A=1, B=2, C=3...）
- **認證只需要一次**，之後直接用 `token.json`
- 大量操作時建議加上延遲，避免 API 配額限制

---

## Conformance Addendum

## When to Use
Python CRUD operations on Google Spreadsheets using gspread. Use when reading, writing, updating, or deleting data in Google Sheets, or when working with Google Spreadsheet APIs.

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
