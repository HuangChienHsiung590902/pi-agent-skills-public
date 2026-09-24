---
name: pi-omni-model-test
description: >-
  透過 pi coding agent + OmniRoute 閘道器，實際呼叫/測試/比較「另一顆 LLM」（例如 ChatGPT/openai-codex、
  DeepSeek、Hunyuan、或 OmniRoute 首頁「免鑑權提供者」清單裡的免費端點）的三種操作方式：
  (1) 用 herdr agent 啟動一個常駐、可持續對話的 pi agent pane；(2) 直接一次性呼叫
  `pi --provider omni --model <id> -p "..." --no-session` 拿乾淨、互不干擾的單次回答，適合批次比較多顆模型；
  (3) 同時開多個 herdr pane 各跑一個 pi，當成可平行工作的 subagent 群，分工處理多個獨立子任務。
  涵蓋：`pi --list-models <關鍵字>` 查模型 id、OmniRoute 首頁「免鑑權提供者」卡片名稱對應的 pi model 前綴
  對照表、Windows 上 herdr agent start / PowerShell 執行原則的已知踩坑、免費模型常見的 429/503/逾時/401/403
  失敗模式、以及如何讓 pi agent 透過 connect-chrome 的 playwright MCP 去瀏覽網站並回報內容。
  當使用者要求「叫另一個 LLM/agent 做事」「測試/比較不同模型能力」「用 pi 呼叫 ChatGPT/DeepSeek/其他免費模型」
  「omniroute 那些免鑑權提供者能不能用」「讓 pi 用 subagent 的方式協同工作」「多開幾個 pi 平行處理」
  「pi 分工/並行跑」時使用。
---

# pi + OmniRoute 呼叫與測試其他 LLM

## 這個 skill 解決什麼問題

`pi`（見 `pi-coding-agent-setup` skill）在**這台機器上**因為 `~/.pi/agent/settings.json` 設了
`defaultProvider: "omni"`，所以預設走 OmniRoute 閘道器——注意這是**這台機器的設定檔覆寫**，不是
pi 這支 CLI 工具本身的內建預設（`pi --help` 裡 `--provider` 選項寫的內建預設是 `google`）。換一台
沒設過這個 settings.json 的機器，裸打 `pi` 不加 `--provider` 可能會打到完全不同的 provider，保險
起見批次測試/寫腳本時**建議永遠明確帶 `--provider omni`**，不要依賴預設值。
OmniRoute 底下聚合了非常多 provider/model（Claude、GPT、DeepSeek、以及首頁「免鑑權提供者」那些不用
註冊的免費端點）。這份 skill 記錄「怎麼實際把工作或問題丟給指定的另一顆模型」的兩種做法、怎麼列出/
對應可用模型、以及一路踩過的坑。

**`--provider omni` vs 直接指定其他 provider（例如 `openai-codex`、`anthropic`）不是同一件事**：
`omni` 是 OmniRoute 這個閘道器本身，底下聚合一堆免費/付費端點（`oc/`、`ddgw/`、`felo/` 等前綴都是
`omni` 底下的 model id）；`openai-codex` 則是**繞過 OmniRoute、直接用你 ChatGPT/Codex 帳號的 OAuth
登入**，兩者是平行的兩種 provider，`pi --provider openai-codex --model gpt-5.5` 裡的
`gpt-5.5` **不是** OmniRoute 的 model id。用 `pi auth check --provider <name> --json` 可以確認某個
provider（不管是 `omni` 還是 `openai-codex`）目前是否 ready。

## 前置條件

- **方式一（herdr agent）只能在目前這個 AI session 本身就跑在 Herdr 管理的 pane 裡才能用**，操作前先確認
  （這行是 bash 語法，在 Bash/Git Bash 工具裡跑；PowerShell 裡直接看 `$env:HERDR_ENV` 是否為 `1`）：
  ```bash
  test "${HERDR_ENV:-}" = 1 && echo "在 Herdr 裡" || echo "不在 Herdr 裡，改用方式二"
  ```
  不在 Herdr 環境裡就直接用方式二，不要嘗試操控不存在的 pane。
