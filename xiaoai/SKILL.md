---
name: xiaoai
description: 小愛整套語音助理堆疊（小黑 LX06 音箱）—— 全部跑在 192.168.2.90 / 10.145.119.19（hostname gigabyte，雙 RTX 4060 Ti）的 Docker：open-xiaoai-bridge + agent-proxy（工具/技能/MCP）+ llama.cpp（Qwythos-9B）+ LLM Wiki 知識庫，四個容器已合併成 compose project `xiaoai`，用 `xiaoai up/down` 一起開關。涵蓋強制的「小愛同學→召喚小黑→問題」語音流程、手機 Termux 語音與 HTTP 控制、工具/技能/MCP 接線、無頭 Tauri 容器化、模型切換、逐層日誌除錯。當使用者提到 小黑、小愛、小爱音箱、open-xiaoai-bridge、agent-proxy、llm-wiki 知識庫、llama-cpp、或要讓音箱回答問題、要開關這組服務時使用。
triggers:
  - 小愛
  - 小爱
  - xiaoai
  - 小黑
  - 召喚小黑
  - 召唤小黑
  - 小爱音箱
  - 小愛音箱
  - open-xiaoai
  - open-xiaoai-bridge
  - agent-proxy
  - llm-wiki
  - llama-cpp
argument-hint: "[up|down|status|ask|summon|logs|switch-model|add-tool|wiki|reconnect]"
---

# xiaoai — 小愛語音助理全棧 Skill

> 本 skill 由以下舊 skill / memory 合併而來（2026-08）：
> `xiaoai-open-bridge-deepseek`（正本）、`llm-wiki` 的 API 部分、
> `omc-learned/llama-cpp-server-setup`（裸機安裝，見「延伸」）、
> memory `reference-gigabyte-gpu-switch` / `reference-llama-vision-test` /
> `project_ollama_docker_10145119019` 的相關段落。
> 沒有被合併進來但相關的：`tauri-wiki-app`（Windows 桌面版 LLM Wiki，跟這裡
> 容器版是不同部署）、`connect-llm-wiki`（pi/opencode 接 MCP）、
> `phone-termux-remote`（手機端連線）、`llama-vision-test`（測 vision）。

---

## 🚨🚨 鐵則 #1（使用者已強調超過五次，再犯不可原諒）🚨🚨

> # 講「召喚小黑」之前，一定要先講「小愛同學」。

**順序永遠是：`小愛同學` → `召喚小黑` → `問題`**

每一次要跟小黑說話（包括重試、重跑、每一輪迴圈）都要完整走這三步。
這不是建議，是硬性規定。寫任何腳本、做任何測試、任何重試邏輯，都不得違反。

- 不可省略「小愛同學」直接講「召喚小黑」。
- 不可自作主張改成其他喚醒詞（例如「你好小黑」）來繞過這一步。
- 不可把「小愛同學」默默改成 `POST /api/wakeup` 就當作完成了；若因技術限制
  必須這樣做，要**先明說並取得同意**，不能假裝照做了。
- 連續對話逾時退出（聽到「小黑，再見」）後，要重新對話也必須從「小愛同學」重來。

## 🚨 鐵則 #2：三句話必須「一條 SSH 連線內連續唸完」

喚醒後的接手視窗只有幾秒。若把三句拆成三條 `ssh` 指令分開送，
每次連線往返就吃掉數秒，等第二句唸出來時視窗早就關了，音箱會直接說「小黑，再見」。

**錯誤寫法**（實測失敗很多次）：
```bash
ssh phone "termux-tts-speak '小爱同学'"
ssh phone "termux-tts-speak '召唤小黑'"     # 太慢，視窗已關
```

**正確寫法**：一條連線把三句跑完，全部唸完之後才去查結果。
```bash
ssh -p 8022 -i ~/.ssh/id_ed25519 10.145.119.96 "
termux-volume music 15 >/dev/null
termux-tts-speak -l zh-CN -r 0.5 -s MUSIC '小爱同学'
termux-tts-speak -l zh-CN -r 0.5 -s MUSIC '召唤小黑'
sleep 2
termux-tts-speak -l zh-CN -r 0.45 -s MUSIC '<問題>'
"
```

## 🚨 鐵則 #3：失敗就換方法，不要重複同一招

使用者明確要求過：同一個做法失敗兩次就必須改變策略並說明原因，
不要用同樣的參數一直重試洗版面。

---

## 架構（2026-08，全部集中在同一台，已不使用 Jetson .242）

```
[小愛音箱 LX06 192.168.2.152]
        │ ws://192.168.2.90:4399
        ▼
[.90] open-xiaoai-bridge  ← VAD / KWS / ASR(SenseVoice) / TTS 派送   :4399 :9092
        │ http://192.168.2.90:8082/v1   (OpenAI 相容)
        ▼
[.90] agent-proxy         ← 工具執行層（skills + MCP + ASR）          :8082
        │ http://192.168.2.90:8081/v1
        ▼
[.90] llama-cpp           ← Qwythos-9B-v2-Q4_K_M（+mmproj，可看圖）  :8081
                          ↑
[.90] llm-wiki            ← 個人知識庫，agent-proxy 的 wiki_* 工具去查  :19828 :6080
```

