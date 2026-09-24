---
name: qingjian-ime-development
description: 維護與打包 Qingjian／清箋跨平台中文輸入法（D:\qingjian-main）的專用流程。當使用者提到 Qingjian、清箋／清简、注音輸入法、拼音候選、Ctrl+數字選字、Windows TSF、macOS IMK、Rust 輸入法編譯、重新打安裝程式或升級現有安裝時，務必載入本 Skill；涵蓋架構定位、按鍵修改、Windows release build、Inno Setup 打包、安裝升級與驗證。
---

# Qingjian 跨平台輸入法開發與打包

## When to Use

使用者要求以下任一事項時使用：

- 修改 Qingjian／清箋輸入法的選字、注音、拼音、雙拼、候選窗或快捷鍵。
- 釐清輸入法功能目前由哪個 crate／平台外殼負責。
- 編譯 Windows Server、TSF DLL、設定程式，或建立新的 Windows 安裝程式。
- 將程式碼變更同步到安裝包，或讓使用者升級目前已安裝版本。
- 排查「修改後沒有生效」「只有數字能選字」「舊 DLL 被程式載入」「安裝後快捷鍵仍是舊行為」等問題。

## 使用者目前的輸入方案偏好

- 此使用者目前固定使用**大千注音輸入法**。後續 Qingjian 的功能修改、除錯、測試、編譯、安裝與回報，預設只處理並描述注音路徑。
- 回覆中不要主動提及全拼、雙拼或其他輸入方案；只有使用者明確要求，或共用 Core 變更會直接影響注音且必須交代時才提及。
- 效能與卡頓測試必須使用實際大千注音鍵序列，不得以其他輸入方案的 CLI 或基準測試結果代替。
- 長句、候選、選字、游標移動、聲調鍵、自動分段與安裝後驗收，全部以注音模式為預設驗收條件。
- 文件或程式中既有的跨方案設計不需要為此刪除；這是後續任務的聚焦規則，不是要求破壞產品原有能力。

## 專案事實與路徑

- 專案根目錄：`D:\qingjian-main`。
- 主要說明：`D:\qingjian-main\CLAUDE.md`，修改程式前完整閱讀；架構與提交約定在 `docs/contributing.md`。
- Core：`crates/qingjian-core`，包含候選生成、注音、拼音、排序與 Engine；不得放平台判斷。
- Windows：
  - `apps/windows/server`：唯一 Engine、IPC 與按鍵分派。
  - `apps/windows/tsf`：TSF DLL，只翻譯 Windows 按鍵並寫回文字。
  - `apps/windows/settings`：Windows 設定程式。
  - `apps/windows/installer`：Inno Setup 腳本與打包腳本。
- macOS：`apps/macos`，IMK 外殼；候選／按鍵處理在 `apps/macos/src/imk/controller`。
- 使用者設定與學習資料不在安裝目錄，通常位於 `%APPDATA%\Qingjian`；升級時不要任意刪除。

## 按鍵規則目前共識

- 一般選字：空白鍵選高亮候選，**左右鍵**移動高亮（到頁邊自動翻頁），`1–9` 選當頁對應候選。**上下鍵按音節（一個字）移動編碼光標**並刷新候選窗；不要再用上下鍵選字，也不要一次只跳一個拼音字母／注音符號。
- 跨平台固定快捷鍵：`Ctrl + 1–9`（macOS 顯示為 `⌃ + 1–9`）選取當頁第 N 個候選。
- Windows 譯詞快捷鍵不可再使用 `Ctrl + 數字`，因為該組合已固定給選字：
  - `Ctrl + Alt + 1–9`：第一個譯詞。
  - `Ctrl + Alt + Shift + 1–9`：第二個譯詞。
