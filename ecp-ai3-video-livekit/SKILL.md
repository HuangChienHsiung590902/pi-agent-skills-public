---
name: ecp-ai3-video-livekit
description: C:\ECP(見 ecp-ai3-instance skill)上把官方 VideoPage 模組(原本用 OpenVidu)改寫成 LiveKit 版的 MVP——後端 VideoGatewayServlet、前端獨立 git repo、Caddy TLS proxy，含 13 個實測踩坑(Caddy 自簽憑證沒被瀏覽器信任、Caddy 用特定 IP 當站台位址導致換網段就 SNI 不符、3 秒客服在場檢查太緊幾乎必炸、LiveKit 單一 UDP port 導致視訊 track publish 逾時、LiveKit ParticipantConnected 事件丟失、CSS 版面塌陷、原廠 recording-tag 殘留迴圈、malformed JSON 500、原廠 GPS 權限檢查誤導、adb am start 網址裡的 & 被裝置 shell 吃掉、手機直式鏡頭塞進橫式小視窗被裁到爆滿、本地視訊 attach race condition、CDP 關分頁留下 LiveKit 幽靈參與者)。附可直接執行的 scripts/(手機開連結、CDP 遠端除錯手機分頁、LiveKit 房間管理踢幽靈)。用在：繼續開發這套 VideoPage LiveKit 功能、除錯視訊通話問題(尤其「雙方都連上但看不到對方視訊」「訊令連得上但一直斷線」「大畫面小畫面看起來一樣」)、要用手機測試、或需要複用其中任何一個踩坑的解法。
---

# C:\ECP 的 VideoPage → LiveKit 改寫(2026-07-11 MVP)

在 `C:\ECP`(見 `ecp-ai3-instance` skill)這套全新安裝上，把官方 `VideoPage_8.5.03.01` 模組(原生用 OpenVidu)
拆掉 OpenVidu、改用 **LiveKit** 重新實作。**跟 `ecp-video-livekit` skill 講的舊實例(C:\com\chainsea)自建版本
是不同架構**：那邊是自己刻的 JoinVideoServlet+單一 webapp 內的 class，這裡是官方模組骨架+兩個獨立 git repo，
不要互相套用細節。

用「Task 拆解 → 實作 → 審查 → 有問題修正 → 再審查」的 subagent pipeline 做完，backend 5 commits、frontend 11
commits，最終全分支審查判定 **Ready to merge**。

## 架構

```
手機/瀏覽器(區網內，同 Wi-Fi)
    │
    ├─ https://192.168.1.106:12822/ecp/VideoPage/            前端靜態頁(index.html/js)
    │     └─ /openapi/*  → VideoGatewayServlet               後端 gateway(見下)
    │
    └─ wss://192.168.1.106:7443                              Caddy(TLS) → reverse_proxy → 127.0.0.1:7880(LiveKit 訊令)
                                                               LiveKit 媒體另外走 7881(TCP)/7882-7892(UDP 範圍)，
                                                               同一台機器區網內直連，沒有另外做 NAT/port forward(跟舊
                                                               ecp-video-livekit 的對外 Cloudflare 情境不同，這套目前
                                                               只在區網內測試)
```

這台機器同時有 **實體 Wi-Fi(`192.168.1.106`)** 跟 **ZeroTier 虛擬網卡(`10.145.119.100`)** 兩張網卡。2026-07-11
最初用的是 ZeroTier IP，後來改成實體區網 IP——**但這不是解決視訊連不上的關鍵**(見踩坑 8，兩個網段都出現過
一樣的錯誤)，純粹是因為區網延遲更低、更貼近真實部署情境。Caddyfile 已經改成萬用 `:7443`(不綁定特定 IP)，
兩個網段的位址理論上都能用，只是 Config.js 目前寫死其中一個。

## Caddy 設定：站台位址要用萬用 `:7443`，不要綁定特定 IP