**同一台機器的兩個 IP**：`192.168.2.90`（內網）與 `10.145.119.19`（ZeroTier），
hostname `gigabyte`，帳號 `hch`，免密碼金鑰。雙 RTX 4060 Ti 16GB。
文件裡兩個 IP 混用是正常的，**指的是同一台**。

歷史：原本 bridge 跑在 Jetson `192.168.2.242`，後端接 DeepSeek/OmniRoute。
現已全數搬到 `.90`，`.242` 的 bridge 容器已停止，不要再改它。

---

## 🆕 一起開 / 一起關（compose project `xiaoai`）

四個容器已被 `/home/hch/xiaoai/compose.yaml` 用 `include:` 收攏成同一個
compose project（project name = `xiaoai`）。**各服務原本的
`docker-compose.yml` 完全沒改動**，單獨進各目錄操作依然正常。

```bash
ssh hch@10.145.119.19 'xiaoai up'       # 一起打開（llama-cpp/agent-proxy/llm-wiki/bridge）
ssh hch@10.145.119.19 'xiaoai down'     # 一起關閉
ssh hch@10.145.119.19 'xiaoai ps'       # 看狀態
ssh hch@10.145.119.19 'xiaoai restart'
ssh hch@10.145.119.19 'xiaoai logs open-xiaoai-bridge'   # -f --tail=100
```

不透過腳本：`docker compose -f /home/hch/xiaoai/compose.yaml up -d`

| 檔案 | 說明 |
|---|---|
| `/home/hch/xiaoai/compose.yaml` | include 四個專案的 compose，`name: xiaoai` |
| `/home/hch/bin/xiaoai` | 一鍵開關腳本（`~/bin` 在互動 shell 的 PATH） |
| `/usr/local/bin/xiaoai` | → symlink，讓 `ssh host 'xiaoai up'` 這種非互動命令也找得到 |

本 skill 附的副本在 `compose/compose.yaml`、`compose/xiaoai.sh`。

### 🔕 不開機自動啟動（2026-08 設定）

四個專案的 `docker-compose.yml` 已全部從 `restart: unless-stopped`
改成 **`restart: "no"`**，所以：

- 主機重開後這四個容器**不會自己起來**，要手動 `xiaoai up`。
- 容器 crash 也不會自動重啟（這是 `"no"` 的附帶效果，使用者接受；
  若之後想要「不開機啟動但 crash 要救」，改成 `restart: on-failure` 即可）。
- 原檔都有備份 `docker-compose.yml.bak-restart`，要還原直接覆回去。

驗證：
```bash
ssh hch@10.145.119.19 'for c in agent-proxy llama-cpp llm-wiki open-xiaoai-bridge; do \
  echo "$c $(docker inspect $c --format "{{.HostConfig.RestartPolicy.Name}}")"; done'
# 四行都要是 no
```

> **坑：改 `restart:` 必須重建容器才生效。** RestartPolicy 是寫在容器
> `HostConfig` 裡的，只改 yaml 不重建，舊容器依然帶著 `unless-stopped`。
> 改完跑一次 `xiaoai down && xiaoai up` 再用上面指令確認。

> 這台還有一個 `llama-server.service`（systemd，**已 disabled**）——是舊的裸機
> llama.cpp 服務，跟這裡的 Docker 版無關，不要把它 enable 起來，會跟
> `llama-cpp` 容器搶 GPU 與 port。

> **坑：改 compose project 歸屬必須重建容器。** 舊容器身上帶著舊 project 的
> `com.docker.compose.project` label，光加 include 檔不會自動歸隊——`docker
> compose ps` 會顯示空的。必須先在各自原目錄 `docker compose down`，再用新的
> compose.yaml `up -d` 一次，容器才會掛上新 label。

> **坑：這台機器上的容器名單變動很快。** 除了這四個，還有 comfyui、vllm、
> ollama、n8n、new-api、mikopbx、audiocpp 等時開時關的容器（多數目前 exited）。
> 任何「現在在跑什麼」的判斷，一律先現場 `docker ps -a` + `nvidia-smi` 確認，
> 不要相信任何文件裡「目前正在運作」的描述。

---

## 存取方式

| 主機 | 角色 | 連線 |
|---|---|---|
| `192.168.2.90` / `10.145.119.19` | 主機（四個容器都在這） | `ssh hch@10.145.119.19`（免密碼金鑰） |
| `192.168.2.152` | 小愛音箱 LX06（已 root） | `plink -ssh -hostkey SHA256:8qX7d7XxQYgXjszhSeqIYE7mUroL/5eD2zUF8AMzmVY -pw '<XIAOAI_ROOT_PASSWORD>' root@192.168.2.152` |
| `10.145.119.96` | 手機 Termux（ZeroTier） | `ssh -p 8022 -i ~/.ssh/id_ed25519 10.145.119.96`；先跑 `phone-termux-remote` 的 `connect.sh` |
| `192.168.2.242` | 舊 Jetson（bridge 已停用） | `plink ... -pw '<SSH_PASSWORD>' hch@192.168.2.242` |

⚠️ `.90` 上 `docker` CLI 偶爾會卡住（`docker ps` 無回應數分鐘）。
改用 Docker Engine API 走 unix socket 最可靠：
```bash
curl --unix-socket /var/run/docker.sock -sS http://localhost/containers/json
```