- 兩種方式都需要 `pi` 已安裝且 `pi`/`pi.cmd` 在 PATH（見 `pi-coding-agent-setup` skill）。
- 走 `--provider omni` 的話，OmniRoute 本身要在跑（預設 `http://localhost:20128`），可以先探測一下：
  ```bash
  curl -s http://localhost:20128/v1/models -o /dev/null -w "%{http_code}\n"
  ```
  印出 `000` 代表根本連不上（OmniRoute 沒開/網路不通），印出非 2xx 的其他碼代表服務有回應但有問題；
  先確認 OmniRoute 服務狀態，不要一開始就懷疑是模型或 pi 的問題。
- 要用瀏覽器工具（見下方「讓 pi agent 用瀏覽器工具」一節）才需要 Chrome CDP 9222 + `mcp.json` 設定，
  純文字問答不需要。
- `--provider openai-codex` 首次使用需要先完成 ChatGPT/Codex 帳號的 OAuth 登入授權（互動模式下 pi
  會引導你開瀏覽器登入一次），登入過一次後 token 會存在 `~/.pi/agent/auth.json`，之後才能像本 skill
  範例一樣直接 `-p --no-session` 呼叫；沒登入過的情況下第一次一定要用互動模式（方式一）走完授權流程。

## 快速決策表

| 需求 | 選哪個 |
|---|---|
| 要它做一件比較長、可能來回好幾輪的任務（例如 SSH 進遠端主機調查） | 方式一：herdr agent |
| 同一個問題丟給好幾顆模型，要公平比較（乾淨起點、不互相污染） | 方式二：`pi -p --no-session` |
| 只是想確認某顆模型「連不連得上、答不答得出來」 | 方式二 + 下面的「最小 smoke test」 |
| 不在 Herdr pane 裡執行 | 只能用方式二 |
| 有好幾個**互相獨立**的子任務，想同時分派出去平行做 | 方式三：多個 pi 並行當 subagent |
| 好幾個子任務會動到**同一個**遠端資源/檔案（例如同一台主機的同一個 compose 專案） | 不要平行，照方式一序列化一個一個做，避免互踩 |

## 兩種呼叫方式，怎麼選

### 方式一：herdr agent（常駐、互動、有對話記憶）

適合「這顆模型要持續做一件比較長的任務」（例如叫它 SSH 進遠端主機做調查、或要來回好幾輪對話）。

```bash
# 1. 切個新 pane（寬 pane 往右切，窄/高 pane 往下切）
herdr pane split --current --direction right --cwd "$PWD" --no-focus
# 從回傳 JSON 的 .result.pane.pane_id 取得 pane id（範例假設拿到 w1:p3，不要照抄，要讀實際回傳值）

# 2. 在該 pane 啟動 pi，指定 provider/model
#    Windows 上 herdr agent start 直接呼叫 Start-Process -FilePath pi 會炸掉，見下方「踩坑」，
#    所以改用 pane run 手動下指令，不要用 `herdr agent start`：
herdr pane run w1:p3 "pi.cmd --provider openai-codex --model gpt-5.5"
#   ↑ 這裡走的是「直接 OAuth 到你 ChatGPT/Codex 帳號」，不是 OmniRoute；
#   若要測 OmniRoute 底下的模型（例如免費端點），改成：
#   herdr pane run w1:p3 "pi.cmd --provider omni --model oc/deepseek-v4-flash-free"

# 3. 等 TUI 啟動好之後，herdr 會自動偵測到 pi agent（herdr agent list 看得到），
#    之後就能用 agent 語意的指令跟它對話：
herdr agent prompt w1:p3 "你的任務內容" --wait --timeout 120000
herdr agent read w1:p3 --source recent-unwrapped --lines 150

# 4. 用完記得收尾，常駐 pane 不會自己消失，會一直佔資源：
herdr pane close w1:p3
```

