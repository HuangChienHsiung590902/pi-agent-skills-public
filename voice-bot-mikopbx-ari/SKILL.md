---
name: voice-bot-mikopbx-ari
description: 把 D:\Github\voice-bot 的 Gemini 語音 bot 透過 Asterisk ARI + External Media 接進 mikopbx2026-docker，讓真人打分機 205-210 或 ACD 代表號 200 就由 AI 接聽。含完整部署流程與所有踩過的坑（音訊雜音、被自動掛斷、佇列派線不動等）的根因與修法。
---

# voice-bot × MikoPBX ARI 整合（AI 語音客服）

把 `D:\Github\voice-bot`（原本只吃本機麥克風的 Gemini Multimodal Live bot）
接進 `mikopbx2026-docker` 電話系統：真人用 SIP 話機撥打**分機 205-210**（或撥
**ACD 代表號 200** 讓系統自動分配到任一空線），Asterisk 透過 ARI 把通話交給
Stasis app `ai-bot`，建立 External Media（RTP）把音訊串到 Python bot，由 Gemini
即時對話。全部跑在 Docker。

**分機號碼配置**（2026-07-13 定案）：
- `201`-`203`：baresip-agents 模擬的假客服測試機（不動，見 `mikopbx2026-docker` skill）
- `204`：使用者自己的 MicroSIP 測試軟體電話（不動）
- `205`-`210`：AI bot 專用分機，各自獨立可直撥，也是 ACD 佇列成員
- `200`：ACD 佇列代表號（`QUEUE-5E3FB45D`，strategy=`leastrecent`），來電自動分配給 205-210 任一空線
- `2001`/`2002`：MikoPBX 內建示範佇列（Sales/Support，成員 201-203，跟 AI bot 無關）

## 架構

```
MicroSIP(204) ──SIP──► MikoPBX(mikopbx2026) ──dialplan [all_peers-custom]
                                              或 Queue(200)→[internal-users-custom]──►
  Stasis(ai-bot) ──ARI WebSocket 事件──► ai-bot 容器(ari_service.py)
  ai-bot 用 ARI 建 mixing bridge + externalMedia(UnicastRTP, 8kHz slin)
  ──RTP(UDP)──► rtp_transport.py ──► pipecat pipeline ──► Gemini Live API
```

- 兩個容器同在 `mikopbx2026-docker_default`（external network）：`ai-bot`
  用容器名 `mikopbx2026:8088` 連 ARI，不對外開 port。
- 音訊格式固定 **8kHz slin，payload type 11**（PT11 是 ASTERISK-28751 的 workaround，
  16kHz slin16 送回 Asterisk 會被靜默丟棄）。
- `ari_service.py` 的 `_handle_stasis_start` 完全不綁分機號碼（只認 Stasis channel
  id），所以從 1 條線擴到 6 條線 + ACD 佇列**完全不用改 application code**，只需要
  在 MikoPBX 那層多開分機、多寫一份 dialplan hook、建一個 Queue。

## 關鍵檔案（都在 D:\Github\voice-bot）

| 檔案 | 職責 |
|---|---|
| `ari_service.py` | ARI 事件迴圈；StasisStart 時建 bridge+externalMedia、answer、跑 `run_call`；StasisEnd 時取消 task |
| `bot.py` | `run_call(rtp_port, call_id)`：一通電話的 pipecat + Gemini pipeline |
| `rtp_transport.py` | pipecat 自訂 transport：UDP 收發 RTP、8k↔16k/24k resample、**網路位元組序轉換**、**20ms 配速** |
| `rtp_packet.py` | RTP 封包 pack/unpack（12-byte header + slin payload） |
| `port_allocator.py` | 每通電話分配一個 40000-40100 的 UDP port |
| `deploy/mikopbx_setup.py` | 一次性設定 MikoPBX（建 205-210、載模組、開 ARI、建 dialplan hook、建 ACD 佇列 200） |
| `Dockerfile` / `docker-compose.yml` | ai-bot 容器（CPU-only torch） |

## 部署流程

