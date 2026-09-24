---
name: usb-storage-agent
description: USB 隨身碟控管系統——Rust Windows agent（D:\CODES\usb-storage-agent）+ aipower 主控台（C:\Aipower）。涵蓋編譯/部署/Windows Service 安裝、政策運作邏輯（登錄檔全域封鎖 Deny_Read/Write/Execute，非個別裝置 CM_Disable_DevNode）、machineCode 自動註冊待審核流程、aipower 端 LocalApi Bearer Token 整合、Jocket WebSocket 即時政策 push、審核狀態下拉選單、主控台 UI（單一選單+對話框+主從頁籤）、requireAdministrator manifest 自動提權、console 模式背景執行+系統匣圖示（tray icon）、Inno Setup 打包安裝程式（installer.iss）、遠端受控電腦部署（AIPOWER_BASE_URL 內網位址設定）、以及一系列實測踩過的坑（message-only window 收不到通知、CM_Disable_DevNode 被系統否決、Stop-Process 殺不掉提升權限行程、usbagent_unit.sql 重跑會清空真實資料、Jocket push 無窮迴圈）。當使用者要求「usb agent 重新編譯」「usb 控管服務裝/解除安裝」「usb agent 連不上 aipower」「隨身碟被鎖住/存取被拒」「usb 服務帳號」「即時通知/Jocket push」「usb agent 打包/安裝程式」「usb agent tray/系統匣圖示」「usb agent 遠端部署/裝到別台電腦」或提到這個專案時使用。
---

# USB Storage Agent（USB 隨身碟控管系統）

企業 DLP 系統：Rust agent 跑在受控端電腦偵測 USB 隨身碟插拔、依政策全域封鎖/放行、稽核檔案存取；aipower（`C:\Aipower`）當政策中心 + 事件稽核中心 + 主控台。完整設計決策記錄在 `C:\Users\HCH\.claude\plans\gleaming-squishing-donut.md`（原始 grill-me 拷問記錄 + 每個 task 的實測過程），這份 skill 是給日常操作/除錯用的濃縮版。

## 系統組成

| 部分 | 位置 | 角色 |
|---|---|---|
| Rust agent | `D:\CODES\usb-storage-agent` | 跑在受控端電腦，偵測裝置/封鎖/稽核/回報 |
| aipower 主控台 | `C:\Aipower`（port 22821/22822） | 政策中心 + 待審核機器 + 事件稽核 UI |
| 服務帳號 | `usb` / `<USB_ACCOUNT_PASSWORD>` | agent 唯一准入門檻，**不要用 administrator**。⚠ 計畫書/早期版本寫的是 `usbagent-svc`，但目前 `C:\Aipower` 資料庫裡實際的帳號是 `usb`（`config.rs` 已同步改過）——之後若又對不上，先用 `SELECT FLoginName FROM TsAccount` 查真正的帳號名稱，不要照抄舊文件 |

## 架構要點（跟直覺不一樣、務必記住的部分）

1. **封鎖機制是全域登錄檔政策，不是個別裝置停用**。`HKLM\SOFTWARE\Policies\Microsoft\Windows\RemovableStorageDevices\{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}` 設 `Deny_Read/Write/Execute=1`——整個「卸除式磁碟」類別一起擋，沒有白名單自動放行個別裝置的能力。原計畫的 `CM_Disable_DevNode`（個別裝置強制停用）**已放棄**：實測一旦裝置掛載，這個 API 幾乎必定被 Windows PnP 子系統否決（`CONFIGRET=23`），連剛插入的瞬間也一樣，不是使用中才會發生。
2. **政策不會追溯套用到已連接裝置**——只影響「政策變更之後才插入」的裝置，需要重新插拔或重開機。
3. **machineCode 沒有 AD 可用**，靠 `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid` + `%COMPUTERNAME%` 組成，agent 首次啟動自動呼叫 `usbagent/api?op=register` 建立 `PENDING` 記錄，IT 要在 aipower 後台把 `FApprovalStatus` 改成 `APPROVED` 才會生效。**未核准 = fail-closed = 封鎖**，包括「從未同步過政策」的情況（連不上 aipower 時也一樣封鎖，不會因為離線就放行）。
4. **aipower 對接方式是 LocalApi Bearer Token，不是 API 2.0**——`ecp-java-core` skill 描述的 `qs/user/token/apply` 那套在這個專案沒用到，走的是 `TsLocalApi` 自訂 Handler（`POST /aipower/openapi/auth/token?op=apply` 換 token，之後帶 `Authorization: Bearer <tokenId>`）。這條路由框架本身不驗證 Authorization，每個 Handler 要自己解析 token。

## Rust 專案結構（`D:\CODES\usb-storage-agent\src`）