`C:\ECP\caddy\Caddyfile` 原本是 `10.145.119.100:7443 { tls ...; reverse_proxy ... }`——Caddy 雖然還是會在
`0.0.0.0:7443` 全介面監聽(TCP 連線進得來)，但**站台比對是靠 TLS SNI／Host 做的**，換一個目的地 IP(例如
改用區網 IP `192.168.1.106`)連過去時，瀏覽器送的 SNI 是 `192.168.1.106`、跟設定檔裡的 `10.145.119.100`
對不上，Caddy 找不到對應站台就直接讓 TLS 交握失敗(`ERR_SSL_PROTOCOL_ERROR`)，**連「不安全但可以點過去」的
警告頁都不會出現**——因為 TCP 有通、只是 TLS 層被拒絕，症狀比純粹的憑證不信任更難判斷。
現在已改成 `:7443 { tls ...; reverse_proxy ... }`(不寫 host，只寫 port)，同一個 cert 對任何目的地 IP 都
會回應，之後只要瀏覽器手動信任過那個 IP:7443 的自簽憑證(見踩坑 0)就能用，不用再為每個網段開一個站台區塊。

## 關鍵檔案

| 用途 | 路徑 |
|---|---|
| 後端原始碼(獨立 git repo) | `C:\ECP\src\com\chainsea\ecp\video\livekit\VideoGatewayServlet.java` |
| 後端部署位置 | `C:\ECP\apache-tomcat\webapps\ecp\WEB-INF\classes\...`(web.xml 註冊 `url-pattern: /openapi/*`) |
| 前端原始碼(獨立 git repo) | `D:\_暫時保留\ECP\AI3_Version\VideoPage_8.5.03.01.202504021824\VideoPage\`(baseline: 官方原廠 as shipped；原本在 `D:\ECP\...`，2026-07-11 被搬到 `D:\_暫時保留\ECP\...`，路徑會變動，找不到就先確認有沒有搬家) |
| 前端部署位置 | `C:\ECP\apache-tomcat\webapps\ecp\VideoPage\`(改完前端後要複製過去，非 symlink) |
| 前端設定 | `VideoPage\config\Config.js` — `GatewayServer`/`LiveKitServer` 目前寫死 `192.168.1.106`(區網 IP，2026-07-11 從 ZeroTier IP `10.145.119.100` 改過來) |
| Caddy 設定 | `C:\ECP\caddy\Caddyfile`(站台位址是萬用 `:7443`，不綁定特定 IP，見下) |
| LiveKit 設定 | `C:\ECP\livekit\config.yaml`(API key/secret 也在這裡；`rtc.port_range_start`/`port_range_end` 已從單一 port 7882 拉寬成 7882-7892，見踩坑 8) |

`.superpowers/sdd/` 目錄(在 `C:\ECP\src` 裡)留有每個 Task 的審查 diff，要回溯某次審查抓到什麼可以直接看那裡。

## 後端目前狀態

只有 `/openapi/getVidoePageToken` 是真的實作(簽發 LiveKit JWT)，其餘端點是 stub，還沒接。

## scripts/ 目錄(固定操作直接執行，不要每次重新手動摸索)

| 檔案 | 用途 |
|---|---|
| `scripts/phone_open_url.sh` | 用 adb 在手機 Chrome 開一個含 `&` 的網址，正確處理踩坑 9 的 shell 吃字問題 |
| `scripts/cdp_eval.mjs` | 對指定 CDP 分頁(手機或任何已知 webSocketDebuggerUrl 的頁面)直接執行 JS，見「遠端除錯手機 Chrome」一節 |
| `scripts/lk_admin.mjs` | LiveKit RoomService admin API(list/remove/clean)，踢除幽靈參與者，見踩坑 12 |

## 十三個實測踩坑(不要重新 debug 一次)

0. **(最容易誤判成別的問題)Caddy 的自簽憑證要先讓瀏覽器手動信任一次，否則訊令/媒體永遠連不上，而且不會跳出「不安全」警告頁讓你點過。**
   `wss://10.145.119.100:7443` 跟 `https://10.145.119.100:12822` 是**兩個不同的憑證(不同 keystore)**，
   瀏覽器對每個「host:port」分開記錄信任例外。直接打開 VideoPage 頁面(12822)本身可能沒問題(該憑證可能
   之前已經被信任過)，但 LiveKit 的 WebSocket(`wss://...:7443/rtc/v1?...`)和 `fetch`
   (`https://...:7443/rtc/v1/validate`)這種**非頂層導覽的請求**，遇到未信任憑證會直接靜默失敗
   (`net::ERR_CERT_AUTHORITY_INVALID`)，**不會**跳出「進階→繼續前往」那種可以點過去的警告頁——那種
   互動頁面只有「直接在網址列打開該網址」這種頂層導覽才會出現。
   **症狀**：console 看到 `ConnectionError: could not establish signal connection: Failed to fetch`，
   `websocket closed`，但打開 VideoPage 首頁本身完全正常，很容易誤判成程式碼又壞了或 LiveKit 沒啟動
   (實際上 LiveKit/Caddy 程序都活著，netstat 也看得到在監聽)。
   **解法**：每一個要測試視訊的瀏覽器(含手機瀏覽器)，都要**先**直接在網址列開一次
   `https://10.145.119.100:7443/`(會看到 LiveKit 回的純文字 `OK`，代表這個 host:port 的憑證例外已經
   被瀏覽器記住)，**之後**再去開 VideoPage 的連結，訊令才連得上。這一步沒做，換新瀏覽器/換裝置/清過
   憑證例外都要重做一次。