```bash
# 1. 一次性設定 MikoPBX（見下方「MikoPBX 設定的 5 個坑」，已全部編進腳本）
cd D:/Github/voice-bot
MIKOPBX_API_KEY=<見 mikopbx2026-docker skill 的 key> \
MIKOPBX_ARI_PASSWORD=<密碼> \
  .venv/Scripts/python.exe deploy/mikopbx_setup.py
docker restart mikopbx2026   # 套用 modules.conf/ari.conf/dialplan 變更

# 2. .env（不進 git）：GEMINI_API_KEY=... / MIKOPBX_ARI_PASSWORD=...

# 3. 起 ai-bot 容器
docker compose up -d --build
docker logs ai-bot --tail 5    # 應看到 "Connected to ARI events for app=ai-bot"

# 4. 用 SIP 話機撥 205（或撥 200 讓 ACD 佇列自動分配到 205-210 任一空線）
```

`create_extensions()` 會先查現有分機列表、跳過已存在的號碼，所以整個腳本可以安全
重跑來補開新分機（例如從 205 擴到 205-210）；`create_acd_queue()` 是 POST（新建），
重跑會建出第二個佇列，要新增/調整佇列成員請直接用 REST API PUT
`call-queues/{id}`（見坑 I）。

改了 `ari_service.py`/`bot.py`/`rtp_transport.py` 後要 `docker compose up -d --build`
重建才會生效（COPY 進 image）。改完先 `python -m py_compile` + `pytest tests/`（16 個）。

---

## 已踩過的坑（按查修順序，這是本 skill 最有價值的部分）

### A. MikoPBX 設定的坑（已編進 deploy/mikopbx_setup.py）

1. **`ari.conf` 的 `enabled=yes` 不夠**：這個 build 的 `modules.conf` 是
   `autoload=no` 白名單，`res_ari*`/`res_stasis*`/`res_websocket_client` 都沒載 →
   ARI 路由 404。要手動 `load =>` 這些模組。
2. **`asterisk-rest-users` REST API 不會產生 `ari.conf` 的 `[username]` 段** →
   401。要直接把 `[ai-bot]` user 段寫進 `ari.conf`。
3. **`ari.conf` 是 sorcery 解析，重複 `[general]` 會整檔拒載** → 要用
   `mode="override"` 寫一份完整自足的檔，不能 `append`。
4. **撥沒註冊裝置的號碼（如 205）走 `[all_peers]` 不是 `[internal]`**（AMI trace
   確認 `Context=all_peers`）→ Stasis hook 要放 `[all_peers-custom]`。
5. **External Media 的 "UnicastRTP" channel 需要 `chan_rtp.so`**（外加依賴
   `res_rtp_multicast.so`），跟 ARI/Stasis 模組是分開的。

### B. ari_service.py：answer 的順序 → 避免 SIP UPDATE

**症狀**：早期版本在建 bridge **之前**就 `answer` caller channel，Asterisk 送出
200 OK 後隔幾十毫秒又送一個 UPDATE（因為 external media leg 較晚才接上、SDP 要補），
部分 SIP client 的對話狀態機會被搞混。
**修法**：把 `POST /channels/{id}/answer` 移到 **bridge + externalMedia + addChannel
都完成之後**才呼叫，這樣只會有一次乾淨的 200 OK、不需要後續 UPDATE。

（注意：完全不 answer 的話 caller 會永遠卡在 `Ring`、RTP 不會流動——answer 是必須的，
只是順序要對。）

### C. ari_service.py：addChannel 422 "Channel not in Stasis application" → 重試

**症狀**：剛建好的 externalMedia channel，Asterisk 內部把它註冊進 Stasis 是
**非同步**的，`POST /channels/externalMedia` 回應可能早於註冊完成，馬上 addChannel
會 422。
**修法**：`_add_channel_to_bridge()` 對 422 做指數退避重試（50ms 起，最多 8 次）。

### D. rtp_transport.py：位元組順序 → **這是「全部都是高頻雜音」的根因**

**症狀**：音訊有雙向流動、頻譜也像語音，但聽起來全是高頻雜音。
**根因**：RFC 3551 的 L16 payload 是**網路位元組序（big-endian）**，但我們原本用
主機的 little-endian 打包/解讀，每個 16-bit 樣本高低位元組顛倒 → 變噪音。
**修法**：收（`_handle_packet`）用 `np.frombuffer(payload, dtype=">i2")`；
送（`write_audio_frame`）用 `pcm_8k.astype(">i2")` 再 `tobytes()`。

