---
name: t25-xampp-dashboard
description: T25 (10.145.119.209) 上用 Bitnami XAMPP 跑的自建文件檢視 dashboard（Excel/Word/PPT/PDF/CSV 線上預覽 + html）。含：SSH 連線方式（port 2222、密碼登入、sshpass 在這台機器的 git bash 會失效要改用 plink）、遠端中文/emoji 顯示亂碼但資料沒壞的判斷方式、Apache 目前非服務、只監聽 88 埠、且是前景模式跑的已知隱患、dashboard 原始碼架構（list.php/file.php 白名單、files/ 目錄的 .htaccess 保護、app.js 依副檔名分派預覽、index.php 的 app.js?v= 快取版本號機制），以及如何安全地新增支援的檔案類型（已示範新增 html 支援並用 sandboxed iframe 隔離）。當使用者提到 T25、10.145.119.209、xampp、htdocs、或那個文件總管/dashboard 網頁打不開、看不到新放的檔案時使用。
---

# T25 (10.145.119.209) — XAMPP + 自建文件 Dashboard

## 這是哪台機器

T25，`10.145.119.209`，跟 `flutter-tray-app-deploy` skill 提到的資產管理 tray app 目標機是同一台實體機器。這個 skill 專門記錄它上面另一個完全獨立的東西：一個用 Bitnami XAMPP 架設、自建 PHP 的文件總管/預覽網頁（在 `C:\xampp\htdocs\dashboard\`）。

## SSH 連線（2026-07-28 實測）

- **Port 2222**，不是預設 22（22 會直接 timeout，防火牆/NAT 只開了 2222）。
- 帳號 `Administrator`，**密碼登入**（不是金鑰）。`flutter-tray-app-deploy` skill 舊筆記寫「金鑰式、已在 known_hosts 信任」是過時資訊——實測那把本機金鑰在這台上 publickey 會被直接拒絕，只有密碼能過。
- **這台機器的 git bash 環境下 `sshpass` 會失敗**：即使密碼正確也回報 `Permission denied`，log 裡會看到 `read_passphrase: can't open /dev/tty`——sshpass 在 MINGW/Git Bash 底下模擬不出 tty，密碼實際上沒送出去。改用 Windows 內建的 **plink（PuTTY）** 才可靠：

```powershell
# 第一次連線需先接受 host key，避免 -batch 模式因無法確認 host key 直接 abort
# 錯誤訊息裡會印出 SHA256 指紋，直接帶入 -hostkey 參數即可跳過互動確認
plink -ssh -batch -hostkey "SHA256:K9qEr+sPWtakF9iCQ5OW8Ug+lYOrDVr616t0+nL7dD8" -P 2222 -pw '密碼' Administrator@10.145.119.209 "指令"
```

密碼跟使用者索取，不要寫死在 skill 檔案裡。

## 連線本身會間歇性整個斷掉，不只是 Apache 的問題

2026-07-28 觀察到：原本刻意留著、用來讓前景模式 Apache 保持活著的那條 SSH session 結束後，不久整台機器連 **port 2222 都直接 timeout**（`plink ... FATAL ERROR: Network error: Connection timed out`），連基本的 `echo alive` 都連不上——不是 Apache/port 88 的問題，是 SSH 本身連不進去。過一段時間後續重試通常會恢復。

**遇到 SSH 連線 timeout 時的判斷順序：**
1. 先分辨是「port 2222 SSH 本身連不上」還是「port 88 Apache 連不上但 SSH 正常」——兩者原因完全不同，前者是網路/機器層級（可能重開機、進入睡眠、防火牆狀態變化），後者才是本 skill 其他章節講的 Apache 前景模式問題。
2. 純網路層級的 timeout **值得重試幾次**，不要一次連不上就斷定機器掛了或防火牆規則被改掉——目前已知它會間歇性恢復。
3. 如果重試多次仍持續連不上，才需要請使用者去現場確認機器是否還開著、或者防火牆/NAT 設定是否真的變了。

## 遠端輸出中文/emoji 顯示亂碼，是顯示問題不是資料問題

