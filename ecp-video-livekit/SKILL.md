---
name: ecp-video-livekit
description: Aipower(C:\com\chainsea) 的視訊客服功能——後端 ChatRoom video 方法、LiveKit media server、Cloudflare Tunnel 對外路由、手機優先的通話網頁(含即時字幕/QR/訊號格)、公開加入端點。用於重啟/除錯視訊通話、擴充通話頁面 UI、或理解這套自建視訊客服系統的架構。
---

# Aipower 視訊客服 (LiveKit)

Aipower(C:\com\chainsea，port 22821)原本沒有視訊客服功能，這是仿照 VRS(手語視訊轉譯中心)的樣子，
用 **LiveKit**(不是 VRS 原本用的 OpenVidu)自建的一套視訊通話系統。詳細的移植過程/踩坑記錄見專案記憶
`project_lab2_video_customer_service_port.md`，本 skill 只記「現在長怎樣、怎麼維護、怎麼繼續擴充」。

## 整體架構

```
使用者手機/瀏覽器
    │
    ├─ https://hch.james-huang.org/aipower/...      (Cloudflare Tunnel → 127.0.0.1:22821 Tomcat)
    │     ├─ /join-video                             公開進入點(不需 ECP 登入)，見下方 JoinVideoServlet
    │     ├─ /videopage/index.html                    實際的視訊通話網頁(純靜態 html/js)
    │     └─ /ecp/page/video/VideoTest.jsp             ECP 選單內的「視訊交談」(需要 ECP 登入)
    │
    └─ wss://livekit.james-huang.org                (Cloudflare Tunnel → 127.0.0.1:7880 LiveKit 訊令)
          LiveKit server 本身另外監聽 7881(TCP)/7882(UDP mux)做媒體流，
          這兩個 port **不走 Cloudflare Tunnel**(免費版不支援任意訪客的原始 TCP/UDP 轉發，
          只有 Cloudflare Spectrum 付費版才行)，要讓外部訪客真的看到影像/聲音，
          必須在路由器上把 TCP 7881 + UDP 7882 port forward 到這台機器的 LAN IP(目前是 192.168.2.110，
          `Get-NetIPAddress` 查 Wi-Fi 介面確認；這組 IP 會因為 DHCP/換網路而變動，port forward 沒跟著
          更新是「signaling 連得上、畫面/聲音出不來」最常見的原因，見下方 Gotcha)。
```

### Gotcha：LAN IP 換了但路由器 port forward 沒跟著改 → signaling 連得上、媒體連不上

**症狀**：ECP 選單「視訊交談」點下去不再是「連線失敗」畫面，而是卡在自拍畫面出現、狀態顯示連線中，
瀏覽器 console 看得到 `connected to Livekit Server`（signaling 經 Cloudflare Tunnel 正常連上），但接著
`[LiveKitApp] connect failed ConnectionError: could not establish pc connection`，畫面/對方一直進不來。

**根因**：signaling(WSS)走 Cloudflare Tunnel，不受 LAN IP 影響；但實際的 WebRTC 媒體(TCP 7881 / UDP 7882)
完全繞過 Tunnel，靠路由器的 port forward 直接指到這台機器的 LAN IP。這台機器的 LAN IP 會因為
DHCP 續約、換 Wi-Fi、換路由器而改變，一旦跟路由器裡設定的目標 IP 對不上，port forward 規則就變成
轉發到一個不存在/別台機器的位址，媒體連線自然建立不起來，但 signaling 完全不受影響，很容易誤判成
「LiveKit 又掛了」或「domain 設定錯」而白繞一圈。

**檢查**：
```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.IPAddress -notlike "169.254.*" -and $_.IPAddress -ne "127.0.0.1" }
```
比對這裡查到的 Wi-Fi IP 跟路由器 port forward 規則裡寫的目標 IP 是否一致。**不一致就是這個問題**，
需要使用者自己到路由器後台把 TCP 7881 + UDP 7882 兩條規則的目標 IP 改成目前的 LAN IP(AI 端無法代勞)。

## 關鍵檔案