### E. rtp_transport.py：送話沒配速 → 破碎/雜音

**症狀**：pipecat 會把一整段 TTS 一次丟給 `write_audio_frame`，原本 for 迴圈把好幾秒
的封包瞬間全灌進 UDP socket，Asterisk 收到的是爆量而非即時串流 → 播放破碎。
**修法**：用 `loop.time()` 維持 `_next_packet_due`，每個 20ms 封包之間
`await asyncio.sleep()` 對齊真實時間節奏。

### F. bot.py：Gemini 聽不到用戶、bot 不回應 → turn-taking 設定

**症狀**：bot 講完開場白後，用戶說話它沒反應。log 有
`GeminiLiveLLMService is not emitting turn frames` warning。
**根因**：realtime service（Gemini Live）預設用伺服器端 VAD，但 pipecat 的
realtime-mode 會把預設的 turn-start 策略丟掉又沒補回來，用戶語音永遠不被判定為一個 turn。
**修法**（依 pipecat 官方文件）：
- `GeminiLiveLLMService.Settings(vad=GeminiVADParams(disabled=True))` 關伺服器端 VAD
- `LLMContextAggregatorPair(context, realtime_service_mode=True, user_params=
  LLMUserAggregatorParams(vad_analyzer=SileroVADAnalyzer(), ...))` 用本地 VAD 驅動 turn
- `SileroVADAnalyzer` 需要 `onnxruntime`（pipecat-ai 基礎依賴已含）。

### G. ✅ 已解：SIP NAT → 約 32 秒被自動掛斷（+ bot 聽不到用戶）

**⚠️ 2026-07-13 更新，數字已變**：下面這節記錄的是最初發現這個問題時的真實數值
（`127.0.0.1` / `local_net=172.18.0.2/31`），原理完全沒變，但**現在的實際值不一樣了**：
- `external_signaling_address`/`external_media_address` 現在是 **`10.145.119.100`**
  （這台機器的 ZeroTier IP），不是 `127.0.0.1`——因為後來也開放讓 ZeroTier 網路上的其他
  機器連進來，docker-proxy 對這兩種來源做的 SNAT 都會變成同一個 gateway `172.18.0.1`，
  Asterisk 分不出誰是誰，只能對外統一回報同一個位址，`10.145.119.100` 本機也連得到，
  所以能同時滿足兩邊。
- `local_net` 現在是 **`172.18.0.4/32`**（只含 baresip，不用再算 Asterisk 自己）。
- 更根本的是：**這三個容器（mikopbx2026/ai-bot/baresip-agents）現在都已經釘死固定
  IP**（`.2`/`.3`/`.4`，見 `mikopbx2026-docker` skill「容器 IP 為什麼要固定」），因為
  實測連單純 `docker restart` 都會讓 IP 互換，光靠「知道現在是哪個 IP」不夠，要從根本
  讓它不再變動。

**症狀**：音訊（bot→用戶）正常，但通話固定約 32 秒後**被系統掛斷**（不是使用者掛的），
AMI 顯示 `Hangup Cause=18 (No user responding) TechCause=408`；且 bot 聽不太到用戶說話。

**根因（tcpdump 確認）**：MicroSIP 在 Windows 主機，透過 Docker 的 `127.0.0.1:5060`
loopback port-forward 連 PBX，經 docker-proxy SNAT 後在容器端來源變成 gateway
`172.18.0.1`。Asterisk（bind `0.0.0.0`）預設把 **Contact 與 SDP 媒體位址都填容器內部 IP
`172.18.0.2`**，但這個位址**從 Windows 主機不可達**（`ping 172.18.0.2` 100% loss）。於是：
- 200 OK 的 `Contact: <sip:172.18.0.2:5060>` → 依 RFC 3261 對 2xx 的 ACK 要送到 Contact
  → ACK 送不到 → Asterisk 一直重傳 200 OK → **32 秒 SIP Timer H 逾時自己拆線**。
- SDP `c=IN IP4 172.18.0.2` → 用戶語音的 RTP 送不到 Asterisk → **bot 聽不到用戶**。