| 檔案 | 職責 |
|---|---|
| `main.rs` | 純進入點：解析 `--install`/`--uninstall`，否則先試 Service 模式失敗就退回 console 模式 |
| `agent.rs` | 主邏輯（console/Service 共用）：訊息迴圈、政策同步、封鎖開關套用 |
| `service.rs` | Windows Service 安裝/解除安裝/control handler |
| `device.rs` | `WM_DEVICECHANGE` 監聽（只註冊 `GUID_DEVINTERFACE_DISK`） |
| `device_id.rs` | 從裝置介面路徑解析 Ven_/Prod_/序號，**強制要求 enumerator=USBSTOR** 否則拒絕（防止誤判系統磁碟） |
| `enforce.rs` | 列舉目前已連接磁碟裝置（純稽核記錄用途） |
| `registry.rs` | 全域封鎖政策讀寫 + MachineGuid 讀取 |
| `audit.rs` | `ReadDirectoryChangesW` 檔案稽核，輪詢卸除式磁碟機代號 |
| `policy.rs` | 政策資料結構 + 本機快取檔讀寫 + 白名單比對（目前僅供稽核記錄） |
| `machine.rs` | machineCode/hostname 組裝 |
| `api.rs` | 呼叫 aipower LocalApi（`ureq`） |
| `events.rs` | 事件佇列（送不出去會放回佇列重試，不會丟事件） |
| `socket.rs` | Jocket WebSocket client：即時接收政策變更 push，斷線自動重連（見下方「Jocket 即時政策 push」） |
| `config.rs` | aipower URL（`AIPOWER_BASE_URLS: &[&str]` 陣列，**2026-08-04 改回只含一筆 `http://192.168.100.145:22821/aipower`**，見下方「多台 aipower / 單一伺服器」）+ 服務帳號帳密（**目前寫死常數，未來要外部化成 DPAPI 加密設定檔**） |
| `paths.rs` | 固定資料目錄 `C:\ProgramData\UsbStorageAgent\`（Service 模式工作目錄是 System32，不能用相對路徑） |
| `log.rs` | 統一輸出到 stdout + `agent.log`（Service 模式沒有 stdout 可看，這是唯一追蹤管道） |
| `tray.rs` | 系統匣圖示（僅 console/手動執行模式，見下方「背景執行 + 系統匣圖示」） |
| `build.rs` | 編譯時內嵌 `requireAdministrator` manifest（見下方「自動提權」） |

## 編譯

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
cd D:\CODES\usb-storage-agent
cargo build          # debug，開發用
cargo build --release
cargo test           # device_id 解析器單元測試
```

前置：Rust toolchain（`rustup`）+ Visual Studio Build Tools（C++ 工作負載，提供 `link.exe`，MSVC target 必需）都已裝在這台機器上，正常情況不需要重裝。

**⚠ `cargo build --release` 若報 `failed to remove file ...usb-storage-agent.exe`（`存取被拒`/os error 5）**：通常是先前測試留下的 exe 還在跑，把檔案鎖住了。因為現在 exe 內嵌了 `requireAdministrator` manifest（見下方「自動提權」），這個殘留行程幾乎一定是提升權限啟動的，非提升權限的 `taskkill`/`Stop-Process` 殺不掉（同下方「終止提升權限行程的正確方式」），要先跑：
```powershell
Start-Process powershell -Verb RunAs -Wait -ArgumentList "-NoProfile -Command taskkill /F /IM usb-storage-agent.exe"
```
確認 `Get-CimInstance Win32_Process -Filter "Name='usb-storage-agent.exe'"` 沒有殘留後再重新 `cargo build --release`。

## 自動提權（2026-08-01）

`build.rs`（`embed-manifest` crate）在編譯時內嵌 `requestedExecutionLevel=requireAdministrator`。之後**任何方式**執行 exe（雙擊、`--install`/`--uninstall`、console 模式）Windows 都會自動跳 UAC，不用再手動 `-Verb RunAs`。Service 模式（被 SCM 以 LocalSystem 啟動）不受影響、不會被 manifest 擋。驗證方式：`mt.exe -inputresource:usb-storage-agent.exe\;#1 -out:x.manifest` 抽出來看有沒有 `requireAdministrator`（`mt.exe` 在 Windows SDK `bin\<ver>\x64\` 底下）。

## Windows Service 操作

**安裝**（以 LocalSystem 身分執行，開機自動啟動，解決開發期每次都要跳 UAC 的問題）：
```powershell
Start-Process -FilePath "D:\CODES\usb-storage-agent\target\debug\usb-storage-agent.exe" -ArgumentList "--install" -Verb RunAs -Wait
```
（manifest 已要求提權，`-Verb RunAs` 其實已經是雙重保險，不加也會自動跳 UAC。）

**啟動/停止/查狀態**（`net start`/`net stop` 也可以，這裡用 PowerShell）：
```powershell
Start-Service -Name UsbStorageAgent    # 或 Stop-Service
Get-Service -Name UsbStorageAgent
```

**解除安裝**：
```powershell
Start-Process -FilePath "D:\CODES\usb-storage-agent\target\debug\usb-storage-agent.exe" -ArgumentList "--uninstall" -Verb RunAs -Wait
```

**Console 模式**（開發期除錯用，不裝服務直接跑，一樣需要系統管理員權限才能寫封鎖政策；雙擊 exe 即可，會自動跳 UAC）：
```powershell
Start-Process -FilePath "D:\CODES\usb-storage-agent\target\debug\usb-storage-agent.exe"
```

## 背景執行 + 系統匣圖示（2026-08-01）

`main.rs` 加了 `#![windows_subsystem = "windows"]`——雙擊/手動執行不再跳出 console 黑窗，log 只能看 `agent.log`（`Get-Content -Wait` 追蹤）不再有 stdout 可看。

Console/手動執行模式（`agent::run(true)`）額外會呼叫 `tray.rs::add()` 加一顆系統匣圖示（圖示 PNG 用 `include_bytes!` 在編譯時內嵌自 `D:\APP\icons8-asset-100.png`，不是執行期讀檔——這樣部署到其他受控端電腦時不需要一起帶著這個 PNG 路徑）。右鍵圖示有「結束」選單可以正常關閉。

