---
name: deepfreeze-symlink-restore
description: 在裝了 Deep Freeze（Faronics）的機器上，讓 C:\Users\<user> 底下指向 D 槽的設定資料夾 symlink、家目錄下的單一設定檔（.claude.json / .gitconfig 等）、使用者環境變數，在每次重開機/凍結還原後自動修復。當 C 槽被磁碟凍結軟體保護、重開機後會清空手動建立的 symlink、設定檔或環境變數時使用。
---

# Deep Freeze Symlink 與環境變數還原

## When to Use
當使用者的 C 槽被 Deep Freeze（或類似磁碟凍結軟體）保護、重開機後會還原成凍結時的快照，導致以下三類東西消失或被清空，需要一個修復機制：(1) 手動建立、指向 D 槽的 symlink（例如 `C:\Users\HCH\.claude -> D:\.system\.claude`）；(2) 使用者層級環境變數（HKCU\Environment，實體存在 `C:\Users\<user>\NTUSER.DAT`，一樣在凍結範圍內），例如 CARGO_HOME/RUSTUP_HOME/PATH 裡新增的自訂路徑；(3) Windows 工作排程器的任務定義本身（存在 `C:\Windows\System32\Tasks\`，同樣在 C 槽凍結範圍內，實測會在重開機後被清空消失，不是可靠的開機自動觸發機制）；(4) **家目錄底下的單一設定「檔案」**（不是資料夾，所以不會被資料夾清單涵蓋到），實測被清空的包括 `C:\Users\<user>\.claude.json`（Claude Code 的已安裝 plugin 註冊表 + 頂層 MCP 設定，被清空後所有 plugin 會靜默消失，即使 `.claude` 資料夾本身安全地在 D 槽）、`.gitconfig`、以及 `.ssh/`（known_hosts 消失會導致 git clone 報 Host key verification failed）。這些即使對應的程式本體裝在 D 槽、重開機後也可能因為 C 槽被還原而讀不到或無法自動觸發。適用於任何「C 槽會被清空，但重要設定/程式要長期保存在未凍結磁碟（如 D 槽）」的情境。

## Procedure
1. 先確認凍結軟體與凍結範圍：用 `tasklist | grep -i frz` 或類似指令找凍結監控程序（如 Faronics 的 FrzState2k.exe），並用登錄檔 `HKLM:\SOFTWARE\WOW6432Node\Faronics\Deep Freeze *` 或磁碟標籤（例如非系統碟命名為 DATA）判斷哪個磁碟是 ThawSpace（不凍結）。可進一步查 `HKLM:\SYSTEM\CurrentControlSet\Services\DeepFrz\Parameters` 的 Enum 清單確認實際被接管的磁碟區 GUID，以及 Parameters 裡的 PersiPath（通常是 `C:\PersiN.sys`，一個由 DeepFrz 驅動即時鎖定佔用的持久化容器檔案，代表 Deep Freeze 有「部分路徑可持久化」的機制，但其內容是加密格式，不建議嘗試讀取或修改）
2. 列出目前 `C:\Users\<user>` 底下所有 symlink 及其目標：用 PowerShell `Get-ChildItem -Force | Where-Object {$_.Attributes -band [System.IO.FileAttributes]::ReparsePoint}`，記錄每個 name -> target 對應表，同時列出需要保護的使用者環境變數目前值與 PATH 裡自訂新增的路徑條目
3. 寫一個 PowerShell 還原腳本（例如 `D:\.system\scripts\restore-symlinks.ps1`），分兩個部分：Part 1 處理 symlink，對每個項目檢查目標是否存在、現有路徑是否已是正確 symlink（用 Get-Item 的 .Target 比對），已正確的跳過（[OK]），不存在或被還原成空資料夾的用 `New-Item -ItemType SymbolicLink` 重建（[FIX]），若偵測到是「有實際內容的真實資料夾」則只記錄警告不要自動刪除（[WARN]）；Part 2 處理環境變數，用 `SetEnvironmentVariable('User')` 冪等地補回缺少或錯誤的值，PATH 用陣列定義必須包含的條目清單、逐一檢查不重複再 append，避免整個覆蓋使用者其他自訂的 PATH 項目；兩部分結果都 append 進同一個日誌檔方便事後檢查
4. 對於家目錄底下的單一設定「檔案」（.claude.json / .gitconfig 等），另寫一段 Part 1b 處理，不能只拄資料夾：先把目前 C 槽的檔案複製到 D 槽當 master，再建 `New-Item -ItemType SymbolicLink` 指回去（檔案也能建 symlink，不只資料夾）。**關鍵差異：必須加 salvage 邏輯** —— 很多程式存設定檔是「寫暗存檔再改名覆蓋」的 atomic write，這會把 symlink 整個換成一個全新的真實檔案，此時最新設定反而在 C 槽那個真實檔裡；若腳本直接刪掉它重建連結，使用者的設定會被靜默回滾。正確做法：偵測到 C 槽是真實檔時，比對 LastWriteTimeUtc，比 D 槽 master 新就先 Copy-Item 覆寫回 master（[SAVE]）再重建 symlink，舊的才丟棄（[INFO]）
5. 手動測試還原腳本本身：先在管理員 PowerShell 下執行一次確認全部 [OK]，再手動刪除一個 symlink、清空一個環境變數模擬凍結還原情境，重新執行確認能自動修復（[FIX]）
5. 不要只依賴 Windows 工作排程器當作唯一的登入自動觸發機制：實測發現即使用 `New-ScheduledTaskTrigger -AtLogOn` + RunLevel Highest 正確註冊且 `Get-ScheduledTask` 顯示 Ready，重開機後任務定義本身（存於 `C:\Windows\System32\Tasks\`）仍然會被清空消失（`Get-ScheduledTask` 查不到，也沒改名），因為任務排程器的任務定義本身也在被凍結的 C 槽範圍內，不能當成可靠的持久化機制，只能當成「如果…就有額外幫助」的選項
6. 若想查證 Deep Freeze 本身有沒有官方的路徑持久化白名單功能（例如透過 DFC.exe 主控台），先上網查證再在 DFC 主控台裡尋找：實測發現 DFC Enterprise Console 的 ThawSpaces 右鍵選單只有 Format/Delete，沒有「特定路徑持久化白名單」這種設定，因為這個功能其實不在 DFC 裡，而是一個另外獨立、免費的官方工具叫 Faronics Data Igloo（會把指定資料夾/登錄檔鍵值用 NTFS junction 重新導向到 ThawSpace），需要另外下載安裝才能用，不是 DFC 內建功能；若使用者覺得安裝新工具麻煩，可直接跳過、改用下一步的桌面捷徑方案
7. 當登入自動觸發不可靠時，改建一組「手動雙擊即可提權執行」的桌面捷徑作為備援：(a) 建一個精簡的 .bat（只做一件事：`powershell -NoProfile -ExecutionPolicy Bypass -File "<elevate腳本路徑>"`，不要在 .bat 裡包裝層層嵌套引號的 Start-Process 指令），(b) 另寫一個專門的 elevate-and-run.ps1，內容只做 `Start-Process powershell -ArgumentList @(...) -Verb RunAs -Wait` 去提權執行真正的還原腳本，執行完後印出完成訊息並用 try/catch 包住 ReadKey 避免沒有互動主控台時報錯，(c) 用 PowerShell 的 WScript.Shell ComObject 建立 .lnk 捷徑到桌面、TargetPath 指向那個 .bat，給一個清楚的中文檔名提醒使用者重開機後雙擊執行
8. 實測整條鏈路（雙擊 .lnk）：先確認使用者的 UAC 是否已關閉（若已關閉，`-Verb RunAs` 會直接以管理員權限執行不會跳確認視窗，不需要一直提醒使用者點「是」），手動模擬損壞後用 Start-Process 呼叫 .lnk 觸發，確認 symlink/環境變數真的被修復，而不是只看到舊的 log 時間戳就誤以為成功

## Pitfalls
- `New-Item -ItemType SymbolicLink` 和 `Register-ScheduledTask` 都需要系統管理員權限，在一般權限的 shell 裡直接呼叫會得到錯誤，記得用 `Start-Process ... -Verb RunAs -Wait` 提權執行；若使用者已關閉 UAC，這會直接以管理員權限執行不會跳確認視窗，不要一直提醒使用者點確認
- 不要對「有實際內容的真實資料夾」自動刪除重建 symlink，可能會誤刪使用者新產生但還沒來得及搬移的資料，腳本應該只在該路徑為空或不存在時才動手，遇到非空真實資料夾應只記錄警告讓人工確認
- PowerShell 印出中文時透過 bash 管線容易亂碼，是編碼轉換問題不代表指令本身失敗，可改用 `Out-File -Encoding utf8` 寫檔後用 Read 工具讀取來確認實際內容
- 只處理「之前手動建立、應該指向 D 槽的」symlink 清單，不要動 Windows 內建的特殊捷徑（如 Application Data、Cookies、My Documents 等 junction），那些是系統機制的一部分，本來就會自動存在
- 任務排程器用 `Get-ScheduledTaskInfo` 查詢時 LastTaskResult 267011 是「尚未執行過」的正常代碼，不是錯誤，不要誤判；但更重要的是別把任務排程器當成可靠的持久化機制，實測確認即使任務注冊成功且 State=Ready，重開機後任務定義本身仍會消失（`Get-ScheduledTask` 查不到，也未改名），因為任務定義檔存於 `C:\Windows\System32\Tasks\`，也在被凍結的 C 槽範圍內
- 使用者層級環境變數（HKCU\Environment）實體存於 `C:\Users\<user>\NTUSER.DAT`，這個檔案也在被凍結的 C 槽範圍內，即使程式本體（如 Rust/GCC）裝在未凍結的 D 槽，重開機後 PATH/CARGO_HOME 之類的環境變數仍有可能失效，導致指令找不到，不能只保護檔案本體而忽略環境變數層面
- 不要嘗試讀取或解析 Deep Freeze 的 PersiN.sys 持久化容器檔，這是驅動即時鎖定使用中的加密格式檔案，強行讀取會報 Device or resource busy，也不應嘗試修改
- PowerShell 腳本若包含中文字串（例如 `Write-Host "按任意鍵..."`），當檔案寫入時的編碼與 PowerShell 實際執行時解析的編碼不一致，可能導致字串裡的引號被錯誤解讀，整個腳本因語法錯誤而從頭沒執行到任何實質邏輯（包括關鍵的 `Start-Process -Verb RunAs`），但外部觀察可能看不到任何明顯錯誤訊息（因為包裝層沒把錯誤回傳給使用者），這種「靜默失敗」最危險，發現方式是比對 log 檔的時間戳是否真的是本次執行產生的，而不是舊的殘留記錄；修正方式是把所有互動提示文字改成純英文，避免字元集問題
- 包裝 .ps1 的 .bat 檔不要在一行裡包裝層層嵌套引號的複雜指令（例如在 .bat 裡直接寫 `powershell -Command "Start-Process ... -ArgumentList '...' -Verb RunAs"` 這種兩層引號嵌套），cmd.exe 對引號跨層解析極易斷裂，會報 'xxx' 不是內部或外部命令。正確做法是 .bat 只做一件事（呼叫一個 .ps1 檔），複雜的提權邏輯全部放到那個 .ps1 裡用 PowerShell 本身的陣列語法（`-ArgumentList @(...)`）處理，不要靠手動拼字串跨層傳遞
- 用 bash 呼叫 `cmd.exe /c` 時只看到 banner 沒看到實際執行輸出，不代表指令沒執行成功，但也不能當成成功的證據，這種情況下應改用直接執行 .ps1 本身（而非透過 cmd）來取得真實錯誤訊息，或直接檢查實際修復結果/log 時間戳作為判斷依據
- 驗證腳本不要寫成 bash 包貝的 `powershell -Command "..."` 內联指令：路徑字串裡的 `\` 與變數插值 `$name` 經過 bash 雙引號 + PowerShell 兩層解析後極易損壞，實測發生過全部 15 個 symlink 明明正常却被報成 BAD 的假警報（而且連續發生兩次）；正確做法是把驗證邏輯寫成一個獨立的 .ps1 檔（例如 verify-restore.ps1）再用 `-File` 執行，完全避開引號嵌套
- 驗證 symlink 是否修復成功時，不要寫死比對 `(Get-Item $p).LinkType -eq 'SymbolicLink'`：用 `New-Item -ItemType SymbolicLink` 對目錄建立、且以管理員權限執行時，實測發現 `Get-Item -Force` 回報的 `.LinkType` 實際值可能是 `Junction` 而不是 `SymbolicLink`（雖然建立時指定的是 SymbolicLink），若驗證腳本寫死比對 `SymbolicLink` 字串會誤判全部失敗（實際上已經修復正確）；正確做法是直接檢查 `.Target` 屬性是否指向預期的 D 槽路徑，不要只比對 `.LinkType` 字串（`.Target` 可能是陣列，要先取 `[0]`）
- 只保護資料夾是不夠的：家目錄底下還有一批單一設定「檔案」同樣在凍結範圍內，而且因為不在資料夾清單裡而很容易被漏掉，實測確認會被清空的至少有 `.claude.json`、`.gitconfig`、`.ssh/`；征兆往往很晤澀（plugin 莫名其妙消失、git clone 突然報 Host key verification failed 或要求輸入密碼），不會直接說「你的設定檔不見了」
- `.ssh/` 被清空後如果 D 槽沒有任何備份，裡面的私鑰就是真的沒了、無法復原，只能重新產金鑰並往後保護；known_hosts 則可以用 `ssh-keyscan -t rsa,ecdsa,ed25519 github.com > <D槽>/known_hosts` 重建。修好後的正確征兆：ssh 錯誤會從 Host key verification failed 變成 Permission denied (publickey)，後者代表主機金鑰已認得、只是沒私鑰，是預期的下一階段錯誤

## Verification
1. 手動刪除任一目標 symlink、清空目標環境變數（如 CARGO_HOME/RUSTUP_HOME）後，直接執行還原腳本或透過提權包裝器觸發，確認都能自動補回
2. 若有建立任務排程器任務，不要只相信 `Get-ScheduledTask` 顯示 Ready 就認為可靠，必須實際重開機一次後再用 `Get-ScheduledTask` 確認任務定義是否仍存在，這是唯一能確認任務排程器真的能跨重開機存活的方式，模擬觸發（`Start-ScheduledTask`）只能驗證任務內容正確，並不能證明它重開機後還在
3. 若改用桌面捷徑備援方案，逐層驗證：(a) .ps1 本身用 `[System.Management.Automation.PSParser]::Tokenize` 或 `powershell -NoProfile -Command` 直接執行確認語法沒問題，(b) 直接執行 .ps1 確認提權邏輯本身有執行，(c) 最後用 Start-Process 呼叫 .lnk 檔模擬真實雙擊情境，確認整條鏈路都正常
4. 每次驗證修復結果時，必須比對 log 檔的時間戳是否為本次執行產生，不能只看 log 尾部內容就誤以為成功（曾發生過腳本因中文字串語法錯誤靜默失敗，但 log 裡看到的卻是之前成功執行留下的舊記錄，造成誤判）
5. 用一個完全乾淨、不繼承目前 shell 環境變數的新 process 去執行目標指令，確認 exit code 0，這比在已有舊環境變數的 session 裡測試更接近真實重新登入情境
6. 最終驗證：實際重開機一次，登入後檢查所有目標 symlink 是否存在且正確、新終端機直接打指令是否不用手動設定任何變數就能執行，若使用桌面捷徑方案，也要確認重開機後雙擊捷徑真的能修復所有項目，這是模擬測試無法完全取代的最後一關
7. 用獨立的只讀驗證腳本 `D:\.system\scripts\verify-restore.ps1` 一次檢完所有項目（資料夾連結 / 檔案連結 / 環境變數 / PATH），每項印 PASS 或 FAIL 並給總結；它寫成實體 .ps1 而非內联指令，就是為了避開 bash/PowerShell 引號嵌套造成的假警報
8. （2026-08-13 實測通過）實際真實重開機一次後，重開前 log 顯示 14 個 symlink 全部 [OK]、重開後未執行捷徑前全部變回 [WARN]/不存在，CARGO_HOME/RUSTUP_HOME/PATH 也全部歸零，確認 Deep Freeze 真的會清空這些項目；雙擊桌面捷徑後，新產生的 log 時間戳確認為本次執行，14 個 symlink 全部修復、環境變數全部補回，證實這套機制能真實擐過 Deep Freeze 重開機清空，不只是理論上有效
9. （2026-08-13 實測通過）檔案層 symlink 的實際行為驗證：Claude Code（`claude plugin list`）與 git（`git config --global`）寫入設定時，**都不會破壞 symlink**，寫入會直接落到 D 槽 master（以 Reparse=True + 時間戳/內容更新確認），代表這兩個程式適用 symlink 方案；salvage 邏輯也實測過：手動把 symlink 換成較新的真實檔後執行腳本，log 出現 [SAVE] 且新設定確實被救回 master、symlink 也正確重建

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