- **`herdr agent prompt --wait` 有 5 秒判定窗**：如果送出 prompt 時 agent 不是 `working` 狀態，
  必須在 5 秒內觀察到狀態變化，否則直接回傳 `agent_prompt_stalled` 而不是繼續等；免費模型第一個
  token 常常超過 5 秒，這種情況下改成不帶 `--wait` 送出、再另外呼叫 `herdr agent wait <pane>
  --timeout <ms>` 分開等待比較保險（本 skill 撰寫過程中就實際遇過一次 `--wait` 逾時，但用
  `herdr agent wait` 補等之後任務其實還在正常跑，並沒有卡死）。

- 對話歷史會累積在同一個 session 裡，**適合連續多輪任務**，但也代表如果你想拿「乾淨的起點」跟另一顆模型
  公平比較，這個模式不合適（後面問的模型會看到前面模型答過的內容，context 也會一直漲）。
- 中途要換模型：`herdr agent send-keys <pane> ctrl+p` 會**循環切到清單裡的下一顆模型**（pi `--help`
  裡對 `--models` 參數的說明是「Comma-separated model patterns for Ctrl+P cycling」，實測按一次就是跳
  到下一顆，不是開一個要再輸入文字篩選的選擇器），不是精準選擇，不能拿來指定切到某一顆特定模型；
  想精準指定，直接照上面步驟 2 重開一個新的 pi 進程帶 `--model` 最可靠。

### 方式二：直接一次性 CLI 呼叫（乾淨、無狀態，適合批次比較）

適合「同一個問題丟給好幾顆模型比較誰答得好」這種場景。不需要 herdr pane：

CMD 開 Luna（互動）：

```bat
pi --provider openai-codex --model gpt-5.6-luna
pi --provider omni --model codex/gpt-5.6-luna-high
```

前者直連 ChatGPT/Codex OAuth；後者走 OmniRoute。Pi 斜線是 `/model gpt-5.6-luna`，不是 CMD。

```bash
# Bash / Git Bash：直接打 pi
pi --provider omni --model oc/deepseek-v4-flash-free -p "你的問題" --no-session
```

```powershell
# PowerShell：要用 pi.cmd（原因見下方「Windows 上的已知踩坑」第 2 點）
pi.cmd --provider omni --model oc/deepseek-v4-flash-free -p "你的問題" --no-session
```

- `-p`（`--print`）：非互動模式，處理完 prompt 直接把回答印到 stdout 然後結束進程。**過程中仍會
  呼叫工具**（包含 MCP 工具），不是純文字模式；若卡在需要人工 approve 的權限關卡，非互動模式下可能
  會卡住或直接失敗，遇到這種狀況才需要改回方式一的互動模式手動處理。
  > 有其他 AI 審閱這份文件時猜測「`oc/` 這類免費端點通常不支援 tool calling，所以瀏覽器自動化範例
  > 對它們可能無效」——**這個猜測已被本 skill 的實測推翻**：`oc/hy3-free`、`oc/deepseek-v4-flash-free`
  > 都實際成功呼叫過檔案讀寫工具跟 Playwright 瀏覽器工具（見下方「讓 pi agent 用瀏覽器工具」一節的
  > 範例就是拿 `oc/hy3-free` 真的跑出來的），免費端點能不能用工具要看該模型實際支不支援 tool calling
  > API，不能直接假設「免費 = 不能用工具」。
- `--no-session`：不寫入 session 檔，每次呼叫都是全新、乾淨的 context，不會被前面問過的模型/問題污染，
  這是能公平比較多顆模型的關鍵。
- 涉及瀏覽器工具、多輪 MCP 呼叫時可能要跑 2~5 分鐘（尤其免費模型本身推論就慢），逾時設寬一點
  （用 Bash 工具的 `timeout` 參數，或直接 `run_in_background: true` 丟到背景跑，完成會自動通知）。
- 批次比較很多顆模型時，建議把每顆的輸出導到各自的檔案，避免混在同一份終端輸出裡難以對照：
  ```bash
  pi --provider omni --model oc/deepseek-v4-flash-free -p "$PROMPT" --no-session > /tmp/oc-deepseek.txt 2>&1
  ```

### 最小 smoke test（先確認 pi / OmniRoute / model id 都通）