### 快速健康檢查
```bash
ssh hch@10.145.119.19 'xiaoai ps; curl -s localhost:8081/health; curl -s localhost:8082/health'
```
`8082/health` 會回目前載入的 skills 清單、upstream、mcp_tools、asr 狀態。

---

## 兩種問小黑的方式

### 方式 A：語音（手機開口說，音箱麥克風聽）— 遵守鐵則 #1/#2

手機腳本：`~/scripts/speak_xiaohei.sh "<問題>"`（本 skill `scripts/speak_xiaohei.sh`）

```bash
ssh -p 8022 -i ~/.ssh/id_ed25519 10.145.119.96 "~/scripts/speak_xiaohei.sh '礼拜天去哪里玩'"
```

語音參數（實測調出來的）：
- `-l zh-CN`：小米雲端 ASR 是簡中調校，用 zh-TW 會誤聽（「召喚小黑」→「没有接受」）
- `-r 0.5`（喚醒/召喚）、`-r 0.4~0.45`（問題）：太快會被切斷
- `-s MUSIC` + `termux-volume music 15`：走音樂串流並開到最大聲
- **問題要短**：長句常被 VAD 切斷，只辨識到前半段

### 方式 B：純 HTTP（不出聲，音箱直接唸答案）— 最穩，但不走喚醒詞

`~/scripts/ask_xiaohei.sh "<問題>"`，或直接：
```bash
# 1) 問 agent-proxy（會自動呼叫工具）
curl -sS http://192.168.2.90:8082/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"x","messages":[{"role":"user","content":"現在幾點"}]}'

# 2) 讓音箱唸出來
curl -sS http://192.168.2.90:9092/api/play/text \
  -H 'Content-Type: application/json' -d '{"text":"...."}'
```

⚠️ 方式 B **沒有**走「小愛同學→召喚小黑」，音箱只是當喇叭用。
要用這個方式前必須先告知使用者，不能拿它假裝完成了語音流程。

---

## 驗證：不要用手機麥克風，要讀音箱自己的紀錄

**手機麥克風在 SSH 背景執行時錄到的是純靜音**（實測 peak=0），
因為 Android 對背景 App 強制靜音麥克風，這是系統政策，無法用權限解決。
另外 TTS 播放時也無法同時錄音，兩者搶同一個音訊裝置。

改用 agent-proxy 提供的兩個端點：

```bash
curl -sS 'http://192.168.2.90:8082/spoken?limit=5'   # 音箱最近「說」了什麼
curl -sS 'http://192.168.2.90:8082/heard?limit=5'    # 音箱最近「聽」到什麼
```

或直接讀 bridge 日誌關鍵字：
```bash
ssh hch@10.145.119.19 "docker logs --since 3m open-xiaoai-bridge 2>&1 | \
  grep -E '收到指令|Recognized|我说|OpenAI Conv|\"Speak\"'"
```
| 日誌關鍵字 | 意義 |
|---|---|
| `XiaoAI wakeup` | 「小愛同學」成功喚醒 |
| `收到指令: 召唤小黑` | 召喚被正確辨識 |
| `before_wakeup returned: openai` | 小黑接手成功 |
| `OpenAI Conv 🎙️ 进入` | 已進入連續對話 |
| `[ASR] Recognized: xxx` | 音箱聽到的問題 |
| `"Speak"` + text | 音箱實際講出的話 |
| `小黑，再见` | 逾時退出，要從「小愛同學」重來 |

---

## 🚨🚨 最重要的調查方法：先分清「哪一層錯了」，不要猜

小黑答錯時，一定要照這個順序查，**每一層都要拿到證據再往下走**。
直接改提示詞是最常見的錯誤做法。

```bash
# 第 1 層：知識庫原始檔對不對？
cat /home/hch/llm-wiki/data/KM/raw/sources/*.md

# 第 2 層：工具實際回傳什麼？（最關鍵，常常真相就在這）
docker exec agent-proxy sh -c 'cd /app && python3 -c "
import asyncio, skills
print(asyncio.run(skills.wiki_search(\"關鍵字\")))"'

# 第 3 層：模型到底有沒有呼叫工具？（沒呼叫 = 編的）
docker logs --since 20m agent-proxy 2>&1 | grep -c 'POST /v1/chat/completions'   # 請求數
docker logs --since 20m agent-proxy 2>&1 | grep -c 'tool call'                  # 工具數
# 兩者落差很大（例如 17 vs 3）= 模型大量跳過工具自己編

# 第 3.5 層：【最重要】bridge 到底送了什麼給 proxy？
docker logs --since 20m agent-proxy 2>&1 | grep 'incoming' | tail -3
# 看 msgs=? 與 extras=?。這行日誌是實作在 main.py 裡的，專治
# 「語音跟 HTTP 行為不一致」——不要再靠猜的。

# 第 4 層：音箱實際聽到/說出什麼？
docker logs --since 20m open-xiaoai-bridge 2>&1 > /tmp/a.log
```