- Windows `Shift + 1–9` 預設仍為刪除候選；macOS 原有 `⌥`／`⇧⌥` 譯詞快捷鍵維持不變。
- `Ctrl + 數字` 只在組句中攔截；無組句時不能搶走應用程式原本的快捷鍵。
- 表達式模式（例如 `v2^3`）中的數字是算式內容，不應被當成選字快捷鍵。
- 注音模式的數字可能是聲調／注音鍵；修改選字時要確認 `is_zhuyin_mode`、`zhuyin_needs_tone` 與壳層分派規則，不能直接破壞注音輸入。
- 使用者要「方向鍵移到要改的字、彈出候選」：上下鍵呼叫 `Engine::move_cursor_syllable_left/right`（不是 `move_cursor_left/right` 逐字母）。遊標不在末尾時，候選只算光標前那段（`ni|hao` 出「你」）。注音用 `zhuyin::decode` 的 unit `keys` 一次跳完整音節（含聲調），`unit_len_before` 不得再把注音當 `plain` 逐符號跳。
- 實作位置：Core `crates/qingjian-core/src/engine/composing.rs`；Windows `apps/windows/server/src/dispatch/key/input.rs` 的 `UP`/`DOWN`；macOS `apps/macos/src/imk/controller/command.rs` 的 `moveUp:`/`moveDown:`。macOS `⌥←/⌥→` 仍是音節跳，與上下鍵相同單位。
- 驗證：`cargo test -p qingjian-core zhuyin_arrows`、`cargo test -p qingjian-windows-server --test engine_loop up_arrow`。使用者文件：`docs/user/getting-started/keys.md`、`first-input.md`。
- 改 Cargo.toml 版本後先不帶 `--locked` 更新 `Cargo.lock`，再跑 `build.ps1`（腳本用 `--locked`）。
- **原始碼過測 ≠ 已安裝**。裝完還要關掉再開記事本／瀏覽器，否則 TSF 仍是舊 DLL。

## Inputs and Outputs

輸入：使用者需求、現有程式碼、目前 target 產物與必要的錯誤訊息。

輸出：

1. 維持 Core／平台層邊界的程式修改。
2. 相應單元測試或整合測試。
3. 使用者可見按鍵變更同步到 `docs/user/getting-started/keys.md`、必要時同步 `docs/user/learning/translation.md`。
4. 若要求安裝包，產出 `target/installer/qingjian-<版本>-windows-x86_64-setup.exe`，並回報實際路徑、大小、雜湊與驗證結果。

## Procedure

### 1. 先確認現況

1. 讀取 `D:\qingjian-main\AGENTS.md`、`CLAUDE.md` 與 `docs/contributing.md`。
2. 搜尋相關按鍵與候選分派：
   ```bash
   rg -n "digit|candidate|translation|delete_candidate|KeyOutcome|index_on_page|zhuyin" \
     crates apps docs
   ```
3. 先確認是 Windows、macOS，還是兩端都要改；不要只改 Server 而忘記 TSF DLL 或 macOS IMK。
4. 讀取現有 `config.toml`／`Config::default()` 與安裝腳本，確認使用者自訂快捷鍵是否會與新固定鍵衝突。

### 2. 修改按鍵時遵守分層

- Windows Server 的核心分派放在 `apps/windows/server/src/dispatch/key/`。
- Windows TSF 的 `would_eat`／`eats_key` 必須與 Server 的分派一致；若 TSF 不先吃 `Ctrl+數字`，按鍵會漏到應用程式，Server 根本收不到。
- macOS 在 `apps/macos/src/imk/controller/mod.rs` 以 key code 與修飾鍵分派；指定候選的上屏邏輯放在既有 `commit.rs`，不要把排序或詞庫邏輯搬進外殼。
- 修改 Windows 預設快捷鍵時，同步：
  - `crates/qingjian-platform/src/config/modifiers.rs` 的修飾鍵常數。
  - `crates/qingjian-platform/src/config/shortcut.rs` 的預設值與衝突規則。
  - `apps/windows/server/src/dispatch/config.rs` 的協議／Router 設定映射。
  - Windows 設定程式的快捷鍵選項，例如 `apps/windows/settings/src/panel/pages/shortcut.rs`。
  - 測試 helper 的預設快捷鍵，避免測試仍以舊 `Ctrl` 譯詞鍵送事件。
- 固定快捷鍵應優先於可配置譯詞快捷鍵判斷，否則同一個 `Ctrl+數字` 可能同時被解讀為選字與譯詞。

### 3. 測試與編譯

先設定可用的 Rust toolchain。專案 `rust-toolchain.toml` 目前指定 `1.96.0`，若目前 shell 找不到 Rust，先確認 cargo/rustc/rustfmt 的實際路徑，不要直接宣稱無法編譯。

最低驗證：

```powershell
cargo check --locked -p qingjian-windows-server -p qingjian-windows-tsf -p qingjian-windows-settings
cargo test --locked -p qingjian-windows-server
cargo fmt --all -- --check
```

Windows release 安裝包還需要 32 位 DLL target：

```powershell
rustup target add i686-pc-windows-msvc
```

建議針對按鍵規則執行：

```powershell
cargo test --locked -p qingjian-windows-server --test engine_loop
```

至少確認：

- `Ctrl+1` 會提交當頁第一個候選，而不是第一條譯詞。
- Windows `Ctrl+Alt+1` 仍能提交第一個譯詞。
- 沒有候選時 `Ctrl+數字` 不會把字元漏進應用程式。
- 非組句時 `Ctrl+數字` 不被輸入法攔截。
- 第二頁的 `Ctrl+1` 會選第二頁第一個候選。
- 注音模式的數字聲調規則沒有被一般選字快捷鍵破壞。