批次測試一大串模型前，先用一個秒回的問題確認基礎設施沒問題，避免把「連不上」誤判成「這題目太難」：

```bash
pi --provider omni --model oc/deepseek-v4-flash-free -p "只回答兩個字：OK" --no-session
```

### 方式三：多個 pi 並行當 subagent 協同工作

當手上有**好幾個互相獨立**的子任務（例如「同時查兩台不同主機的狀態」「同一個問題丟給好幾顆模型比較」
「一批獨立檔案各自要做分析」），可以比照方式一，開好幾個 herdr pane，每個 pane 各跑一個 pi，當成一群
可以真的同時工作的 subagent，而不是一個一個排隊做。跟 Claude Code 內建的 Agent tool（subagent）是同一種
概念，只是換一套機制（herdr pane + pi CLI）實作，操作起來完全可見（使用者看得到每個 pane 在做什麼）。

**固定版面規則：先切一個垂直 pane，再在這個 pane 裡往下切橫向 pane，橫向最多 3 個。**
不要從原本 pane 連續往右切，否則會變成多欄而不是同一欄內的橫向工作區。

```bash
# 版面簡寫：1 欄、3 個橫向 pane
/panel c1 r3

# 3. 確認每個都被偵測成 pi agent（idle 狀態）
herdr agent list

# 4. 【關鍵】送出任務時不要對每個 pane 都用 --wait 依序等，那樣會退化成序列執行、
#    完全失去平行的意義。正確做法二選一：
#    (a) 全部用 run_in_background: true 包住 Bash 呼叫，讓每個 --wait 各自在背景等，
#        Claude 這邊會在全部完成時各自收到通知，等於同時在跑：
herdr agent prompt <pane_A> "子任務 A 的內容" --wait --timeout 120000    # 包在 run_in_background 裡
herdr agent prompt <pane_B> "子任務 B 的內容" --wait --timeout 120000    # 也包在 run_in_background 裡
#    (b) 或者全部先不帶 --wait 送出（fire-and-forget），之後用 herdr agent list
#        輪詢每個 pane 的 agent_status 是否從 working 變成 idle，再逐一 herdr agent read。

# 5. 各自收工後讀結果、彙整
herdr agent read <pane_A> --source recent-unwrapped --lines 200
herdr agent read <pane_B> --source recent-unwrapped --lines 200

# 6. 全部收尾，逐一關閉 pane，不要留著佔資源
herdr pane close <pane_A>
herdr pane close <pane_B>
```

- **能不能平行，先看子任務之間會不會互踩**：純查詢/唯讀（例如「查 A 主機」+「查 B 主機」、「同一題丟
  給 3 顆模型比較」）幾乎都能安全平行；但如果多個子任務會**寫入/修改同一個目標**（例如都要改同一台主機
  上同一個 docker compose 專案），平行執行有競態風險（兩個 pi 同時 `docker compose up`、同時改同一個檔案），
  這種情況改回方式一序列化處理，一個做完確認沒問題，再做下一個。
- **`--wait` 逾時不等於任務失敗**：實測中 `herdr agent prompt --wait --timeout <N>` 常常在 Bash 工具層
  自己先逾時噴 `{"error":{"code":"timeout","message":"timed out waiting for agent status"}}`，但這只代表
  「這次等待動作本身超過設定時間」，不代表 pi 那邊卡死或任務失敗——遇到這個錯誤，先 `herdr agent list`
  看該 pane 的 `agent_status`，如果還是 `working` 就代表任務仍在正常進行，改用 `herdr agent wait <pane>
  --timeout <N>`（一樣包 `run_in_background: true`）繼續等，不要因為看到 timeout 就誤判失敗、砍掉重來。
  多個 pane 平行跑、且任務本身牽涉多輪 SSH/工具呼叫時（例如逐一驗證好幾個服務的 health check），單次
  `--wait` 逾時是常態，不是異常。
- **每個 pane 各自累積獨立的對話歷史**，彼此互不污染（跟方式一單一 pane 的行為一致），這也是「當 subagent
  群」這個比喻成立的原因——各自有自己的 context，只是共用同一台機器的 pi/herdr 基礎設施。