（baresip 201/202/203 沒事：它們在 Docker 網路**內部**，連得到 172.18.0.2。）

**踩過的死路**：把 MicroSIP `[Account1]` 的 `proxy=` 設 `127.0.0.1`（outbound proxy）。
outbound proxy 只對「對話外」請求預載 route（401 的 ACK 確實帶了 `Route: <sip:127.0.0.1;lr>`），
但 2xx 的 ACK 走**對話 route set**（由 200 OK 的 Record-Route 建立），Asterisk 當 UAS
不會 Record-Route → 對話 route set 空 → ACK 仍直送 Contact 172.18.0.2 → **沒用**。
（這條 proxy 設定留著無害，但不是解法。）

**正解（伺服器端）**：在 PJSIP 的 `transport-udp` 加 3 行，讓 Asterisk 對「外部」peer
廣播 MicroSIP 連得到的 `127.0.0.1`，對「內部」peer（baresip）維持容器內部位址：
```
[transport-udp]
...
local_net=172.18.0.2/31            ; 含容器 .2 與 baresip .3，排除 gateway .1（MicroSIP 來源）
external_signaling_address=127.0.0.1  ; 對外部 peer 的 Contact/Via 位址
external_media_address=127.0.0.1      ; 對外部 peer 的 SDP 媒體位址
```
套用後驗證：`pjsip show transport transport-udp` 三欄有值；撥 205 後 tcpdump 看到
`Contact: <sip:127.0.0.1:5060>` + `c=IN IP4 127.0.0.1` + 用戶的 ACK 有到、200 OK 不再
重傳、Asterisk 不再送 BYE；baresip 仍是 `172.18.0.3`（不受影響）。**32 秒掛斷解決。**

**為何 local_net 是 `172.18.0.2/31`**：`/31` = .2–.3 兩個位址，剛好涵蓋 Asterisk 自己(.2)
與唯一的內部 SIP peer baresip(.3)，排除 MicroSIP 的 SNAT 來源 gateway(.1)。docker-proxy
會把所有外部流量 SNAT 成 gateway，所以「外部 client」在容器端一律是 .1——這是能區分內外的
唯一線索。若未來內部 peer 拿到 .4+ 要擴大 local_net。

**✅ 持久化（已完成）**：透過 MikoPBX custom-files（pjsip.conf = **id 45**）以 `mode="append"`
附加，並用 Asterisk 的 **`[transport-udp](+)`** 合併語法（`(+)` = 併入既有物件；直接寫第二個
`[transport-udp]` 會被 sorcery 當「duplicate object」拒絕，跟 ari.conf 同款）：
```
[transport-udp](+)
local_net=172.18.0.2/31
external_signaling_address=127.0.0.1
external_media_address=127.0.0.1
```
已編進 `deploy/mikopbx_setup.py` 的 `apply_sip_nat_workaround()`（`main()` 會呼叫，設
`SKIP_SIP_NAT=1` 可在正式部署跳過）。**已實測 `docker restart mikopbx2026` 後 transport 自動
帶回三個設定、baresip/MicroSIP 正常註冊。**

**為何不用 MikoPBX 原生「外部主機/NAT」設定**：它的 `local_net` 是從網卡拓樸自動產生**整個
LAN 網段**（172.18.0.0/16），會把 MicroSIP 的 SNAT 來源 gateway(.1) 也算進 local → 破壞修法。
只有 custom-files 的 `(+)` 能精準指定 `local_net=172.18.0.2/31`。

（`scripts/apply_sip_nat_fix.sh` 是「尚未跑過 deploy 持久化、想立刻 live 套用」時的快捷工具；
持久化做好後正常情況不需要它。）

**本質**：這是「Asterisk 在 Docker-Desktop loopback NAT 後」的**測試拓樸產物**；正式部署
（MikoPBX 有真實可達 IP）不會發生，也就不需要這組 external_*/local_net 設定。

### H. bot 聽不到用戶、不回應 → 先驗證是不是 client 送靜音（別急著怪程式）

**症狀**：bot 講完開場白後，用戶說話 bot 沒反應。log 沒有 `👤 用戶:` 或內容是 None。
（**注意先排除坑 F/G**：F 是 turn-taking 沒設 start 策略、G 是媒體位址不可達；兩者修好後
才輪到這個。）