### 4. 建立 Windows 安裝程式

正式打包使用專案腳本，不要手動只編 Server 或只複製 DLL：

```powershell
Set-Location D:\qingjian-main
$env:QINGJIAN_ISCC = 'C:\Users\Administrator\AppData\Local\Programs\Inno Setup 7\ISCC.exe'
pwsh -NoProfile -ExecutionPolicy Bypass -File apps\windows\installer\build.ps1
```

腳本會：

1. release 編譯 64 位 Server、TSF DLL、Settings。
2. 編譯 `i686-pc-windows-msvc` 32 位 TSF DLL。
3. 準備 Windows App Runtime。
4. 讀取 `apps/windows/server/Cargo.toml` 版本。
5. 用 `apps/windows/installer/qingjian.iss` 產生安裝程式。

若只修改程式或安裝腳本、release 產物已確認是最新，可使用：

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File apps\windows\installer\build.ps1 -SkipBuild
```

`-SkipBuild` 只有在 release 產物確實包含最新修改時才能使用；否則會把舊二進位檔重新封裝成看似新的安裝包。

Inno Setup 可透過 `QINGJIAN_ISCC` 指定；不可因 PATH 找不到 `iscc.exe` 就判定打包流程壞掉。

### 5. 安裝與升級

- 通常不需要先移除舊版；安裝程式支援升級，會保留 `%APPDATA%\Qingjian` 使用者設定與學習資料。
- 安裝程式讓 DLL 按版本並排安裝並重新註冊 TSF；不要自行對舊版 DLL 執行 `regsvr32 /u`，以免把整個 CLSID／輸入法註冊移除。
- TSF DLL 會被載入每個有文字輸入框的應用程式。安裝完成後，至少關閉並重新開啟要測試的程式；若仍載入舊 DLL，可登出／重新登入或重開機。
- 若要快速測試、不想直接改正式安裝，使用：
  ```powershell
  pwsh -NoProfile -ExecutionPolicy Bypass -File apps\windows\scripts\test-local.ps1
  ```
  這會把已安裝目錄複製到 `%USERPROFILE%\qingjian-devtest`，替換 Server／DLL 並重新註冊，避免直接破壞 `Program Files` 安裝。
- 編譯前若 `qingjian_tsf.dll` 被宿主程式載入，先切到其他輸入法並關閉記事本、瀏覽器、IDE 等宿主；用 `tasklist /m qingjian_tsf.dll` 查鎖定者。

### 6. Windows 懸浮狀態列與右鍵功能選單

Windows 的懸浮狀態列不是 Windows 設定程式本身，而是由 Server UI 執行緒建立的原生分層視窗：

- 狀態列視窗：`apps/windows/server/src/ui/status/mod.rs`
- 擺放與滑鼠事件：`apps/windows/server/src/ui/status/placement/mod.rs`
- 狀態列配置：`[status_bar] enabled = true`、`x`、`y`
- 設定程式入口：`apps/windows/settings/src/main.rs`
- 設定頁初始分頁：`apps/windows/settings/src/panel/component.rs` 的 `Component::Input` 與 `--page` 參數

狀態條切換按鈕右側會顯示「訓練 N」。數量定義、開始個人訓練與未簽名 Server 的部署限制見 `qingjian-personal-training`，不要在本 Skill 另寫一套計數規則。

目前右鍵選單應包含：

- 開始個人訓練

- 打開設置
- GPT / AI 設置（啟動 `qingjian-settings.exe --page cloud`）
- AI 協助選字（打勾＝`[predict] enabled`，開啟時一併打開 `auto_correction` 與 `sentence`）
- 切換中 / 英
- 切換全角 / 半角標點

狀態列格子固定為 `[中/英][。][AI][⚙]`：標點格永遠顯示「。」，亮起＝全形、灰色＝半形，不要再用 `，。` / `,.` 切換文字，否則寬度會跳。

新增或修改右鍵選單時：

1. 在狀態列 window procedure 處理 `WM_RBUTTONUP`，用螢幕座標呼叫 `TrackPopupMenu`。
2. 建立 popup menu 後必須 `DestroyMenu`；選單關閉後補送 `WM_NULL`，避免滑鼠訊息被系統選單吃掉。
3. 不要讓右鍵選單搶走文字輸入焦點；狀態列仍應保留 `WS_EX_NOACTIVATE` 與 `WM_MOUSEACTIVATE = MA_NOACTIVATE`。
4. 若右鍵後要開特定設定頁，設定程式必須接受 `--page <tag>`，而且 `Component::Input`、`create`、`view` 的型別要一致。
5. 懸浮狀態列預設可以是關閉的；測試右鍵前確認目前使用者的 `%APPDATA%\Qingjian\config.toml` 有：
   ```toml
   [status_bar]
   enabled = true
   ```
6. 修改 Server UI 後必須重新編譯與重啟 `qingjian-server.exe`；若狀態列仍不存在，先查 Server 版本與狀態列設定，不要先判定右鍵事件失效。

### 7. GPT / DeepSeek 等 OpenAI 相容服務

GPT、DeepSeek 或其他 OpenAI 相容服務的設定集中在 `PredictConfig`：

- 設定資料結構：`crates/qingjian-predict/src/config.rs`
- HTTP / chat client：`crates/qingjian-predict/src/chat_client.rs`
- Windows 設定頁：`apps/windows/settings/src/panel/pages/cloud.rs`
- 設定訊息與落盤：`apps/windows/settings/src/panel/message.rs`、`component.rs`

本機私人模型（HCH，優先於公網 DeepSeek）。8080 上一次只會有一顆 GGUF（27B／30B-A3B／Spark 輪流，見 `qwen-spark-docker-desktop-switch`）。

2026-09-21 起清箋實際接入的是 30B-A3B：

```toml
[predict]
enabled = true
base_url = "http://10.145.119.19:8080/v1"
model = "/models/Qwen3-30B-A3B-Instruct-2507-Q4_K_M.gguf"
max_tokens = 512
timeout_ms = 20000
debounce_ms = 300
reasoning_effort = "none"
sentence = true
auto_correction = true
```

切回 27B 容器後改成 `/models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf`。

- 模型 id **必須**先 `GET http://10.145.119.19:8080/v1/models` 對過再寫。寫成 `QW//models/...` 會連得上服務但選不到模型。
- `[predict].model` 留空或填 `auto` 時，`qingjian-predict` 會查詢 `{base_url}/models`，使用回傳的第一個非空模型 ID，並在該客戶端生命週期內快取；若服務沒有模型則回報 `API returned no available models`。
- Windows「雲服務」設定頁的「模型」現在是下拉式選單：第一項「自動選擇」，其餘由「重新整理」按鈕從 `/models` 動態載入；選定模型立即寫回 `%APPDATA%\Qingjian\config.toml`，不應手動猜測模型名稱。
- 模型清單查詢走背景執行緒，不阻塞設定視窗；和聊天請求共用設定的 timeout。查詢模型不要求 API key（但實際聊天仍須依服務需要提供）。
- 不要用 Pi 的 `PI_MODEL`（例如 `xai/grok-4.6`）判斷輸入法；看 `%APPDATA%\Qingjian\config.toml` 與 `Local\Qingjian\logs\server.*.log` 的 `雲聯想已接入 model=...`。
- 這台是 llama.cpp／OpenAI 相容；預設會走思考欄，`reasoning_effort` 必須 `none`，否則聯想正文常是空的。
- 27B 單次常要 1.5–3 秒；30B-A3B 聯想實測常在 0.4–0.6 秒。`timeout_ms = 5000` 仍可能不夠，維持 20000。
- 使用者打太快時，回覆會被 Core 當成過期丟掉（日誌：`丟棄過期的聯想結果`）。測 AI 協助要打完停約 2 秒。
- 模型回的 `sentence` 若與 `local_sentence` 相同，畫面不另顯示雲端句；只有不同且 `confidence >= 0.9` 才自動上屏。
- 高信心自動修正上屏**必須**走 `Engine::accept_prediction`，才會寫入 `user-phrases.tsv` 與個人 n-gram。只把字塞進 TSF 不會學習。
- 狀態列「AI」格與設定「AI 協助選字」都寫 `[predict] enabled`。API Key 只在 `%APPDATA%\Qingjian`，**禁止**寫進 Skill。
- 改 `config.toml` 後 Server 會熱載入；日誌應出現 `雲聯想已接入 model=/models/...`。
- 若用設定頁下拉選取固定模型，Server 熱載入後應使用該 ID；若選「自動選擇」，下次建立雲端客戶端時重新查 `/models`。