**⚠ 這顆 tray icon 只在 console/手動執行模式生效**——`service.rs` 呼叫 `agent::run(false)`，刻意不開。原因是 Windows Service 跑在 **Session 0**，跟使用者的桌面 session 是隔離的，`Shell_NotifyIconW` 在服務行程裡呼叫不會顯示任何東西，這是 Windows 自 Vista 以來的架構限制，不是能繞過的 bug。正式部署（Windows Service）沒有 tray，只有開發期在自己電腦上手動跑才看得到。

## ⚠ 終止提升權限行程的正確方式（踩過的坑）

用非提升權限的 `Get-Process X | Stop-Process -Force -ErrorAction SilentlyContinue` **無法終止**用 `-Verb RunAs` 啟動的行程——一般權限行程沒有權限終止提升權限行程，而 `SilentlyContinue` 會把這個失敗吞掉，讓人誤以為已經關閉。實測曾因此讓一個測試用 agent 在背景多跑了 15 分鐘，反覆切換全域封鎖政策，使用者的隨身碟被鎖住卻沒人發現。

**正確做法**：
```powershell
taskkill /F /IM usb-storage-agent.exe
# 或明確提升 Stop-Process 本身
Start-Process powershell -Verb RunAs -Wait -ArgumentList "-NoProfile -Command Stop-Process -Name usb-storage-agent -Force"
```
關閉後務必用 `Get-CimInstance Win32_Process -Filter "Name='usb-storage-agent.exe'"` 確認真的沒有殘留，**不要只看指令有沒有報錯**。

## 排查：隨身碟突然被鎖住/存取被拒

1. 檢查是否有殘留行程（用上面的 `Get-CimInstance` 指令，不要只用 `Get-Process`）。
2. 檢查全域封鎖政策是否還在：
   ```powershell
   Test-Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\RemovableStorageDevices\{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}'
   ```
3. 手動解除（需要 UAC）：
   ```powershell
   Start-Process powershell -Verb RunAs -Wait -ArgumentList "-NoProfile -Command Remove-Item -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\RemovableStorageDevices\{53f5630d-b6bf-11d0-94f2-00a0c91efb8b}' -Recurse -Force"
   ```
4. **政策不會追溯套用到已連接裝置**——如果解除政策後隨身碟還是打不開，通常是要拔插一次讓它重新掛載。

## 排查：aipower 端連不上

