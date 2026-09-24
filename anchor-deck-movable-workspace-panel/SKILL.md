---
name: anchor-deck-movable-workspace-panel
description: 讓 Anchor Deck MCP 的「MCP 工作區」選單面板與右上角觸發按鈕可以被拖曳移動，並用 localStorage 記住位置；同時隱藏面板內 Codex／Claude／WorkBuddy chips 與「選擇元素」說明。當使用者說「MCP 工作區選單可以移動」「面板擋住畫面」「讓工作區選單拖曳」「記住面板位置」「隱藏選擇元素說明」，或 Anchor Deck 的 workspace panel／floating button 需要可移動、可還原位置時使用。適用於 Windows 已安裝 Anchor Deck MCP（Bun compiled EXE）的本機 binary patch 流程。
compatibility: 需要 Windows、已安裝 Anchor Deck MCP、管理員權限（覆蓋 Program Files）、Node.js + Playwright（可選，用於 CDP 驗證）。
---

# Anchor Deck 可移動工作區選單

## When to Use

- 使用者要求「MCP 工作區」選單／面板可以移動、拖曳。
- 使用者抱怨面板或右上角按鈕擋住畫面。
- 使用者要求拖曳後的位置在重新整理、重啟瀏覽器後仍保留。
- 使用者要求隱藏面板內的 Codex／Claude／WorkBuddy chips 或「選擇元素」說明。
- 需要回復到不可移動的原始版本。

這個 skill 只處理 **UI 行為**（面板／按鈕可移動 + 位置持久化 + 說明隱藏）。寬版工作區、頁面尺寸等問題改用 `anchor-deck-wide-workspace` skill，不要混在一起改。

## Inputs and Outputs

### Inputs

- 已安裝的 Anchor Deck MCP EXE：

```text
C:\Program Files\Anchor Deck MCP\Anchor Deck MCP.exe
C:\Program Files\Anchor Deck MCP\anchor-deck-mcp.exe
```

- 可工作的原始備份（作為 patch 基底）：

```text
C:\Users\HCH\AppData\Local\Temp\anchor-deck-mcp-backups\install-20260916-151259\
```

若該備份不存在，先從目前安裝檔複製一份當基底，並明確標記為新基底。

### Outputs

- 候選 patched EXE（放在 Temp，不直接碰 Program Files）：

```text
C:\Users\HCH\AppData\Local\Temp\anchor-deck-pos-persist\
```

- 安裝後的可移動版本 + 安裝前備份 + 安裝 log。
- 驗證結果（smoke test、可選 CDP 拖曳測試、使用者手動確認）。

## 核心原理

### 為什麼不能直接改 `ensureButton()`

Anchor Deck MCP 是 **Bun compiled EXE**：JavaScript bundle 與 Bun runtime 打包在同一個執行檔。直接替換整個 bundled 函式（例如 `ensureButton()`）或插入變長內容，會破壞 bytecode／offset，導致 EXE 無法啟動（曾實際發生多次）。

安全原則是：

1. **只替換有足夠空間的既有 script 區段**，且新內容必須 **小於或等於原區段長度**，不足處用空白 padding 補齊，維持 EXE 檔案大小不變。
2. 優先使用 **事件委派**（`document.addEventListener(..., true)`）而非在按鈕建立時綁定事件。因為 `workspace-ui.js` 會在啟動後**動態建立**按鈕與面板，頁面初始載入時 `querySelector` 可能找不到元素（曾實際導致拖曳無效）。
3. 外觀與行為分開：拖曳是行為，圓形／矩形是外觀。使用者只要求移動時，**不要**改外觀。

### 可移動的兩個目標

```text
.codex-workspace-floating-button   ← 右上角觸發按鈕
.codex-workspace-panel             ← 開啟後的選單面板
```

面板的拖曳把手是標題列：

```text
.codex-workspace-title
```

但排除關閉鈕：

```text
.codex-workspace-close
```

### 位置持久化

拖曳結束時寫入：

```text
localStorage["anchor-deck-ui-pos"]
```

格式：

```json
{ "b": [x, y], "p": [x, y] }
```

- `b` = 按鈕座標
- `p` = 面板座標

讀取時機：頁面載入、面板開啟、以及一個 500ms 的輪詢（因為面板是動態建立的）。套用時同時設 `left`、`top` 並把 `right` 設為 `auto`，否則原本的 `right: 20px` 會把元素拉回右上角。

### 隱藏說明區塊

用精準 CSS 等長替換（把 `padding` 換成 `display: none` 並補空白），只動這兩個 rule：

```css
.codex-workspace-selection-hint { ... }
.codex-workspace-client-list { ... }
```

不要為了隱藏說明而替換整個 `ensureButton()`。

## Procedure

### 1. 建立候選 patch

執行 bundled 腳本（會從基底備份產生候選 EXE 到 Temp）：

```powershell
python -X utf8 "D:\OB\skills\anchor-deck-movable-workspace-panel\scripts\build-movable-panel-patch.py"
```

腳本會：

