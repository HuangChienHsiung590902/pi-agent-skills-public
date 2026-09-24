---
name: mikopbx2026-docker
description: 本機用 Docker 跑的 MikoPBX 2026 沙盒實例（D:\APP\mikopbx2026-docker）——啟動/檢查容器、Web UI 登入、REST API 呼叫（含 OpenAPI 規格抓取）、CTI/ACD 開發現況（AMI 帳號、Call Queue 設定）。當使用者提到「mikopbx」「mikopbx2026-docker」「這個 PBX 容器」或要用這台機器測 REST API/CTI/ACD 時使用。跟 `mikopbx` skill（10.145.119.64 那台真實環境）是不同機器，不要混用。
---

# mikopbx2026-docker（本機 Docker 沙盒）

## 快速參考

| 項目 | 值 |
|---|---|
| docker-compose.yml | `D:\APP\mikopbx2026-docker\docker-compose.yml` |
| Container 名稱 | `mikopbx2026` |
| Web UI (HTTP) | `http://127.0.0.1:18080` |
| Web UI (HTTPS，自簽憑證) | `https://localhost:18443` |
| REST API Base | `http://127.0.0.1:18080/pbxcore/api/v3` |
| 目前 API Key（Full access） | `9064b2498e369c884e1eb4e3484c7f1d45c1c88076ac407260935fb595e93deb` |
| AMI (Asterisk Manager) | `127.0.0.1:5038`（只綁 loopback，見下方 docker-compose 設定） |
| AMI 帳號 | `1cami` / `ZuRxqYsTJrnQeMFyNUbp`（原廠內建的 MIKO Panel 整合帳號） |
| ARI 帳號 | `ai-bot` / `tCPPOgxFqidxiF2B5IJp`（voice-bot 專案 Task 5 建立，供 `deploy/mikopbx_setup.py` 設定的 ARI Stasis app 使用，分機 205 → `Stasis(ai-bot,${EXTEN})`；Task 7 的 docker-compose.yml 要沿用同一組密碼） |
| SIP (host 可連) | `127.0.0.1:5060/udp` + `10.145.119.100:5060/udp`（ZeroTier IP，跨機也能連），RTP 同樣雙綁 `10000-10800/udp`（給 host 上的真實軟體電話如 MicroSIP 用） |
| 分機 204 | `MicroSIP Test`，密碼透過 `GET /sip/204:getSecret` 取得，專門給 host 上的 MicroSIP 測試用（見 `microsip-control` skill） |
| baresip ctrl_tcp | `127.0.0.1:4444`（JSON-RPC over netstring，可遠端下 `dial` 等指令，見下方） |
| 版本 | MikoPBX 2026.2.118 |
| Docker network 固定 IP | `mikopbx2026-docker_default`（172.18.0.0/16）：`mikopbx2026`=.2、`ai-bot`=.3、`baresip-agents`=.4，全部寫死（見下方「容器 IP 為什麼要固定」） |

## 開放 10.145.119.100（ZeroTier）註冊，不只 127.0.0.1（2026-07-13）

**原本**：SIP(5060/udp)、RTP(10000-10800/udp) 只綁 `127.0.0.1`，只有這台機器上的軟體
電話（MicroSIP）連得到，ZeroTier 網路上的其他機器連不進來。

**改法**：`D:\APP\mikopbx2026-docker\docker-compose.yml` 的 `ports:` 幫 SIP/RTP 各多加
一條綁 `10.145.119.100`（這台機器的 ZeroTier IP，見 `Get-NetIPAddress` 查出來的），
跟原本的 `127.0.0.1` 綁定並存（同一個 container port，兩個 host IP，compose 的
`ports:` list 本來就支援一對多）：
```yaml
ports:
  - "127.0.0.1:5060:5060/udp"
  - "127.0.0.1:10000-10800:10000-10800/udp"
  - "10.145.119.100:5060:5060/udp"
  - "10.145.119.100:10000-10800:10000-10800/udp"
```
改完要 `docker compose up -d`（port 對應變更需要 recreate 容器，單純 restart 不會套用）。