**關鍵教訓：先確認「用戶的聲音有沒有真的以非靜音送達 bot」，再往上游查。**
不要被表象誤導——曾發生：MicroSIP 麥克風位準列看起來有跳（其實那是**音量滑桿填色**，不是即時
位準表），但實際送出的是靜音。

**診斷法（objective，已寫成腳本）**：
1. bot 端 debug 擷取（`RTP_DEBUG_CAPTURE=1`）錄到的 `/tmp/rtp_rx.raw` 用 `scripts/analyze_rtp.py`
   看——但**注意它只錄前 5 秒**，而 bot 開場白約 8-10 秒，前 5 秒剛好是「用戶在聽、還沒開口」的
   靜音段，會誤判。要嘛加大擷取上限，要嘛改用下一步。
2. **在 Asterisk 端抓 caller-leg RTP、A-law 解碼看 RMS**（最可靠）：
   ```bash
   MSYS_NO_PATHCONV=1 docker exec -d mikopbx2026 sh -c \
     "timeout 7 tcpdump -i any -n 'udp portrange 10000-10800' -w /tmp/rtp_live.pcap"
   # <撥 205，等開場白結束後持續講話 6 秒>
   MSYS_NO_PATHCONV=1 docker cp mikopbx2026:/tmp/rtp_live.pcap <本機>
   python scripts/decode_caller_rtp.py <本機>/rtp_live.pcap   # 自動選埠、判定 SILENCE / 有語音
   ```
   RMS≈8、只有 2 種值 = **client 送的是靜音** → 問題在 client 端麥克風，Asterisk/bot 全是好的。
3. **逐支麥克風錄音比對**（確認是哪支、是不是選錯/靜音）：
   ```bash
   bash scripts/test_mics.sh     # ffmpeg 逐一錄每支啟用中的麥克風，回報哪支收得到聲音
   ```

**實際根因（2026-07-13 本案）**：用戶戴 Jabra EVOLVE 30 II 耳麥，MicroSIP 用「預設」錄音裝置
＝Jabra（headset 常被 Windows 設為預設通訊裝置），但 **Jabra 麥克風桿朝上=自動靜音**，收到的是
靜音；用戶的聲音其實只有內建 Intel 麥克風排列收得到（RMS 326 vs Jabra 3.3）。
**修法**：把 Jabra 桿子轉到嘴邊解除靜音，或在 MicroSIP Settings 把錄音裝置改成內建麥克風排列。
（MicroSIP.ini `audioInputDevice=""` = 系統預設；改成明確裝置可避免抓到靜音的那支。）

### I. ✅ 已解：擴到多線（205-210）+ ACD 佇列（200）

**目標**：從單一分機 205 擴成 6 條 AI bot 線（205-210），加一個 ACD 佇列代表號 200，
來電自動分配給任一空線。`ari_service.py` 完全不用改（見架構節說明），但 MikoPBX
這層踩了兩個大坑：

**坑 I-1：`extensions.conf` custom-files 誤用 `mode="override"` 差點砍光整個 PBX
dialplan。** ari.conf 用 override 是對的（MikoPBX 生成的 ari.conf 本來就很小、
自成一體），但 extensions.conf **不一樣**——MikoPBX 會先生成一份 700+ 行的完整
base dialplan（`[internal]`、`[globals]`、每個員工的 `Goto()`、兩個示範佇列的
`Queue()` 呼叫……），原本用 `mode="append"` 接在後面。改成 override 會**整個蓋掉
那 700+ 行**，只剩自己寫的內容（親身驗證：檔案從 707 行掉到 49 行，`[internal]`
context 直接消失，201-204、2001/2002 全部斷線）。
**教訓**：custom-files 是不是能用 override，要看 MikoPBX 對那個檔案的生成邏輯是
「一份小的自足檔」還是「一份大的自動生成 base + 使用者附加內容」——不確定就先
`GET /custom-files/{id}` 看目前的 `mode` 欄位，維持原 mode 最安全。**壞掉後的救法**：
把 mode 改回 append、重新 PUT 內容、`docker restart mikopbx2026`，base dialplan
會在下次啟動時重新生成。