`C:\Aipower` 的 Tomcat（`java.exe`）沒在跑，只剩孤兒的內嵌 MariaDB（`mariadbd.exe`）活著，是這個環境已知會發生的狀況（見 `ecp-server-startup` skill 的 mariadbd「重啟必炸」問題背景）。判斷方式：
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:22821/aipower/   # 200=正常，連線失敗=Tomcat沒起來
powershell -Command "Get-CimInstance Win32_Process -Filter \"Name='mariadbd.exe'\""  # DB 通常還活著
```
Agent 端連不上時**依設計走 fail-closed（封鎖），不是放行**——這不是 bug，是計畫要的行為。要恢復連線需要重啟 `C:\Aipower\server.bat`（見 `ecp-server-startup` skill）。

## 密碼/帳號注意事項

- **絕對不要碰 `administrator` 帳號密碼**，即使只是為了清鎖定計數器。有專用 `ecp-pwd` skill 但那是給真的需要重設密碼的情境用，不要因為測試方便就順手用它動 administrator。
- 需要驗證 aipower API 時一律用 `usb`（密碼 `<USB_ACCOUNT_PASSWORD>`），不要借用 administrator。
- `usb` 密碼同樣可能因錯誤嘗試被鎖，鎖了才用 `ecp-pwd` skill 重設（這個帳號被重設沒有上面那條限制，只有 administrator 不行動）；`ecp-pwd` 的 `reset_password.py` 在這個環境（`C:\Aipower`，非 `C:\com\chainsea`）跑要另外帶 `--port`/`--client`（見下方腳本用法，自動偵測邏輯是照 Lab2 環境寫的，找不到這裡的 mariadbd）。

## aipower 端（Java）

六大核心模組原始碼在 `C:\Aipower\tool\src\com\chainsea\ecp\usbagent\`，3 個 Unit：`Ecp.UsbAgentMachine`/`Ecp.UsbAgentDevice`/`Ecp.UsbAgentEvent`。DB 註冊腳本 `C:\Aipower\tool\src\usbagent_unit.sql`（冪等，可重複執行）。改完 class 要重新編譯（`javac` + `C:\Aipower\jdk`）、複製到 `WEB-INF\classes`、重啟 `server.bat`——完整流程見 `ecp-java-core` skill 與計畫檔 Part A 段落。

核准/拒絕待審核機器：直接用 aipower 後台「受控機器」清單，開啟一筆記錄改 `FApprovalStatus` 存檔即可，是標準 EntityForm，沒有另外寫自訂按鈕。**`FApprovalStatus` 是下拉選單**（`TsField.FType='ComboBox-SelectOnly'` + `FDictionaryId` 指到一組 `TsDictionary`/`TsDictionaryItem`：`FValue`=PENDING/APPROVED/REJECTED 對應儲存值，`FText`=待審核/已核准/已拒絕 對應顯示文字），不是自由輸入文字——照抄本機資料庫裡其他欄位既有的同一套 `ComboBox-SelectOnly` + `FDictionaryId` 慣例即可，不需要另外寫程式碼。

### ⚠ `usbagent_unit.sql` 是「冪等」不等於「安全重跑」——會清空真實資料

腳本開頭的「1. 實體表」段落對三張真正存資料的表（`TcUsbAgentMachine`/`TcUsbAgentDevice`/`TcUsbAgentEvent`）寫的是 `DROP TABLE IF EXISTS` + `CREATE TABLE`——**每次重跑整份腳本都會把裡面的真實資料整個砍掉重建**，不是只重建 `TsField`/`TsMenu` 這些 metadata。曾經只是想改一個欄位的下拉選單設定，就重跑了整份腳本，結果把當時已註冊的機器、白名單、稽核事件全部清空。

**只改 metadata（欄位型態、選單、頁面版面）時，寫針對性的 `UPDATE`/`INSERT`，不要重跑整份 `usbagent_unit.sql`。** 只有真的要「砍掉重練」整個模組（例如改了 `TcXxx` 的欄位結構本身）才需要重跑整份腳本，且重跑前務必先確認使用者知情、真實資料可以接受被清空。

### 主控台 UI：單一選單 + 對話框 + 主從頁籤（2026-07-31 UI 整併）

原本「受控機器」「裝置白名單」「稽核事件」是三個各自獨立的左側選單項目，使用者要求「盡可能在一個或兩個表單來控制就好了，有要設定的部分用 dialog 就好了」，整併成：
- 左側選單只留**一個**「USB控管」入口（指到受控機器列表 `@M_PLIST`）。
- 「裝置白名單」改成受控機器列表工具列上的按鈕，用 `Utility.openDialog("Ecp.UsbAgentDevice.List.page", null, {...})` 開對話框（自訂 JS 檔 `ecp/page/usbagentmachine/UsbAgentMachineList.js`，docroot 路徑慣例是「Unit 對應的頁面資料夾」）。
- 「稽核事件」改成**主從頁籤**內嵌在受控機器表單裡（`TsPage.FMasterUnitId`+`FIsSlavePage=b'1'`+`FRelationId`），比照 `Ecp.Task.Form` 底下 `Ecp.Task.ActivityList` 的頁籤機制——**`TsRelation` 必須成對插入**（兩個方向互相指對方的 `FOppositeId`），只插一個方向會讓框架啟動時的 `RelationRoutine.complementRelations()` 自動補完邏輯壞掉、整個 aipower 開機直接炸掉（已實測踩過，修法是啟動獨立 embedded DB 手動補另一個方向）。

### Jocket 即時政策 push（2026-07-31，取代單純輪詢，端到端驗證通過）

**動機**：原本政策變更靠 agent 每 60 秒輪詢 aipower，IT 核准/拒絕後最慢要等 60 秒才生效。改用 aipower 內建的 `com.jeedsoft.jocket`（WebSocket 抽象層，框架自己也在用，`OnlineUserServiceImpl` 踢除 session 時就是靠它推播）做即時 push，60 秒輪詢**仍保留**當安全網（push 連線斷線期間的保底，不因為改用 push 就整個拿掉輪詢——DLP 這種安全關鍵的東西不該只依賴單一機制）。

**協定**（`jeedsoft-jocket-2.2.1.jar`，反編譯確認，跟 `aipower-docker-local` skill 記錄的協定完全一致）：
1. `POST /aipower/jocket/create`，參數**必須放在 URL query string**，不能放 POST body（body 會讀不到，`getConfig()` 找不到 config，回「Jocket configuration not found.」——這個坑真的踩過，curl 手動測試時最容易犯）：`jocket_path=/usbagent-control&tokenId=<token>&machineCode=<code>`，回應 `{"sessionId":...,"pingInterval":25000,...}`。
2. **立刻背景送一個 `POST jocket/poll?s=<sessionId>`**（會被伺服器掛起，不等它完成）——這是純 WebSocket 打不通的關鍵：`Jocket.send()` 送訊息前，連線必須先在伺服器端的 `connections` map 裡，但 WebSocket 剛連上只會進 `probingConnections`；唯一能把它「升級」進 `connections` 的路徑藏在處理這個 long-polling 回應的地方。給它 ~400ms 讓請求先掛上去，再開 WebSocket。
3. 開 `ws://.../jocket/ws?s=<sessionId>`，連上後**立刻送** `{"type":"upgrade"}`。
4. Client 每 `pingInterval` 毫秒主動送 `{"type":"ping"}`（server 被動回 pong，不送心跳連線會被視為死掉）。
5. Server 端呼叫 `Jocket.send(sessionId, "policy_changed", {})` 推播；client 收到 `{"type":"message","name":"policy_changed",...}` 就立刻重新同步政策。

**Rust 端**：`src/socket.rs`（新增 `tungstenite` 依賴，同步阻塞式，跟既有執行緒模型一致，不拉 tokio），斷線就整段重連（含重新握手），10 秒後重試。

