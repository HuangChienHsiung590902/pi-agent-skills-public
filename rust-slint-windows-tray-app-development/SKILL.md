---
name: rust-slint-windows-tray-app-development
description: >-
  開發與維護 Rust + Slint Windows 桌面應用程式，特別適用於系統匣常駐、可收合 Sidebar、SQLite 可攜式設定、
  OmniRoute OpenAI-compatible LLM Chat、Windows icon、設計階段 UI 迭代與 release EXE 編譯驗證。
---

# Rust + Slint Windows 常駐桌面應用程式開發

## When to Use

當需要在 Windows 上建立或維護以下類型的應用程式時使用本 Skill：

- Rust + Slint GUI 桌面程式。
- 關閉視窗後仍留在 Windows system tray 的常駐工具。
- 有左側功能選單、頁面切換與 Sidebar 收合功能的工具。
- 將設定與可攜式資料放在 EXE 同目錄的應用程式。
- 透過 OpenAI-compatible API 呼叫 OmniRoute 的 LLM Chat。
- 使用 SQLite 儲存 Base URL、API Key、Model 等設定。
- 需要把 Rust 專案編譯成沒有 Console 視窗的 Windows GUI `.exe`。
- 需要在設計階段只修改原始碼、不反覆產生正式 EXE 的 UI 迭代。

不適用於：

- Windows Service 或需要服務管理員生命週期的背景服務。
- 對外公開的 Web/API agent service。
- 需要直接修改 OmniRoute 伺服器、Provider 或 Pi `models.json` 的工作；那些設定應依 `omniroute-native` 或 `pi-omniroute-model-definition` Skill 處理。

## Inputs and Outputs

### Inputs