解碼 bridge 日誌的 Speak 內容（日誌是 `\uXXXX` 轉義，直接 grep 會是亂碼）：
```bash
python3 - <<'PY'
import re, json
raw = open('/tmp/a.log', encoding='utf-8', errors='replace').read()
for m in re.finditer(r'Recognized: (.+)', raw):
    print('聽到:', m.group(1).strip())
for m in re.finditer(r'MP3","text":"(.*?)"\}\}', raw):
    s = m.group(1)
    try:
        s = json.loads('"' + s + '"')
    except Exception:
        pass
    print('說出:', s)
PY
```

---

## 🚨🚨 坑：對話歷史污染——錯誤答案會自我複製（最隱密、最害）

症狀：**同一句話走 HTTP 永遠正確，走語音就一直答錯**，而且錯得很一致。
實際發生：問「沈世蘭在哪裡工作」，音箱反覆答「台中」（正確：花蓮），
問面部特徵答「酒窩」（正確：會長毛的痣）。

真正原因（靠 `incoming` 日誌才看得到）：
```
incoming: msgs=22  extras={'reasoning_effort': 'low', ...}
```
bridge 每次把 **22 則對話歷史**一起送給模型，而那裡面塞滿了之前的錯誤答案。
模型看到自己講過「在台中工作」，就直接沿用，不再呼叫工具。
**錯誤一旦進入歷史就會自我複製，越答越偏。**

這也是為什麼 HTTP 測不出來：curl 是乾淨的單則請求，根本沒有歷史。

修法（三處一起，已套用）：
| 項目 | 改動 | 原因 |
|---|---|---|
| `config.py` → `history_max_messages` | 20 → **4** | 大幅縮短歷史，錯誤不會長期殘留 |
| `config.py` → `extra_body` | `{"reasoning_effort":"low"}` → **`{}`** | 壓低推理會讓模型偷懶跳過工具 |
| `main.py` | 歷史有 ≥2 則 assistant 時**注入提醒** | 明示「歷史不是可靠來源，這一題必須重新查證」 |

> 教訓：只要看到「**HTTP 對、語音錯**」，**第一個要懷疑的就是對話歷史**，
> 不是提示詞、不是知識庫、也不是工具。先看 `incoming: msgs=?`。

## 🚨 坑：bridge 的 system_prompt 會蓋掉 agent-proxy 的工具指示

症狀：**同一句話走 HTTP 答得出來，走語音卻說「這個我幫不上忙」**。

原因：`config.py` 的 `openai` 區塊有自己的 `system_prompt` 與 `rule_prompt`，
會覆蓋 agent-proxy 的 `TOOL_SYSTEM_HINT`，模型根本不知道有工具/知識庫。

**改工具相關行為時，兩邊都要改：**
- `/home/hch/agent-proxy/app/main.py` → `TOOL_SYSTEM_HINT`
- `/home/hch/open-xiaoai-bridge/config.py` → `openai.system_prompt` 與 `rule_prompt`

另外 `rule_prompt` 原本限「80 字以內」，會把工具查到的步驟壓縮掉（例如漏掉
「等兩分鐘」），已放寬到 150 字並要求照工具內容回答。

## 🚨 坑：模型「假裝查過知識庫」並編造答案（比拒答更難抓）

實際發生：問「第一帥是誰」，音箱答「我查過知識庫，第一帥是**林俊傑**」。
但 `grep -c 'tool call'` 顯示那幾次請求的工具呼叫次數是 **0**——它在說謊。

重點：**不要相信模型自述的「我查過了」，一律以 tool call 日誌為準。**
另一個典型症狀：5 次 `POST /v1/chat/completions` 但只有 1 次 tool call。

修法（兩層都要）：
1. `main.py` 的 `claims_lookup_without_tool()`：答案裡出現「我查過知識庫、
   根據知識庫、我查到…」但這一輪 `used` 為空，就判定為編造，
   加 `tool_choice="required"` 強制重問。（與只抓拒答的 `looks_like_refusal()`
   並存，兩種病徵不同）
2. bridge `system_prompt` 加：「你沒有記憶，也不記得之前查過什麼。絕對不可以說
   『我查過知識庫』，除非你在這一次回答裡真的呼叫了 wiki_search。」

## 🚨 坑：概念頁只有 slug，模型會依拼音編造人名

實際發生：問「第二帥是誰」，音箱答「**李建明**」（正確是黃建銘）。

原因：簡體查詢命中的是**概念頁**「帥哥排名」，而概念頁裡人名只寫 slug：
```
- 全世界最帥的人：huang-chien-hsiung（第一名）
- 第二帥的人：huang-chien-ming（第二名）
```
中文名字只存在**實體頁**，但它排後面被截掉了。模型被要求用中文回答，
看到 `huang-chien-ming` 就自己音譯成「李建明」。

這也解釋了為什麼**走 HTTP 答對、走語音答錯**——兩邊命中的頁面順序不同。

修法：
1. `_wiki_query()` 將 `wiki/entities/` 排序到最前面，並額外補抓實體頁。
   **實體頁才是名字與數值的權威來源，概念頁只引用代號。**
2. 提示詞加：人名地名專有名詞必須**一字不差照抄工具原文**，不准改字、
   換同音字、自行音譯；若只有拼音代號就說只有代號，不准猜。

## 🚨 坑：shell heredoc 會寫錯中文字

實際發生：透過 `ssh ... <<'EOF'` 寫入知識庫，「沈世**蘭**」被寫成「沈世**然**」，
導致後端建了一個名字錯誤的實體頁，小黑就一直答錯名字。