**Java 端部署（`@JocketServerEndpoint` 註解本身不會被自動掃描）**：
- `integration/UsbAgentControlEndpoint.java`：`onOpen()` 驗證 `tokenId`（跟 `LocalApiAuth` 同一套 `OnlineUserHome.getService().getItem(tokenId)`），把 `machineCode -> sessionId` 存進行程內 `ConcurrentHashMap`（純記憶體，無落地，webapp 重啟全部重來，這是刻意的最小可行版本）。
- `integration/UsbAgentStartupListener.java`（`@WebListener`）：`contextInitialized()` 呼叫 `JocketDeployer.deploy(UsbAgentControlEndpoint.class)`——**這一步是必要的**，光有 `@JocketServerEndpoint` 註解不會自動掛上去，一定要有個 `ServletContextListener` 明確呼叫 `deploy()`。
- **驗證部署是否成功的方法**：看 `C:\Aipower\apache-tomcat\extension\aipower\log\info.log`（不是 `catalina.log`，Quicksilver 框架的 slf4j log 走自己的 log4j2 設定，輸出到這個獨立檔案）裡的 `[Jocket] Service started. tree structure:` 那段多行輸出，成功會多一行 `|---usbagent-control (com.chainsea.ecp.usbagent.integration.UsbAgentControlEndpoint)`。**這個 log 要看框架自己的 `JocketHome.initialize()` 印出來的最終版本**（在 `ApplicationListener.contextInitialized()` 裡，會把我們的 `deploy()` 結果跟框架自己的 `inner`/`mobile` 端點合併印出），不要只看我們自己 `UsbAgentStartupListener` 那行 log（那行只代表我們的 `deploy()` 呼叫本身沒丟例外，不代表框架後續有沒有正確合併進最終的樹）。

**⚠ 踩過的坑：push 觸發無窮迴圈**——`UsbAgentMachineServiceImpl.doUpdate()` 一開始無條件呼叫 `pushPolicyChanged()`，但 agent 自己的 `usbagent/api?op=register`（每次同步都會呼叫，用來刷新 `FHostname`/`FLastSeenTime`）**也會呼叫 `update()`**，變成「IT 存檔 → push → agent 收到立刻重新同步 → 同步流程呼叫 register → update() → 又 push → ...」的無窮迴圈（實測 `agent.log` 瞬間狂印數百行）。**修法**：`doUpdate()` 裡先 `getItem()` 讀出更新前的 `FApprovalStatus`，跟更新後的值比對，**只有真的變了才 push**，不是每次 `update()` 都 push。任何要在 `doUpdate`/`doCreate` 裡加「變更後通知外部系統」邏輯的模組，都要記得這個「自己觸發自己」的迴圈風險。

## 打包成安裝程式（Inno Setup，2026-08-01）

`installer\installer.iss`（Inno Setup 6，`winget install --id JRSoftware.InnoSetup` 裝的，裝在 `C:\Users\HCH\AppData\Local\Programs\Inno Setup 6\ISCC.exe`，不是 Program Files）。

```powershell
cd D:\CODES\usb-storage-agent
cargo build --release
& "C:\Users\HCH\AppData\Local\Programs\Inno Setup 6\ISCC.exe" installer\installer.iss
# 產出 installer\output\UsbStorageAgentSetup.exe
```

安裝程式邏輯：`PrivilegesRequired=admin`（整個安裝程式本身就是提升權限跑的，裡面再呼叫一次 exe 的 `--install` 不會跳第二次 UAC，子行程繼承同一個已提升權杖）→ 複製 exe 到 `{autopf}\UsbStorageAgent` → 跑 `--install`（註冊服務）→ `net start UsbStorageAgent`。解除安裝時跑 `--uninstall`（`service.rs::uninstall()` 本身就會先停止再標記移除）。

圖示已經在編譯期 `include_bytes!` 內嵌進 exe，安裝程式**不需要**額外帶 `D:\APP\icons8-asset-100.png` 這個檔案一起發布。

## 遠端部署（2026-08-01）

這套系統本來就是設計給多台受控端電腦裝的：agent 裝上去 → 用 machineCode 自動向 aipower 註冊（`PENDING`）→ IT 在 aipower 後台「USB控管」清單核准 → 政策生效（Jocket push 即時、斷線退回 60 秒輪詢）→ 裝置插拔/檔案存取事件回報到 aipower 集中檢視。

**⚠ `config.rs` 的 `AIPOWER_BASE_URL` 原本寫死 `http://127.0.0.1:22821/aipower`**（只有 agent 跟 aipower 同一台機器才連得到），已改成內網位址 `http://10.145.119.100:22821/aipower`——這是編譯期常數，燒進 exe 裡，**遠端電腦跟這個位址之間要能連得到**（同內網或 VPN；原始設計是「僅內網、不掛 Cloudflare Tunnel」，不是打算公網存取）。

要部署到新的受控端電腦：`cargo build --release` → `ISCC.exe installer\installer.iss` 產出 `UsbStorageAgentSetup.exe` → 拿去目標電腦跑（見上方「打包成安裝程式」）。**如果之後又要換一個 aipower 位址，記得改完 `config.rs` 要重新 build + 重新跑一次 ISCC 產生新的安裝程式**，不是改設定檔就好——目前沒有外部化設定檔這回事。

## 遠端部署到新受控端電腦（plink/pscp 免互動連線，2026-08-04）

這個環境（MSYS/Git Bash）跑一般 `ssh`/`pscp`（OpenSSH 版）密碼登入會失敗——`read_passphrase: can't open /dev/tty`，因為沒有真正的 tty 可以互動輸入密碼，`sshpass` 也一樣過不了（`Permission denied` 但密碼其實是對的）。**改用 PuTTY 的 `plink`/`pscp`**（`C:\Program Files (x86)\PuTTY\`），這兩支工具吃 `-pw` 參數不需要 tty：

```bash
# 第一次連線要先信任 host key（互動模式會問，記下 fingerprint 後改用 -hostkey 走非互動）
plink -pw '<password>' -hostkey "SHA256:<fingerprint>" Administrator@<ip> "<command>"