- 子任務數量抓 2~4 個是比較實際的範圍；開太多 pane 一來使用者畫面會很亂，二來如果子任務最終都要
  SSH 到同一台遠端主機，過多平行連線本身也可能造成負擔或互相干擾。

### 固定流程：可見多模型 runner column

當使用者要求「讓多個 LLM 同時工作、要看得到過程」時，預設採用以下流程，而不是只做背景 `pi -p`：

1. 先確認目前是在 Herdr 管理的 pane（`HERDR_ENV=1`），並用 `herdr pane current --current` 取得原始 pane。
2. 從原始 pane **向右建立一個獨立 column**；後續只在這個右側 column 內向下分割。不要把左側主 pane 切開。
3. 每個模型一個 pane，右側 pane 依序上下排列，並用 `herdr pane layout --current` 驗證高度相同或只差 1 列。
4. 用 `herdr pane run <pane> "pi.cmd --provider omni --model <model-id> ..."` 啟動；Windows 優先使用 `pi.cmd`，不要依賴 `pi.ps1`。
5. 若只是執行單一任務，臨時 runner pane 啟動時加 `--env PI_DISABLE_HUD=1`。這只關閉該 pane 的 Pi HUD/footer，保留模型輸出與必要功能，不影響左側主 pane。
6. 從回傳 JSON 讀取實際 pane ID，不自行推算 ID；所有模型啟動後再用 `herdr agent prompt <pane> ... --wait --timeout <N>` **並行**送出任務。
7. 任務完成後先讀取各 pane 的 `recent-unwrapped` 輸出並確認 agent 已進入 `idle`／`done`，再逐一執行 `herdr pane close <created-pane-id>`。只關閉本次建立的 pane，不關閉使用者原本的 pane。

範例（兩個模型、右側上下等高、完成後清理）：

```bash
# 右側 column：從回傳 JSON 讀出 RIGHT_ID
herdr pane split --current --direction right --ratio 0.5 --no-focus --cwd "$PWD" --env PI_DISABLE_HUD=1
# 只對 RIGHT_ID 向下分割；必要時用 pane layout/resize 調成 50/50
herdr pane split --pane "$RIGHT_ID" --direction down --ratio 0.5 --no-focus --cwd "$PWD" --env PI_DISABLE_HUD=1

herdr pane run "$TOP_ID" "pi.cmd --provider omni --model codex/gpt-5.6-luna-high"
herdr pane run "$BOTTOM_ID" "pi.cmd --provider omni --model codex/gpt-5.6-sol-high"

# 兩個 prompt 要同時送出；不可一個 --wait 完成後才送下一個
herdr agent prompt "$TOP_ID" "任務 A" --wait --timeout 120000 &
herdr agent prompt "$BOTTOM_ID" "任務 B" --wait --timeout 120000 &
wait
herdr pane read "$TOP_ID" --source recent-unwrapped --lines 200
herdr pane read "$BOTTOM_ID" --source recent-unwrapped --lines 200
herdr pane close "$TOP_ID"
herdr pane close "$BOTTOM_ID"
```

> 注意：`PI_DISABLE_HUD=1` 應透過 `herdr pane split --env PI_DISABLE_HUD=1` 傳入臨時 pane 環境。這是本機 HUD
> extension 支援的 runner opt-out，不要為了這個需求修改全域 HUD 設定。若要保留特定功能，改用 `--no-skills`、
> `--mcp-config` 或明確 `--skill` 控制資源，不要任意移除主 pane 的設定。

## 本機自建 provider（非 OmniRoute 內建的已連接服務，例如 llama.cpp/vLLM）要先加進 OmniRoute 才能選到

若要測試的不是 OmniRoute 內建的 provider，而是自己部署的本機/遠端服務（例如自己跑的 llama.cpp、vLLM），
先進 OmniRoute 網頁後台把它登記成一個 OpenAI 相容節點，重點有兩個容易漏掉的步驟：