→ 寫中文進知識庫一律用 **Python 直接寫 UTF-8 檔**，不要用 heredoc：
```bash
ssh hch@10.145.119.19 "python3 - <<'PY'
text = '''# 標題
內容……
'''
with open('/home/hch/llm-wiki/data/KM/raw/sources/x.md', 'w', encoding='utf-8') as f:
    f.write(text)
PY"
```
寫完**一定要 `cat` 回來確認字沒錯**，再跑 rescan。
若已產生錯誤實體頁，要手動刪除：
```bash
docker exec llm-wiki ls /data/KM/wiki/entities/
docker exec llm-wiki rm -f /data/KM/wiki/entities/錯誤名.md
```

---

## LLM Wiki（個人知識庫，小黑的資料來源）

官方**沒有** Docker 版。做法：拿 release 的 Linux `.deb`，在 Debian 容器裝
WebKit/GTK，用 **Xvfb 虛擬顯示器**讓這個 GUI 程式無頭跑起來。

```
/home/hch/llm-wiki/
├── Dockerfile          # debian + webkit2gtk-4.1 + xvfb + x11vnc/noVNC + socat
├── entrypoint.sh       # 寫 apiConfig → Xvfb → openbox → VNC → llm-wiki → socat
├── docker-compose.yml  # user: "1000:1000"、shm_size 512m、image llm-wiki:0.6.8
├── data/               # 持久化（專案、DB、設定）
└── docs/               # host 端丟檔案處，GUI 裡匯入成 project source
```

| 項目 | 值 |
|---|---|
| REST API | `http://192.168.2.90:19828/api/v1` |
| Token | `<KB_BEARER_TOKEN>`（compose 的 `LLM_WIKI_API_TOKEN`） |
| 授權 header | `Authorization: Bearer <KB_BEARER_TOKEN>` |
| 健康檢查 | `GET /health`（唯一免授權端點） |
| 網頁 GUI | `http://192.168.2.90:6080/vnc.html`（noVNC，無密碼） |
| 專案 | `KM`，id `9e42dddf-3b29-4a2d-af88-43cfbe61d939`（`current` 亦可代指當前專案） |
| 來源目錄 | `/home/hch/llm-wiki/data/KM/raw/sources/`（可直接 scp） |

### 常用 API 端點

| 方法 | 端點 | 說明 |
|------|------|------|
| GET | `/projects` | 列出所有專案 |
| GET | `/projects/{id}/files?root=wiki` | 列出 wiki 頁面 |
| GET | `/projects/{id}/files/content?path=wiki/xxx.md` | 取得檔案內容（**搜尋後一定要再讀這個**） |
| POST | `/projects/{id}/search` | 搜尋 |
| POST | `/projects/{id}/chat` | 問檢索代理（較慢；`images` 欄位可傳圖，見 `tauri-wiki-app`） |
| POST | `/projects/{id}/sources/rescan` | 重新掃描來源目錄建索引 |
| GET | `/projects/{id}/graph` | 知識圖譜 |

### 容器化踩過的坑（全部實測）

1. **設定檔在 `XDG_DATA_HOME` 不是 config 目錄** —— 實際路徑
   `/data/share/com.llmwiki.app/app-state.json`。
2. **API 寫死綁 `127.0.0.1`**，`allowLanAccess` 只能在 GUI 開。
   → 用 **socat** 轉發 `0.0.0.0:19829 → 127.0.0.1:19828`，compose 發佈 `19828:19829`。
3. **要能 scp 就要 `user: "1000:1000"`**，否則容器寫出 root 檔主機動不了。
   但非 root 會讓 dbus 壞掉 → 要在 image 裡先建好 `/data`、`/run/dbus`、
   `/run/user/1000` 並 chown，改用 session bus。
4. **GUI 要加 openbox** —— 沒視窗管理員的話 VNC 裡視窗拖不動、對話框按不到。
5. **建專案沒有 API**，只能進 noVNC 點。REST API 是**唯讀**：
   `/projects`、`/search`、`/chat`、`/graph`、`/files`、`/sources/rescan`。
6. **Tauri/WebKit 需要 `shm_size: 512m`**，Docker 預設 64MB 不夠。

### 新增知識
```bash
scp 文件 hch@10.145.119.19:/home/hch/llm-wiki/data/KM/raw/sources/
curl -sS -X POST -H 'Authorization: Bearer <KB_BEARER_TOKEN>' \
  http://192.168.2.90:19828/api/v1/projects/9e42dddf-3b29-4a2d-af88-43cfbe61d939/sources/rescan \
  -H 'Content-Type: application/json' -d '{}'
```
後端會自動抽取實體、建概念頁、做雙向連結（約 30–60 秒）。

### 坑：搜尋結果只有 YAML 標頭，沒有真內容
`/search` 回傳的 snippet 幾乎都是 `type: entity  tags: [...]` 這種 metadata，
真正的事實（密碼、位置、步驟）在頁面內文。
→ `wiki_search` 必須搜到後**再用 `/files/content` 把內文讀出來**，
並過濾掉 front-matter 與 `index/overview/log` 這種無用頁。
這是「小黑查不到知識庫」的主因之一。