- 遠端是繁體中文 Windows，`cmd.exe` 主控台預設用 Big5(950) codepage。透過 `plink ... "dir"` 這種直接執行 `cmd` 指令的輸出，中文檔名/資料夾名會變亂碼（例如 `�ϺЬO`），**這只是終端機顯示壞掉，磁碟上的實際檔名是對的**，看到亂碼不代表資料壞了。
- `type 某檔案.php`（cmd 直接吐出 raw bytes）通常沒事，因為原始碼檔案本身是 UTF-8、cmd 沒有重新編碼，是原樣吐出。查看原始碼內容優先用這招。
- 若改用 `powershell -Command "Get-Content ... "` 再印到主控台，PowerShell 自己的主控台輸出一樣會套用 Big5，中文（尤其超出 Big5 範圍的 emoji/特殊符號）一樣會被打成 `?`。
- **要修改遠端檔案內容**：全程在 PowerShell 腳本內用 `[System.IO.File]::ReadAllText/WriteAllText` 處理，不要讓要寫入的中文字串經過主控台這種有損的來回路徑。用 `-EncodedCommand`（Base64 包住 UTF-16 編碼的腳本）把腳本送過去最保險；腳本裡直接寫死要替換成的新字串（自己 authoring 的內容一定正確），比對舊內容時只用純 ASCII 或常見繁體中文片段當「錨點」去 `.Replace()`，不要嘗試比對 emoji 這種容易在傳輸過程中被打壞的字元。寫回檔案務必用 `New-Object System.Text.UTF8Encoding($false)`（不加 BOM）——PHP 檔案開頭如果混進 BOM 會導致 `headers already sent` 之類的執行期錯誤。

## Apache 現況與已知隱患

- **沒有安裝成 Windows 服務**（`sc query apache2.4` 回 1060 找不到服務）。連 `apache_stop.bat` 裡都還留著沒被替換掉的 Bitnami 安裝樣板變數 `@@BITROCK_INSTALLDIR@@`，代表這台當初裝機時這個環節本來就沒收尾好，`apache_stop.bat` 目前其實是壞的。
- `httpd.conf` 設定 `Listen 88`，**不是預設的 80**。任何要給的連結都要記得加 `:88`（例如 `http://10.145.119.209:88/xxx`），直接省略埠號會連不上。
- 啟動方式是 `C:\xampp\apache_start.bat`，這支 bat **前景執行** `apache\bin\httpd.exe`（沒用 `-k start`），會佔住呼叫它的那個 console/SSH session 不放。2026-07-28 當時的做法是刻意留著那條 SSH 連線卡在背景讓 Apache 活著，使用者被告知風險後選擇「先維持現狀」，還沒改成正式 Windows 服務。
  - **後續影響**：那條維持 Apache 存活的 SSH session 一旦斷線，Apache 大概率會被一起砍掉。之後如果回報「網站打不開/dashboard 連不上」，先用 `tasklist | findstr httpd` 確認 Apache 是不是已經掛了，是的話重新跑一次 `apache_start.bat`（一樣要留著那條連線）。
  - 如果要一勞永逸解決，做法是裝成真正的 Windows 服務：`C:\xampp\apache\bin\httpd.exe -k install` 之後用 `net start`／`sc`管理，這樣重開機、斷線都不受影響——但這是之前特意問過使用者、對方選擇先不做的事，要做之前先跟使用者確認一次。
- 設定語法正確與否可用 `C:\xampp\apache\bin\httpd.exe -t` 快速驗證，改完設定檔或懷疑設定壞掉時先跑這個。

## `dashboard` 這個自建網頁在幹嘛