1. **LiveKit SDK：`ParticipantConnected` 事件對「加入時房間已有人」會直接丟棄，不是延後發送。**
   `emitWhenConnected` 內部邏輯是連線還在 `Connecting` 階段時事件被丟掉，只有 `Connected` 之後才真的 emit；
   而「房間已存在的參與者清單」是在連線完成**之前**處理的，所以第二個(含之後)加入者收不到既有參與者的
   `ParticipantConnected`，導致對方的 UI 綁定(例如 leave 按鈕)沒跑到。**解法**：連線後不能只依賴這個事件，
   要顯式走一次 `room.remoteParticipants` 把已存在的參與者手動跑一次綁定邏輯。這是靠反查 SDK 原始碼
   + Playwright 實測才抓到的，不是文件寫的。

2. **手機瀏覽器要用鏡頭/麥克風，一定要 secure context(HTTPS/WSS)。** LiveKit server 本身只給 plain
   ws(7880)，解法是在前面架 **Caddy** 做 TLS termination 再 reverse_proxy 回 127.0.0.1:7880。

3. **CSS `height:100%` 在沒有完整祖先高度鏈時不會撐滿 viewport。** 視訊容器改用
   `position:fixed; inset:0` 才真正滿版，不要再用 `height:100%` 疊代測試。

4. **原廠 VideoPage 有一個輪詢 `#recording-tag` DOM 元素的迴圈**，Task 5 把這個元素拿掉重刻 UI 後，
   舊迴圈還留著繼續跑，找不到元素就狂發 `ReConnectVideo`/network-error 的假錯誤 toast。改寫前端時，
   拿掉原廠某個 DOM 元素一定要順便找出所有還在監看它的輪詢/事件邏輯一起拔掉。

5. **Token endpoint 的 JSON body 沒做防呆，畸形 JSON 直接炸 500。** 新寫的後端 endpoint 只要吃
   request body 就要包 try/catch 明確處理 parse 失敗，不要讓例外直接冒出去變成裸 500。

6. **原廠遺留的「開場先跳一個攝影機/GPS 權限提示」邏輯會誤導判斷，而且 GPS 檢查跟視訊完全無關。**
   `App.js` 的 `checkPermissionsAndShowDialog()`（開場就會跑）只是用 `navigator.permissions.query()`
   問瀏覽器「**之前**有沒有記住授權」，只要不是已經記住的 `granted`（包含「還沒問過」的 `prompt` 狀態，
   也就是任何人第一次造訪都會是這個狀態），就跳出「無法啟用攝影機」的提示——這只是裝飾性的提前告知，
   不是真正的錯誤，真正的授權要等後面 `VideoController.js` 呼叫 `setCameraEnabled(true)` 時瀏覽器跳出
   的原生請求才算數。原本這段還會**一併檢查 `geolocation` 權限**、把「GPS」加進提示訊息裡——這是
   OpenVidu 原廠版本某個跟浮水印/定位打卡有關的舊功能殘留，跟 LiveKit 視訊本身毫無關係，2026-07-11
   已經把 `checkUserGPS()` 整段拿掉（`App.js` 原始碼+部署位置都改了），提示現在只會提攝影機。

