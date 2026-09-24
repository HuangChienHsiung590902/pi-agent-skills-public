---
name: docker-gigabyte-ui
description: >-
  維護 D:\\docker-gigabyte-ui 的 Rust + Slint Windows Docker 遠端管理 GUI；用於修改或除錯
  10.145.119.19 容器狀態同步、Docker heartbeat、host network 容器 port 顯示、啟停重啟操作、
  Windows 黑框視窗、MSVC 編譯與 release EXE 交付。
---

# Gigabyte Docker UI 維護

## When to Use

當使用者提到以下任一項時使用本 Skill：

- `D:\\docker-gigabyte-ui` 或 `docker-gigabyte-ui.exe`。
- Rust + Slint 的 Gigabyte Docker 遠端控制桌面 GUI。
- 遠端 Docker 主機 `10.145.119.19`、容器列表、GPU 使用量、啟動／停止／重啟容器。
- UI 狀態沒有反映伺服器端、需要 heartbeat、輪詢或最後同步時間。
- host network 容器顯示沒有 port，但啟動參數其實有 `--port`。
- Windows GUI 不應跳出黑色 console 視窗。
- 需要修復、檢查、編譯或交付這個專案的 release EXE。

不適用於：

- 一般遠端 Docker 管理或 Docker context／備份／清理；使用 `docker-remote-control`。
- ComfyUI、Spark、llama.cpp 本身的服務部署；使用對應的服務 Skill。
- 一般 Rust + Slint 新專案；可參考 `rust-slint-windows-tray-app-development`。

## Project Facts

```text
Source:       D:\\docker-gigabyte-ui
Main Rust:    D:\\docker-gigabyte-ui\\src\\main.rs
Slint UI:     D:\\docker-gigabyte-ui\\ui\\app.slint
Manifest:     D:\\docker-gigabyte-ui\\Cargo.toml
Toolchain:    D:\\docker-gigabyte-ui\\rust-toolchain.toml
Release EXE:  D:\\docker-gigabyte-ui\\docker-gigabyte-ui.exe
Remote:       hch@10.145.119.19
```

目前程式是 Rust + Slint，不是 Web UI。它由 SSH 呼叫遠端 shell，再取得 Docker inspect 與 GPU 資訊。不要假設本機 Docker Desktop context 會影響它；程式固定透過 SSH 操作 `10.145.119.19`。

## Architecture

### Snapshot and heartbeat

`src/main.rs` 的 `LIST_CMD` 會依序輸出三個 section：

```text
---HEARTBEAT---
<Docker ServerVersion>
---CONTAINERS---
<container rows>
---GPU---
<nvidia-smi rows>
```

heartbeat 使用：

```bash
docker info --format '{{.ServerVersion}}' || exit 1
```

如果 SSH 失敗、Docker daemon 失敗或 heartbeat 沒有回傳版本，整次 snapshot 視為失敗。UI 顯示離線、錯誤原因與最後成功同步秒數；不要把失敗 snapshot 當成空的容器列表，避免讓使用者誤以為遠端沒有容器。

UI 啟動時立即同步，之後用 Slint `Timer` 每 5 秒執行一次背景同步。手動重新整理、啟動／停止／重啟後的 snapshot 與 heartbeat 共用 `AtomicBool`，避免 SSH 請求重疊。背景 heartbeat 不應把 `loading` 設成 true，否則按鈕會每五秒閃爍或被鎖住。

成功同步會更新：

- 容器實際狀態與 exit code。
- Docker Server version。
- GPU memory／utilization。
- 容器 ports 與可開啟 URL。
- 選取中的容器資料。
- `server-online`、最後成功同步時間與狀態列文字。

失敗時保留上一份容器資料供檢視，但將 `server-online` 設為 false，並停用啟動、停止、重啟與開啟網址功能。

### Container port handling

一般 bridge／compose 容器從：

```text
.HostConfig.PortBindings
```

解析 host port。

但是 `network_mode: host` 的容器通常會回傳空的 `PortBindings`，即使程式實際用 command argument 監聽 port。例如 `llama-cpp-spark-x25-4b` 的 inspect 可能是：

```text
NetworkMode=host
Cmd=["--host","0.0.0.0","--port","8083", ...]
```

因此 `parse_container_line()` 會額外輸出並解析：

- `.HostConfig.NetworkMode`
- `.Config.Cmd` 的 JSON

當 port bindings 為空且 network mode 是 `host` 時，從 command arguments 解析：

- `--port 8083`
- `--port=8083`
- `-p 8083`