1. **只加 Provider Node 不夠，還得再加一層 Connection（API 金鑰）**，否則 `/v1/models` 不會列出這顆模型，網頁上
   該 provider 會顯示「0 個連線」。llama.cpp 本機部署通常不需要真的驗證，API key 欄位隨便填一個字串即可。
2. **OmniRoute 加好不代表 pi 馬上看得到**，必須在 pi 的 TUI 對話輸入框手動打：
   ```text
   /omni sync
   ```
   才會重新同步本地 model catalog（`~/.pi/agent/models.json`），之後 `pi --list-models <prefix>` 才查得到。不要看到
   OmniRoute 已新增完就假設 pi 自動同步。
   > 注意：若在 Herdr pane 裡用 `herdr pane run <pane> "/omni sync"` 送這個指令，要加 `MSYS_NO_PATHCONV=1`，
   > 原因見下方「Windows 上的已知踩坑」第 5 點。

## 怎麼列出可用模型

```bash
pi --list-models <關鍵字>      # 模糊搜尋，例如 pi --list-models gpt
pi --list-models "oc/"         # 找特定 provider 前綴（注意這是模糊比對，不是嚴格 prefix 過濾）
pi update --models             # 重新整理 model catalog（omniroute-pi-ext-integration 同步進來的）
pi auth check --provider <name> --json   # 確認某 provider 的憑證/連線是否 ready
```

## OmniRoute 首頁「免鑑權提供者」卡片 ↔ pi model 前綴對照表

OmniRoute 網頁後台（`http://localhost:20128/dashboard/providers` 或首頁的「免鑑權提供者」卡片區）列出
一批不用註冊、開放使用的免費端點。卡片上的顯示名稱跟 pi 這邊 `--list-models` 看到的 provider 前綴**不是
同一套命名**，實測對照如下（**2026-08-13 的快照，不保證長期有效**——provider 清單、model id、連線狀況
都可能隨時變動，使用前務必先用 `pi --list-models` 或最小 smoke test 重新確認一次，不要直接照表操課）：

| OmniRoute 卡片名稱 | pi model 前綴 | 備註 |
|---|---|---|
| OpenCode Free | `oc/` | 目前有 6 顆：`deepseek-v4-flash-free`、`hy3-free`、`mimo-v2.5-free`、`nemotron-3-ultra-free`、`north-mini-code-free`、`big-pickle` |
| DuckDuckGo AI Chat | `ddgw/` | 例如 `ddgw/claude-haiku-4-5`、`ddgw/gpt-5.4-mini` |
| Felo | `felo/` | 例如 `felo/felo-chat`、`felo/felo-search`、`felo/felo-document` |
| Chipotle Pepper AI (Free) | `pepper/pepper-1` | 只有這一顆 |
| MiMoCode (Free) | `mcode/mimo-auto` | 只有這一顆；**注意跟 `oc/mimo-v2.5-free` 不是同一個東西**，`mcode/` 和 `oc/` 是兩個不同 provider，只是剛好都叫 mimo |
| The Old LLM (Free) | `tllm/` | 前綴下掛了一大票代理模型（`GPT_5`、`claude_sonnet_4`、`gemini_3_pro`、`openrouter_*` 等），但這些都是同一個底層 provider，壞掉是整批一起壞 |
| Veo AI Free | `veo-free/` | 純影片生成（`veo-free/veo`、`veo-free/seedance`），不是文字對話模型，不能拿來測文字任務 |
| AI Horde | 目前找不到 | `pi --list-models` 完全搜不到，`pi update --models` 刷新後也沒有。**以下是未實際驗證過的推測**：可能要先在 OmniRoute 網頁上手動按「測試」啟用連線才會同步進 pi 的 catalog——真的要用這顆時務必先按網頁上的測試按鈕再重新整理 catalog，不要假設這個推測一定對 |
| Augment (Auggie CLI) | 目前找不到 | 同上，同樣是未驗證的推測 |

## 免費模型常見的失敗模式（基礎設施層問題，不代表模型能力差）

實測下來，免費/無鑑權端點大部分的問題出在「連不連得上」而不是「答得好不好」：