7. **`WebChatControl.doCheckMainAgentConnect()` 的「客服在場」檢查只給 3 秒，而且完全靠一趟 data channel
   來回，真實網路下幾乎必炸。** 一般使用者(`FROM_USER`)加入房間後會廣播 `USER_CHECK` 訊息問「客服在嗎」，
   同時起一個 3 秒計時器；3 秒後如果沒收到客服回應的 `MAIN_AGENT_CHECK`(靠設定 `App.AgentMainChecked`)，
   就直接呼叫 `session.disconnect()`、跳「找不到客服」錯誤、關閉頁面。**這個 3 秒的 data channel 來回在
   真實裝置(尤其手機)上經常來不及**——WebRTC data channel 建立本身需要時間，即使客服早就已經在房間裡
   穩定連著，也常常來不及在 3 秒內完成一趟真正的訊息往返，導致「明明客服已經在，客戶還是被踢」。
   **症狀**：console 看得到雙方都 `publishing track` 成功(代表本地媒體已經抓到)，然後客戶端自己
   `disconnect from room`，網址列被加上 `&action=disconnect`；由於 `App.js` 的 `initView()` 一看到
   `action=disconnect` 就直接顯示「視訊服務已結束」不會重新嘗試加入，**同一個網址重新整理只會一直卡在
   這個畫面**，必須換一個乾淨(不含 `&action=disconnect`)的連結才能重新加入。
   **解法**：2026-07-11 已修改 `WebChatControl.js` 的 `doCheckMainAgentConnect()`——除了原本的
   `App.AgentMainChecked`，額外用 `VideoController.session.remoteParticipants.size > 0`(LiveKit client
   本地就有的房間狀態，不需要等 data channel)當作更即時可靠的訊號，兩種訊號任一成立就算通過；逾時判斷
   也從「一次性 3 秒」改成「每秒檢查一次、最多 8 次(共 8 秒)」，兩邊都改了(源碼+部署位置)。**測試雙向
   通話時務必記得**：要嘛讓「客服」那一方用 `&from=agent&isMain=1` 加入且先於客戶穩定連上，要嘛就準備
   接受這個 3→8 秒緩衝依然可能不夠、需要再往上調。

8. **(這次卡最久、最容易誤判成網路問題)LiveKit 的 `rtc.port_range_start`/`port_range_end` 原本設成同一個
   UDP port(都是 7882)，導致視訊 track 常常 publish 逾時。** 症狀是簽章、訊令、`publishing track` 全部
   正常，雙方也都能看到**自己的**本地畫面，但**永遠收不到對方的視訊/音訊**(沒有任何 `TrackSubscribed`)。
   LiveKit server 自己的 log(`C:\ECP\server_debug.log`，server.bat 啟動 LiveKit 時的輸出都會導進這個檔案)
   會看到關鍵一行：
   ```
   ERROR ... supervisor error on publication ... "trackID": "TR_VC...", "error": "publish time out"
   ```
   (`TR_VC` 開頭是視訊 track、`TR_AM` 開頭是音訊——當時只有視訊逾時，音訊靠重連後勉強擠過去，這是判斷
   「音訊還行、視訊完全不行」的關鍵線索)。**一開始誤以為是 ZeroTier 虛擬網卡疊加封裝造成的頻寬/延遲問題，
   把 Config.js/Caddyfile 全部改成走實體區網 IP 後，同樣的 `publish time out` 錯誤在區網底下一樣重現**，
   證明跟 ZeroTier 無關，真正瓶頸是**所有音訊視訊、不管幾個參與者，全部都要擠過同一個 UDP port**。
   **解法**：把 `config.yaml` 的 `port_range_end` 從 7882 拉寬到 7892(給 LiveKit 十個 UDP port 可以用)，
   重啟 LiveKit(`taskkill /F /IM livekit-server.exe` 後用 `livekit-server.exe --config config.yaml` 重開，
   `netstat -ano | findstr UDP` 確認多個 788x port 都在監聽)，雙向視訊立刻就正常了。**除錯這類「本地畫面
   都有、就是收不到對方」的問題時，直接查 LiveKit server 自己的 log 找 `error`/`publish time out`/
   `TrackSubscribed`，比在瀏覽器 console 或猜網路架構快得多**——瀏覽器端只看得到「連不上」，看不到
   server 端真正卡在哪個 track、哪個階段。