不要只把 `.HostConfig.PortBindings` 當作所有容器的 port 來源。此修正需要 `serde_json` dependency。

### Windows no-console behavior

`src/main.rs` 必須保留：

```rust
#![cfg_attr(windows, windows_subsystem = "windows")]
```

這是 release GUI EXE 不開 console 的第一層設定。

所有由 GUI 建立的子程序也要在 Windows 加上：

```rust
use std::os::windows::process::CommandExt;
const CREATE_NO_WINDOW: u32 = 0x08000000;
command.creation_flags(CREATE_NO_WINDOW);
```

目前至少適用於：

- SSH 子程序。
- `cmd /C start` 開啟瀏覽器的子程序。

只修 `windows_subsystem` 而沒有修 SSH child process，heartbeat／手動刷新仍可能跳黑框；只修 SSH 而沒有 GUI subsystem，EXE 啟動時仍可能有 console。

### Icon design and rendering stability

圖示要兼顧「漂亮」與 Windows 小尺寸辨識度，不要直接把大型 Docker Whale 圖片當成唯一圖示。建議採用原創的深藍控制台風格：

- 深藍背景、細邊框與柔和陰影。
- 2–3 層堆疊的 container/service 模組。
- 每個模組有簡單狀態指示燈。
- 底部可加入連線波形或監控線，但不要放過多細節。
- 16×16、24×24、32×32 時仍要看得出主要輪廓。
- 保留透明背景，避免 Windows 桌面出現黑色或白色方塊。

圖示產生流程：

1. 先把既有 `assets/app.png`、`assets/app.ico` 與根目錄 EXE 備份到：
   ```text
   C:\Users\HCH\AppData\Local\Temp\docker-gigabyte-ui-icon-<timestamp>\\
   ```
2. 產生一張 512×512 原始 PNG，再縮放輸出 `16/24/32/48/64/128/256/512` PNG frame。
3. 用 Windows `System.Drawing` 或等效工具把每個 PNG frame 封裝成同格式的 Windows ICO；不要只把單一 512×512 PNG 改名成 `.ico`。
4. 將 512×512 PNG 放到 `assets/app.png`，讓 Slint 視窗與程式內 Image 使用；將多尺寸 ICO 放到 `assets/app.ico`，讓 Windows EXE 與工作列使用。
5. 圖示資源異動後必須重新執行 `cargo build --release`，再把 `target/release/docker-gigabyte-ui.exe` 複製到專案根目錄。
6. 若畫面出現大量白色橫線、閃爍、破碎或視窗假死，先判斷是 GPU renderer 問題，不要繼續反覆換 ICO。Windows + Intel Arc 環境可在 `MainWindow::new()` 前設定：
   ```rust
   #[cfg(windows)]
   std::env::set_var("SLINT_BACKEND", "software");
   ```
   這會改用 Slint software renderer，通常會增加少量 CPU 使用量，但可避開 femtovg/OpenGL 顯示驅動造成的畫面破碎。

## Inputs and Outputs

### Inputs

- `D:\docker-gigabyte-ui` 的 Rust／Slint 原始碼、Cargo toolchain 與目前 GUI 行為。
- 遠端 `hch@10.145.119.19` 的 Docker、GPU、heartbeat、容器與 port 實際狀態。
- 使用者指定的修復、編譯、圖示或 release 交付範圍。

### Outputs

- 經驗證的 Rust／Slint 修改或 release EXE（只有使用者要求交付時才產生）。
- Docker heartbeat、容器狀態、GPU、port、Windows no-console 與 GUI 行為的驗證結果。
- 實際修改檔案、建置結果、遠端操作結果與仍待處理的問題。

## Procedure

1. 確認工作目錄與檔案：
   ```text
   D:\\docker-gigabyte-ui\\Cargo.toml
   D:\\docker-gigabyte-ui\\src\\main.rs
   D:\\docker-gigabyte-ui\\ui\\app.slint
   D:\\docker-gigabyte-ui\\build.rs
   D:\\docker-gigabyte-ui\\rust-toolchain.toml
   ```
2. 先檢查現況：讀取 `main.rs`、`app.slint`、`Cargo.toml`，確認不要覆蓋使用者未要求的 UI 或 Docker 行為。
3. 若修改狀態同步，維持 `docker info` heartbeat、5 秒 timer、in-flight guard、online/offline UI 狀態與最後成功同步時間。
4. 若修改 port 顯示，先查遠端容器：
   ```bash
   ssh -o ConnectTimeout=8 -o BatchMode=yes hch@10.145.119.19 \
     "docker inspect <container> --format 'Network={{.HostConfig.NetworkMode}}|Ports={{json .HostConfig.PortBindings}}|Cmd={{json .Config.Cmd}}'"
   ```
   若是 host network，從 command 中解析 `--port`，不要建立虛假的 Docker port binding。
