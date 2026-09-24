---
name: debug-vrs
description: 啟動 D:\SFAA\fix\room_monitor.py 原始碼（不是打包好的 exe）來即時除錯，然後接管它的 Chrome 進行操作。當使用者要求「debug vrs」、「除錯 room_monitor」、「跑 room_monitor.py 來測」、「用原始碼測 SFAA 監控」時使用。
---

# Debug VRS（跑 room_monitor.py 原始碼 + 接管 Chrome）

用途：改完 `D:\SFAA\fix\room_monitor.py` 的程式碼後，不想重新 PyInstaller 打包成 exe 才能測，直接跑 `.py` 原始碼，馬上看到修改結果，然後接管它開出來的 Chrome 繼續操作/觀察。

背景知識見 `sfaa-ecp-room-monitor` skill（`C:\Users\HCH\.claude\skills\omc-learned\sfaa-ecp-room-monitor.md`）——本 skill 只負責「啟動 + 接管」這個操作流程，系統本身的架構/坑都在那份文件。

**工具名稱（2026-07-08 起，2026-07-24 簡化）**：本文全篇提到的裸 `browser_tabs`/`browser_click`/`browser_snapshot`/`browser_evaluate` 一律指 `mcp__playwright-vrs__*`（固定接 9222，VRS 專用）。舊版同時存在的 `mcp__playwright-aipower__*`（接舊版 `C:\com\chainsea`專用）已於 2026-07-24 隨該安裝消失一同從 `~/.claude.json` 移除，目前只剩 `playwright-vrs` 一個 server，不再需要用 `select:mcp__playwright-vrs__browser_tabs` 這種精確寫法避開同名工具互撞。細節見 `connect-chrome` skill。

## 🔴 鐵則：做任何操作之前一定要先切未就緒（建檔中）

不管接下來要點什麼、改什麼設定、測試什麼功能，**接管 Chrome 後的第一件事永遠是先切「未就緒」→「建檔中」**，才能開始做其他操作。這是使用者明確要求的硬性規則（2026-07-06），目的是避免帳號在你操作/測試期間被 ACD 派入真實客戶的新服務。

直接跑腳本 `scripts/set_unready.py`（同目錄），不要每次都用 `browser_snapshot`/`browser_click` 手動摸索一遍：

```powershell
python "C:\Users\HCH\.claude\skills\debug-vrs\scripts/set_unready.py"
```

`scripts/set_service_category.py` 已經內建這一步（登入後立即未就緒/建檔中），但**任何其他手動操作也一律要先跑這支腳本**，不是只有那支腳本才需要。

**已知的安全限制（已知非 bug）**：如果帳號已經是「未就緒(建檔中)」，重新選一次同樣的值有時會點擊失敗（腳本會印出「看到「建檔中」但點擊失敗」並以 exit code 1 結束）——這是因為下拉選單裡目前已生效的選項本身可能不可再點擊，不是誤點到別的地方。腳本失敗時**不會**有任何頁面導覽或狀態改變（已驗證），所以看到這個特定錯誤訊息、且當下本來就已經是未就緒(建檔中) 時，可以視為「目標狀態已達成，無需再做動作」，不必重跑。

**2026-07-06 事故記錄**：第一版腳本用 `document.querySelectorAll('iframe')` + `contentDocument` 手動爬 frame 樹算絕對座標、再 `page.mouse.click(x,y)`（抄自 `scripts/set_service_category.py` 那支腳本的手法）。結果在一次測試中，「未就緒」回報點擊成功，但實際點歪，把正在跑的 PROD 分派主控台整頁導覽到完全不同的 `vrs.sfaa.gov.tw` CRM 頁面，中間卡了一個 `beforeunload` 對話框，一度讓 `room_monitor.py` 找不到它依賴的頁面。現在的版本改用 Playwright 自己的 `frame.get_by_text(...).click()`（跟 MCP 內部驅動 `browser_click` 一樣的機制，會自己處理跨 frame 定位、不用手動算座標），並且每一步點擊後都驗證預期的下一個元素有沒有出現、網址有沒有意外改變，任何一步不對就立刻中止、不再繼續點。

## ⚠️ 開始前必查：現在是不是有真人在服務中

`launch_chrome()` 開頭就會 `taskkill /F /IM chrome.exe`，會直接砍斷任何正在進行的真實客服對話（視訊/文字）。**在殺任何 process 之前**，若目前有 Chrome 在跑（`Get-NetTCPConnection ... 9222` 有東西），务必先接管看一下有没有真人在服務中：

