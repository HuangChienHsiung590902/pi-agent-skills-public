---
name: windows-restore-center-ui
description: 維護與改善 D:\\app\\還原軟體\\WindowsRestoreCenter.exe 的 Flet Windows 桌面介面；當使用者提到 Windows 重灌恢復中心、WindowsRestoreCenter、還原軟體、分類選單、軟件恢復清單、搜尋軟件、分類收合、畫面空間利用、重新打包恢復中心或要調整這套恢復工具 UI 時使用。涵蓋原始碼修改、分類清單互動、搜尋篩選、響應式版面與 EXE 重新打包，不要只修改截圖或另做無關的示範頁面。
---

# Windows Restore Center UI

## When to Use

使用者要求下列任一類工作時載入本 Skill：

- 修改「Windows 重灌恢復中心」的畫面、分類選單、按鈕、清單或搜尋功能。
- 讓軟件分類可以展開／收合，或改善視窗空間利用率。
- 新增或修正軟件名稱、ID、分類、發行者的搜尋篩選。
- 修改 `D:\app\還原軟體` 內的 `WindowsRestoreCenter.exe`，並需要同步更新可執行檔。
- 重新打包或驗證這套 Flet 桌面程式。

這是既有桌面程式的維護 Skill，不是建立全新 UI 原型。優先保留掃描、快照、選取、保存與 winget/npm 恢復流程，只改使用者要求的介面行為。

## Project facts

- 原始碼：`D:\app\還原軟體\reinstall\restore-gui\restore_gui.py`
- Python 依賴：`D:\app\還原軟體\reinstall\restore-gui\requirements.txt`
- 初始套件清單：`D:\app\還原軟體\reinstall\restore-gui\packages.json`
- 建置輸出：`D:\app\還原軟體\reinstall\restore-gui\dist\WindowsRestoreCenter.exe`
- 使用者直接開啟的 EXE：`D:\app\還原軟體\WindowsRestoreCenter.exe`
- 另一份部署副本：`D:\app\還原軟體\reinstall\WindowsRestoreCenter\WindowsRestoreCenter.exe`
- 目前框架：Flet `1.0.0`，Python `3.13`，PyInstaller 由 Flet pack 呼叫。

## Inputs and Outputs

### Inputs

- 使用者對 UI 的具體需求，例如搜尋、分類收合、雙欄清單、放大視窗或改善空白。
- 目前的 `restore_gui.py`、截圖與執行結果。
- 若要打包，需確認來源 `packages.json` 存在，且目標 EXE 沒有被使用者程序鎖定。

### Outputs

- 修改後的 `restore_gui.py`。
- 若使用者要求可直接使用，重新產生並複製 `WindowsRestoreCenter.exe`。
- 回報實際修改的檔案、驗證命令與尚未驗證的限制。

## UI baseline

目前採用以下介面方向，除非使用者明確要求不同風格：

- 淺色、圓角、柔和藍色系；避免僵硬的滿版細線表格。
- 視窗預設約 `1280 × 900`，最低約 `980 × 680`，讓清單有足夠垂直空間。
- 上方保留標題、目前狀態、主要操作按鈕與進度列。
- 軟件清單是主要工作區，執行紀錄放在右側窄欄，避免紀錄區吃掉整個清單高度。
- 分類使用 Flet `ExpansionTile` 展開／收合；分類內容使用 `ResponsiveRow`，寬視窗雙欄、小視窗自動回單欄。
- 搜尋列必須明顯可見，放在清單上方；至少搜尋 `name`、`id`、`category`、`publisher`。
- 搜尋後顯示「顯示 X / Y 個項目」，沒有結果時顯示清楚的空狀態與下一步提示。
- 搜尋結果只影響目前顯示與「全選／清除目前選擇」的操作範圍，不得意外清除未顯示項目的選取狀態。
- 勾選狀態以穩定的 `kind:id` key 保存，不要依賴畫面順序。

## Procedure

1. **先確認來源與目前狀態**
   - 讀取 `restore_gui.py`、`requirements.txt`、打包 bat；確認目前 Flet 版本與輸出位置。
   - 檢查 `D:\app\還原軟體\WindowsRestoreCenter.exe` 是否正在執行。若要覆蓋被鎖定的 EXE，先告知使用者並避免強制關閉其正在使用的程式。
   - 不要直接把修改只寫進已打包 EXE；永久修改必須在 `restore_gui.py`。

2. **將需求拆成不互相破壞的 UI 變更**
   - 搜尋：保留一個可輸入的 `TextField`，以 `on_change` 即時重建清單；加入清除搜尋操作。
   - 分類：以分類建立群組；每組用 `ExpansionTile`，群組內用 `ResponsiveRow` 或等價的可重排容器。
   - 空間：讓清單區 `expand=True`，避免固定過大的空白；紀錄區與清單並排但保留可讀寬度。
   - 狀態：分類收合、搜尋結果數量、選取數量都要在畫面上反映。
   - 行為：不要改變 `scan_current_software`、`merge_catalog`、`save_catalog`、winget/npm 安裝命令，除非使用者另外要求。