**⚠️ 關鍵限制，不是單純多開一個 port 就好**：Docker 對外發佈的 port，不管從哪個
host IP 連進來，容器內部看到的來源 IP **一律被 SNAT 成同一個 bridge gateway
（172.18.0.1）**——Asterisk 沒辦法分辨「這是本機用 127.0.0.1 連的」還是「這是別台機器
用 10.145.119.100 連的」，兩者在容器端長得一模一樣。這代表 PJSIP transport 的
`external_signaling_address`/`external_media_address`（回報給對方的 Contact/SDP 位址，
見 `voice-bot-mikopbx-ari` skill 坑 G）**沒辦法同時對兩種來源回報不同位址**，只能
二選一或找一個兩邊都通的答案。

**最後選擇把這個位址從 `127.0.0.1` 整個換成 `10.145.119.100`**（不是兩個都設、也不是
維持 127.0.0.1 不動）：`10.145.119.100` 是這台機器自己的真實網卡（ZeroTier 虛擬網卡），
本機的 MicroSIP 一樣連得到（連自己的真實 IP 沒問題，等於 hairpin），同時 ZeroTier
網路上的其他機器也連得到——一個位址同時滿足兩邊，比嘗試維持兩個位址並存簡單、也是
唯一真的可行的做法。相關的 `local_net`/`external_*` 設定改動在 voice-bot 的
`deploy/mikopbx_setup.py`（`PJSIP_TRANSPORT_NAT_APPEND`），不在這個 skill 管的
docker-compose.yml 範圍內，但兩邊是同一件事的兩半，缺一不可：這裡開 port，
那邊改 Asterisk 要回報的位址。

## 容器 IP 為什麼要固定（2026-07-13 踩過）

**這個 bridge network 的動態 IP 分配完全不穩定**——不只 `docker compose up -d --force-recreate`，連單純 `docker restart mikopbx2026` 都實測會讓 `mikopbx2026` 跟 `ai-bot` 的 IP互換。任何寫死內部 IP 的設定（例如 voice-bot 的 PJSIP NAT workaround `local_net`）都會在下次重啟時悄悄失效，症狀通常是「本來好好的突然又 32 秒斷線」或「佇列派線又不動了」，而且不會有任何錯誤訊息提示你 IP 變了。

**修法**：三個容器全部釘死固定 IP，一勞永逸：
- `mikopbx2026-docker/docker-compose.yml`：`networks.default.ipam.config.subnet` 指定
  `172.18.0.0/16`，`mikopbx2026` 服務加 `networks.default.ipv4_address: 172.18.0.2`。
- `voice-bot/docker-compose.yml`：`ai-bot` 服務的 `networks:` 從 list 形式改成
  mapping 形式，加 `mikopbx2026-docker_default.ipv4_address: 172.18.0.3`。
- `scripts/start_baresip_agents.ps1`：`docker run` 加 `--ip 172.18.0.4`。

**套用時的坑**：改完 compose 檔案的 network 定義（加了 `ipam.config`）後，Compose 會
判斷需要重建整個 network，但 `docker compose up -d` 沒辦法移除還有其他（非本
compose 專案管的）容器掛在上面的 network——`baresip-agents`（不是 compose 管的，用
`docker run` 啟動）跟 `ai-bot`（不同 compose 專案）都會擋住。**要先把兩者都
`docker rm -f` 掉，讓 network 上沒有任何 endpoint，才能重建成功**；重建完再依序用
各自的啟動方式（`docker compose up -d` / `scripts/start_baresip_agents.ps1`）把它們帶回來，
這次靠靜態 IP 保證它們會拿回原本的位址。設定資料都在 named volume（不在容器本身），
整個 rm+recreate 對 mikopbx2026 是安全的。

## 推 image 到 Docker Hub

用 `scripts/push_images_to_dockerhub.ps1`，不要每次重新想 tag/push 的組合：

```powershell
# 先登入一次（互動式，只能人工做）：
! docker login
.\scripts\scripts/push_images_to_dockerhub.ps1
```