### 坑：知識庫的「寫法」會決定後端怎麼標註它
檔名叫 `scptest.txt`、內容像隨手筆記時，建索引時會自動判定為
「測試/玩笑性質內容，不建立實體頁」，並寫進 wiki 說「不具備實質知識價值」。
模型讀到這個負面標註，就會答得吞吞吐吐或拒答。
→ 要當正式知識用，文件就要**寫得像正式資料**：檔名有意義、標題明確、
並直接聲明「這是正式認定，不是玩笑也不是測試資料」。修改後要 rescan。

### 坑：主觀/排名類問題不會觸發知識庫
問「最帥的人是誰」，模型會答「這很主觀……網路投票有李奧納多」，根本不查。
→ 提示詞需明列：「最帥、最好、最厲害、第一名、誰最…」這類主觀排名問題
**也必須先查知識庫**，因為那是在問主人記錄的看法，不是問模型意見或網路投票；
且查到就要肯定地講，不准加「這只是測試內容」這類評論。

> 📌 **不要跟這兩個搞混**：
> - `tauri-wiki-app` skill 講的是 **Windows 桌面版** LLM Wiki（`%APPDATA%`、
>   `Bearer` token 存在 `app-state.json`、傳圖 OCR、patch bundled mcp-server）。
> - `wiki` skill 是 OMC 內建、純 markdown 存 `.omc/wiki/*.md` 的**完全不同系統**。
> - `connect-llm-wiki` 是把桌面版接進 pi / opencode 當 MCP 工具源。

---

## agent-proxy（工具/技能/MCP 執行層）

部署位置：`/home/hch/agent-proxy`（container `agent-proxy`，port 8082）

```
agent-proxy/
├── docker-compose.yml   # TZ、runtime: nvidia、掛 docker.sock 與 models、MODEL_OVERRIDE
├── Dockerfile           # 含 ffmpeg（ASR 轉檔用）
├── requirements.txt     # fastapi/httpx/mcp/sherpa-onnx
├── mcp.json             # MCP server 設定（目前空 `{"mcpServers":{}}`）
└── app/
    ├── main.py          # OpenAI 相容端點 + 工具迴圈 + /spoken /heard /asr /summon
    ├── skills.py        # 內建白名單技能
    ├── asr.py           # SenseVoice 本地 ASR
    └── mcp_client.py    # MCP client，工具名為 mcp__<server>__<tool>
```

重要環境變數（compose 內）：`UPSTREAM_BASE_URL=http://192.168.2.90:8081/v1`、
`MODEL_OVERRIDE=/models/Qwythos-9B-v2-Q4_K_M.gguf`、`MAX_TOOL_ROUNDS=4`、
`WIKI_BASE_URL` / `WIKI_API_TOKEN` / `WIKI_PROJECT_ID`、
`ASR_MODEL_DIR=/models/sherpa-onnx-sense-voice-...`。

### 內建技能（`/health` 可即時查詢清單）

| 技能 | 用途 |
|---|---|
| `get_time` | 現在時間、日期、星期 |
| `get_weather` | 城市天氣＋未來三天預報（可答「明天」） |
| `get_stock` | 股票／指數／匯率／加密貨幣（台股要 `.TW`） |
| `web_search` | 網路搜尋，多層 fallback |
| `web_fetch` | 讀取指定網址內容 |
| `calculate` | 安全算式計算 |
| `docker_status` | 本機容器狀態 |
| `gpu_status` | GPU 使用率／記憶體／溫度 |
| `host_status` | CPU 負載／記憶體／磁碟 |
| `wiki_search` | **查個人知識庫**，關鍵字搜尋，會把頁面內文讀出來 |
| `wiki_ask` | 丟完整問題給知識庫的檢索代理彙整（較慢） |

### 新增技能
編輯 `app/skills.py` 的 `SKILLS`（handler + JSON schema），然後重建：
```bash
ssh hch@10.145.119.19 "cd /home/hch/agent-proxy && docker compose build && \
  docker rm -f agent-proxy; docker compose up -d"
```
（重建完若想回到 `xiaoai` project 群組，記得改用 `xiaoai up` 起。）

### 新增 MCP server
編輯 `/home/hch/agent-proxy/mcp.json`（Claude Desktop 相容格式）後重啟容器：
```json
{ "mcpServers": {
    "filesystem": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/data"] }
} }
```
工具會自動以 `mcp__filesystem__<tool>` 註冊給模型。

### 強制「先查再答」
`main.py` 的 `TOOL_SYSTEM_HINT` 規定所有事實性問題都要先呼叫工具；
`looks_like_refusal()` + `claims_lookup_without_tool()` 兩個偵測會加
`tool_choice=required` 再問一次。輸出會經 `sanitize_for_speech()`
清掉 Markdown / emoji / 網址，避免唸出奇怪符號。

---

## 常見問題與已知坑（bridge / 音箱）

### 坑 1：小黑收到問題卻不回答（最容易踩）
`config.py` 一次性問答必須帶 `wait_response=True`：
```python
await app.send_to_openai_and_play_reply(rest, wait_response=True)
```
少了它就是射後不理，答案回來時沒人接手，音箱永遠不出聲。