公網 DeepSeek 備援（沒有私人模型時）：

```toml
[predict]
enabled = true
base_url = "https://api.deepseek.com"
model = "deepseek-v4-flash"
max_tokens = 512
reasoning_effort = "none"
sentence = true
```

規則：

- API Key 不得寫進 Skill、git、安裝包或一般回覆；使用者設定放 `%APPDATA%\Qingjian\.env`，由 `api_key_env` 指向的環境變數讀取。
- `max_tokens = 0` 代表使用服務商／程式預設值；要改善長句補全可設定 512 或 1024，但仍受模型與服務商限制。
- 連線測試使用固定短回覆上限，不要把一般聯想的 token 設定誤套到連線測試。
- 修改設定結構時同步 Windows 設定頁、設定模板與使用者文件，並確認舊設定檔因 `serde(default)` 仍能載入。
- 修改雲端 prompt 時維持隱私邊界：只傳必要的拼音、候選與設定允許的前後文；密碼框／私密輸入不得送出。

### 8. 長句整句轉換優化

長句品質與短句不同，優先檢查 `crates/qingjian-core/src/sentence/`：

- `SPAN_CANDIDATES`：每個詞圖格子保留的普通候選數。
- `ABBREVIATED_SPAN_CANDIDATES`：含簡拼位置時保留的候選數。
- `BEAM_WIDTH`：短句束搜尋寬度。
- `LONG_BEAM_WIDTH` / `LONG_SENTENCE_SYLLABLES`：長句使用較寬束搜尋的門檻。
- `RESCORE_PATHS`：神經重排可看到的候選路徑數，位於 `crates/qingjian-core/src/engine/mod.rs`。