**坑 I-2：佇列成員（Queue member）撥號路徑跟直撥不一樣，需要多一個 dialplan
context + 手動校正裝置狀態。**
- 直撥分機（如 MicroSIP 打 205）：MikoPBX 的 PJSIP endpoint 都設
  `context=all_peers`，所以外部來電落在 `[all_peers-custom]`（見坑 A.4）。
- Queue 的成員撥號用 `Local/<ext>@internal/n`（`queue show`/`queues.conf` 都看得到
  這個格式），這個 Local channel 的第二腳走 `[internal]` context，而 `[internal]`
  對每個員工只是 `Goto(internal-users,<ext>,1)`——真正檢查
  `${CONTEXT}-custom`（GosubIf）的地方在 **`internal-users` context 的 priority
  15**，此時 `${CONTEXT}` 是 `internal-users`，要找的是 `internal-users-custom`，
  不是 `internal-custom`（原本以為的「順便留一份」完全沒被用到，是條死路）。
  **沒有這個 hook 時的症狀**：MikoPBX 幫每個員工自動建了一個「沒人註冊」的
  PJSIP endpoint（哪怕只是純 Stasis 分機），`internal-users` 的 dialplan 靠
  `PJSIP_ENDPOINT(${EXTEN},auth)` 判斷「這個分機有沒有裝置」——因為 endpoint
  客觀存在（只是沒 contact），這個判斷是「有」，於是**不會**走去
  `internal-num-undefined`（那條路才會查 `all_peers-custom` 之類的替代路徑），
  而是繼續往下試 `Dial()`，因為沒有註冊的 contact 直接 `CHANUNAVAIL`，整通電話
  安靜地失敗——caller 端聽起來就是「一直響、沒人接」。
- MikoPBX 也會幫每個分機自動產生
  `hint:<ext>@internal-hints = PJSIP/<ext>&PJSIP/<ext>-WS&PJSIP/<ext>-TLS&Custom:<ext>`。
  因為 PJSIP 部分永遠是 unregistered endpoint，組合出來的 device state 永遠是
  **Unavailable**（`queue show` 會看到成員狀態是紅字 `Unavailable`），而
  **Asterisk 的 Queue app 在成員狀態 Unavailable 時根本不會嘗試撥打它**——
  這才是「打 200 永遠沒人接」的真正根因（比坑本身更隱蔽：dialplan hook 沒補齊
  只是必要條件之一，devstate 沒校正一樣打不通）。

**修法（兩步都要）**：
1. `deploy/mikopbx_setup.py` 的 `DIALPLAN_HOOK` 現在會同時寫進三個 context：
   `[all_peers-custom]`（直撥）、`[internal-users-custom]`（**佇列必需**）、
   `[internal-custom]`（保留，無害但目前沒被用到的路徑）。
2. 手動把每個 AI bot 分機的 Custom device state 校正成可用：
   ```bash
   docker exec mikopbx2026 asterisk -rx "devstate change Custom:205 NOT_INUSE"
   # ... 206, 207, 208, 209, 210 依此類推
   ```
   `queue show` 應該從紅字 `Unavailable` 變綠字 `Not in use`。**已實測這組
   Custom device state 會存在 astdb、`docker restart mikopbx2026` 後自動留著**
   （不像 extensions.conf 是每次開機從 DB 重新生成），所以正常情況下設一次就好；
   如果哪天重開機後佇列又不動了，先查 `devstate list` 有沒有掉、沒有的話重跑上面
   的指令即可。

**驗證方式**：`queue show` 看 6 個成員都是綠字 `Not in use`；實際撥 200，
`core show channels` 應該看到 `Local/<ext>@internal;2` 進了
`Stasis(ai-bot,<ext>)` 而不是卡在 `internal-users` Ring 不動；`docker logs ai-bot`
應該看到 `incoming call on Local/<ext>@internal-...;2`。AMI Originate 模擬
（`node ami_demo.js originate 200`）看不出真實結果——它是拿 Local channel 假裝
來電源頭，訊令路徑跟真實 SIP 來電差太多，這次直接被誤導了，**佇列這種東西一定要
拿真實話機（MicroSIP）實際撥打驗證，不能只信 AMI 模擬**。

