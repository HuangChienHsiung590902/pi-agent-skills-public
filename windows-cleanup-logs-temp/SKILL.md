---
name: "windows-cleanup-logs-temp"
description: "清除 Windows 系統暫存檔、快取、Windows Update 快取與各類系統 Log，讓系統維持乾淨（適用需要管理員權限的 Windows 環境，含 UAC 已停用的情境）"
version: 1
created: "2026-08-14"
updated: "2026-08-14"
---
## When to Use
當使用者要求「清理 Windows 系統」、「清暫存檔」、「清 log」、「讓電腦像新的一樣乾淨」、或要求清除 CBS.log / WindowsUpdate log / Event Viewer 記錄時使用。適用於本機 Windows 環境（透過 Bash tool 呼叫 powershell），且目前 shell 通常是非管理員權限，需要用 Start-Process -Verb RunAs 或 UAC 已停用時直接提權執行寫入類操作。

## Procedure
1. 【第一步：先掃描不要直接刪】用 PowerShell 分項統計以下路徑大小，彙整成表格給使用者看，讓使用者確認要清哪些，不要不問就刪：
  - 系統暫存: C:\Windows\Temp
  - 使用者暫存: $env:LOCALAPPDATA\Temp (或指定使用者路徑 C:\Users\<user>\AppData\Local\Temp)
  - Windows Update 快取: C:\Windows\SoftwareDistribution\Download
  - 縮圖快取: $env:LOCALAPPDATA\Microsoft\Windows\Explorer (filter thumbcache*)
  - 瀏覽器快取 (Chrome 範例): $env:LOCALAPPDATA\Google\Chrome\User Data\Default\Cache
  - 資源回收筒 (New-Object Shell.Application 的 Namespace(10))
  - Windows.old、系統還原點 (Get-ComputerRestorePoint)
  - 各類系統 log: C:\Windows\Logs\CBS, WindowsUpdate, MoSetup, WinREAgent, MeasuredBoot, NetSetup, waasmedic; C:\Windows\Panther; C:\Windows\INF\setupapi*.log; C:\Windows\Logs\DISM\dism.log
2. 【第二步：確認清除範圍】把掃描結果分兩類呈現給使用者：✅ 可安全清除（暫存、快取、系統 log，不影響已安裝軟體與個人資料）；⚠️ 需要使用者確認（瀏覽器登入資料、已安裝軟體清單、開機啟動項等）。等使用者明確同意才動手，一次做一批，不要一口氣全部下手。
3. 【第三步：寫成 .ps1 腳本再執行，不要用一行 inline 指令】因為 Bash tool 傳遞 PowerShell 多行/含特殊符號指令容易被 shell 轉義搞爛（unexpected EOF、變數插值出錯等），穩定作法是: 用 Write 工具把清除邏輯寫成一個 .ps1 檔案（例如放在使用者桌面 C:\Users\<user>\Desktop\xxx.ps1），每個清除動作用 try/catch 包起來，清除前後都記錄大小，最後用 Out-File 把結果寫到一個 result.txt 檔案。
4. 【第四步：以管理員權限執行】目前 shell 通常是一般使用者權限，直接 Remove-Item 系統資料夾會被拒絕 (Access denied / SecurityException)。用以下方式提權執行：
  powershell -Command "Start-Process powershell -Verb RunAs -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File','<ps1路徑>'"
  若使用者的 UAC 已停用，這行會直接以管理員權限跑完，不會跳提示視窗；若 UAC 有開啟，會跳 UAC 授權視窗，需請使用者點「是」。執行後用 sleep 等待數秒（依清除量調整 10~20 秒）再去讀 result.txt。
5. 【第五步：清除 Windows Update 快取前要先停服務】清 C:\Windows\SoftwareDistribution\Download 前必須先 Stop-Service wuauserv, bits（否則檔案被鎖定刪不掉），清完後要 Start-Service 把兩個服務啟動回來，並在驗證階段確認服務狀態恢復 Running。
6. 【第六步：清除 Event Viewer 事件記錄】用 wevtutil el 列出所有 log channel，逐一 wevtutil cl "<logname>" 清空。這個動作本身會在 System log 留下一筆 Event ID 104（記錄檔已清除）的紀錄，這是 Windows 機制上無法避免的，屬正常現象，要跟使用者說明清楚，不要誤以為清除失敗。
7. 【第七步：驗證清除結果】清除完成後，重新對每個路徑跑一次大小統計，並列出前後對照表給使用者看實際釋放的空間（MB/GB）。同時檢查關鍵服務 (wuauserv, bits, TrustedInstaller) 是否恢復正常運作（bits 和 TrustedInstaller 顯示 Stopped 是正常的，它們是按需啟動服務）。
8. 【第八步：清理暫存腳本】清除任務完成、驗證無誤後，用 rm -f 把桌面上暫時建立的 .ps1 腳本與 result.txt 記錄檔一併刪除，不要留下操作痕跡在使用者桌面。

## Pitfalls
- 不要用 Bash tool 直接塞多行 PowerShell 含變數插值的 inline 指令（例如用 \$var 跳脫），容易出現 'unexpected EOF while looking for matching' 這類 shell 解析錯誤；一律先寫成 .ps1 檔案再執行最穩定。
- 不要沒有掃描、沒有跟使用者確認清單就直接刪除系統資料夾，尤其是已安裝軟體清單、瀏覽器登入資料這類會影響使用者體感的項目，務必先列出來給使用者選。
- 刪除 C:\Windows\SoftwareDistribution\Download 前忘記停用 wuauserv/bits 服務，會導致部分檔案刪除失敗（檔案被佔用）。
- 清除系統資料夾（C:\Windows\Temp、C:\Windows\Logs 等）需要管理員權限，一般 shell 直接 Remove-Item 會出現 'Access to the path is denied' 或 'Requested registry access is not allowed'，必須用 Start-Process -Verb RunAs 提權執行。
- 事件記錄清除後，System log 一定會留下 wevtutil 清除動作本身產生的 Event ID 104 紀錄，這是正常現象不是清除失敗，不要誤判。
- 清除軟體殘留（如解除安裝後的登錄檔/資料夾殘留）時，要先用 Test-Path / Get-ChildItem 逐一列出實際找到的殘留項，避免用模糊比對條件誤刪其他無關的系統原生檔案（例如檔名中恰好包含類似字母的 Windows 內建驅動，如 dfsc.sys 跟 Deep Freeze 無關）。
- 不要一次把所有清理項目（暫存、log、瀏覽器資料、已安裝軟體、開機啟動項等）混在同一批直接執行，應分批進行，每批做完先回報結果給使用者，再詢問是否繼續下一批。

## Verification
1. 清除後重新執行大小統計指令，確認目標資料夾大小明顯下降（通常只剩幾百 KB 的執行中殘留檔案，屬正常）。
2. 確認 wuauserv 服務狀態為 Running（若清過 Windows Update 快取）。
3. 確認沒有跳出任何 Access Denied 或 SecurityException 錯誤訊息在 result.txt 裡；若有，代表提權執行沒有成功，需要重新確認 UAC 狀態或改用其他提權方式。
4. 彙整一份「清除前 → 清除後」的大小對照表回報給使用者，讓使用者清楚知道實際釋放了多少空間。
5. 確認暫時建立的 .ps1 與 result.txt 檔案已從桌面移除，沒有留下操作痕跡。

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