| 用途 | 路徑 |
|---|---|
| LiveKit 執行檔+設定 | `C:\com\chainsea\livekit\livekit-server.exe`、`livekit\config.yaml` |
| LiveKit token 產生工具 | `com.chainsea.ecp.video.LiveKitTokenUtil`(純 JDK `javax.crypto` 手刻 HS256 JWT，無外部依賴) |
| 公開進入點 servlet | `com.chainsea.ecp.video.JoinVideoServlet`，註冊在 `WEB-INF/web.xml`，url-pattern `/join-video` |
| ChatRoom 視訊方法 | `ChatRoomActionImpl`/`ChatRoomServiceImpl` 新增 `replyVideoInvite`/`getDownloadVideoUrl`/`getVideoJoinUrl` |
| 視訊業務 class(8個) | `com/chainsea/ecp/video/{VideoCaptureHome,Action,ActionImpl,Dao,DaoImpl,Model,Service,ServiceImpl}` |
| 通話網頁(手機優先 UI) | `apache-tomcat\webapps\aipower\videopage\index.html` + `js\livekit-app.js` |
| LiveKit 瀏覽器 SDK | `videopage\lib\livekit\livekit-client.umd.min.js`(livekit-client 2.20.1，全域變數 `LivekitClient`) |
| QR code 瀏覽器函式庫 | `videopage\lib\qrcode\qrcode.min.js`(全域變數 `QRCode`，`QRCode.toCanvas(canvas, text, opts, cb)`) |
| ECP 選單版通話頁 | `apache-tomcat\webapps\aipower\ecp\page\video\VideoTest.jsp` |
| 原始碼 git 倉庫 | `C:\com\chainsea\tool\src\*.java`，獨立 git repo 在 `C:\com\chainsea\tool\.git` |
| Cloudflare Tunnel 設定 | `C:\Users\HCH\.cloudflared\config.yml` |

## LiveKit 認證資訊(正式，非 dev 模式)

```
API Key:    apikey6444104edbe8
API Secret: 3b13897416961d6999f3cc0952d4294e0f788b9f2602a128
WS URL:     wss://livekit.james-huang.org (對外) / ws://127.0.0.1:7880 (本機)
```
這組 key/secret 目前**寫死**在三個地方，換掉的話三處都要改：`livekit\config.yaml`、
`ChatRoomServiceImpl` 的 `LIVEKIT_API_KEY`/`LIVEKIT_API_SECRET` 常數、`JoinVideoServlet` 的同名常數。

## 重啟/維護

**LiveKit server 現在已經寫進 `server.bat`**(在 Redis 之後、cbm-lite 之前用 `start "LiveKit" /B` 背景啟動)，
正常透過 `server.bat` 啟動 aipower 時會自動一起帶起來，不用另外手動處理。

如果只是 LiveKit 自己掛掉(常見症狀：視訊交談點下去「連線失敗，請確認網路連線後重試」)，先檢查：
```powershell
tasklist /FI "IMAGENAME eq livekit-server.exe"
netstat -ano | findstr ":7880 :7881"
```
沒有的話手動重啟：
```powershell
Start-Process -FilePath 'C:\com\chainsea\livekit\livekit-server.exe' -ArgumentList '--config','C:\com\chainsea\livekit\config.yaml' -WorkingDirectory 'C:\com\chainsea\livekit' -WindowStyle Hidden
```

重啟整個 aipower(Tomcat)時，記得先確認沒有殘留的孤兒 `mariadbd.exe`(殺 Tomcat 不會自動殺掉它衍生的
mariadbd 子行程)，細節見 `ecp-server-startup` skill 的相關章節。

修改 `com.chainsea.ecp.video.*` 或 `ChatRoomServiceImpl`/`ChatRoomActionImpl` 原始碼後的編譯部署流程：
```bash
JAVAC="C:/com/chainsea/jdk/bin/javac.exe"
"$JAVAC" -d "C:/com/chainsea/tool/classes_new" \
  -classpath "C:/com/chainsea/apache-tomcat/webapps/aipower/WEB-INF/classes;C:/com/chainsea/apache-tomcat/webapps/aipower/WEB-INF/lib/*;C:/com/chainsea/apache-tomcat/lib/*" \
  -encoding UTF-8 "C:/com/chainsea/tool/src/<改動的檔案>.java"
# 再把 tool/classes_new 底下對應的 .class 複製回 WEB-INF/classes 同樣的相對路徑，然後重啟 aipower
```
`web.xml`(新增/修改 servlet mapping)的改動需要重啟 aipower 才會生效；純 `.jsp`/`.html`/`.js` 靜態檔案
**不需要重啟**，改完直接生效(但瀏覽器可能快取 `.js`，靜態頁面用 `?v=日期字母` 的 query string 做 cache-busting，
每次改 `js/livekit-app.js` 記得同步把 `index.html` 裡的 `<script src=...?v=xxx>` 版本號往後遞增一碼)。

## 前端功能清單(videopage/index.html + livekit-app.js)

- 手機優先版面：遠端畫面全螢幕、本地自拍畫面右上角小視窗(鏡像)、底部大按鈕操作列、支援 iPhone
  safe-area-inset(瀏海/手勢列)
- 訊號格(電話訊號樣式)：兩側都有，綁定 LiveKit `RoomEvent.ConnectionQualityChanged`，非自製假數據
- 切換前後鏡頭：`room.switchActiveDevice('videoinput', deviceId)`，只有偵測到多顆鏡頭才顯示按鈕
- 即時字幕(語音)：瀏覽器內建 `SpeechRecognition`/`webkitSpeechRecognition`(`lang: 'zh-TW'`)，不需要
  任何第三方 API 金鑰，但辨識品質看瀏覽器自己接的服務，Safari 支援度較差