# 上傳檔案（同樣要帶 -hostkey，否則第一次會卡在 host key 確認掉進 batch 模式直接失敗）
pscp -pw '<password>' -hostkey "SHA256:<fingerprint>" "本機路徑(Windows格式 D:\...)" Administrator@<ip>:"C:\Temp\目標檔名"
```

**踩過的坑**：
- 純 `ssh`/`sshpass` 在這個環境一律失敗，不要浪費時間重試，直接换 plink/pscp。
- `pscp` 第一次沒帶 `-hostkey` 會因為要互動確認主機金鑰而在 batch 模式直接 `Command aborted`，之後傳輸會卡住直到 timeout（曾經卡滿 60 秒）——**一律先用 `plink` 連一次拿到 fingerprint，之後 `pscp` 也要帶 `-hostkey`**，不要假設信任快取會生效。
- `plink -batch` 模式對未知 host key 會直接報錯退出，不會像互動模式那樣問要不要信任，適合腳本化重複呼叫。
- 遠端執行 PowerShell 指令時，中文輸出（`Get-ItemProperty` 的日期、`DisplayName` 等）會亂碼（plink 用的編碼跟 PowerShell 主機的 code page 對不上），是顯示問題不影響指令實際執行結果，數字/英文欄位（`fDenyTSConnections=0`、`Enabled=True`）還是看得懂，不用特別去修編碼。

### 啟用遠端桌面（RDP）

新裝的機器 RDP 預設關閉，兩件事都要做（缺一不可）：

```powershell
# 1. 允許 RDP 連線（登錄檔）
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name 'fDenyTSConnections' -Value 0

# 2. 開防火牆規則——⚠ 中文版 Windows 的 DisplayGroup 不是英文 "Remote Desktop"，
#    用 Get-NetFirewallRule 篩 DisplayName 找不到會直接報錯（ObjectNotFound）。
#    改用規則的 Name（英文，語言無關）：
Enable-NetFirewallRule -Name RemoteDesktop-UserMode-In-TCP,RemoteDesktop-UserMode-In-UDP
```

先用 `Get-NetFirewallRule | Where-Object {$_.Name -like '*RemoteDesktop*'} | Select Name,DisplayName,Enabled` 確認規則實際的 `Name`（不要猜 `DisplayGroup`），不同語言版本的 Windows 顯示名稱會不一樣，但 `Name` 欄位是穩定的英文識別碼。

### 部署到新機器的完整流程（重現版本升級 0.1.0→0.1.1 的實際步驟）

1. **改版本號**（兩個地方都要改，`Cargo.toml` 也要同步改，否則已安裝程式清單顯示的版本跟原始碼實際內容對不上）：
   - `Cargo.toml`：`version = "0.1.1"`
   - `installer\installer.iss`：`#define MyAppVersion "0.1.1"`
2. `cargo build --release`
3. `& "C:\Users\HCH\AppData\Local\Programs\Inno Setup 6\ISCC.exe" installer\installer.iss`
4. `pscp` 把 `installer\output\UsbStorageAgentSetup.exe` 傳到目標機器 `C:\Temp\`（目錄要先用 `plink` 跑 `New-Item -ItemType Directory -Path C:\Temp -Force` 確保存在）
5. `plink` 遠端靜默安裝：`C:\Temp\UsbStorageAgentSetup.exe /VERYSILENT /SUPPRESSMSGBOXES /NORESTART`（Inno Setup 標準靜默參數，不會跳 UI，`[Run]` 段落的 `--install`/`net start` 照樣會執行）
6. 驗證（三個都要查，任一個沒對上就表示安裝過程有問題）：
   ```powershell
   Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Where-Object {$_.DisplayName -like '*Usb*Storage*'} | Select DisplayName,DisplayVersion
   Get-Process usb-storage-agent -ErrorAction SilentlyContinue | Select Id,StartTime   # PID/啟動時間要是新的
   Get-Service UsbStorageAgent | Select Status   # 要是 Running
   ```

## 相關

- 完整決策記錄與逐項實測過程：`C:\Users\HCH\.claude\plans\gleaming-squishing-donut.md`
- `ecp-java-core`：aipower 六大核心開發規範
- `ecp-server-startup`：`C:\Aipower`/`server.bat` 啟動、mariadbd 重啟必炸的 patch
- `ecp-pwd`：帳號密碼重設（**administrator 不要用**，見上）
- `D:\USB-Storage-Block`：舊版陽春 PowerShell 封鎖腳本，`registry.rs` 的登錄檔機碼直接沿用它已驗證過的路徑/數值

## 多台 aipower 架構與服務意外 crash 導致 fail-closed（2026-08-04）

`AIPOWER_BASE_URLS` 是陣列，支援同時對接多台 aipower（每台獨立進行「換 token → 註冊 → 拉政策 → 回報事件 → Jocket」），但只要**任一台**回報未核准，全域封鎖開關就套用封鎖（**AND 邏輯**，最嚴格，見 `agent.rs::combine_policies()`）——這是計畫書明訂的刻意設計，不算 bug。

**實測發現問題**：曾同時接 `10.145.119.100` + `192.168.100.145` 兩台，實際運作中兩台核准狀態常常對不齊（其中一台暫時連不上、或審核狀態沒同步），AND 邏輯下造成「明明已經核准卻還是被鎖」的混亂情況，非常難排查。**2026-08-04 使用者決定改回只對接單一台**（`192.168.100.145`），簡化成單一資料來源，避免混亂。`agent.rs` 仍保留「陣列」設計以支援未來若真的需要多台，只是目前陣列長度固定為 1。改回單一伺服器後記得清理殘留的 `policy_cache_1.json`（對應已被移除那台伺服器的快取）。

**另一個實測案例**：服務意外終止（Windows Event ID 7034），重啟時撞上其中一台伺服器連線失敗的空窗期，agent fail-closed 退回本機舊快取（`policy_cache.json` 記錄舊的 REJECTED），AND 邏輯下整體封鎖——從 aipower 後台看起來明明已核准，實際卻還是被鎖，很容易誤判為政策沒生效。**真正根因**：程式本身有 panic 風險點（未裝 panic hook）+ Mutex 中毒後無法自癒（一旦某個執行緒 panic 時持有 lock，其他執行緒 `.lock().unwrap()` 會連鎖 panic，整個服務崩潰後自動重啟，重啟期間剛好撞上伺服器連線失敗）。

**修復**（不引入 `parking_lot`，用標準庫最小改動）：`main.rs` 新增 `install_panic_hook()`，panic 訊息寫入 `agent.log`；`agent.rs`/`events.rs`/`log.rs` 各自新增 `lock_xxx()` 包裝函式，用 `unwrap_or_else(|poisoned| poisoned.into_inner())` 處理 Mutex 中毒，不讓一個執行緒 panic 拖垮其他執行緒。**不要去改「fail-closed」本身的邏輯**，只修「服務為何會 crash」這個真正問題。

## 重新打包/重裝完整流程

```powershell
# 1. 移除舊服務（需提升權限）
Start-Process cmd -ArgumentList '/c "C:\Program Files\UsbStorageAgent\usb-storage-agent.exe" --uninstall' -Verb RunAs -Wait