### 坑 2：講「小黑…」卻被小愛搶答
原本路由只認 `text == "召唤小黑"` 完全相等。已改為模糊比對，
涵蓋 ASR 常見誤聽（小嘿/小赫/小鹤/小何…）並自動去掉召喚詞留下問題。

### 坑 3：連續對話太快逾時
`config.py` → `wakeup.timeout`，原本 20 秒，**已改成 120 秒**。

### 坑 4：時間差 8 小時
agent-proxy 需要 `TZ=Asia/Taipei` 並掛 `/etc/localtime`，否則回 UTC。

### 坑 5：容器內沒有 docker CLI / 看不到 GPU
`docker_status` 改用 Docker Engine API 走 socket；
`gpu_status` 需要 compose 加 `runtime: nvidia` + `NVIDIA_DRIVER_CAPABILITIES=utility`。

### 坑 6：搜尋結果過時（例如總統答成舊的）
`web_search` 先查即時網頁，再加 **Wikidata 現任職位查詢**（SPARQL，P39 且無結束日期），
最後才回退維基百科摘要。維基百科開頭段落常年久未更新，不能單靠它。

### 坑 7：音箱麥克風會錄到旁邊真人講話
聊天時使用者若在旁邊講話，會蓋掉手機的問題，小黑會回「你說的有點亂」。
測試時請使用者保持安靜，或把手機靠近音箱。

### 坑 8：`docker-compose up` 報 `KeyError: 'ContainerConfig'`
舊的 docker-compose **1.29.2**（llm-wiki / bridge 兩個專案還在用）對既有容器
重建會炸。先 `docker rm -f <name>` 再 `docker compose up -d`。
（主機同時裝有 compose v5.1.4，`docker compose` 子命令走的是新版，較穩。）

---

## 模型切換

模型檔在 `/home/hch/models/`（目前有 Qwythos-9B 系列、Qwen3-4B/8B/14B/32B、
qwen2.5-7b 等 GGUF）。改 `/home/hch/llama-cpp-docker/docker-compose.yml` 的
`-m`（與 vision 用的 `--mmproj`）：

```bash
ssh hch@10.145.119.19 "sed -i 's/Qwythos-9B-v2-Q4_K_M.gguf/Qwen3-14B-Q4_K_M.gguf/g' \
  /home/hch/llama-cpp-docker/docker-compose.yml && xiaoai down && xiaoai up"
```

**三處要一起改，否則行為不一致：**
| 位置 | 欄位 |
|---|---|
| `/home/hch/llama-cpp-docker/docker-compose.yml` | `-m` / `--mmproj` |
| `/home/hch/agent-proxy/docker-compose.yml` | `MODEL_OVERRIDE` |
| `/home/hch/open-xiaoai-bridge/config.py` | `openai.model` |

> ⚠️ 目前 `config.py` 的 `model` 還寫著 `/models/Qwen3-14B-Q4_K_M.gguf`，
> 但 agent-proxy 的 `MODEL_OVERRIDE`（Qwythos-9B-v2）會蓋過它，所以實際跑的是
> Qwythos。這個不一致無害但容易誤導，之後有動到再一起改正。

llama-cpp 目前參數：`-ngl 99 -c 16384 --temp 0.6 --top-p 0.95 --top-k 20
--repeat-penalty 1.05 --jinja`，掛 `--mmproj` 所以**這台 llama.cpp 有視覺能力**。

實測速度（雙 RTX 4060 Ti 16GB）：4B≈93 t/s、8B≈66 t/s、14B≈38 t/s、
32B 需跨兩張卡會更慢。語音助理建議停在 14B 以內。

---

## 音箱重新指向

```bash
plink -ssh -hostkey SHA256:8qX7d7XxQYgXjszhSeqIYE7mUroL/5eD2zUF8AMzmVY -pw '<XIAOAI_ROOT_PASSWORD>' root@192.168.2.152 \
  "echo 'ws://192.168.2.90:4399' > /data/open-xiaoai/server.txt && \
   kill -9 \$(ps w | grep '[o]pen-xiaoai/client' | awk '{print \$1}'); sleep 1; \
   /data/open-xiaoai/client ws://192.168.2.90:4399 > /dev/null 2>&1 &"
```
重開機後 `/data/init.sh` 會重讀 `server.txt`，設定會保留。

韌體版本：目前 `LX06_1.88.221_patched`，最新為 `LX06_1.94.13`（需實體 USB 刷機，無法遠端）。

## 手機端腳本

| 腳本 | 用途 |
|---|---|
| `~/scripts/speak_xiaohei.sh` | 語音版：手機唸小愛同學→召喚小黑→問題（遵守鐵則） |
| `~/scripts/ask_xiaohei.sh` | HTTP 版：不出聲，音箱直接唸答案 |

Termux 沒有 python，只有 `node`，JSON 處理一律用 node 寫。
手機連線本身見 skill `phone-termux-remote`。

---

## GPU 與其他共用這台機器的服務

這台（gigabyte，雙 RTX 4060 Ti 各 16GB）同時放了很多東西：comfyui-h3、
vllm-qwythos、ollama、n8n、new-api、mikopbx2026、audiocpp 等，多數目前是
exited 狀態，但隨時可能被開起來搶 VRAM。