推 3 個 image 到帳號 `hch590902`，統一前綴 `mikopbx2026-`（方便辨識這整套是同一個
專案的東西）：
- `hch590902/mikopbx2026-voice-bot-ai-bot`（voice-bot 的 ai-bot 自建 image）
- `hch590902/mikopbx2026-baresip-agents`（假客服測試容器）
- `hch590902/mikopbx2026-mikopbx`（官方 `mikopbx/mikopbx:latest` 重新 tag，方便版本
  釘選/備份，不是原創但一起收在同個命名空間下好找）

腳本冪等，image 內容沒變的話重跑只會顯示 `Layer already exists`，秒回。

## 容器管理

用 `scripts/manage_container.ps1`，不要每次重新想指令：

```powershell
.\scripts\scripts/manage_container.ps1 -Action status       # 檢查容器是否在跑
.\scripts\scripts/manage_container.ps1 -Action start         # docker compose up -d
.\scripts\scripts/manage_container.ps1 -Action restart       # force-recreate
.\scripts\scripts/manage_container.ps1 -Action logs          # 最近60行log
.\scripts\scripts/manage_container.ps1 -Action wait-ready    # 輪詢等 web UI 就緒（最多120秒）
```

## 開機自動啟動（2026-07-13 設定）

`mikopbx2026` / `baresip-agents` 的 restart policy 是 `no`（刻意不用 `unless-stopped`，
避免設定壞掉時無限重開），Windows 重開機後**不會**自動起來，需要手動 `docker start`。

已設定成全自動，不用再手動處理：
1. **Docker Desktop 本身**：`C:\Users\HCH\AppData\Roaming\Docker\settings-store.json` 的
   `AutoStart` 已設為 `true`（登入時自動啟動 Docker Desktop）。
2. **容器**：Windows 工作排程器任務 **`MikoPBX-AutoStart`**（觸發：使用者 `HCH` 登入時）
   執行 `scripts/startup_after_boot.ps1`——先輪詢等 `docker info` 就緒（最多 180 秒），
   再對 `mikopbx2026`/`baresip-agents` 執行 `docker start`（已在跑就略過）。
   - log：`%TEMP%\mikopbx-startup.log`
   - `ai-bot`（在 D:\Github\voice-bot）不需要處理：它的 restart policy 是
     `unless-stopped`，docker daemon 一起來就會自己重啟，連不上 `mikopbx2026` 時只會
     重試（已實測：`mikopbx2026` 晚起來也沒關係，ai-bot 重試到它就緒後會自動連上 ARI）。
   - 手動補跑：`powershell -File scripts\scripts/startup_after_boot.ps1`；查看/觸發排程任務：
     `Get-ScheduledTask -TaskName MikoPBX-AutoStart` / `Start-ScheduledTask -TaskName MikoPBX-AutoStart`。

**✅ IP 依賴警告已解除（2026-07-13）**：曾經是「動態 IP 可能因重建而改變」的風險，
現在三個容器（mikopbx2026=.2、ai-bot=.3、baresip-agents=.4）都已釘死固定 IP（見上方
「容器 IP 為什麼要固定」），voice-bot 的 SIP NAT 修正（`local_net=172.18.0.4/32`，見
`voice-bot-mikopbx-ari` skill 坑 G）不會再因為重啟/重建而失效。以下是舊版風險記錄
（保留供理解教訓，實際已不用擔心）：`docker start`（停止後啟動，不重建容器）不會換 IP，
已實測沒問題；但如果哪天用 `docker compose down` 再 `up`、或砍掉容器重建，IP 分配可能改變，NAT 修正就可能失效
（症狀：撥 205 又開始 32 秒被掛斷，或 bot 聽不到用戶）——出現這狀況先檢查
`docker exec mikopbx2026 asterisk -rx "pjsip show transport transport-udp"` 的
`local_net`/`external_*` 是否還對得上目前的 `docker network inspect mikopbx2026-docker_default` IP。

## 已踩過的坑