**多線併發測試工具**：讓 baresip-agents（201/202/203）主動撥打 200 來壓測 ACD 派線，
用 `scripts/dial_agents_to_queue.js`（在 mikopbx2026-docker skill）：
```bash
node dial_agents_to_queue.js 200 201,202,203   # 三支同時撥 200
node dial_agents_to_queue.js hangup            # 個別掛斷
```
關鍵眉角：ctrl_tcp 的 `dial` 指令永遠用「目前選定的 UA」發話，預設是 accounts 檔案
第一個帳號（201），**單純送 `dial` 三次不會變成三支不同分機在打**（會全部從 201
發出）；要先送 `uafind <完整 AOR，例如 sip:202@mikopbx2026>` 切換 UA、確認回應帶對
的 `accountaor` 後再 `dial`，純數字（`202`）餵給 `uafind` 選不到、悄悄還是用 201。

### J. ✅ 已解：多線併發時 RTP channel 變孤兒 — ARI `POST hangup` 對這批 channel 持續 404

**症狀**：多線併發測試（4 通同時打 200）結束後，`core show channels` 留下好幾個
`UnicastRTP/ai-bot-*` channel 沒被清掉，`docker logs ai-bot` 看到
`ARI POST /channels/<id>/hangup -> 404: {"message":"Resource not found"}`——
但那個 channel id 明明還活著（`core show channels` 查得到）。

**踩過的死路**：一開始以為是跟 addChannel 422 同款的「Asterisk 內部狀態還沒跟上
API 回應」時間差，比照 `_add_channel_to_bridge` 加了 5 次指數退避重試——**沒用**，
重試 5 次全部照樣 404，證明不是暫時性 race，是持續性狀況。

**根因**：這個 Asterisk build 的 ARI `POST /channels/{id}/hangup` 子路徑，對「caller
已經先離開 bridge」之後的 External Media channel 有 bug，查不到那個 channel（不論
重試幾次），但 channel 客觀上還在跑。改用標準 ARI hangup 端點 **`DELETE
/channels/{id}`** 對同一個 id 測試，**立刻成功（204）**。

**修法**：`ari_service.py` 新增 `_hangup()` 統一用 `DELETE /channels/{id}`，取代原本
三處 `POST /channels/{id}/hangup`（`_handle_stasis_start` 的 no-free-port 拒絕路徑、
setup 失敗清理路徑、`_run()` finally 區塊的正常掛斷路徑）。已用 3 通併發真實電話
驗證：修好後 `core show channels` 完全清空、`docker logs` 無任何 404/ERROR。

**教訓**：同一個症狀（回應碼錯誤）先假設「暫時性競態」再假設「端點本身有問題」——
重試次數用完還是 100% 失敗，就該懷疑不是 race，換一個等價的 API 端點試試看，
不要一路加重試次數硬幹。

### K. ⏳ 待查：baresip 主動撥出的電話固定約 32 秒自斷（新現象，跟坑 G 不是同一個根因）

**症狀**：baresip-agents（201/202/203）用 `dial_agents_to_queue.js` 主動撥打 200
時，電話會正常接通（有 `CALL_ESTABLISHED` 事件、Asterisk 端也是 `Up`），但固定在
約 32 秒後 caller 端自己掛斷（`ai_service.py` log 顯示 `caller hung up`，即
StasisEnd 是 caller 那端先觸發，不是 bot 主動掛的）。已重現兩次，兩次都是 3 通同時
撥出、都在 32 秒上下斷。**MicroSIP 直撥 205（或撥 200 被派線）完全正常，可以撐好
幾分鐘的真實對話，只有 baresip 主動撥出這個方向會斷。**

**已排除的可能性**：不是坑 J 的孤兒 channel 問題（那個修好後這個現象照樣發生，
只是這次乾淨收尾沒留孤兒）；理論上也不該是坑 G 的 NAT 問題——baresip 在
`mikopbx2026-docker_default` 網路內部，不經過 Docker loopback port-forward，
Contact/SDP 位址對它來說本來就是可達的容器內部 IP。