3. **修改後先做快速檢查**
   - 執行：
     ```text
     python -m py_compile D:/app/還原軟體/reinstall/restore-gui/restore_gui.py
     ```
   - 檢查關鍵 UI 元件仍存在：搜尋欄、`ExpansionTile`、響應式分類內容、清除搜尋、空結果提示。
   - 若只修改 Python，先不要急著覆蓋正式 EXE；先完成語法檢查與必要的本機啟動檢查。

4. **重新打包（使用者要求交付 EXE 時）**
   - 在 `D:\app\還原軟體\reinstall\restore-gui` 執行既有 bat，或使用等價命令：
     ```text
     flet pack restore_gui.py -n WindowsRestoreCenter --product-name "Windows Restore Center" --file-description "Scan and restore selected Windows software" --product-version 1.1.0.0 --add-data "packages.json:." --distpath ".\\dist" -y
     ```
   - 確認 `dist\\WindowsRestoreCenter.exe` 存在且時間戳是本次建置。
   - 將已驗證的輸出複製到 `D:\app\還原軟體\WindowsRestoreCenter.exe`。若使用者也依賴 `reinstall\\WindowsRestoreCenter` 副本，再同步複製該副本。
   - 不要把 `software_catalog.json` 打包進 EXE；它是執行時快照，必須留在 EXE 旁邊並可獨立備份。

5. **回報結果**
   - 明確列出修改的原始碼與 EXE 路徑。
   - 說明已通過的語法／打包檢查。
   - 若沒有實際開啟 GUI 驗證，必須說明「尚未完成互動式畫面驗證」，不要宣稱已確認視覺效果。

## Rules and Limitations

- 一律以繁體中文回報。
- 不要刪除使用者的 `software_catalog.json`、`restore.log` 或 `packages.json`。
- 不要在沒有使用者要求時改變掃描來源、恢復命令、資料格式或自動勾選邏輯。
- 不要在 EXE 被鎖定時強制終止使用者正在使用的恢復中心；先回報並等待使用者關閉。
- 不要只修改 `D:\app\還原軟體\WindowsRestoreCenter.exe` 而沒有同步更新原始碼。
- 搜尋只做本機已載入項目的篩選，不要為了搜尋功能新增網路服務或外部資料來源。
- UI 改動應優先採增量修改，保留背景執行緒、queue、進度列與日誌流程。
- 若 Flet API 在版本更新後變動，先以本機安裝版本的 introspection／小型語法檢查確認，不要猜測參數名稱。

## Pitfalls

- `selected_ids` 必須在重建搜尋結果或展開／收合分類後保留；不要在 `rebuild_list()` 裡重設選取集合。
- lambda 綁定迴圈變數時要使用預設參數，例如 `lambda e, k=key: ...`，避免所有勾選框都操作最後一筆。
- 搜尋結果為空時不能讓清單高度塌陷到看不見；要保留明確空狀態。
- 固定清單高度與固定紀錄高度會造成「畫面沒放什麼就沒空間」；主要清單應使用可伸展區域。
- 執行掃描或恢復時不要讓搜尋重建破壞背景工作；按鈕鎖定與完成後恢復要維持原流程。
- `flet pack` 的 `--add-data` 在 Windows 仍使用 `source:destination` 格式；打包後一定要確認 `packages.json` 被包含。
- `version` 欄位可能含舊機器的臨時路徑；優先使用現行 Flet pack 命令產生的版本資訊，不要盲目沿用舊 `.spec`。
- 使用者直接開啟的檔案是根目錄 EXE；只更新 `dist` 不代表使用者已拿到新版。

## Verification

至少完成下列與本次變更相符的檢查：

1. `restore_gui.py` 存在，且 `python -m py_compile` 成功。
2. 原始碼可搜尋到搜尋欄、分類收合元件與清單重建邏輯。
3. 若重新打包：`dist\\WindowsRestoreCenter.exe` 存在，且建置時間戳為本次操作。
4. 若已部署：`D:\app\還原軟體\WindowsRestoreCenter.exe` 存在，且檔案大小／時間戳與本次打包輸出一致。
5. 回報是否有實際互動式 GUI 驗證；若沒有，明確保留這項限制。

## Current implementation reference

目前已完成並可作為基準的介面變更包括：

- 分類使用可展開／收合的 `ExpansionTile`。
- 分類項目使用 `ResponsiveRow`，寬視窗以雙欄排列。
- 搜尋欄可搜尋名稱、ID、分類與發行者，並顯示結果數量。
- 清單與執行紀錄左右分欄，主要清單使用 `expand=True`。
- 空搜尋結果顯示「找不到符合的軟件」提示。
- 已使用 `python -m py_compile` 驗證，並以 Flet pack 產生新版 EXE。