### 1. Port 對應千萬別改成 `18080:80` / `18443:443`

`docker-compose.yml` 裡的 `WEB_PORT`/`WEB_HTTPS_PORT` 環境變數**只在容器第一次初始化、`/storage` volume 是空的那一刻**生效，並被寫入持久化設定，之後永遠固定在那個值（這個實例是 **8080/8443**）。容器重開機後 nginx 一律監聽 8080/8443，跟環境變數當下寫什麼無關。

**正確設定**（已修正在 docker-compose.yml 裡，不要再改回 80/443）：
```yaml
ports:
  - "18080:8080"
  - "18443:8443"
```

若未來要砍掉 volume 重建全新實例，第一次啟動時的 `WEB_PORT`/`WEB_HTTPS_PORT` env var 才會決定這個「永久值」是多少。

### 1b. AMI port 5038 預設沒對外開放

容器內 Asterisk AMI 監聽 `0.0.0.0:5038`，但 docker-compose.yml 預設沒 publish 這個 port，host 連不到（連 container 內部 IP 也連不到，Docker Desktop on Windows 的網路隔離擋住）。已加上：
```yaml
ports:
  - "18080:8080"
  - "18443:8443"
  - "127.0.0.1:5038:5038"   # 只綁 loopback，不對外網開放
```
改完要 `docker compose up -d`（不是單純 restart，port 對應變更需要 recreate 容器）。

### 1c. 要讓 host 上的真實軟體電話（MicroSIP 等）打進來，SIP+RTP 也要開

跟 AMI 一樣的道理，SIP port 5060 預設沒 publish；還需要額外開 Asterisk 的 RTP port range（在容器內 `/etc/asterisk/rtp.conf` 查，這個實例是 `10000-10800`），否則 SIP 訊令能連但沒有雙向音訊。目前設定：
```yaml
ports:
  - "127.0.0.1:5060:5060/udp"
  - "127.0.0.1:10000-10800:10000-10800/udp"
```
`docker port mikopbx2026` 會把 800 個 RTP port 全部列出來，一行一行看很正常，不用擔心。

### 2. Module marketplace 頁面會卡住 5 個 file chooser 對話框

開啟 `http://127.0.0.1:18080/admin-cabinet/pbx-extension-modules/index/#/marketplace` 時，頁面 JS 會連續觸發 5 個原生 file chooser（不明原因，可能是某些模組圖示的 lazy-load bug）。用 Playwright 操作這頁時會卡住，`browser_take_screenshot` 等工具會回報 modal state 錯誤。

**解法**：用 `browser_file_upload` 呼叫 5 次、不帶 `paths`（等於依序取消每個 file chooser），畫面才會正常渲染：
```
browser_file_upload({}) × 5
```

### 3. 登入表單：用 Enter 送出可能導致 "Authorization error"

用 Playwright headless 腳本測試 admin/admin 登入時，`page.press('input[name="password"]', 'Enter')` 曾經導致 "Authorization error, you have 9 attempts left"，懷疑是繞過了頁面 JS 的送出邏輯。若要腳本化登入，改點擊 "Authorize" 按鈕而非按 Enter。

**更穩的替代方案**：直接用已開啟、有登入 session 的 Chrome（見 `connect-chrome` skill，port 9222 CDP），不必每次重新輸入帳密——這個 Chrome profile 對這台 PBX 已經是登入狀態。

### 4. API keys 頁面「Save」按鈕靠 `.fill()` 填欄位會一直是 disabled

在 `/admin-cabinet/api-keys/modify/{id}` 編輯頁，用 Playwright `.fill()` 改 Description 欄位後，`#submitbutton` 仍然是 `disabled` class（React 表單的 dirty-state 沒被觸發）。

**解法**：填完後用 `.type(' ')` 再 `.press('Backspace')` 補一次真實鍵盤事件，`disabled` class 才會消失：
```js
const field = page.getByRole('textbox', { name: 'e.g., CRM system integration' });
await field.fill('...');
await field.press('End');
await field.type(' ');
await field.press('Backspace');
```

## REST API 使用方式