**懷疑方向（尚未證實）**：32 秒這個數字本身就是 SIP Timer H 的經典特徵，值得先用
`scripts/capture_sip.sh` 抓 baresip↔mikopbx2026 之間的 SIP 訊令，看 200 OK 的 ACK
有沒有正常送達；也可能是 baresip 本身作為 UAC 撥出時的某個逾時設定（`config` 檔目前
只調過 audio_player/audio_source，没碰過任何 timer 相關參數）。**下次要查這個，先
`scripts/capture_sip.sh start` → 用 `dial_agents_to_queue.js` 撥號 → 斷線後 `scripts/capture_sip.sh
stop`，比對有沒有 ACK 缺失、200 OK 重傳、或 baresip 自己主動送 BYE。**

---

## 診斷工具（scripts/）

固定重複的診斷動作已寫成腳本，不要每次重想：

| 腳本 | 用途 |
|---|---|
| `scripts/apply_sip_nat_fix.sh` | 套用坑 G 的 PJSIP NAT 修正（live，重啟後需重跑）；冪等、會驗證並列出分機 |
| `scripts/cleanup_channels.sh` | 掛掉所有殘留的 `UnicastRTP/ai-bot-*` channel（debug 後常留一堆） |
| `scripts/capture_sip.sh {start\|stop}` | 在 mikopbx2026 內用 tcpdump 抓 port 5060 SIP，stop 會 dump 出可讀訊令 |
| `scripts/ami_trace.py` | 連 AMI（1cami）即時追 dialplan/Hangup 事件，看是誰、什麼 Cause 掛的 |
| `scripts/analyze_rtp.py` | 分析 RTP 原始音訊：位元組序、RMS、頻譜（判斷雜音是編碼壞還是 byte order） |
| `scripts/decode_caller_rtp.py` | 解 caller-leg pcap 的 A-law/mu-law，判定 client 送的是靜音還是真語音（坑 H） |
| `scripts/test_mics.sh` | ffmpeg 逐支錄 Windows 麥克風，找出哪支實際收得到聲音（坑 H） |
| `mikopbx2026-docker` skill 的 `scripts/dial_agents_to_queue.js` | 讓 baresip-agents（201/202/203）主動撥打指定分機/佇列，測試 ACD 併發派線（坑 I/K） |

RTP 音訊擷取：`rtp_transport.py` 認 `RTP_DEBUG_CAPTURE=1` 環境變數，會把收/送的
8kHz PCM 寫到容器 `/tmp/rtp_rx.raw` / `/tmp/rtp_tx.raw`（各約 5 秒上限），
`docker cp` 出來後用 `scripts/analyze_rtp.py` 分析。**正式使用要拿掉這個環境變數。**

常用即時檢查：
```bash
docker exec mikopbx2026 asterisk -rx "core show channels concise"   # State 欄：Ring=沒接通, Up=已接通
docker exec mikopbx2026 asterisk -rx "bridge show all"
docker logs ai-bot --since 3m | grep -E "incoming call|👤|🤖|👋|ERROR"
```

## 測試拓樸備忘

- MicroSIP 在 Windows 主機（`C:\Users\HCH\AppData\Local\MicroSIP\MicroSIP.exe`），
  帳號 204，設定檔 `C:\Users\HCH\AppData\Roaming\MicroSIP\MicroSIP.ini`（UTF-16LE）。
  改設定檔要先關掉 MicroSIP（它結束時會覆寫 ini），改完再開。見 `microsip-control` skill。
- mikopbx2026 SIP 綁 `127.0.0.1:5060`（udp），Web `18080`，ARI `8088`（容器內）。
  詳見 `mikopbx2026-docker` skill。

## 相關 skill

- `mikopbx2026-docker`：PBX 容器管理、REST API、API key。
- `microsip-control`：MicroSIP 的 CLI 控制與撥號。
- `pipecat`：pipecat 框架參考。

---

## Conformance Addendum

## When to Use
把 D:\Github\voice-bot 的 Gemini 語音 bot 透過 Asterisk ARI + External Media 接進 mikopbx2026-docker，讓真人打分機 205-210 或 ACD 代表號 200 就由 AI 接聽。含完整部署流程與所有踩過的坑（音訊雜音、被自動掛斷、佇列派線不動等）的根因與修法。

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