```javascript
// browser_evaluate，在已接管的 Chrome 上跑
() => {
  const iframes = [...document.querySelectorAll('iframe')];
  for (const f of iframes) {
    try {
      const zone = f.contentDocument && f.contentDocument.getElementById('LeftZone');
      if (zone) {
        const rooms = [...zone.children].filter(el => !['ChatRoomMainMenu','CleanRoomMenu'].includes(el.id));
        return rooms.map(r => ({ id: r.id, leave: r.getAttribute('leave'), text: (r.innerText||'').slice(0,200) }));
      }
    } catch(e) {}
  }
  return 'LeftZone not found';
}
```

`leave: null` 的房間＝進行中，**不要砍**。如果有真人在服務中，跟使用者確認可以繼續再動手，不要自己假設現在是安全的測試時段。

## 步驟

### 1. 停掉所有現存的 room_monitor（exe + 舊的守護子程序 + 舊的 .py 執行），避免雙開搶燈

`room_monitor.py`（2026-07-06 起）啟動時會自己 spawn 一個帶 `--guardian <PID>` 的守護子程序——只殺主程序，守護者會在 ~10 秒內把它救回來，變成打不死。兩個都要殺：

```powershell
Get-CimInstance Win32_Process -Filter "Name='room_monitor.exe'" | ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }
Get-CimInstance Win32_Process -Filter "Name='python.exe' OR Name='pythonw.exe'" |
    Where-Object { $_.CommandLine -like '*room_monitor.py*' } |
    ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }
Start-Sleep -Seconds 2
```

### 2. 啟動 `.py` 原始碼（不是 exe）

```powershell
Start-Process python -ArgumentList "-u", "D:\SFAA\fix\room_monitor.py"
```

不用管視窗顯示與否——`setup_log()` 一開始就把 stdout/stderr 重導向到 log 檔，不看 console 也看得到輸出。

**log 路徑跟 exe 不一樣**：因為沒有 `frozen`，`setup_log()` 用 `os.path.abspath(__file__)` 當 base，所以這次的 log 會寫到：
- `D:\SFAA\fix\room_monitor.log`（一般執行 log）
- `D:\SFAA\fix\dropout_diag.log`（`attach_diagnostics` 診斷 log）
- `D:\SFAA\fix\dispatch_log.jsonl`（派線紀錄）
- `D:\SFAA\watchdog.log`（守護子程序，這個路徑寫死不受 frozen 影響）