認證：`Authorization: Bearer <api-key>`，key 在 UI「System → API keys」建立（`/admin-cabinet/api-keys/index/`）。

```bash
KEY="9064b2498e369c884e1eb4e3484c7f1d45c1c88076ac407260935fb595e93deb"
BASE="http://127.0.0.1:18080/pbxcore/api/v3"
curl -s -H "Authorization: Bearer $KEY" "$BASE/extensions"
```

**抓完整 OpenAPI 規格 + 依分類列出全部 262 個 endpoint**：
```bash
node scripts/fetch_openapi_summary.js [apiKey] [baseUrl]
```
（不帶參數會用上表的預設 key/baseUrl）。輸出涵蓋 45 個分類：Extensions、Call queues、IVR menu、SIP/IAX providers、CDR、Firewall、System operations（含 `executeBashCommand`/`executeSqlRequest`，權限等同 root，妥善保管 key）等。

## CTI / ACD 開發現況

- **AMI 帳號 `1cami` 已經預先建立**（原廠內建，非我們新增），描述是「The user for integration into the MIKO Panel」，讀寫權限涵蓋 `call, originate, reporting`——這是 MikoPBX 原廠設計給外部 CTI 面板串接用的帳號，做 ACD/CTI 開發直接沿用最快，不用重新申請。密碼見上方快速參考表。
- **ARI users 目前是空的**（0 筆）——如果要走 ARI（Stasis app，完全自訂通話路由）而非 AMI，需要先建立 ARI 使用者。
- **已有 2 個 Call Queue，成員已掛好**（透過 REST API PATCH `members: [{"extension":"201"}]` 這種格式設定，`CallQueueMember` schema 只吃 `extension` 欄位，`represent` 是唯讀）：
  - `2001` Sales office（strategy: ringall）→ members: 201 (Smith James), 202 (Brown Brandon)
  - `2002` Technical Support Department（strategy: random）→ members: 203 (Collins Melanie)
- **201/202/203 已經有自動應答的軟體電話註冊**（見下方 baresip-agents），MikoPBX `sip:getStatuses` 會顯示三支都是 `Available`，AMI `QueueStatus` 的 member `Status` 不再是 5（Unavailable）。撥進 queue 會真的振鈴、被自動接聽、SIP 訊令層完整走完 ANSWER。

### 自動應答測試坐席：baresip-agents（`baresip-agents/` 目錄）