- llama-cpp 用 `deploy.resources.reservations.devices: count: all`（看得到兩張卡）。
- `~/bin/gpu-switch` 二進位仍在（2026-07-22 建立，舊版是 llama.cpp/ollama/whisper
  三選一的互斥機制），但**現在是否還有效、行為是否改過都需現場查證**，
  不要假設它還是原本的邏輯。
- 若 llama-cpp 回應異常慢或 OOM，先 `nvidia-smi` 看是不是別的容器把卡吃掉了，
  不要一開始就去改推理參數。
- **不要**把 Qwythos 拿去當 embedding 模型（`--task embed`）——它是 roleplay
  微調，沒有對比學習/檢索目標，向量品質很差；要本地 embedding 請另外部署
  Qwen3-Embedding-0.6B 之類的專用小模型。

---

## 延伸 / 相關 skill

| 主題 | 去哪裡 |
|---|---|
| llama.cpp **裸機**安裝（CUDA toolkit、build、systemd、GGUF 下載） | `omc-learned/llama-cpp-server-setup.md` |
| 測 llama.cpp 的**圖片辨識/OCR** 能力（經 opencode CLI） | `llama-vision-test` |
| Windows **桌面版** LLM Wiki（傳圖 OCR、patch mcp-server） | `tauri-wiki-app` |
| 把 LLM Wiki 接進 **pi / opencode** 當 MCP | `connect-llm-wiki` |
| 遠端 llm-wiki **noVNC**（`:6080/vnc.html` 開關／驗證） | `llm-wiki-novnc` |
| 手機 Termux / ADB 遠端 | `phone-termux-remote` |
| vLLM 版 Qwythos 部署（另一條路線，非本堆疊） | `vllm-qwythos-deploy` |
| 這台機器的 Docker 遠端管理 | `docker-remote-control` |

---

## 本 skill 附的可部署原始碼

```
compose/       compose.yaml（xiaoai project）、xiaoai.sh（一鍵開關腳本）
scripts/       scripts/speak_xiaohei.sh、scripts/ask_xiaohei.sh（手機端）
agent-proxy/   Dockerfile、requirements.txt、docker-compose.yml、mcp.json
               app/{main,skills,mcp_client,asr}.py
llama-cpp/     docker-compose.yml
llm-wiki/      Dockerfile、entrypoint.sh、docker-compose.yml（無頭 Tauri 方案）
bridge/        config.py、system_prompt.txt
```
scp 到主機對應目錄即可重建整套環境。

---

## 實測驗證紀錄（語音，走完整鐵則流程）

| 問題 | 音箱回答 | 來源 |
|---|---|---|
| 家里的无线网络密码是多少 | 密碼是 12345678 | ✅ 知識庫 |
| 路由器放在哪里 | 放在客厅的电视柜上 | ✅ 知識庫 |
| 台湾有几个县市 | 台湾目前有22个县市 | ✅ 模型 |
| 全世界最帅的人是谁 | 全世界最帅的人是黄建雄 | ✅ 知識庫 |
| 第一帅跟第二帅分别是谁 | 第一帅是黄建雄，第二帅是黄建铭 | ✅ 知識庫 |
| 网络不通怎么办 | ✗ 未照知識庫（漏「等兩分鐘」） | ⚠ 待修 |

### 除錯成功案例（三次答錯，三個不同原因）

| 症狀 | 真正原因 | 修法 |
|---|---|---|
| 答「這很主觀、網路投票…」 | 主觀問題不觸發工具 | 提示詞明列主觀/排名也要查 |
| 答「李建明」（應為黃建銘） | 命中概念頁，只有 slug 無中文名 | 實體頁排序優先 + 禁止音譯 |
| 答「林俊傑」並聲稱查過 | 根本沒呼叫工具，完全編造 | 編造偵測 + 強制 tool_choice |

**教訓：三次都是「人名答錯」，但根因完全不同。每次都必須重新逐層排查，
不能套用上一次的結論。**

### 尚未完全解決的問題（誠實記錄）
「網路不通怎麼辦」這類**通用操作步驟問題**，走 HTTP 會正確照知識庫回答
（含「等兩分鐘」），但走語音時模型仍可能不呼叫 wiki_search 就自己編。
已確認工具本身沒問題。新增的編造偵測可接住部分情況，但若模型沒講
「我查過」就抓不到。待試：對這類問題在 proxy 側主動先查知識庫再交給模型。

---

## Conformance Addendum

## When to Use
小愛整套語音助理堆疊（小黑 LX06 音箱）—— 全部跑在 192.168.2.90 / 10.145.119.19（hostname gigabyte，雙 RTX 4060 Ti）的 Docker：open-xiaoai-bridge + agent-proxy（工具/技能/MCP）+ llama.cpp（Qwythos-9B）+ LLM Wiki 知識庫，四個容器已合併成 compose project `xiaoai`，用 `xiaoai up/down` 一起開關。涵蓋強制的「小愛同學→召喚小黑→問題」語音流程、手機 Termux 語音與 HTTP 控制、工具/技能/MCP 接線、無頭 Tauri 容器化、模型切換、逐層日誌除錯。當使用者提到 小黑、小愛、小爱音箱、open-xiaoai-bridge、agent-proxy、llm-wiki 知識庫、llama-cpp、或要讓音箱回答問題、要開關這組服務時使用。

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