9. **`adb shell am start -a android.intent.action.VIEW -d "https://x?a=1&b=2"` 的 `&` 會被裝置端 shell
   當成背景執行符號吃掉，即使本機端已經用雙引號/單引號包住整個網址。** adb 會把所有參數重新組成一串字串
   丟給手機的 `/system/bin/sh -c` 執行，本機端的 quoting 保護不了裝置端；沒跳脫的 `&` 一到裝置端就被當成
   「前一個指令丟到背景執行」，URL 從 `&` 之後(例如 `joinName=HCH`)整個消失，網址列只剩前半段，App 端會
   拿到空的 `joinName` 而 fallback 成別的預設值。
   **踩過的錯誤解法**：以為用 bash 參數替換把 `&` 轉成 `\&`(`${url//&/\\\\&}`)就能解決——本機端用 `xxd`
   確認位元組完全正確(`\x5c\x26`)，但這個值一旦透過**變數**傳給 `adb.exe`(Windows 原生執行檔)，反斜線會在
   某個環節被吃掉或讓裝置端解析錯亂，實測直接把網址從反斜線處砍斷，比原本更糟；懷疑跟 Git Bash 幫原生
   exe 組 Windows command-line 字串的規則有關，沒有再深究根因。
   **真正可靠的解法**：在本機端幫整個網址多包一層「裝置端」單引號，讓 `-d` 的實際內容變成
   `-d 'https://x?a=1&b=2'` 送到手機，裝置端的 `sh -c` 看到單引號就不會把 `&` 當背景符號——直接用純變數
   (不用任何跳脫)即可：`adb shell am start -a android.intent.action.VIEW -d "'$url'"`。已包成
   `scripts/phone_open_url.sh`，之後手機測試一律呼叫這支 script，不要重新手動組指令。

10. **手機鏡頭拍出來是直式(9:16)，塞進原本寫死橫式(`aspect-ratio:16/9`)的本地小預覽框，`object-fit:cover`
    會把畫面裁到只剩中間一小條、視覺上像整個放大到只看得到臉。** `object-fit:contain` 雖然能看到完整畫面，
    但會在左右留下大片黑邊，使用者觀感上也不喜歡；如果直接寫死改成 `aspect-ratio:9/16`(直式)也只解決一半
    ——**手機轉成橫式時鏡頭輸出會跟著變成橫式，寫死直式框反而重現同一個裁切問題，這時候應該要跟桌面版
    看起來一模一樣(桌面版本來就是橫式框+橫式 webcam)**。**最終解法**：`#localVideoWrap` 不寫死
    `aspect-ratio`，改成在 `VideoController.js` 的 `connectAndPublish()` 裡，本地 `<video>` attach 完成後
    監聽它的 `resize` 事件(track 尺寸改變時觸發，裝置旋轉也算)，即時把 `wrap.style.aspectRatio` 設成
    `el.videoWidth + '/' + el.videoHeight`，讓框的形狀**動態跟著實際鏡頭輸出尺寸走**——直放時是 9:16、
    橫放時自動變回 16:9(跟桌面版同一套 `object-fit:cover` 邏輯，橫放時呈現方式完全一致)，`object-fit`
    固定用 `cover` 即可，不需要黑邊也不需要裁切變形。用 adb 強制旋轉測過雙向都正確
    (`adb shell settings put system user_rotation 1/0`，配合 CDP 查
    `document.getElementById('localVideo').videoWidth/videoHeight` 確認)。同時加了兩顆「拉近/拉遠」按鈕
    (`#btnLocalZoomIn`/`#btnLocalZoomOut`，`App.js` 的 `$(document).ready()` 內)，用 CSS `transform: scale()`
    疊加微調(最小值鎖定在 1，不能往外縮出黑邊，只能往內裁近)，讓使用者自己微調框內構圖。`#localVideo` 的
    `track.attach()` 每次都會整個換新的 `<video>` 元素(但保留 id)，所以 zoom 按鈕的 handler、以及
    `syncLocalAspect` 都要用當下這個新元素的參照，不能快取舊的。