- 從基底備份讀取兩個 EXE。
- 在每個 EXE 的**兩個** `data-codex-collab-navigation` script 區段（外層協作頁與內層各一個）插入拖曳 + localStorage 腳本，等長替換。
- 等長替換隱藏說明區塊的兩個 CSS rule。
- 輸出候選到 `C:\Users\HCH\AppData\Local\Temp\anchor-deck-pos-persist\`。
- 若新內容超過區段長度會直接報錯，**不要**硬塞。

### 2. Smoke test 候選（不碰正式安裝）

```powershell
& "C:\Users\HCH\AppData\Local\Temp\anchor-deck-pos-persist\anchor-deck-mcp.exe" version
$p = Start-Process -FilePath "C:\Users\HCH\AppData\Local\Temp\anchor-deck-pos-persist\Anchor Deck MCP.exe" -PassThru -WindowStyle Hidden
Start-Sleep 4
# 期望：version 印出 0.2.6；短暫啟動後 exit code 0（偵測到既有 server 而正常退出）
```

若短暫啟動的 exit code 不是 0，**不要安裝**，回到步驟 1 檢查 patch。

### 3. 安裝（需管理員）

執行 bundled 安裝腳本（會先備份、停程序、覆蓋、重啟、驗證啟動）：

```powershell
powershell -ExecutionPolicy Bypass -File "D:\OB\skills\anchor-deck-movable-workspace-panel\scripts\install-movable-panel-patch.ps1"
```

注意：這個腳本會 `Stop-Process` 正在跑的 Anchor Deck MCP，會中斷使用者目前的工作區連線。執行前先告知使用者。

安裝 log：

```text
C:\Users\HCH\AppData\Local\Temp\install-anchor-deck-pos-persist.log
```

備份目錄：

```text
C:\Users\HCH\AppData\Local\Temp\anchor-deck-mcp-backups\pos-persist-<timestamp>\
```

### 4. 驗證

優先請使用者手動測試（最可靠）：

```text
Ctrl + Shift + R
1. 點右上角「MCP 工作區」開啟面板
2. 按住面板頂部標題列拖曳 → 面板跟著移動
3. 關掉面板，按住按鈕拖曳 → 按鈕跟著移動
4. Ctrl + Shift + R 重新整理 → 兩者回到拖過的位置
```

若使用者的 Chrome 有開 debug port 9222，可用 bundled CDP 腳本自動化驗證：

```powershell
$env:NODE_PATH = "D:\.system\.mcp\npm\node_modules"
node "D:\OB\skills\anchor-deck-movable-workspace-panel\scripts\test-panel-drag-cdp.cjs"
```

注意：`NODE_PATH` 必須指向 `...\node_modules`（不是 `...\npm`），否則 `Cannot find module 'playwright'`。若 9222 未開（`ECONNREFUSED`），不要強行開 Chrome，改請使用者手動測試。

CDP 腳本會回報：

```json
{ "panelMoved": true, "btnMoved": true, ... }
```

兩者皆為 `true` 才算通過。

### 5. 回復原始版本

若使用者不要可移動版本，從安裝前備份回復：

```text
C:\Users\HCH\AppData\Local\Temp\anchor-deck-mcp-backups\pos-persist-<timestamp>\*.original
```

複製回 `C:\Program Files\Anchor Deck MCP\` 並重啟（需管理員）。

## Rules and Limitations

- 外觀與行為分開：使用者只要求「可移動」時，保持原本矩形外觀，不要改成圓形／橢圓／白點。
- 面板開啟後必須維持矩形、可捲動、可閱讀；不要把整個面板做成圓形。
- 拖曳事件必須用事件委派（capture phase），不能只在初始載入時對按鈕綁定事件。
- 拖曳結束要阻止同一次 pointer 操作誤觸 click（用 `dataset.dragged` 旗標 + capture click listener）。
- 位置必須寫入 localStorage，且拖曳結束才寫入；讀取時同時設 `left`、`top`、`right:auto`。
- binary patch 必須維持 EXE 檔案長度；新內容超過區段長度就報錯，不要硬塞。
- 不要替換整個 `ensureButton()` 或插入變長內容；這會破壞 Bun bundle 導致 EXE 無法啟動。
- 安裝前必須備份；啟動失敗立即回復備份。
- 安裝會中斷使用者目前工作區連線，執行前先告知。
- 寬版工作區、頁面尺寸問題改用 `anchor-deck-wide-workspace` skill。

## Pitfalls

- **初始載入找不到按鈕**：`workspace-ui.js` 動態建立按鈕與面板，初始 `querySelector` 回傳 `null`，拖曳事件綁不到。曾實際導致「裝了但不能拖」。解法是事件委派 + 輪詢還原位置。
- **`right: 20px` 拉回右上角**：只設 `left`/`top` 不夠，必須同時把 `right` 設為 `auto`，否則元素被拉回原位。
- **NODE_PATH 指錯**：要指 `...\node_modules`，指成 `...\npm` 會 `Cannot find module 'playwright'`。
- **9222 未開**：CDP 驗證會 `ECONNREFUSED`；這時改請使用者手動測試，不要強行開 Chrome。
- **變長 patch 破壞 Bun bundle**：曾實際發生 EXE 無法啟動。永遠等長替換 + padding。
- **把「可移動」誤做成「改外觀」**：使用者要的是移動，不是圓形。外觀變動要單獨確認。

## Verification

1. 候選 EXE 的 `anchor-deck-mcp.exe version` 印出正確版本。
2. 候選 `Anchor Deck MCP.exe` 短暫啟動 exit code 為 0。
3. 候選與安裝檔的檔案大小與基底相同（等長）。
4. 安裝 log 顯示 SUCCESS 與新 PID。
5. 手動或 CDP 驗證：面板與按鈕皆可拖曳（`panelMoved`、`btnMoved` 為 true）。
6. 重新整理後位置還原（localStorage 生效）。
7. 說明區塊（chips + 選擇元素）已隱藏。
8. 外觀維持矩形，未被改成圓形。