- 即時字幕(打字)：底部輸入列，Enter 或送出鍵觸發，跟語音字幕共用同一套顯示元件跟 LiveKit 資料通道
- 字幕跨參與者同步：都是透過 `room.localParticipant.publishData(payload, {reliable:false, topic:'caption'})`
  廣播 JSON `{text, isFinal}`，對方用 `RoomEvent.DataReceived` 監聽 `topic==='caption'`
- 字幕外觀：全部統一靠右對齊，用顏色區分「我」(綠)跟對方(藍)，不用左右分兩邊；容器
  `overflow:hidden`+`justify-content:flex-end`確保永遠貼底部、舊訊息自動被裁掉，不會被輸入列/按鈕擋到
- 按鈕關閉狀態的視覺語言：統一用「同一個圖示疊一條斜線(`.off-slash`)」表示關閉/靜音，不用反白背景
- 房間 QR Code：畫面內建、可收合成小圖示，展開後只顯示 QR code 本身(無多餘文字)，QR 內容固定寫死
  `https://hch.james-huang.org/aipower/join-video?room=<目前房間名>`(不能用 `window.location.origin`，
  否則本機測試時 QR 會指向 127.0.0.1，手機掃了連不到)

## 公開加入端點(`/join-video`)——給 QR code/外部分享用

`JoinVideoServlet` 是特意繞開 Quicksilver 框架做的：這個 webapp 的 `*.jsp` 請求會被某個框架層攔截包裝成
ECP 登入畫面(即使該 `.jsp` 根本沒有註冊在 TsMenu/TsPage 裡也一樣)，所以想要「不用 ECP 帳號也能加入視訊」
的公開連結，**不能用 JSP**，必須是直接在 `web.xml` 註冊 url-pattern 的原生 Servlet(`javax.servlet.*`，
這個 Tomcat 是 9.0.93，還不是 jakarta.*)。這個 servlet 每次被訪問都會現生成一個全新的 1 小時效期 token，
所以連結本身可以印出來/做成固定 QR code 永久有效，不會過期。

沒帶 `?room=` 參數時預設加入共用房間 `workbench-shared-room`；要分房間就在網址後面加
`?room=<任意字串>`。

## ECP 選單版(「視訊交談」，需要登入)

在客戶關係群組底下新增了 TsMenu/TsPage/TsRoleMenu(比照既有的「文字交談」抄一份 pattern)，
FParentId 用的是 `e308e57a-3819-4c35-99bc-87e576bb7db6`(客戶關係群組)。**改完 TsMenu/TsPage 這類
metadata 後，選單本身有 cache，單純重整瀏覽器頁面不會生效，要重啟 aipower 才會重新載入。**
`VideoTest.jsp` 用 session id 當房間名稱(`workbench-<sessionId>`)，是**每人一間房**的即興通話 demo，
不是「客服接聽客戶來電邀請」的正式進線流程。

## 還沒做的東西(誠實記錄，不要假裝完成)

1. **工作檯「電話控制」面板完全沒有邀請/接受視訊的 UI 按鈕**——後端 API 跟通話頁面都能動，但
   aipower 自己的客服工作檯裡沒有任何觸發入口，agent 端沒有「邀請客戶視訊」的按鈕，客戶端也沒有
   「接受/拒絕視訊邀請」的彈窗。這是目前最大的缺口，要做的話是新的 ECP 工作檯前端 JS 開發(彈窗+
   postMessage 串接 videopage iframe)，不是後端問題。
2. `getDownloadVideoUrl` 只是回傳「not supported yet」的 stub，真要做通話錄影下載需要接 LiveKit 的
   Egress API，是獨立一塊工作。
3. `VideoSTTTaskRecord`(視訊會議轉文字，ecp.main 原生的雲端 STT 功能，跟這裡自己接瀏覽器 Web Speech
   API 是兩回事)完全沒搬，8 個 class 都還在 ecp.main 那邊沒動。
4. 路由器 port forward(TCP 7881 + UDP 7882 → 192.168.11.205)是否已經真的設定好，需要跟使用者確認，
   AI 端無法代勞(那是路由器後台操作)。沒做的話：訊令連得上，但實際影像/聲音傳不過去。

## 相關 skill

`chainsea-feature-migration`(這次移植一開始用的方法論)、`ecp-server-startup`(重啟 aipower 前先清孤兒 mariadbd)、
`cloudflare-tunnel`(新增子網域路由、UAC 重啟陷阱)。

---

## Conformance Addendum

## When to Use
Aipower(C:\com\chainsea) 的視訊客服功能——後端 ChatRoom video 方法、LiveKit media server、Cloudflare Tunnel 對外路由、手機優先的通話網頁(含即時字幕/QR/訊號格)、公開加入端點。用於重啟/除錯視訊通話、擴充通話頁面 UI、或理解這套自建視訊客服系統的架構。

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