11. **(尚未修)`VideoController.connectAndPublish()` 裡，`setCameraEnabled(true)` resolve 後立刻同步
    `forEach(room.localParticipant.videoTrackPublications)` 把本地鏡頭塞進 `#localVideo`，這個 Map 的
    實際填入時機跟 `setCameraEnabled` 的 promise resolve 時機不保證同步，手機鏡頭初始化(解析度協商、
    自動對焦)通常比 PC 內建 webcam 慢，容易卡在「forEach 執行當下 Map 還是空的」這個時間差，導致
    `#localVideo` 永遠沒被換成真正的 `<video>` 元素、`srcObject` 一直是 `null`——但因為遠端(對方)收得到
    這個裝置的視訊軌是正常的(只有「自己看自己」的小視窗失敗)，很容易誤判成別的問題。用 CDP 直接查
    `document.getElementById('localVideo').srcObject` 是否為 `null`、`readyState` 是否為 0，可以快速
    確認是不是踩到這個 race condition(見下方「遠端除錯手機 Chrome」)。**正確修法(還沒做)**：改成監聽
    `LocalTrackPublished` 事件來附加 `#localVideo`，不要依賴 `setCameraEnabled` resolve 後的同步 forEach。

12. **CDP `Target.closeTarget`(`curl http://127.0.0.1:PORT/json/close/<id>`)關掉手機 Chrome 分頁，不會
    可靠觸發該分頁的 `beforeunload` → `VideoController.session.disconnect()`，導致該參與者變成 LiveKit
    房間裡的「幽靈」——WebSocket 連線還活著、繼續發送舊的視訊軌，但沒有真人在操作。** 前端目前的
    `TrackSubscribed` handler 邏輯是「不管哪個遠端參與者的視訊軌被訂閱，都直接塞進同一個 `#remoteVideo`」
    (`VideoController.js`)，完全沒有 per-participant 邏輯，房間裡一旦累積多個同名幽靈(尤其反覆用同一個
    `joinName` 重新整理測試時)，大畫面會被其中一個(通常是最後訂閱成功的那個)蓋掉，症狀是「大畫面跟小
    畫面看起來根本是同一段影像」——這其實不是本地/遠端接反(用 CDP 查 `srcObject.getVideoTracks()[0].label`
    可以證明：本機鏡頭有真實裝置名稱如 `"camera 1, facing front"`，遠端軌永遠是匿名 UUID，接線邏輯是對
    的)，而是幽靈參與者的舊鏡頭畫面剛好長得跟現在的本地鏡頭很像(同一支手機、同一個房間)。
    **解法**：用 LiveKit RoomService 的 admin API(`ListParticipants`/`RemoveParticipant`，twirp/HTTP，
    API key/secret 在 `C:\ECP\livekit\config.yaml`)手動清幽靈，已包成 `scripts/lk_admin.mjs`
    (`node scripts/lk_admin.mjs clean <room> <要保留的identity>`)。

    **踢除幽靈時「連自己這個活著的分頁也斷線」的副作用，根因已查清楚，是計時器洩漏 bug(已修)：**
    `VideoController.js` 的 `room.on(RoomEvent.ParticipantConnected, () => App.doConnectCreate())`
    ——這個 handler 不是只在自己剛加入時跑一次，**房間裡任何其他人加入，都會讓所有已經在場的 `FROM_USER`
    端重新執行一次 `doConnectCreate()`**。而 `doConnectCreate()`(`App.js`)跟它呼叫的
    `WebChatControl.doCheckMainAgentConnect()` 原本都是**無條件**`App.IdelCheck = setInterval(...)`／
    `App.HostCheck = setInterval(...)`，沒有先清掉舊的參照就直接覆蓋——舊的 interval 就這樣變成孤兒，
    繼續在背景每秒 tick、各自倒數自己的 8 次逾時，彼此不知道對方存在。這次測試期間反覆開了幾十個分頁
    加入同一個房間(每個都是一次 `ParticipantConnected`)，活著的那個分頁背後其實累積了一堆孤兒
    `HostCheck` 計時器；呼叫 admin API 改變房間成員(不管是踢幽靈還是任何人加入/離開)都可能讓
    `room.remoteParticipants` 在 LiveKit client 內部 resync 的瞬間短暫讀空，只要剛好有任何一個孤兒
    計時器在那個瞬間跑到第 8 次，就會誤判「客服不在」而呼叫 `session.disconnect()`——跟到底踢的是不是
    幽靈、是不是自己都無關，純粹是機率問題(孤兒計時器越多、累積測試時間越長，中獎機率越高)。
    真實客服情境一個房間通常只有客戶+一位客服兩人，最多只累積一顆 `HostCheck`，8 秒內自然清掉，不會踩到；
    但只要同一個房間被反覆加入退出測試夠多次，這個 bug 遲早會發作。
    **已修**：`doConnectCreate()`(`App.js`)跟 `doCheckMainAgentConnect()`(`WebChatControl.js`)都改成
    「先檢查並清掉舊的 `App.IdelCheck`/`App.HostCheck`，再建立新的」，兩邊(源碼+部署位置)都已套用。
    用 Playwright 在 PC 端模擬「連續 3 次有人加入房間」(直接呼叫 3 次 `App.doConnectCreate()`)驗證過：
    修好後只會觸發一次「找不到客服」的斷線流程(console 只有一筆 `closeP4Page event send.`)，不會像
    修之前那樣可能疊加出多次獨立倒數、多次斷線嘗試。

