---
name: rust-dev-on-frozen-c-drive
description: 在 C 槽被 Deep Freeze 等磁碟凍結軟體保護的 Windows 機器上，把 Rust 工具鏈與 C 編譯器完整安裝到未凍結的磁碟（如 D 槽），避免重開機後整套開發環境消失。
---

# 凍結 C 槽上的 Rust 開發環境安裝

## When to Use
當使用者的系統碟（通常是 C）被磁碟凍結軟體保護、重開機會還原到快照狀態，但想安裝 Rust 開發環境（rustup/cargo/rustc）且需要能編譯依賴原生 C 程式碼的 crate（例如 rusqlite 的 bundled sqlite），必須確保整套工具鏈（含 C 編譯器）都裝在不會被清空的磁碟上。

## Procedure
1. 用環境變數把 rustup/cargo 的安裝路徑導到非系統碟，例如：`CARGO_HOME=D:\.system\rust\cargo`, `RUSTUP_HOME=D:\.system\rust\rustup`，用 PowerShell 的 `[System.Environment]::SetEnvironmentVariable(name, value, 'User')` 設成永久使用者環境變數（不要只 export，重開機/新 shell 也要生效）
2. 下載官方 rustup-init.exe（https://static.rust-lang.org/rustup/dist/x86_64-pc-windows-msvc/rustup-init.exe），在已設定好上述環境變數的 shell 中執行 `rustup-init.exe -y --default-toolchain stable --profile default --no-modify-path`
3. 把 cargo 的 bin 目錄（如 `D:\.system\rust\cargo\bin`）加進使用者 PATH 環境變數，同樣用 SetEnvironmentVariable 永久設定
4. 檢查是否已有 C 編譯器（cl.exe / gcc.exe），若沒有 MSVC Build Tools 也沒有 MinGW，且不想在會被凍結的 C 槽裝笨重的 Visual Studio 安裝程式，改用 winget 安裝可攜式 MinGW-w64（例如 BrechtSanders.WinLibs.POSIX.UCRT），關鍵是加上 `--location "D:\<某路徑>"` 參數，讓 winget 把整個工具鏈解壓縮到 D 槽而不是預設的 C 槽 Program Files
5. 注意 ABI 相容性：rustup 預設裝的是 MSVC host triple（stable-x86_64-pc-windows-msvc），但 MinGW-w64 gcc 是 GNU ABI，兩者連結器不相容。必須額外安裝 GNU 版工具鏈：`rustup toolchain install stable-x86_64-pc-windows-gnu`，並用 `rustup default stable-x86_64-pc-windows-gnu` 設為預設，才能讓 cargo build 呼叫 MinGW gcc 成功連結
6. 確認 MinGW 的 bin 目錄也在 PATH 中（winget 安裝時通常會自動處理，但要驗證）
7. 建立測試 Rust 專案（`cargo new`），加入需要原生編譯的依賴（如 rusqlite 的 bundled feature）做 `cargo build` 驗證整條鏈路，確認能成功編譯出執行檔

## Pitfalls
- 不要只用 `export` 設定 CARGO_HOME/RUSTUP_HOME/PATH，那只在當前 shell session 有效，換一個新終端機或重開機後就會遺失，必須用 PowerShell 的 User 層級永久環境變數
- rustup-init.exe 預設會裝到 `C:\Users\<user>\.cargo` 和 `.rustup`，如果沒有先設定好 CARGO_HOME/RUSTUP_HOME 環境變數就直接執行安裝程式，整套工具鏈會裝進會被凍結清空的 C 槽
- MSVC 版和 GNU 版 Rust 工具鏈不能混用 gcc 連結器，如果編譯時出現找不到 link.exe 或連結器相關錯誤，先檢查 `rustc --version --verbose` 的 host 是 msvc 還是 gnu，是否跟目前的 C 編譯器（MSVC cl.exe 或 MinGW gcc.exe）配對正確
- `winget install` 若不加 `--location` 參數，很多套件會裝到 C 槽的 Program Files 或 AppData，一定要顯式指定安裝路徑到未凍結磁碟
- 在會被凍結的 C 槽裝 Visual Studio Build Tools 這種大型安裝程式風險高（裝完可能重開機就沒了、又佔用大量時間下載安裝），優先考慮免安裝的可攜式 MinGW-w64 方案

## Verification
1. `rustc --version` 和 `cargo --version` 在新開的終端機（不手動 export 任何變數）也能直接執行，不需要打完整路徑
2. `rustc --version --verbose` 顯示的 host 是 x86_64-pc-windows-gnu（若搭配 MinGW gcc）
3. 檢查 `C:\Users\<user>\.cargo` 和 `.rustup` 確實不存在（`ls` 回報 No such file or directory），代表沒有東西誤裝到 C 槽
4. 在測試專案中 `cargo build` 一個含原生依賴（如 rusqlite bundled）的套件能編譯成功且能執行

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