用 [baresip](https://github.com/baresip/baresip)（透過 OPAL/PTLib 之外的另一套跨平台 CLI 軟電話，比 sipcmd2 更適合這個場景，見對話紀錄的工具比較）跑一個 Docker 容器，同時註冊 201/202/203 三支分機、全部設 `answermode=auto` 自動接聽，掛在跟 `mikopbx2026` **同一個 Docker network**（`mikopbx2026-docker_default`）上，不需要額外對外開 SIP port。

**啟動**（會自動 build image，已存在就跳過）：
```powershell
.\scripts\scripts/start_baresip_agents.ps1
```

**關鍵設定重點**（都已經寫進 `baresip-agents/config` / `accounts`，不用重新摸索）：
- `module_path /usr/lib/baresip/modules`：debian 套件安裝的 baresip 預設用相對路徑找模組（`./g711.so`），不加這行全部載入失敗
- `audio_player`/`audio_source` 用 `aufile,/root/.baresip/tone.wav`（8kHz mono 16bit PCM，60秒 440Hz 音調，`tone.wav` 就在 `baresip-agents/` 目錄下，一起掛進 volume）。**已修復音訊問題，見下方**。
- 三個帳號密碼是各分機的 SIP secret，透過 REST API `GET /sip/{id}:getSecret` 取得（會隨 MikoPBX 重灌而改變，`accounts` 檔案裡的密碼跟目前這個 docker volume 綁定）。
- **Windows 上用 Git Bash 跑 `docker run -v /root/.baresip:...` 會被 MSYS 路徑轉換搞爛**（容器內路徑被誤轉成 Windows 路徑），要嘛加 `MSYS_NO_PATHCONV=1` 前綴，要嘛（更穩）直接用 PowerShell 執行 `scripts/start_baresip_agents.ps1`。

**清掉殘留測試通話**：baresip 只負責自動接聽，沒有自動掛斷邏輯，示範完記得用 AMI `Hangup` 清掉，見下方 demo 流程。

**✅ 已修復：`ausine` 模組只支援 48kHz，跟 8kHz codec（pcmu/pcma）衝突，答應後立刻斷線** —— 曾經是這個環境最大的坑：baresip 在 SIP 層答應了（`call: answering call ... with 200`），但 `audio: start_source failed (ausine.-): Operation not supported` 讓通話立刻在自己內部斷開，Asterisk 那邊看到的是「答應了又馬上斷」，佇列判定 `NOANSWER` 並不斷重新振鈴（`PJSIP/201-xxxx` channel 編號一直往上跳，子 channel 怎麼 Hangup 都清不完，要找最上層來源 channel 砍才斷得乾淨）。

**修法**：把 `audio_player`/`audio_source` 從 `ausine,-` 換成 `aufile,/root/.baresip/tone.wav`（一個實際的 8kHz wav 檔案，取樣率直接匹配協商出來的 codec，不用 resample），`module` 那行也對應從 `ausine.so` 換成 `aufile.so`。換完之後，MicroSIP(204) 撥打 Sales queue(2001) 觸發 ringall，201 真的完整接聽並穩定通話 31 秒（`disposition: ANSWERED`），不再秒斷。要重新產生 `tone.wav`（例如想換頻率/長度）：
```bash
python -c "
import wave, struct, math
sr, duration, freq = 8000, 60, 440
with wave.open('baresip-agents/tone.wav', 'w') as w:
    w.setnchannels(1); w.setsampwidth(2); w.setframerate(sr)
    for i in range(sr*duration):
        w.writeframes(struct.pack('<h', int(3000*math.sin(2*math.pi*freq*i/sr))))
"
```

### 用 ctrl_tcp 讓 baresip 主動撥號（不只被動應答）

baresip 不只能自動接聽，也能主動撥號——透過 `ctrl_tcp` 模組（port 4444），協定是 **JSON-RPC over netstring**（不是純文字指令）：
```js
const net = require("net");
const msg = JSON.stringify({command:"dial", params:"10003246@mikopbx2026"});
const netstring = Buffer.byteLength(msg) + ":" + msg + ",";
const sock = net.createConnection(4444, "127.0.0.1", () => sock.write(netstring));
sock.on("data", c => console.log(c.toString()));
```
會用 accounts 檔案裡**第一個帳號**（201）發起撥號。回應格式：`{"response":true,"ok":true,"data":""}`，同時會收到一串 `{"event":true,"type":"CALL_LOCAL_SDP",...}` 這類事件推播。容器內建了 `nc`/`socat`（見 Dockerfile），也可以在 `docker exec` 進容器後用這兩個工具手動測。

**要指定用哪一支帳號撥號**（不是永遠用 201）：ctrl_tcp 的 `dial` 一律用「目前選定的
UA」，要先送 `uafind` 切換，且**參數要給完整 AOR（`sip:202@mikopbx2026`），純數字
`202` 選不到**（會悄悄還是用 201，不會報錯，很難發現）。已包成腳本
`scripts/dial_agents_to_queue.js`（用於 voice-bot 的 ACD 併發測試，見
`voice-bot-mikopbx-ari` skill 坑 I）：
```bash
node scripts/dial_agents_to_queue.js <目標分機> [201,202,203]   # 預設三支都打
node scripts/dial_agents_to_queue.js 200 202,203                # 只讓 202/203 打 200
node scripts/dial_agents_to_queue.js hangup                      # 逐支切換並掛斷
```

**⏳ 待查現象**：baresip 主動撥出的電話目前會固定在約 32 秒後自己斷線（不是被叫方
掛的），已重現兩次，根因未查（詳見 `voice-bot-mikopbx-ari` skill 坑 K）——被動接聽
（真人打 201/202/203）沒有這個問題，只有 baresip 當 caller 主動撥出這個方向會斷。

### 已驗證可行的 demo 流程（`scripts/ami_demo.js`）

```bash
node scripts/ami_demo.js queue-status        # 連 AMI，列出兩個 queue 目前的 member 即時狀態
node scripts/ami_demo.js originate 2001      # 模擬一通電話撥進 2001，即時印出完整 AMI 事件流（Newchannel→Dial→Queue→MusicOnHold...）
```

`originate` 模式會用 `Local/10003246@internal`（Echo test 分機）當模擬來源。**先啟動 baresip-agents 再跑這個 demo**，效果差很多：
- 沒啟動 baresip-agents 時：卡在 MusicOnHold 一直等（Sales queue 沒設 timeout redirect），因為沒有 member 可振鈴
- 啟動後：ringall 策略會同時振鈴 201 和 202，事件流會看到兩邊都 `Newchannel`/`DialBegin`，其中一支 `DialStatus: ANSWER`，另一支變成 `NOANSWER`/`CANCEL`——**這是 ringall 的正常行為**（同時振鈴、搶先接的贏，另一支被取消），不是 bug

**腳本跑完 20 秒後不會自動掛斷**（baresip 只負責自動接聽，沒有自動掛斷邏輯），需要手動清理殘留通話：
```bash
# 查目前活躍通話（這個才是 ground truth，比 getActiveCalls 準）
curl -s -H "Authorization: Bearer $KEY" "$BASE/pbx-status:getActiveChannels"
# 用 AMI Hangup 清掉（Channel 名稱從上面的事件流或 getActiveChannels 取得，通常要連 Local channel 那端跟 PJSIP/xxx 那端都各 Hangup 一次）
```
注意：`getActiveCalls`（CDR-based）跟 `getActiveChannels`（Asterisk 即時 channel）在用 Local channel 模擬的測試場景下可能不同步，`getActiveCalls` 可能會留一筆 `endtime` 空白的殘影記錄（無害，只是顯示層，不影響系統）。

**容器重開機注意**：如果 `mikopbx2026` 容器被 `docker compose up -d --force-recreate` 重建過，Docker network 也可能跟著變動，這時 `baresip-agents` 需要重新執行 `scripts/start_baresip_agents.ps1`（腳本裡有 `docker rm -f` 起手式，重跑安全）才能確保還在同一個 network 上、SIP 註冊還有效。

### AMI vs ARI 該選哪個

| | AMI | ARI |
|---|---|---|
| 適合 | 標準 ACD 面板：坐席登入/小休、來電彈窗、Click2Dial、報表——直接搭配內建 Queue | 需要超出 Asterisk queue strategy 能表達的客製路由邏輯 |
| 起手式 | 直接用 `1cami` 帳號連 port 5038（見 `scripts/ami_demo.js`） | 先建 ARI user，走 Stasis app 完全接管通話 |

多數 ACD 需求（坐席狀態 + 彈窗 + Click2Dial + 報表）建議先走 **AMI + 內建 Queue**，成本最低。

## 相關 Skill

- `mikopbx`：另一台**真實**環境（10.145.119.64:8080），跟這個本機 Docker 沙盒是不同機器，不要混用登入資訊或 API key。
- `connect-chrome`：接管已開啟的 Chrome（port 9222）操作這台 PBX 的 Web UI，保留既有登入 session。

---

## Conformance Addendum

## When to Use
本機用 Docker 跑的 MikoPBX 2026 沙盒實例（D:\APP\mikopbx2026-docker）——啟動/檢查容器、Web UI 登入、REST API 呼叫（含 OpenAPI 規格抓取）、CTI/ACD 開發現況（AMI 帳號、Call Queue 設定）。當使用者提到「mikopbx」「mikopbx2026-docker」「這個 PBX 容器」或要用這台機器測 REST API/CTI/ACD 時使用。跟 `mikopbx` skill（10.145.119.64 那台真實環境）是不同機器，不要混用。

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