長句優化應遵守：

1. 優先增加候選保留與長句 beam，不要先盲目提高 AI 呼叫頻率。
2. 用既有 `sentence::viterbi` 測試與 CLI 整句評測比較準確率及延遲。
3. 不能只測第一個候選；要檢查 3–20 字長句、簡拼、模糊音、個人 n-gram、專有詞與低頻詞。
4. 神經模型只能重排已產生的路徑；如果正確路徑在 beam 階段被淘汰，增加 `max_tokens` 不會修好本地整句。
5. 改搜尋常數後要觀察輸入延遲與記憶體，避免讓每個按鍵同步等待模型。

### 9. Windows 注音長組句卡死的診斷與防護

Windows TSF 按鍵是同步往返 Server。每一鍵都會重新查詢候選、建立 frame 並繪製候選窗；組句緩衝若無上限，長句可能讓單鍵延遲逐步增加，宿主應用看起來像整個卡死。

#### 已確認案例（2026-09-22）

- 症狀：使用大千注音連續輸入長句，前段正常，二十多鍵後逐步變慢，最後單鍵等待數十秒。
- 日誌證據：`%LOCALAPPDATA%\Qingjian\logs\tsf.<日期>.log` 的相鄰「收鍵」時間從約 0.2–1 秒增加到 2.5 秒、5 秒、10 秒，最後約 34 秒；Server 並未崩潰，候選仍有回應。
- 關鍵判斷：不要只看 Core CLI 的全拼效能。注音會先解碼成拼音再走整句查詢，實際負載與全拼不同；使用者截圖中的 preedit 若為 `ㄅㄆㄇ…`，必須按注音路徑重現。
- 第一次失敗修法：只設全模式共用 64 鍵上限。注音在約 30 鍵前已嚴重變慢，因此 64 鍵保護根本來不及觸發。
- 第二次失敗修法：將注音上限改為 24 鍵，並在第 25 鍵前自動上屏高亮候選。這雖阻止卡頓，卻違反輸入法基本行為：使用者未按 `Space`、候選數字或 `Enter`，文字仍被擅自提交。實測造成「前半段突然變成中文、後半段留下注音」的混合內容，必須撤回。

目前正確作法：**長注音絕不自動上屏、絕不自動拆段。**

- `apps/windows/server/src/dispatch/key/input.rs` 不得保留 `push_bounded` 或任何按鍵數達門檻即 `commit_highlighted()`／`take_raw()` 的邏輯。
- 注音內容只可在使用者明確按 `Space`、候選數字或 `Enter` 時提交；長組句仍完整保留，讓使用者可繼續輸入、退格或自行選字。
- 在 `apps/windows/server/src/dispatch/composed/mod.rs` 設 `ZHUYIN_BACKGROUND_LIMIT = 16`：超過後仍同步查本地候選，但不再發雲端聯想，也不安排神經重排；同時取消既有背景任務。這降低額外查詢、候選重畫和背景工作疊加造成的延遲，且不改寫使用者文本。
- 長句真正的效能瓶頸仍須分開量測注音 decode、整句 Viterbi、候選標註和候選窗渲染；背景工作止損不等於已完成根本效能優化。

測試至少包含：

```powershell
cargo test -p qingjian-windows-server long_zhuyin_composition_never_commits_without_user_confirmation
cargo test -p qingjian-windows-server
```

該回歸測試需連續輸入至少 30 個注音按鍵，逐鍵斷言 `commit == None`，並確認 preedit 仍完整保留所有注音字元。