# 2. 重新編譯 + 打包
cd D:\CODES\usb-storage-agent
cargo build --release
& "C:\Users\HCH\AppData\Local\Programs\Inno Setup 6\ISCC.exe" installer\installer.iss

# 3. 執行安裝程式（靜默安裝）
Start-Process -FilePath "D:\CODES\usb-storage-agent\installer\output\UsbStorageAgentSetup.exe" -ArgumentList '/VERYSILENT','/SUPPRESSMSGBOXES','/NORESTART' -Verb RunAs -Wait
```

安裝後用 `sc qc UsbStorageAgent` 確認 `START_TYPE=AUTO_START`、`SERVICE_START_NAME=LocalSystem`，並比對 `Program Files` 裡的 exe 與 `target\release` 裡的 exe 檔案大小/時間是否一致，確保真的是最新版本。

## ⚠ 軟體模擬拔插（`Disable-PnpDevice`/`Enable-PnpDevice`）實測證實不可靠（2026-08-04）

為了避免每次測試都要真的拔插 USB，曾嘗試用 `Disable-PnpDevice -InstanceId ...` / `Enable-PnpDevice -InstanceId ...` 來「軟體模擬拔插」，**經過多輪系統性測試（包含 USB 底層節點 `USB\VID_xxx&PID_xxx\...` 與 Disk class 節點 `USBSTOR\DISK&VEN_...` 兩種，且連續執行 4 次以上）完全無法讓封鎖/解鎖政策真正生效**。

**驗證方法**：用 Windows 事件記錄比對——每次真正重新掛載都會產生一筆 NTFS 磁碟區健康檢查事件（`wevtutil qe System /q:"*[System[(EventID=98)]]" /c:1 /rd:true /f:text`），對照軟體模擬拔插前後的事件時間戳，發現即使執行了好幾輪 `Disable-PnpDevice`/`Enable-PnpDevice`，最新事件時間從頭到尾**完全沒更新**——代表 Windows 從頭到尾都沒把這當成「真正的裝置移除又插入」，僅改變了 PnP 樹上的邏輯啟用狀態，沒有真正切斷 USB 供電/通訊，儲存過濾驅動根本不會重新讀取政策。與原開發者當初放棄 `CM_Disable_DevNode` 的結論完全吻合（見上方「架構要點」第 1 點）。

**結論**：**不要用軟體模擬拔插來測試封鎖/解鎖是否生效**，唯一可靠的方法是使用者實體拔插（或重新安裝/重啟服務，實測發現有機會間接觸發真正的重新枚舉）。若日後真的需要自動化測試，要先找到能真正觸發 USB 供電中斷的方法（例如帶 hub 電控開關的實體裝置），軟體層級的 PnP API 已確認不行。

## ⚠ 受控機器列表「刪除」失敗：不能刪除 USB 控管，因為存在相關聯的稽核事件（2026-08-04）

aipower 後台「受控機器」列表選定一台機器按「刪除」會彈「不能刪除USB控管，因為存在相關聯的USB控管-稽核事件」錯誤。這是 Quicksilver 框架預設行為：`TcUsbAgentMachine`（受控機器）跟 `TcUsbAgentEvent`（稽核事件，主從頁籤關係）之間有外鍵關聯，只要這台機器產生過任何插拔/存取稽核事件（幾乎不可能沒有，只要裝過 agent 就會產生），框架就拒絕直接刪除。

**目前尚未實作「級聯刪除」邏輯**（順道同時刪掉關聯的稽核事件/白名單裝置），下次需要支援使用者從後台直接刪除受控機器時，要在 `UsbAgentMachineServiceImpl`（`D:\Aipower\tool\src\com\chainsea\ecp\usbagent\machine\service\impl\UsbAgentMachineServiceImpl.java`，目前只改了 `doUpdate()` 加 Jocket push）重寫 `doDelete()`：
1. override `doDelete(ServiceContext ctx, UUID id)`
2. 先用 `UsbAgentEventService`/`UsbAgentDeviceService`（同層級兩個 sibling package，見 rust 專案結構表或 `com.chainsea.ecp.usbagent.event`/`device` package）查出所有 `FMachineId=id` 的稽核事件/白名單裝置，逐筆（或批量）刪除
3. 再呼叫 `super.doDelete(ctx, id)` 刪除機器主記錄
4. 改完 class 要重新編譯（`javac` + `D:\Aipower\jdk`）、複製到 `WEB-INF\classes`、重啟 `server.bat`（完整流程見 `ecp-java-core` skill）

**暫時繞過方法**（不改程式碼、需要即時刪除時）：直接在 aipower 後台把那台機器的 `FApprovalStatus` 改成「已拒絕」就能達到「封鎖這台機器」的實質效果，不一定需要真的從資料庫刪除那筆記錄。若非得從資料庫層級移除（例如要清空隨身碟SN重用等情境），可以直接在 MariaDB 手動執行：
```sql
DELETE FROM TcUsbAgentEvent WHERE FMachineId = '<要刪的機器 id>';
DELETE FROM TcUsbAgentDevice WHERE FMachineId = '<要刪的機器 id>';
DELETE FROM TcUsbAgentMachine WHERE FId = '<要刪的機器 id>';
```
此法繞過了框架層的商業邏輯驗證，**只在確定要從資料庫徹底清除、且使用者知情同意的情況下才用**，不要預設用這條路來繞過 UI 限制。
- `claude-code-chat-history-limit`：對話訊息數超過 800 則上限的處理方式（今天這種高頻 CDP/PowerShell 逐條指令除錯任務容易碰到）

## ⚠ 刪除受控機器後的政策行為：等同重置成待審核，不會自動放行（2026-08-04 確認）

級聯刪除（見上方「不能刪除USB控管」章節）讓「刪除」這個 UI 操作本身可以成功執行，但**刪除後那台機器的 USB 政策不會變成放行**——已跟使用者確認過這是刻意設計，不是 bug：

- `UsbAgentLocalApiHandler.policy()` 對「查不到機器」（含被刪除）的處理**維持原本 fail-closed 邏輯**（`approvalStatus="UNKNOWN"`(未同步過)/`allowed=false`），跟系統一貫的「未知一律不放行」設計保持一致，**沒有**為刪除加特例放行邏輯。
- 因為 agent 端 `sync_with_aipower()` 每次同步都會呼叫 `register`（不只是啟動時一次，見 `agent.rs`），而 `register()` 在查不到機器時會**自動重新建立一筆新的 `PENDING` 記錄**——這代表無論 `policy()` 怎麼處理「機器不存在」這個瞬間狀態，很快就會被 `register` 蓋成 `PENDING`。
- **實務結論**：從 aipower 後台刪除一台受控機器，效果上等同於「重置成待審核狀態」，那台機器的 USB 讀寫會維持封鎖（因為 PENDING 也是 fail-closed），**需要你之後再次從「受控機器」列表核准它，才能恢復讀寫**。不要預期刪除後 USB 會自動變成可用。
- 曾經一度改成「機器不存在時直接放行」（`allowed=true`），但因為會被前述的 `register` 自動重建行為完全抵銷（沒有實際效果），且使用者確認「刪掉他時原本的狀態別變」才是想要的行為，故撤銷改動、恢復原邏輯。

---

## Conformance Addendum

## When to Use
USB 隨身碟控管系統——Rust Windows agent（D:\CODES\usb-storage-agent）+ aipower 主控台（C:\Aipower）。涵蓋編譯/部署/Windows Service 安裝、政策運作邏輯（登錄檔全域封鎖 Deny_Read/Write/Execute，非個別裝置 CM_Disable_DevNode）、machineCode 自動註冊待審核流程、aipower 端 LocalApi Bearer Token 整合、Jocket WebSocket 即時政策 push、審核狀態下拉選單、主控台 UI（單一選單+對話框+主從頁籤）、requireAdministrator manifest 自動提權、console 模式背景執行+系統匣圖示（tray icon）、Inno Setup 打包安裝程式（installer.iss）、遠端受控電腦部署（AIPOWER_BASE_URL 內網位址設定）、以及一系列實測踩過的坑（message-only window 收不到通知、CM_Disable_DevNode 被系統否決、Stop-Process 殺不掉提升權限行程、usbagent_unit.sql 重跑會清空真實資料、Jocket push 無窮迴圈）。當使用者要求「usb agent 重新編譯」「usb 控管服務裝/解除安裝」「usb agent 連不上 aipower」「隨身碟被鎖住/存取被拒」「usb 服務帳號」「即時通知/Jocket push」「usb agent 打包/安裝程式」「usb agent tray/系統匣圖示」「usb agent 遠端部署/裝到別台電腦」或提到這個專案時使用。

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