5. 若修改 Windows 子程序，使用 `CommandExt::creation_flags(CREATE_NO_WINDOW)`；不要移除 `windows_subsystem = "windows"`。
6. 使用 MSVC toolchain 編譯。`rust-toolchain.toml` 應保持：
   ```toml
   [toolchain]
   channel = "stable-x86_64-pc-windows-msvc"
   ```
7. 若修改圖示，先依照 Icon design and rendering stability 產生並檢查多尺寸 `app.ico` 與 `app.png`；不要覆蓋尚未備份的使用者資產。
8. 先做格式與型別檢查，再依使用者需求決定是否交付 release EXE。
9. 只有使用者要求「編譯／產生 exe／修好可執行檔」時，才把 release binary 複製到專案根目錄。

## Development and Build Commands

在 Git Bash／bash 環境中，Rust toolchain 位於本機 D 槽時可用：

```bash
cd /d/docker-gigabyte-ui
RUSTUP_TOOLCHAIN=stable-x86_64-pc-windows-msvc \
  /d/.system/rust/cargo/bin/cargo.exe fmt -- --check
RUSTUP_TOOLCHAIN=stable-x86_64-pc-windows-msvc \
  /d/.system/rust/cargo/bin/cargo.exe check
```

交付 release EXE：

```bash
cd /d/docker-gigabyte-ui
RUSTUP_TOOLCHAIN=stable-x86_64-pc-windows-msvc \
  /d/.system/rust/cargo/bin/cargo.exe build --release
cp -f target/release/docker-gigabyte-ui.exe docker-gigabyte-ui.exe
```

MSVC toolchain 是必要的。若改回 GNU toolchain，可能遇到：

```text
error calling dlltool 'dlltool.exe': program not found
```

不要為了繞過這個問題刪除 heartbeat 或改動產品程式；改用可用的 MSVC toolchain。若環境的 toolchain 位置不同，先查 `rustup toolchain list`，再依實際路徑調整命令。

## Remote Diagnostics

目前 UI 連到的遠端主機是 `10.145.119.19`。查容器現況：

```bash
ssh -o ConnectTimeout=8 -o BatchMode=yes hch@10.145.119.19 \
  "docker ps -a --format '{{.Names}}|{{.Status}}|{{.Ports}}'"
```

查特定容器完整 port／network／command：

```bash
ssh -o ConnectTimeout=8 -o BatchMode=yes hch@10.145.119.19 \
  "docker inspect <name> --format 'Network={{.HostConfig.NetworkMode}}|Ports={{json .Config.ExposedPorts}}|Host={{json .HostConfig.PortBindings}}|Cmd={{json .Config.Cmd}}'"
```

若 host network 的 command 是 `--port 8083`，UI URL 應是：

```text
http://10.145.119.19:8083
```

不要把 `Config.ExposedPorts` 誤當成 host port；也不要把容器內的 exposed port 自動當成可從遠端主機連線的 port，除非 network mode／command／實際服務已確認。

## Pitfalls

- Heartbeat 只能確認 SSH 與 Docker daemon，不代表每個應用服務的 HTTP endpoint 一定正常；UI 的「在線」是 Docker online，不是應用層健康檢查。
- 不要只輪詢 heartbeat 而不更新 container snapshot；容器狀態仍需要 `docker ps`／`docker inspect`。
- 不要讓每次背景同步把 `loading` 設成 true；`loading` 應主要保留給手動刷新與啟停操作。
- 不要在 snapshot 失敗時清空舊資料；顯示離線與最後成功同步時間更安全。
- `network_mode: host` 沒有 `HostConfig.PortBindings` 是正常現象，不等於容器沒有可用 port。
- 不要只從 `.Config.ExposedPorts` 推測 host port；host mode 服務常由 `Cmd` 的 `--port` 決定。
- 不要移除 `CREATE_NO_WINDOW`；即使 `windows_subsystem` 存在，SSH／cmd child process 仍可能產生黑框。
- 不要在 SSH command 中輸出 API key、密碼或其他 credentials；本專案目前不需要認證密碼參數。
- 不要使用舊版 GNU toolchain 來編譯這個 Windows 專案，除非已確認完整 MinGW／`dlltool.exe`／linker 可用。
- 不要把舊的 `docker-gigabyte-ui.exe` 當成新 build；每次交付都要檢查 build exit code 與檔案修改時間。
- 不要只把大型 PNG 改副檔名成 `.ico`；Windows EXE 圖示要有多尺寸 ICO entries，否則可能模糊、比例錯誤或載入異常。
- 圖示換完後若截圖出現橫向白色破碎線，優先檢查 Slint GPU renderer／顯示驅動；ICO 本身通常不會造成執行期畫面撕裂。
- 使用 `SLINT_BACKEND=software` 時，必須保留原始碼註解與測試紀錄，避免未來維護者誤以為這是圖示專用設定。
- 不要直接以 `git diff` 判斷版本；目前 `D:\\docker-gigabyte-ui` 可能不是 Git repository，應以實際檔案與 build 結果為準。