路徑：`C:\xampp\htdocs\dashboard\`，一個給員工瀏覽/預覽 Office 文件（Excel/Word/PPT/PDF/CSV）的內部工具，需登入（`auth.php`／`login.php`）。前端用到 `chart.umd.min.js`（試算表）、`mammoth.browser.min.js`（docx→html）、`pdf.min.js`+`pdf.worker.min.js`（PDF）、`jszip.min.js`+`xlsx.full.min.js`（pptx/xlsx 解析）。

**架構重點：**

- 真正的檔案放在 `dashboard\files\` 底下（子資料夾如「範例」「SFAA」等，對應前端「檔案目錄」樹狀結構）。
- `dashboard\files\.htaccess` 設 `Require all denied`——**刻意擋掉所有直接網址存取**，因為裡面可能有員工個資（例如「員工名單.csv」），必須全部透過登入後的 `file.php` 才能讀到內容。**不要為了圖方便拿掉這條 `.htaccess`**，除非使用者明確接受「這個資料夾裡的東西會變成任何人有連結就能公開下載」這個後果。
- `list.php`（列出目錄樹用）跟 `file.php`（實際讀取單一檔案內容，登入後才給）各自寫死一份 `$ALLOWED` 副檔名白名單陣列，**兩邊必須同步改**，否則會出現「清單看得到但點了打不開」或反過來的狀況。目前（2026-07-28）白名單：`xlsx, xls, csv, ods, docx, pptx, pdf, html, htm`（html/htm 是這次新加的）。
- 前端 `app.js` 的 `openDocument()` 依副檔名分派到對應的 render 函式（`renderDocx`／`renderPptx`／`renderPdf`／新加的 `renderHtml`），沒對應到的副檔名會顯示「不支援的檔案類型：.xxx」。**新增一種副檔名支援，三個地方都要改：`list.php` 白名單、`file.php` 白名單（+ 視需要調整 MIME type）、`app.js` 的 `openDocument()` 分派邏輯。**
- **`index.php` 用 `<script src="app.js?v=13"></script>` 這種版本號手動做快取破壞（cache-busting）。改完 `app.js` 內容後一定要記得把這個 `v=` 數字往上加一，不然瀏覽器會繼續用快取的舊版 JS，等於改了跟沒改一樣**（2026-07-28 就因為忘了這步，使用者回報「html 檔案清單看得到但點了顯示不支援」，後來才發現是瀏覽器快取住了改之前的 `app.js`）。

## html 支援是怎麼加的（可當作之後加其他副檔名的範例）

使用者原本想直接把 `.html` 檔丟進 `dashboard\files\` 底下用瀏覽器開，卡在兩層：
1. `list.php` 白名單沒有 html → 檔案總管清單根本不會列出來（但檔案其實已經好好躺在磁碟上，不是沒傳成功）。
2. 就算繞過清單直接訪問 `files/xxx.html`，會被 `.htaccess` 的 `Require all denied` 擋掉。

一開始的權宜做法是另外開一個不受保護的 `C:\xampp\htdocs\open\` 資料夾放公開網頁，但使用者要的是「大家都放同一個地方、不要搞特殊」，並明確表示「html 也可以當成檔案處理，登入了才能開就好」，於是改成把 html 整合進同一套受保護系統：

1. `list.php`、`file.php` 的 `$ALLOWED` 都加入 `'html', 'htm'`。
2. `file.php` 針對 html/htm 回傳 `Content-Type: text/plain; charset=utf-8`（配合原本就有的 `X-Content-Type-Options: nosniff`）——**刻意不讓它以 `text/html` 直接被瀏覽器當網頁執行**。原因：`file.php` 跟 dashboard 主站同源，如果讓人直接用網址開一個內含惡意 `<script>`／`onerror=` 之類手法的上傳 html，會在使用者已登入的 session 同源環境下執行，等於幫攻擊者開了一個 XSS 後門。前端 `openDocument()` 本來就是靠檔名（不是看 HTTP response header）判斷副檔名，所以後端故意回傳 `text/plain` 完全不影響應用程式自己的預覽功能，只防住「繞過 UI 直接開連結」這條路。
3. `app.js` 新增 `renderHtml(buf)`：把抓回來的內容用 `new TextDecoder('utf-8').decode(buf)` 轉成字串，塞進一個 **sandboxed `<iframe srcdoc="...">`**（`sandbox="allow-scripts allow-popups"`，刻意不給 `allow-same-origin`）。沒有 `allow-same-origin` 的 srcdoc iframe 會被瀏覽器當成一個獨立的 opaque origin，裡面就算真的有惡意 script 也讀不到 `document.cookie`、connect 不回 dashboard 的同源 API——等於把使用者上傳的 html「隔離著顯示」而不是「信任著執行」。
4. **改完別忘了把 `index.php` 的 `app.js?v=` 版本號往上加**，否則瀏覽器不會抓到新版 `app.js`（見上一節）。

之後如果要再開放別的副檔名（例如 `.txt`、`.md`），照這個模式走：list.php + file.php 白名單同步加、想清楚要不要也做隔離渲染（信任的格式如 txt 可以不用 iframe 隔離，直接當純文字顯示即可）、記得升版本號。

## 相關 skill

`flutter-tray-app-deploy`（同一台機器上的資產管理 tray app，其 SSH 連線資訊筆記已過時，實際連線方式以本 skill 為準）、`ssh-passwordless-windows`（一般性的 Windows SSH 金鑰設定坑，這台是密碼登入的個案不適用）。

---

## Conformance Addendum

## When to Use
T25 (10.145.119.209) 上用 Bitnami XAMPP 跑的自建文件檢視 dashboard（Excel/Word/PPT/PDF/CSV 線上預覽 + html）。含：SSH 連線方式（port 2222、密碼登入、sshpass 在這台機器的 git bash 會失效要改用 plink）、遠端中文/emoji 顯示亂碼但資料沒壞的判斷方式、Apache 目前非服務、只監聽 88 埠、且是前景模式跑的已知隱患、dashboard 原始碼架構（list.php/file.php 白名單、files/ 目錄的 .htaccess 保護、app.js 依副檔名分派預覽、index.php 的 app.js?v= 快取版本號機制），以及如何安全地新增支援的檔案類型（已示範新增 html 支援並用 sandboxed iframe 隔離）。當使用者提到 T25、10.145.119.209、xampp、htdocs、或那個文件總管/dashboard 網頁打不開、看不到新放的檔案時使用。

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