不是 `D:\SFAA\fix\dist\` 底下那幾份（那是 exe 專用的）。

### 3. 啟動後才檢查 9222（不要用啟動前的舊結果）

跟 `sfaa-ecp-room-monitor` skill 裡的「已知坑」一樣：`taskkill` + `sleep(2)` + Chrome 重新啟動需要幾秒鐘，啟動指令送出後**稍等幾秒**再檢查：

```powershell
Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue | Where-Object { $_.LocalPort -eq 9222 }
```

### 4. 接管 Chrome（走 `connect-chrome` skill 的正常流程）

port 9222 開了之後直接接管（`playwright-vrs` 是固定常駐的 MCP server，不用再檢查 `.mcp.json`/`enabledPlugins`，那是舊架構的步驟）：

```
mcp__playwright-vrs__browser_tabs(action: "list")
```

看到 VRS/ECP 頁面即代表接管成功，接下來就能用 `browser_snapshot`/`browser_click`/`browser_evaluate` 正常操作、觀察 log 找 bug。**接管後第一件事永遠是先跑 `python "C:\Users\HCH\.claude\skills\debug-vrs\scripts/set_unready.py"` 切未就緒/建檔中**（見上方鐵則），不要用 `browser_snapshot`/`browser_click` 手動摸索，才能繼續做其他事。

### 4.5 常見操作：切換服務類別（1001/2001/2002 等）

不要每次都用 `browser_snapshot`/`browser_click` 手動摸索一遍——已經寫成腳本 `scripts/set_service_category.py`（同目錄），直接跑：

```powershell
python "C:\Users\HCH\.claude\skills\debug-vrs\scripts/set_service_category.py" --enable 2001,2002 --disable 1001 --keep-unready
```

腳本會自己：登入服務(若未登入) → 切未就緒/建檔中 → 開服務類別設定 → 勾選/取消勾選指定編號 → 按確定 → (`--keep-unready` 的話)因為 ECP 按確定後會自動變回就緒，再切一次未就緒。技術細節都寫在 `scripts/vrs_common.py` 和腳本開頭的註解裡。

**編號語意（重要）**：1001 是「正式環境」服務類別，2001/2002 是測試用；除非明確要開正式服務，否則**不要勾選 1001**，`scripts/set_service_category.py` 的預設值就是 `--enable 2001,2002 --disable 1001`。

**這支腳本的存在本身就是規則的示範**（見全域 `C:\Users\HCH\.claude\CLAUDE.md` 的「固定/重複性工作要寫成腳本」鐵則）：第一次是花時間用 `browser_snapshot`/`browser_click`/`browser_evaluate` 慢慢摸索出正確流程，摸清楚後就寫成腳本存起來，下次直接跑，不必重新推理 UI。

**2026-07-06 事故記錄（同一個 debug session 連續三次調校才穩定）**：

1. 第一版腳本（跟 `scripts/set_unready.py` 最初版一樣）用 `document.querySelectorAll('iframe')` + `contentDocument` 手動爬 frame 樹算絕對座標、`page.mouse.click(x, y)` 點擊。第一次在切未就緒步驟誤點導覽到 `vrs.sfaa.gov.tw` CRM 頁面；修好那步後，第二次在開服務類別設定步驟又誤點導覽、還順帶把帳號登出服務。改用 Playwright 自己的 `page.frames` + `frame.get_by_text(...)`，不手動算座標，且每次點擊後都驗證網址沒有意外改變、預期的下一個元素有出現，任何一步不對就中止不再繼續點——這部分修好後就沒再誤點導覽過。
2. 勾選格的點擊目標一開始猜錯兩次：先猜是資料列（`startswith(code)` 那個 `<tr>`）的第一個 `<td>`——實測是「編號」文字格，點了沒反應；改猜資料表格的 `previousElementSibling`——實測每個 zone `<div>` 裡只有一個 `<table>`，table 本身沒有 sibling，找不到目標。最後用 `browser_evaluate` 直接把 DOM 挖出來看，才確認真正結構：

   ```
   <div class="JuiList">
     <div class="JuiListZone2">  <!-- 勾選欄，獨立的 <table> -->
       <table><tr><td class="JuiListRowNumberCell">1</td>
                  <td class="JuiListCheckCell">...</td></tr> ...</table>
     </div>
     <div class="JuiListZone3">  <!-- 資料欄，另一個獨立的 <table> -->
       <table><tr><td>1001</td>...</tr> ...</table>
     </div>
   </div>
   ```

   勾選欄跟資料欄是兩個**左右並排、各自獨立**的 `<table>`，只靠列的順序（index）對應，不是同一張表格的欄位。正確做法：先找到資料列在自己表格裡的第幾列（idx），再從同一個 `.JuiList` 容器裡的 `.JuiListCheckCell`（勾選格唯一的 class，順序跟資料列一致）取第 idx 個，用 `frame.evaluate_handle()` 拿到該格的 ElementHandle 後 `.click()`（真實滑鼠事件，JS `element.click()` 對這個自訂 grid 沒用）。已寫進 `scripts/vrs_common.py` 的 `find_checkbox_handle()`，並在 1001/2001/2002 三個編號、雙方向切換都實測成功。

**已知限制**：如果目標值本來就已經是當前狀態（例如 1001 本來就沒勾、要求 disable 1001），腳本會偵測到並跳過，印出「已經是目標狀態」；「未就緒」下拉選單也有類似情況，見 `scripts/set_unready.py` 段落。

**如果這支腳本失敗或跟預期不符**（ECP 改版、UI 結構變了等）：回去用 `browser_snapshot`/`browser_evaluate` 手動排查、找出新的正確做法，**然後務必回頭修改這支腳本**（或視情況另存一支新腳本），並更新這份 SKILL.md 說明新的用法——不要修好這一次就算了，下次還是要能直接跑腳本，不必再重新推理一遍。

### 5. 結束 debug session

系統匣圖示右鍵「結束」會連同守護子程序一起收掉（`exit_app()` 內部會先 `_kill_guardian()`）。若要直接砍 process 收工，記得比照步驟 1，主程序跟 `--guardian` 子程序都要殺，不然守護者會自己重啟一個回來。

---

## Conformance Addendum

## When to Use
啟動 D:\SFAA\fix\room_monitor.py 原始碼（不是打包好的 exe）來即時除錯，然後接管它的 Chrome 進行操作。當使用者要求「debug vrs」、「除錯 room_monitor」、「跑 room_monitor.py 來測」、「用原始碼測 SFAA 監控」時使用。

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