## 遠端除錯手機 Chrome(CDP，不用 Playwright 也能查手機分頁的 DOM/JS 狀態)

手機端沒有 Playwright MCP 可以接管，但只要手機開了 USB 偵錯，Chrome 會在
`@chrome_devtools_remote` 這個 abstract socket 上開 DevTools Protocol：

```bash
MSYS_NO_PATHCONV=1 adb forward tcp:9333 localabstract:chrome_devtools_remote
curl -s http://127.0.0.1:9333/json          # 列出手機上所有分頁的 id / url / webSocketDebuggerUrl
node scripts/cdp_eval.mjs "ws://127.0.0.1:9333/devtools/page/<id>" "location.href"
node scripts/cdp_eval.mjs "ws://127.0.0.1:9333/devtools/page/<id>" "fetch('/some/url').then(r=>r.text())" --await
```

這比截圖+肉眼判讀準確得多，尤其是查「這個 `<video>` 元素的 `srcObject`/`videoTracks`/`readyState` 到底是
什麼」這種截圖完全看不出來的狀態(參見踩坑 11、12 的除錯過程)。`/json` 的 `url` 欄位偶爾會是**分頁建立當下**
的快取值、不是即時的(例如 App 內用 `history.replaceState` 改過網址後，`/json` 可能還顯示舊網址)，要拿到
準確的當前網址一律用 `scripts/cdp_eval.mjs ... "location.href"` 現查，不要相信 `/json` 列表裡的 `url` 欄位。

## 還沒做的

- VideoGatewayServlet 大部分端點還是 stub。
- 錄影(LiveKit Egress)：最後一輪審查有確認「架構上可行」，但還沒真的接。
- 目前只能區網內測試(Config.js 寫死 IP)，還沒接對外網域/Tunnel。
- 每個新裝置/新瀏覽器第一次測試前，都要記得先手動信任一次 `https://192.168.1.106:7443`(見踩坑 0)。
- 踩坑 11 的本地視訊 attach race condition 還沒修(需要改成監聽 `LocalTrackPublished`)。
- 踩坑 12 的多參與者/幽靈畫面覆蓋問題本身還沒修(`TrackSubscribed` 目前還是「最後訂閱的贏」，沒有
  per-participant tile；但「踢幽靈連帶斷自己」那個計時器洩漏副作用已經修好，見踩坑 12 內文)。