- 參考專案 `C:\Users\HCH\rust-slint-tray-demo\` 與本次要改的 UI／Rust／設定需求。
- 任務屬於設計階段或交付階段。

### Outputs

- 實際修改的原始碼（通常是 `ui/app.slint`、必要時 `src/main.rs`／`Cargo.toml`）。
- `cargo check` 或使用者明確要求時的 release EXE，以及 SQLite／OmniRoute 驗證結果。

## Current Reference Project

目前參考專案：

```text
C:\Users\HCH\rust-slint-tray-demo\
```

專案主要檔案：

```text
Cargo.toml
Cargo.lock
build.rs
src\main.rs
ui\app.slint
assets\app.ico
settings.sqlite3
```

檔案責任：

| 檔案 | 責任 |
|---|---|
| `Cargo.toml` | Rust、Slint、tray、HTTP、序列化與 SQLite 依賴 |
| `build.rs` | 編譯 Slint UI；Windows 上編譯 icon/resource |
| `src/main.rs` | 建立視窗、system tray、事件迴圈、SQLite、OmniRoute API |
| `ui/app.slint` | UI 版面、功能選單、頁面狀態、輸入控制與 callback |
| `assets/app.ico` | Windows 應用程式與 system tray 圖示來源 |
| `settings.sqlite3` | 可攜式設定資料庫；不應公開上傳 |

## Scope Control

每次工作先區分「設計階段」與「交付階段」：

### 設計階段

只修改必要的正式原始碼：

- `ui/app.slint`
- 必要時 `src/main.rs`
- 必要時 `Cargo.toml`

除非使用者明確要求，不執行：

- `cargo build --release`
- 複製或覆蓋正式 `.exe`
- 啟動新編譯的程式
- 新增安裝程式
- 修改 OmniRoute 服務設定

可執行 `cargo check` 作為語法與型別驗證，但如果使用者明確要求「只設計、不編譯」，不可執行 build 或 run。

### 交付階段

使用者明確要求「編譯」「產生 exe」「可以測試」後才：

1. 確認舊程式沒有鎖住 EXE。
2. 建立專用暫存 target 目錄。
3. 執行 `cargo check` 或 `cargo build --release`。
4. 驗證輸出。
5. 只有正式交付需要時才將 EXE 複製到專案根目錄。

## Project Setup

### Cargo dependencies

基本依賴：

```toml
[dependencies]
slint = "1.9"
tray-icon = "0.19"
reqwest = { version = "0.12", default-features = false, features = ["blocking", "json", "rustls-tls"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
rusqlite = { version = "0.32", features = ["bundled"] }

[build-dependencies]
slint-build = "1.9"

[target.'cfg(windows)'.build-dependencies]
winresource = "0.1"
```

`rusqlite` 使用 `bundled`，讓 Windows EXE 不依賴另外安裝的 SQLite DLL。

`reqwest` 目前採 blocking client，但網路請求必須放在背景 thread；不要在 Slint UI callback 直接執行同步 HTTP，否則會卡住畫面。

### build.rs

最小配置：

```rust
fn main() {
    slint_build::compile("ui/app.slint").expect("failed to compile Slint UI");

    #[cfg(windows)]
    {
        let mut resource = winresource::WindowsResource::new();
        resource.set_icon("assets/app.ico");
        resource.compile().expect("failed to compile Windows resources");
    }
}
```

`ui/app.slint` 路徑以專案根目錄為基準；`build.rs` 由 Cargo 執行時會在正確的 package root 下解析它。

### Windows GUI subsystem

在 `src/main.rs` 最上方加入：

```rust
#![cfg_attr(windows, windows_subsystem = "windows")]
```

這會讓 release EXE 成為 Windows GUI 程式，不額外開啟 Console 視窗。開發期間若需要讀取錯誤輸出，可暫時拿掉它，但交付 GUI EXE 前要恢復。

## Slint UI Architecture

### Root component

```slint
export component MainWindow inherits Window {
    title: "Rust + Slint 常駐示範";
    width: 860px;
    height: 540px;

    property<int> current-page: 0;
    property<bool> menu-collapsed: false;

    callback hide-to-tray();
    callback quit-app();
}
```

頁面狀態用 `current-page` 控制，避免為每個頁面建立多個 Window：

```slint
visible: root.current-page == 4;
```

### Properties exposed to Rust

若 Rust 需要存取或修改 property，使用公開的 `in-out property`：

```slint
in-out property<string> omni-base-url: "http://127.0.0.1:20128/v1";
in-out property<string> omni-api-key: "";
in-out property<string> omni-model: "";
in-out property<string> chat-input: "";
in-out property<string> chat-log: "";
in-out property<string> chat-status: "尚未開始對話";
in-out property<bool> chat-sending: false;
```

只寫 `property<string>` 時，產生的 Rust getter/setter 可能是 private；這會造成：

```text
error[E0624]: method ... is private
```

需要由 Rust 讀寫的 property 一律使用 `in-out property`。

### TextInput vs LineEdit

`std-widgets.slint` 的 `LineEdit` 在不同 style 下可能把文字顏色交給原生 palette 控制，導致淡黃色背景上出現白字。

當需要明確指定文字顏色時，使用低階 `TextInput`：

```slint
TextInput {
    text <=> root.omni-model;
    color: #000000;
    single-line: true;
}
```

注意：`TextInput` 支援 `color`；`LineEdit` 不一定支援 `color` 或 `foreground`。

對話輸出如果要固定黑色，使用一般 `Text`：

```slint
Text {
    text: root.chat-log;
    color: #000000;
    wrap: word-wrap;
    vertical-alignment: top;
}
```

`TextEdit` 並不是任意版本都接受 `color` 屬性；若需唯讀輸出且文字不多，`Text` 是最直接的方案。

### High-contrast rule

淡黃色背景不可使用白色或近白色文字：

- 淡黃色／米黃色背景：文字使用 `#000000`、`#171512` 或深棕色。
- 黑色 Sidebar：文字使用暖黃色或米黃色，例如 `#f4d06f`、`#d8cda9`。
- 輸入框文字明確指定 `color: #000000`。
- placeholder 使用深暖灰，例如 `#75694c`。
- 不要假設外層 `Text` 顏色會穿透原生 input widget。

### Sidebar collapse

Sidebar 收合至少需要：

```slint
property<bool> menu-collapsed: false;
```

Sidebar 與內容區寬度都使用同一個條件：

```slint
width: root.menu-collapsed ? 78px : 224px;
```

```slint
x: root.menu-collapsed ? 78px : 224px;
width: parent.width - (root.menu-collapsed ? 78px : 224px);
```

收合控制建議放在 Logo／品牌列右側，而不是底部版本資訊旁：

```slint
TouchArea {
    clicked => { root.menu-collapsed = !root.menu-collapsed; }
}
```

收合時隱藏選單標籤、保留圖示；圖示按鈕仍要有足夠點擊範圍。

## Windows System Tray

### 建立 tray icon

`tray-icon` 的典型配置：

```rust
let menu = Menu::new();
let show_item = MenuItem::new("顯示視窗", true, None);
let quit_item = MenuItem::new("結束程式", true, None);
menu.append_items(&[
    &show_item,
    &PredefinedMenuItem::separator(),
    &quit_item,
])?;

let _tray = TrayIconBuilder::new()
    .with_menu(Box::new(menu))
    .with_tooltip("Rust + Slint 常駐示範")
    .with_icon(tray_icon())
    .build()?;
```

將 tray object 綁定在 `main` scope 中，直到事件迴圈結束前不可 drop；常見做法是保留 `_tray` 變數。

### Close behavior

關閉視窗時隱藏，而不是結束：

```rust
let window_for_close = window.clone_strong();
window.window().on_close_requested(move || {
    window_for_close.window().hide().ok();
    CloseRequestResponse::HideWindow
});
```

事件迴圈必須使用：

```rust
window.show()?;
slint::run_event_loop_until_quit()?;
```

不要用會在最後一個視窗隱藏後結束的錯誤生命週期模式；否則視窗消失時 tray 也會消失。

### Tray event polling

若使用 timer polling：

```rust
let timer = Timer::default();
timer.start(TimerMode::Repeated, Duration::from_millis(100), move || {
    while let Ok(event) = MenuEvent::receiver().try_recv() {
        // enqueue Show or Quit
    }
});
```

UI 相關操作在 Slint event loop 執行緒完成。不要從 tray callback 直接操作非 UI thread-safe 的 Slint strong handle。

## SQLite Portable Settings

### Database path

需求是讓資料庫和 EXE 一起攜帶，因此設定檔使用：

```text
settings.sqlite3
```

開發時若目前工作目錄有 `Cargo.toml`，使用目前專案根目錄；發佈後使用 EXE 所在目錄：

```rust
fn project_directory() -> Result<PathBuf, Box<dyn std::error::Error>> {
    let current = std::env::current_dir()?;
    if current.join("Cargo.toml").is_file() {
        return Ok(current);
    }

    std::env::current_exe()?
        .parent()
        .map(Path::to_path_buf)
        .ok_or_else(|| "無法取得執行檔所在目錄".into())
}
```

注意：便攜式模式需要資料夾可寫；若 EXE 放在 `Program Files`，一般使用者可能無法建立或更新 SQLite。需要時應讓使用者選擇可寫資料夾，或改用 AppData；不能默默假設所有 EXE 目錄都可寫。

### Schema

```sql
CREATE TABLE IF NOT EXISTS app_settings (
    key TEXT PRIMARY KEY NOT NULL,
    value TEXT NOT NULL
);
```

目前 OmniRoute 設定 key：

```text
omniroute_base_url
omniroute_api_key
omniroute_model
```

初始化預設值：

```rust
let defaults = [
    ("omniroute_base_url", "http://127.0.0.1:20128/v1"),
    ("omniroute_api_key", ""),
    ("omniroute_model", ""),
];
```

用 `INSERT OR IGNORE` 初始化，避免每次啟動覆蓋既有設定。

### API key handling

目前簡化版本可以存入專案內 SQLite，但要明確告知：

- API Key 是明文或可被本機使用者讀取的資料。
- 不要提交 `settings.sqlite3` 到公開 Git。
- 不要把 API Key 寫入 log、錯誤字串或畫面訊息。
- 正式產品若有需求，改用 Windows DPAPI 加密後再放 SQLite；不要在未被要求時擴大實作範圍。

## OmniRoute LLM Chat

### Endpoint

OmniRoute 是 OpenAI-compatible gateway，預設：

```text
http://127.0.0.1:20128/v1/chat/completions
```

OmniRoute `/api/health` 可作為服務健康檢查，但 `/v1/models` 可能啟用 API key 驗證；HTTP 401 不代表服務未啟動。

### Request

非串流的最小 request：

```json
{
  "model": "QW/qwen3.8-27b",
  "messages": [
    {"role": "user", "content": "你好"}
  ],
  "stream": false
}
```

Rust 結構：

```rust
#[derive(serde::Serialize)]
struct ChatRequest {
    model: String,
    messages: Vec<ChatRequestMessage>,
    stream: bool,
}

#[derive(serde::Serialize, Clone)]
struct ChatRequestMessage {
    role: String,
    content: String,
}
```

Endpoint 組合時去掉 Base URL 尾端 `/`：

```rust
let endpoint = format!("{}/chat/completions", base_url.trim_end_matches('/'));
```

### Response and errors

至少處理：

- HTTP 200：讀取 `choices[0].message.content`。
- HTTP 401：API Key 無效或缺少。
- HTTP 404：Base URL 或路徑錯誤。
- HTTP 429：provider rate limit。
- HTTP 503：OmniRoute admission busy 或上游暫時不可用。
- JSON 格式錯誤：不要直接 `unwrap`，顯示安全的格式錯誤。
- choices 空陣列或 content 空值：顯示「LLM 回應沒有可顯示的文字」。

不要把完整 response body 或 request header（尤其 Authorization）寫入 log。

### Background thread and UI handoff

`reqwest::blocking` 呼叫不可直接跑在 Slint UI callback；使用背景 thread：

```rust
let window_weak = window.as_weak();
thread::spawn(move || {
    let result = request_chat(...);
    let _ = window_weak.upgrade_in_event_loop(move |window| {
        // 在 Slint 執行緒更新 property
        window.set_chat_status(...);
    });
});
```

不要將 `MainWindow` strong handle 捕捉進 `thread::spawn` 或 `slint::invoke_from_event_loop` closure；Slint component 不是 `Send`。正確做法是：

```rust
let window_weak = window.as_weak();
window_weak.upgrade_in_event_loop(move |window| { ... });
```

這是重要的編譯錯誤修正：直接捕捉 `MainWindow` 會造成大量 `E0277`，例如 `UnsafeCell ... cannot be sent between threads safely`。

### Conversation state

初版可把聊天紀錄放在：

```rust
Arc<Mutex<Vec<ChatMessage>>>
```

聊天紀錄只放記憶體；SQLite 只存連線設定。這樣可避免未經要求就增加對話表、migration、資料清除策略與隱私風險。

UI 可將歷史格式化成：

```text
你
...

LLM
...
```

對話過長時才另行設計 ScrollView、訊息 model 或歷史壓縮；不要在初版偷偷加入複雜聊天資料層。

## Typography and Colors

目前採用暖黃＋黑色方向：

```text
canvas: #f7f0d2
accent: #f4d06f
black: #171512
card: #fbf6df
ink: #000000 / #171512
body: #554c38 / #75694c
sidebar-muted: #d8cda9 / #b7aa87
success: #2f6b42
```

字型：

```text
標題：Noto Serif TC
一般 UI：Noto Sans TC
```

若 Windows 字型檔存在但 Slint render backend 顯示效果不同，要以實際執行畫面確認；不要只因字型檔名存在就宣稱一定使用成功。

## Procedure

1. 先依 Scope Control 區分設計階段與交付階段；未要求交付就不要 `cargo build --release` 或覆蓋正式 EXE。
2. 只改完成需求所需的正式原始碼。
3. 用下面 Development Commands 做 `cargo check`；使用者要求執行或交付時才 `cargo run`／release build。
4. 依 Verification 檢查 UI、tray 生命週期與 SQLite／OmniRoute 行為。
5. 測試與 Cargo target 放在 `%TEMP%\pi-work\<task-id>\`，不要污染專案目錄。

## Development Commands

### 檢查原始碼

在專案根目錄：

```powershell
cd C:\Users\HCH\rust-slint-tray-demo
$env:CARGO_TARGET_DIR="$env:TEMP\pi-work\rust-slint-tray-demo-check"
cargo check
```

`CARGO_TARGET_DIR` 必須導向專案外的暫存目錄，避免 target、incremental cache 與中間產物污染專案。

### 開發執行

```powershell
cd C:\Users\HCH\rust-slint-tray-demo
$env:CARGO_TARGET_DIR="$env:TEMP\pi-work\rust-slint-tray-demo-run"
cargo run
```

此命令會讀取 `ui/app.slint`、編譯 Rust、建立或更新專案根目錄的 `settings.sqlite3`，並啟動 GUI。

### Release EXE

只有使用者明確要求產生 EXE 時才執行：

```powershell
cd C:\Users\HCH\rust-slint-tray-demo
$env:CARGO_TARGET_DIR="$env:TEMP\pi-work\rust-slint-tray-demo-build"
cargo build --release
Copy-Item `
  "$env:TEMP\pi-work\rust-slint-tray-demo-build\release\rust-slint-tray-demo.exe" `
  ".\rust-slint-tray-demo.exe" `
  -Force
```

複製前確認舊 EXE 沒有被正在執行的程式鎖住。不要用 `taskkill` 作為一般流程，除非確定沒有服務 supervisor 或提升權限問題；先正常從 tray 選單結束。

## Verification

### Static verification

確認：

```text
Cargo.toml 存在且依賴正確
build.rs 存在
src/main.rs 存在
ui/app.slint 存在
assets/app.ico 存在且為有效 ICO
```

確認 UI：

```text
LLM Chat 選單存在
current-page 包含 0～4 頁
Sidebar menu-collapsed 存在
Base URL/API Key/Model bindings 存在
TextInput input 欄位使用 color: #000000
Chat output 使用黑色文字
```

確認 Rust：

```text
settings.sqlite3 初始化
app_settings table
OmniRoute chat/completions endpoint
Bearer API key
background thread
Weak::upgrade_in_event_loop
run_event_loop_until_quit
```

### Cargo verification

```powershell
$env:CARGO_TARGET_DIR="$env:TEMP\pi-work\rust-slint-windows-tray-check"
cargo check
```

成功標準：

```text
Finished `dev` profile
exit code 0
```

### Runtime verification

開發執行後確認：

1. 主視窗可開啟。
2. Sidebar 可收合與展開。
3. `設定` 頁的 Base URL、API Key、Model 文字清楚可見。
4. 點 `儲存設定` 後 SQLite 中 key/value 更新。
5. 點 `LLM Chat` 可看到聊天頁。
6. OmniRoute 服務與 API Key/Model 正確時，訊息可收到回覆。
7. API 失敗時顯示錯誤，不洩漏 API Key。
8. 關閉主視窗後程序仍存在，tray icon 仍存在。
9. tray 選單可重新顯示與真正結束。

### Release verification

```powershell
Get-Item .\rust-slint-tray-demo.exe
```

並確認 EXE 是 Windows GUI subsystem，而不是 Console subsystem；可用 `file` 或 PE 工具確認。

發佈攜帶至少包含：

```text
rust-slint-tray-demo.exe
settings.sqlite3
```

若 SQLite 尚未存在，第一次啟動會自動建立；若要攜帶既有設定，必須一起複製資料庫。

## Pitfalls

### Slint API pitfalls

- `std-widgets.slint` 不一定匯出名為 `Text` 的 widget；一般 `Text` 是 Slint built-in，不要從 `std-widgets.slint` 匯入它。
- `Button` 不一定有 `color` 或 `background` 屬性；需要客製樣式時使用 `Rectangle + Text + TouchArea`。
- `TextEdit` 不一定有 `color`；要固定顏色可改用 `Text`，或深入使用底層 `TextInput`／style component。
- `LineEdit` 的 API 屬性不等同低階 `TextInput`；`foreground` 不是通用可設定屬性。
- `CloseRequestResponse::KeepWindow` 不存在時，Slint 1.17 使用 `HideWindow` 或 `KeepWindowShown`，依實際 API 版本確認。
- 由 Rust 存取的 Slint property 必須公開為 `in-out property`。

### Threading pitfalls

- 不可把 Slint `MainWindow` strong handle 捕捉到背景 thread。
- 不可把 strong handle 捕捉進要求 `Send` 的 `invoke_from_event_loop` closure。
- 使用 `window.as_weak()` 搭配 `upgrade_in_event_loop`。
- 網路請求、SQLite 慢操作不要阻塞 UI event loop；SQLite 初始化很小可在啟動時執行，HTTP 一定放背景 thread。

### Tray lifecycle pitfalls

- `TrayIcon` object drop 後圖示會消失；保持 `_tray` 存活。
- 使用 `window.run()` 搭配隱藏最後一個視窗可能使事件迴圈結束；常駐工具使用 `run_event_loop_until_quit()`。
- 關閉視窗的 callback 必須回傳隱藏行為，不要在一般關閉操作中呼叫 `quit_event_loop()`。
- GUI 的「結束程式」與 tray 的「結束程式」才應真正停止 process。

### OmniRoute pitfalls

- `/api/health` 回 200 不代表 `/v1/models` 不需要 API key。
- `/v1/models` HTTP 401 通常表示 authentication required，不是服務未啟動。
- Base URL 若已包含 `/v1`，不要再拼接第二個 `/v1`。
- 不要把 `contextWindow / 2` 當作上游模型的 `max_tokens`；那是 Pi 未定義模型的常見陷阱，與這個直接 API client 的設定不同。
- 有些 OmniRoute response 可能是 SSE，即使 request 沒有明確預期；初版非串流 client 若遇到 SSE，應明確報格式問題，不能靜默解析錯誤。

### Portable database pitfalls

- EXE 位於 `Program Files` 等唯讀位置時，專案內 SQLite 可能無法寫入。
- `settings.sqlite3` 可能包含 API Key；不要公開分享。
- 不要在每次啟動時重設 defaults，必須使用 `INSERT OR IGNORE`。
- 不要為了「未來可能需要」建立多餘 migration、對話表或加密抽象層。

### Build and temporary files

- 所有 Cargo target、Log、測試輸入輸出與中間產物放在：
  ```text
  %TEMP%\pi-work\<task-id>\
  ```
- 不要把 target 目錄放進專案。
- 正式 `.exe` 只有在使用者要求交付時才複製到專案目錄。
- 若 build 逾時，先檢查是否仍有 cargo/rustc process，再重新執行；不要直接宣稱 build 成功。
- 如果編譯失敗，回報真實錯誤與目前產物狀態，不要把舊 EXE 當成新版本。

## Rules and Limitations

- 設計要求只改 `ui/app.slint` 時，不要改 Rust。
- SQLite 設定層需求只改設定讀寫，不要順便做 DPAPI、migration、設定同步或安裝程式。
- Chat 初版先使用非串流 API；除非使用者要求，不要加入 SSE parser、停止生成、token counter 或多模型管理。
- UI 美化以使用者指定的 DesignMD 方向為準；不要重新引入與目前配色衝突的漸層、白字或過大的 Hero。
- 任何會修改系統設定、啟停 OmniRoute、安裝套件、刪除資料或覆蓋既有產物的操作，先列出風險並取得確認。

## Output Checklist

完成一次工作後回報：

- 實際修改的檔案。
- 只修改原始碼還是也產生了 EXE。
- SQLite 是否建立／更新。
- `cargo check` 或 `cargo build` 是否執行及結果。
- 暫存路徑。
- 尚未完成或需要使用者手動測試的項目。
- 若有 OmniRoute 測試，說明 HTTP status；不可洩漏 API Key。