| 錯誤 | 意思 | 該怎麼看待 |
|---|---|---|
| `429 rate_limit_exceeded` | 免費額度/QPS 被打滿 | 通常可以隔一陣子（幾十秒到幾分鐘）重試，但也可能持續失敗一整段時間 |
| `503 chat_admission_busy` | 服務端當下滿載 | 跟 429 類似，重試看運氣 |
| 逾時無回應（進程 timeout） | 可能是真的在算、也可能是掛住 | 拉長 timeout 重試一次，還是不行就換模型，不要無限重試 |
| `401 Model X is not supported` | 通常代表本地 catalog 過期，或後端已下架這個 model id | 先 `pi update --models` 重新整理 catalog 再試一次；再失敗就視為這顆目前不可用，換別顆，不用一直重試 |
| `403 ... blocked by Vercel for this server egress IP` | 整個 provider 的 server 端出口 IP 被目標網站擋了 | provider 層級問題，換哪個底層模型都一樣會擋，別再試同一個 provider 的其他 model |
| `502 bad_gateway` | 上游服務本身連不上 | 服務本身掛了，換模型 |

## 讓 pi agent 用瀏覽器工具瀏覽網站並回報

先確認 `~/.pi/agent/mcp.json` 有掛 `playwright` server（指到 CDP `http://127.0.0.1:9222`，設定方式見
`connect-chrome` skill 的 pi 那一段），並確認 debug Chrome 的 9222 port 已開（`connect-chrome` skill
步驟 1）。接著不管用方式一還是方式二，prompt 裡明確要求「真的呼叫工具」，模型才不會用猜的敷衍：

```text
請用你可以用的瀏覽器/playwright工具，連線到已經開啟的 Chrome（CDP 已接管），開一個新分頁導航到
<URL>，然後實際瀏覽這個網站，告訴我有哪些功能/選單/頁面。請務必真的呼叫工具去看網頁內容，不要用猜的。
看完後不要關掉分頁。
```

這類任務因為要跑好幾輪工具呼叫（導航、截圖/snapshot、點選單），免費模型常常要 2~5 分鐘，務必用背景執行
或設寬鬆 timeout，不要以為卡住了就砍掉重來。

## 出題技巧：怎麼問才比得出模型程度差異

單純叫模型「寫檔案」「做加法」這種任務型指令，只要模型還算聽得懂中文、工具呼叫沒壞，幾乎每顆免費模型
都能做到，比不出真正的推理/知識深度差異。要真的看出差距，題目要滿足：

- **需要跳脫直覺的推理**，不是查表/背誦就能矇對（例如經典的「三開關三燈泡只能進房一次」燈泡溫度題，
  比單純數學計算更能篩出會不會「多想一步」）
- **有明確、可驗證的正確答案**，方便你事後逐一核對每顆模型的回答對不對，而不是憑印象打分
- 如果是拿來當 coding agent 用，額外測「多步驟指令 + 精確工具操作」（例如「新建檔案但不要動到另一個
  已存在的檔案」這種細節要求），比單純問答更貼近實際會被交付的任務型態
- **經典題（例如「三開關三燈泡」）容易被模型在訓練資料裡背過答案**，答對不一定代表真的在推理。正式
  要嚴謹比較時，最好把經典題改幾個條件（例如燈泡數量、多加一個限制）做成變形題，或乾脆自己出一題，
  避免測到的其實是記憶力而不是推理力

## Windows 上的已知踩坑

1. **`herdr agent start` 在 Windows 會炸**：底層用 `Start-Process -FilePath pi`，但 `pi` 是 npm 產生的
   `.ps1`/`.cmd` shim 不是原生 exe，會報「%1 不是有效的 Win32 應用程式」。解法：不要用
   `herdr agent start`，改用 `herdr pane run <pane> "pi.cmd --provider ... --model ..."` 手動啟動，
   herdr 會自動偵測到這是 pi agent。