## 本地小視窗直式化 + 拉近/拉遠按鈕(2026-07-11，已完成)

`#localVideoWrap` 從寫死橫式 `aspect-ratio:16/9` 改成 `9/16`(配合手機直式鏡頭)，`object-fit` 用 `cover`；
另外加了 `#btnLocalZoomIn`/`#btnLocalZoomOut` 兩顆按鈕，用 CSS `transform:scale()` 微調(最小值鎖定 1，只能
往內裁近、不會露出黑邊)。細節見踩坑 10，程式碼在 `index.html`(`#localVideoWrap` 區塊)跟 `App.js`
(`$(document).ready()` 內 `applyLocalZoom`)。

## 用手機測試時的額外提醒

- 手機瀏覽器測試一律用 `scripts/phone_open_url.sh "<url>"` 開連結，不要手動重組 `adb shell am start` 指令
  ——網址帶 `&` 時手動組指令很容易踩到踩坑 9。截圖用 `adb shell screencap -p /sdcard/x.png` 再 `adb pull`；
  **Git Bash 會把 `/sdcard/...` 誤轉成 Windows 路徑**，所有 adb 指令前面要加 `MSYS_NO_PATHCONV=1` 才不會出錯。
- 要精準點擊網頁按鈕，優先用「遠端除錯手機 Chrome(CDP)」一節的 `scripts/cdp_eval.mjs` 直接在頁面 context 內
  `document.querySelector(...).click()`，比算座標點擊更準、也不會因為版面 reflow 點歪；只有在沒辦法用
  JS 觸發(例如原生瀏覽器 UI)時才退回用 `adb shell uiautomator dump` 抓 `bounds="[x1,y1][x2,y2]"` 算座標。
- **絕對不要用 `pm clear com.android.chrome` 去清分頁/憑證信任狀態**——這個指令會清掉整個 Chrome 的資料，
  包含使用者原本登入的 Google 帳號工作階段，等於逼使用者重新登入。只是想清分頁的話用 `am force-stop com.android.chrome`
  (只結束程序，不清資料，重開會保留分頁)；只是想清某個網站的憑證信任例外，要請使用者自己在 Chrome 設定裡手動清除，
  沒有安全的 adb 捷徑。

## 相關 skill

`ecp-ai3-instance`(這套實例本身怎麼啟動/登入)、`ecp-video-livekit`(舊實例的不同架構版本，別搞混)、
`superpowers:subagent-driven-development`(這次用的 Task→實作→審查 pipeline 方法論)。

---

## Conformance Addendum

## When to Use
C:\ECP(見 ecp-ai3-instance skill)上把官方 VideoPage 模組(原本用 OpenVidu)改寫成 LiveKit 版的 MVP——後端 VideoGatewayServlet、前端獨立 git repo、Caddy TLS proxy，含 13 個實測踩坑(Caddy 自簽憑證沒被瀏覽器信任、Caddy 用特定 IP 當站台位址導致換網段就 SNI 不符、3 秒客服在場檢查太緊幾乎必炸、LiveKit 單一 UDP port 導致視訊 track publish 逾時、LiveKit ParticipantConnected 事件丟失、CSS 版面塌陷、原廠 recording-tag 殘留迴圈、malformed JSON 500、原廠 GPS 權限檢查誤導、adb am start 網址裡的 & 被裝置 shell 吃掉、手機直式鏡頭塞進橫式小視窗被裁到爆滿、本地視訊 attach race condition、CDP 關分頁留下 LiveKit 幽靈參與者)。附可直接執行的 scripts/(手機開連結、CDP 遠端除錯手機分頁、LiveKit 房間管理踢幽靈)。用在：繼續開發這套 VideoPage LiveKit 功能、除錯視訊通話問題(尤其「雙方都連上但看不到對方視訊」「訊令連得上但一直斷線」「大畫面小畫面看起來一樣」)、要用手機測試、或需要複用其中任何一個踩坑的解法。

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