快速部署只有 Server 原始碼變更時，可在確認協議未變的前提下：

```powershell
$env:QINGJIAN_UIACCESS = '0'
cargo build --release --locked -p qingjian-windows-server
Stop-Process -Name qingjian-server -Force -ErrorAction SilentlyContinue
Copy-Item D:\qingjian-main\target\release\qingjian-server.exe `
  'C:\Program Files\Qingjian\qingjian-server.exe' -Force
Start-Process 'C:\Program Files\Qingjian\qingjian-server.exe' `
  -WorkingDirectory 'C:\Program Files\Qingjian'
```

部署後必須比較 build 與安裝檔 SHA-256、確認新 PID／啟動時間，並看最新 Server／TSF 日誌。若修改協議、TSF 吃鍵規則或正式發版，不能只複製 Server，仍須走完整安裝包流程。

使用者文件同步：`docs/user/getting-started/keys.md` 應說明較長注音不會自行上屏；長輸入時只暫停額外背景聯想與重排以維持按鍵響應。

### 10. 版本與安裝包

每次程式碼或使用者可見功能更新，都要遞增版本並產生新的檔名，禁止用同一個 `.exe` 覆蓋不同程式碼版本：

1. Windows 三個 package 的版本一起改：
   ```text
   apps/windows/server/Cargo.toml
   apps/windows/tsf/Cargo.toml
   apps/windows/settings/Cargo.toml
   ```
2. 版本號同步至 `Cargo.lock`（用 cargo check/build 更新，不要手改依賴版本）。
3. 在 `CHANGELOG.md` 新增對應版本段落。
4. 執行完整 release build + Inno Setup，輸出檔名必須包含新版本，例如：
   ```text
   target/installer/qingjian-0.1.7-dev-windows-x86_64-setup.exe
   ```
5. 回報新檔案的大小與 SHA-256；保留舊版本安裝包。

### 11. AI 自動修正與候選評分策略

使用者的主要目標可能不是「多看幾個候選」，而是：輸入注音／拼音（即使有錯字、漏字、多字或音節混淆）後，由 AI 判斷高信心的正確句子並自動上屏。

正確的分工是：

```text
本地 Engine：快速解析、模糊音、詞庫、個人學習、基本整句
生成式 LLM：根據 letters + 上下文產生修正候選或完整句子
OpenJEV：只能在候選已經生成後做 NLI／一致性評分，不負責生成文字
Qingjian：只有高信心結果才自動替換 marked text 並上屏
```

目前相關資料流：

- `PredictionRequest` 位於 `crates/qingjian-core/src/engine/prediction/request.rs`。
- OpenAI 相容生成器位於 `crates/qingjian-predict/src/chat_client.rs` 與 `prompt.rs`。
- 非同步回覆經 `Prediction` 回到 Windows Server 的 `dispatch/composed`，再由 TSF Poll 取回。
- 自動修正應透過 TSF 的異步 `RequestEditSession` 替換整段 composition，不可在輪詢 callback 直接寫宿主文件。

自動修正的安全規則：

1. Prompt 必須要求模型輸出 `confidence`（0–1）與單一可直接上屏的句子；不要輸出解釋或多個版本。
2. 只有 `confidence >= 0.9`、結果與目前輸入仍屬同一個最新 prediction sequence，且目前 composition 尚未變更時，才允許自動上屏。
3. 不確定時保留候選視窗，不自動替換；使用者可按空白、數字或 Enter 走原本選字流程。
4. 密碼框、私密輸入、英文模式、表達式、問字與原始直輸段不可啟用自動修正。
5. 自動修正結果必須記錄來源與輸入摘要，不能把未確認的結果寫成一般本地選字學習；若要學習，應等使用者明確接受或提供撤銷機制。
6. 先以生成式 DeepSeek／GPT 產生 2–5 個候選，再由本地語言模型或可選的 OpenJEV 做第二階段評分；不要每個按鍵直接呼叫 OpenJEV。

OpenJEV（`http://10.145.119.19:8090`）是 NLI cross-encoder：

- 適合 `premise`／`hypothesis` 一致性、候選句子 rerank、已有參考答案的 grade。
- 不會生成句子，不能單獨把 `nihooma` 變成「你好吗」。
- CPU 延遲高，不適合阻塞打字；若使用，只對已縮小的候選背景評分，結果晚到不應阻塞輸入。
- 真正的文字生成與拼音錯誤修正仍由 OpenAI 相容的 DeepSeek／GPT 類模型負責。

### 12. 選字與改字機制重構