2. **PowerShell 執行原則封鎖 `pi.ps1`**：如果這台機器的指令碼執行原則是停用/受限狀態，PowerShell 裡
   直接打 `pi` 會報「因為這個系統上已停用指令碼執行，所以無法載入 ...pi.ps1」。解法：改打 `pi.cmd`
   （批次檔包裝，不受這個限制）。Git Bash（Bash 工具）不受影響，可以直接打 `pi`。
3. **一旦某個 PowerShell pane 因連續指令失敗，殘留的多行錯誤訊息可能被逐行當成新指令「回放」執行**，
   畫面會變成一長串無關的 parser error，越修越亂。解法：`herdr pane send-keys <pane> enter` 後
   `herdr pane run <pane> "cls"` 清畫面重新開始，不要在髒掉的畫面上繼續疊指令。
4. **`herdr pane read` 的 `agent_status` 有時會短暫顯示 pane 標題變成 `Windows PowerShell`**（看起來
   像 pi 已經退出/當掉），但實際上只是偵測時機問題，真正結果要用 `herdr pane read --source
   recent-unwrapped` 讀內容確認，不要只看 `terminal_title` 就判斷 agent 已經死掉。
5. **`herdr pane run <pane> "/exit"`（或任何以 `/` 開頭的 pi 指令）在 Git Bash 裡會被 path conversion 改寫**：
   實測送進去的實際字串變成 `C:/Program Files/Git/exit`，pi 拿到的不是 `/exit` 指令而是一堇文字，會被當成一般
   prompt 送去模型回答（或在模型已噴錯時堆進重試佇列）。解法：在這類指令前加 `MSYS_NO_PATHCONV=1` 關掩 Git Bash 的
   自動路徑轉換：
   ```bash
   MSYS_NO_PATHCONV=1 herdr pane run <pane> "/exit"
   ```
   同樣道理也適用於 `herdr agent prompt <agent> "/omni sync"` 這類具有 `/` 前綴的命令。
6. **Pane 卡在舊的重試佇列時，送 `/exit` 也會被堆進佇列排在最後面，不會立即生效**：如果前一輪命令（例如上一個問题）跟下游
   服務連不上（429/502 一直 retry），`agent_status` 會長時間保持 `working`，這時候先 `send-keys esc` → 確認
   `idle` → 再送 `/exit`，依然可能持續被舊的重試佇列告碸（發送後 依然進入 `working` 並印出上一堇問題的錯誤訊息）。
   **最簡單可靠的做法：不要跟 pi 內部佇列發提寗上，直接 `herdr pane close <pane>` 關掉整個 pane 再 `herdr pane
   split` 重開一個**，比等它自己消化完舊佇列快得多，也不用猜它到底還有幾個重試。

---

## Conformance Addendum

## When to Use
透過 pi coding agent + OmniRoute 閘道器，實際呼叫/測試/比較「另一顆 LLM」（例如 ChatGPT/openai-codex、 DeepSeek、Hunyuan、或 OmniRoute 首頁「免鑑權提供者」清單裡的免費端點）的三種操作方式： (1) 用 herdr agent 啟動一個常駐、可持續對話的 pi agent pane；(2) 直接一次性呼叫 `pi --provider omni --model <id> -p "..." --no-session` 拿乾淨、互不干擾的單次回答，適合批次比較多顆模型； (3) 同時開多個 herdr pane 各跑一個 pi，當成可平行工作的 subagent 群，分工處理多個獨立子任務。 涵蓋：`pi --list-models <關鍵字>` 查模型 id、OmniRoute 首頁「免鑑權提供者」卡片名稱對應的 pi model 前綴 對照表、Windows 上 herdr agent start / PowerShell 執行原則的已知踩坑、免費模型常見的 429/503/逾時/401/403 失敗模式、以及如何讓 pi agent 透過 connect-chrome 的 playwright MCP 去瀏覽網站並回報內容。 當使用者要求「叫另一個 LLM/agent 做事」「測試/比較不同模型能力」「用 pi 呼叫 ChatGPT/DeepSeek/其他免費模型」 「omniroute 那些免鑑權提供者能不能用」「讓 pi 用 subagent 的方式協同工作」「多開幾個 pi 平行處理」 「pi 分工/並行跑」時使用。

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