## Verification

### Static checks

確認以下項目存在：

```text
D:\\docker-gigabyte-ui\\Cargo.toml
D:\\docker-gigabyte-ui\\Cargo.lock
D:\\docker-gigabyte-ui\\build.rs
D:\\docker-gigabyte-ui\\src\\main.rs
D:\\docker-gigabyte-ui\\ui\\app.slint
D:\\docker-gigabyte-ui\\assets\\app.ico
```

搜尋確認：

```text
---HEARTBEAT---
docker info --format
Duration::from_secs(5)
AtomicBool
server-online
CREATE_NO_WINDOW
windows_subsystem = "windows"
SLINT_BACKEND
software renderer
NetworkMode / host
--port / --port=
serde_json
```

圖示資源確認：

```text
assets/app.ico：存在，且包含多個 16/24/32/48/64/128/256 尺寸 entries
assets/app.png：存在，透明 PNG，通常為 512×512
```

若可用，使用 `System.Drawing.Icon.ExtractAssociatedIcon()` 確認 release EXE 能讀到關聯圖示；不要只以檔案存在判定資源嵌入成功。

### Cargo checks

成功標準：

```text
cargo fmt -- --check  exit code 0
cargo check          Finished `dev` profile ... exit code 0
cargo build --release Finished `release` profile ... exit code 0
```

### Runtime checks

啟動 `D:\\docker-gigabyte-ui\\docker-gigabyte-ui.exe` 後確認：

1. 不會跳出黑色 console 視窗。
2. 標題列顯示 `10.145.119.19`。
3. 遠端可用時顯示綠點、Docker 版本與同步容器數量。
4. 每約 5 秒容器狀態會重新同步。
5. 在遠端停止或啟動容器後，下一輪同步會更新 UI 的狀態與操作按鈕。
6. 遠端失聯時顯示紅點、離線與最後成功同步時間。
7. 離線時啟動／停止／重啟／開啟 URL 按鈕停用。
8. `host` network 且 command 使用 `--port 8083` 的容器顯示 `8083`，不是 `-`。
9. 選取該容器後，開啟網址使用 `http://10.145.119.19:8083`。
10. GPU 容器仍顯示 GPU 標記與 `nvidia-smi` 資訊。

## Output Checklist

完成任務後回報：

- 實際修改的檔案。
- 是否修改 heartbeat、port parsing 或 Windows process flags。
- `cargo fmt`、`cargo check`、`cargo build --release` 的實際結果。
- 圖示資產是否已更新，以及是否保留暫存備份路徑。
- 是否修改 `SLINT_BACKEND`；若有，說明是為了解決哪一類 renderer／顯示驅動問題。
- release EXE 是否已複製到 `D:\\docker-gigabyte-ui\\docker-gigabyte-ui.exe`。
- 是否完成 EXE 關聯圖示讀取與啟動穩定性測試。
- 是否完成遠端 runtime 測試；若未測試，明確標示需要使用者手動確認。
- 不要回報未實際執行的測試，也不要把舊 EXE 當成新版本。

## Rules and Limitations

- `D:\\docker-gigabyte-ui` 是本專案原始碼與交付 EXE 位置；不要把設定或程式碼散落到其他 agent 目錄。
- 遠端 Docker 操作只針對本專案已指定的 `hch@10.145.119.19`，不要自行切換其他 Docker context。
- 變更 UI、heartbeat、port parsing 或 process flags 時採最小修改原則。
- 不要執行破壞性 Docker 指令，例如刪除容器、volume、image 或整個 compose project；這個 Skill 只維護控制 GUI。
- 不要暴露 SSH credentials、Docker Hub credentials、API keys 或遠端服務的敏感設定。