當使用者說選字不好、改字不順、不要歷史包袱，優先把它視為 Core 候選／組句狀態機問題，不要只調高詞頻或只改候選窗外觀。

這次已驗證的產品方向：

- Core 在整句轉換時保留最多 5 條完整 Viterbi 路徑，不再只暴露唯一首選；替代路徑必須去重，且不能讓模糊音／敲錯路徑蓋過精確詞級候選。
- 完整句候選的候選資料仍要帶完整 `syllables` 與 `kind`，讓 `commit` 能依實際被選的路徑重算 `sentence_words`；不能只靠重新取首選，否則替代句的學習與 n-gram 會錯。
- 游標移到中間時，`Composition::scope()` 只查游標前的音節；選字後透過既有 `consumed_by`／`drain_prefix` 只消耗已選部分，後半段拼音必須保留並重新查詢。
- 上下鍵按完整音節移動，左右鍵移動候選高亮；不要把上下鍵退回逐字母移動，也不要讓候選選擇直接破壞後半段組句。
- 跨平台測試至少涵蓋：整句替代路徑可直接選、移到中間改字後後半段仍在、替代路徑上屏後學習正常、候選不重複、注音與雙拼既有行為不退化。
- 這次實作涉及：`crates/qingjian-core/src/engine/mod.rs`、`engine/query/mod.rs`、`engine/commit/mod.rs`、Core 測試，以及 Windows `apps/windows/server/tests/engine_loop/`。

不要把「候選變多」當成完成條件：替代候選必須可以實際上屏、消耗正確的輸入、保留正確的後續組句，並且不破壞學習、糾錯、繁體對映與翻頁。

### 13. 回報結果

回報時明確寫出：

- 修改的檔案與行為。
- 使用的驗證命令及成功／失敗原因。
- 安裝包完整路徑。
- 安裝包大小與 SHA-256。
- 是否需要重新開啟應用程式、登出或重開機。
- 若沒有實際執行編譯或打包，不要聲稱已產出安裝程式。

## Rules and Limitations

- Core 與平台外殼嚴格解耦；平台層不能加入排序、詞庫查詢、翻譯或文字轉換邏輯。
- Windows Server 與 TSF DLL 必須使用同一份源碼／協議版本建置；只更新其中一個會導致 IPC 或候選 frame 不相容。
- 不要在沒有確認 target 產物時間與內容時使用 `-SkipBuild`。
- 不要為了讓快捷鍵測試通過而刪掉注音、表達式、問字、英文模式或修飾鍵衝突規則。
- 不要任意刪除使用者設定、學習檔、API 設定或 `%APPDATA%\Qingjian`。
- `QINGJIAN_UIACCESS=1` 只在本機有受信任簽章、需要 UWP／高層級候選窗的情境使用；未簽名測試建議使用 `QINGJIAN_UIACCESS=0`。
- 不把密碼、API key、憑證或使用者學習資料寫進 Skill、安裝包回報或 git。

## Pitfalls

- **只改 Server 不改 TSF**：TSF 的 `OnTestKeyDown` 會先決定是否吃鍵；Server 可能永遠收不到新快捷鍵。
- **把 Windows 舊 `Ctrl+數字` 譯詞設定留著**：會與跨平台 `Ctrl+數字` 選字衝突；預設譯詞應改成 `Ctrl+Alt` 系列。
- **只更新文件沒有重新編譯**：正在執行的 Server 和已載入宿主內的 DLL 仍是舊版本。
- **只重啟 Server 沒重開宿主應用**：TSF DLL 仍可能是舊版，導致看起來像快捷鍵修改沒生效。
- **32 位 target 缺失**：release 64 位編譯成功仍可能在打包前因 `i686-pc-windows-msvc` 缺少 `core/std` 而失敗。
- **PowerShell 執行原則阻擋腳本**：使用 `-NoProfile -ExecutionPolicy Bypass` 執行專案腳本；不要為了單次打包永久降低整機安全策略。
- **Inno Setup 不在 PATH**：用 `QINGJIAN_ISCC` 指到實際 `ISCC.exe`。
- **編碼造成 PowerShell 輸出亂碼**：不要把亂碼輸出當成編譯失敗；以 exit code、產物、Inno 的 `Successful compile` 為準。
- **`cargo fmt --check` 顯示既有全專案差異**：先分辨是既有格式差異還是本次檔案造成，不要為了本次快捷鍵需求大範圍重排無關檔案。
- **只改上下鍵為 `move_cursor_left`（字母）**：注音一次只跳 ㄋ／ㄧ／ˇ，不像「移到一個字」。必須走 `move_cursor_syllable_*`，且注音不要當 `plain`。
- **`build.ps1 --locked` 在改版本後失敗**：先 `cargo build --offline -p qingjian-windows-server ...` 寫入 lock，再打包。
- **模型下拉選單是空的**：先按「重新整理」，確認 `base_url` 可達且 `{base_url}/models` 回傳 `data[].id`；不要只看 `/health`。模型清單是背景查詢，等待狀態完成後再選取。
- **選了模型但 Server 仍用舊模型**：確認設定檔已寫入完整模型 ID、Server 熱載入日誌是否更新；若已開啟輸入框的應用仍載入舊 TSF DLL，關閉並重新開啟該宿主程式。
- **qingjian-server.exe 測試要系統管理員**：`cargo test -p qingjian-windows-server up_arrow` 可能撞 bin 的 os error 740；按鍵迴圈測用 `--test engine_loop`。
- **長句卡死使用非注音測試代替**：其他方案的測試結果不能代表 Windows 大千注音同步路徑安全；必須從 TSF 日誌的逐鍵時間戳辨認延遲曲線，並以實際注音鍵序列測試。
- **以自動上屏處理長注音**：不論門檻是 24 鍵或其他數字，未經使用者確認就 commit 都會造成文字被切成前段中文、後段注音。長句保護只能減少背景工作或最佳化查詢，不能自動提交。
- **只看 Server 還活著就排除卡死**：這類問題通常不是 crash，而是單鍵同步處理耗時數秒到數十秒。程序仍存在、日誌最後仍回候選，不代表使用體驗沒有卡死。

## Verification

1. `D:\qingjian-main\CLAUDE.md`、`docs/contributing.md` 與本 Skill 路徑存在。
2. `cargo check --locked -p qingjian-windows-server -p qingjian-windows-tsf -p qingjian-windows-settings` 成功。
3. Windows Server 的按鍵測試涵蓋 `Ctrl+數字` 選字與 Windows 譯詞快捷鍵。
4. `target\release\qingjian-server.exe`、`target\release\qingjian_tsf.dll`、`target\release\qingjian-settings.exe` 與 `target\i686-pc-windows-msvc\release\qingjian_tsf.dll` 存在且為本次 build 產物。
5. Inno Setup 輸出 `target\installer\qingjian-<版本>-windows-x86_64-setup.exe`，記錄 SHA-256。
6. 安裝後重新開啟文字輸入宿主，實測 `nihao` 後按 `Ctrl+1`，確認上屏的是當頁第一個候選而不是譯詞。
6.1 實測組句中 `↑`：`nihao` 後上鍵，候選含「你」，preedit 仍為 `ni'hao`。
6.2 實測整句替代路徑：`nihao` 的候選頁必須能看到至少兩個覆蓋兩音節的組合，數字選第二條後仍能正常上屏。
6.3 實測中間改字：`nihao` 按 `↑` 後選第二個「ni」候選，結果只上屏前一字，剩餘 `hao` 仍在 preedit。
6.4 目前已驗證安裝包（2026-09-21）：`D:\qingjian-main\target\installer\qingjian-2.0.7-dev-windows-x86_64-setup.exe`，98,608,696 bytes，SHA-256 `d5a9b0a43bcb965c743fd72abfc1f420deb5eed60ecb1fb7b5b551b54824f6b6`。這代表已完成打包，不代表已安裝；若安裝程式被 UAC／互動視窗中止，必須如實回報。
6.5 模型下拉選單版本已驗證（2026-09-22）：`D:\qingjian-main\target\installer\qingjian-2.0.8-dev-windows-x86_64-setup.exe`，98,681,612 bytes，SHA-256 `69AFD8291F976637E1C96393DB659185E495F662C422B3A2449BD1F955D8319E`；已以靜默安裝成功更新至 `C:\Program Files\Qingjian\`，並確認 `qingjian-settings.exe`、`qingjian-server.exe` 與 32/64 位 TSF DLL 已更新。
6.6 注音長組句修正版（2026-09-22）：`D:\qingjian-main\target\installer\qingjian-2.0.10-dev-windows-x86_64-setup.exe`，98,590,128 bytes，SHA-256 `d11ef2619267133fe9c58540c8b9c74b7b74adc538df48b685384ebe7886afc7`。此版移除 24 鍵自動上屏，超過 16 鍵時只停止背景聯想與神經重排；安裝後須以超過 30 個注音按鍵驗證未按確認鍵不會有任何 commit。
7. 若本 Skill 被更新，執行：
   ```powershell
   python D:\OB\skills\skill-creator\scripts\quick_validate.py D:\OB\skills\qingjian-ime-development
   python D:\OB\skills\obsidian-mcp-skill-router\scripts\rebuild_skills_index.py
   ```
