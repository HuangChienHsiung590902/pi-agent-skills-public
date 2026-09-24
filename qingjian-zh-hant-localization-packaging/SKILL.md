---
name: qingjian-zh-hant-localization-packaging
description: 當使用者要求將 D:\qingjian-main Qingjian／清簡輸入法的程式文字統一為臺灣繁體中文，或重新編譯、打包 Windows 安裝程式時使用；涵蓋 OpenCC 轉換、UI 用語校正、Windows release build、32/64 位 TSF 與 Inno Setup 驗證。
---

# Qingjian 臺灣繁體中文在地化與 Windows 打包

## When to Use

使用者提出以下需求時載入：

- 將 Qingjian／清簡輸入法程式碼、設定頁或偏好設定中的簡體中文改成繁體中文。
- 將「云服务、接口地址、API 密钥、配置文件、环境变量」等用語改成臺灣繁體中文。
- 修改 `D:\qingjian-main` 後重新建置 Windows Server、TSF DLL、設定程式或安裝包。
- 需要把本次修改打成新的 `.exe` 安裝程式。

本 Skill 只處理程式文字與 Windows 打包；按鍵規則、輸入法核心功能與一般 Qingjian 開發流程仍遵循 `qingjian-ime-development`。

## 固定環境

- 專案根目錄：`D:\qingjian-main`
- Windows 安裝器腳本：`D:\qingjian-main\apps\windows\installer\build.ps1`
- Inno Setup 7：通常位於：
  `C:\Users\Administrator\AppData\Local\Programs\Inno Setup 7\ISCC.exe`
- Rust MSVC 工具鏈可能位於：
  `D:\MY-DISK-C\rust\rustup\toolchains\stable-x86_64-pc-windows-msvc\bin`
- Rust Cargo：
  `D:\MY-DISK-C\rust\cargo\bin`
- 正式產物：
  - `target\release\qingjian-server.exe`
  - `target\release\qingjian-settings.exe`
  - `target\release\qingjian_tsf.dll`
  - `target\i686-pc-windows-msvc\release\qingjian_tsf.dll`
  - `target\installer\qingjian-<版本>-windows-x86_64-setup.exe`

## Procedure

### 1. 先讀取專案規範與既有 Skill

先完整讀取：

```text
D:\OB\skills\qingjian-ime-development\SKILL.md
D:\qingjian-main\AGENTS.md
D:\qingjian-main\CLAUDE.md
D:\qingjian-main\docs\contributing.md
```

確認目前版本、工作樹狀態、Windows Server 是否正在執行，以及 `ISCC.exe` 是否存在。不要把終端機中文亂碼誤判成編譯錯誤，應以 exit code、產物與 Inno Setup 的 `Successful compile` 為準。

### 2. 搜尋簡體文字並建立備份

先限定在程式碼與使用者可見 UI，避免直接修改詞庫資料、模型、API 欄位、設定鍵名、拼音資料與測試協議內容：

```bash
rg -n --text -i "云|启用|开启|关闭|接口|密钥|配置文件|环境变量|默认|输出|联想|词格|整句|自动修正" \
  D:/qingjian-main/apps D:/qingjian-main/crates D:/qingjian-main/tools
```

修改前備份至少包含：

```text
apps/windows/settings/src
apps/macos/src/preferences
apps/windows/server/src
apps/windows/tsf/src
crates/qingjian-predict/src
crates/qingjian-platform/src
```

### 3. 執行簡體到臺灣繁體轉換

專案已使用 `ferrous-opencc`，優先採用 `BuiltinConfig::S2twp` 做批次轉換。不要直接用沒有臺灣詞彙轉換的 `S2t`，否則「软件、默认、文件、网络」可能只變成一般繁體而不是臺灣用語。

轉換時：

- 只處理 UTF-8、程式碼註解、使用者可見字串與文件。
- 不修改 Rust identifier、TOML key、JSON/API 欄位、環境變數名稱、檔名、URL、模型 ID、測試協議值。
- 不轉換詞庫、`.qj`、模型檔、學習資料與使用者設定。
- OpenCC 轉換後要人工檢查 UI 文字，尤其是「介面位址、API 金鑰、設定檔、環境變數、測試連線、程序」等詞。

建議用暫存 Rust helper 呼叫 `ferrous_opencc::OpenCC::from_config(BuiltinConfig::S2twp)`；若 Cargo 工具鏈無法直接執行，先使用 MSVC `rustc.exe` 與 Cargo 的絕對路徑，不要任意重新安裝 Rust。

### 4. 人工校正臺灣用語

至少檢查並統一：

| 原文 | 目標用語 |
|---|---|
| 云服务 | 雲服務 |
| 启用／开启／关闭 | 啟用／開啟／關閉 |
| 接口地址 | 介面位址 |
| API 密钥 | API 金鑰 |
| 配置文件 | 設定檔 |
| 环境变量 | 環境變數 |
| 默认值 | 預設值 |
| 测试连接 | 測試連線 |
| 进程 | 程序 |
| 联想 | 聯想 |
| 词库 | 詞庫 |
| 输出 | 輸出 |
| 文件 | 檔案（依語境判斷；技術協議中的 file 可保留英文） |
| 网络 | 網路 |
| 软件 | 軟體 |
| 屏幕 | 螢幕 |
| 数据 | 資料 |

「雲端」「整句補全」「本地整句模型」等產品術語要前後一致；API、preedit、token、Server、TSF、OpenCC 等技術名稱不要翻譯成不自然的中文。

### 5. 編譯驗證

Windows 本機可用下列環境執行：

```powershell
$env:PATH = "D:\MY-DISK-C\rust\cargo\bin;D:\MY-DISK-C\rust\rustup\toolchains\stable-x86_64-pc-windows-msvc\bin;$env:PATH"
cargo check --locked -p qingjian-windows-server -p qingjian-windows-tsf -p qingjian-windows-settings
```

若使用絕對路徑，需確保 `rustc.exe` 與 `cargo.exe` 是同一個 MSVC toolchain。最低驗證必須成功，才可進入打包。

### 6. 建立新的 Windows 安裝包

不要用 `-SkipBuild`，除非已確認 release 產物就是本次修改後產物。正式流程：

```powershell
$env:QINGJIAN_ISCC = 'C:\Users\Administrator\AppData\Local\Programs\Inno Setup 7\ISCC.exe'
$env:PATH = "D:\MY-DISK-C\rust\cargo\bin;D:\MY-DISK-C\rust\rustup\toolchains\stable-x86_64-pc-windows-msvc\bin;$env:PATH"
pwsh -NoProfile -ExecutionPolicy Bypass -File D:\qingjian-main\apps\windows\installer\build.ps1
```

若 checkout 沒有 `.git`，`build.ps1` 在開發版組合版本後綴時可能因 `git rev-parse` 失敗而中止。此時不要偽造 git hash；可暫時使用只回傳失敗的 `git` shim，讓安裝包採用原本的 `0.1.x-dev` 版本，並在回報中明確說明沒有 git metadata。

若要用 `-SkipBuild` 重新編 Inno Setup，仍要先確認以下四個產物的時間與內容是本次 release build：

```text
target\release\qingjian-server.exe
target\release\qingjian-settings.exe
target\release\qingjian_tsf.dll
target\i686-pc-windows-msvc\release\qingjian_tsf.dll
```

### 7. 記錄產物與雜湊

```bash
ls -lh D:/qingjian-main/target/installer/qingjian-*-windows-x86_64-setup.exe
sha256sum D:/qingjian-main/target/installer/qingjian-*-windows-x86_64-setup.exe
```

回報時包含完整路徑、版本、檔案大小、SHA-256、是否簽章、`uiAccess` 狀態與實際驗證命令。不要聲稱已安裝，除非使用者明確要求安裝並且確實執行安裝程式。

## Pitfalls

- 不要把 `API_KEY`、`QINGJIAN_API_KEY`、TOML key、模型 ID、URL 或 JSON 欄位翻譯掉。
- 不要對詞庫、語言模型、`.qj`、`.qjm` 或使用者學習資料做全文 OpenCC 轉換。
- 不要只改 Windows 設定頁而漏掉 macOS 偏好設定頁；使用者可見文字應保持跨平台一致。
- 不要用一般 `S2t` 取代 `S2twp`，否則臺灣用語不完整。
- 不要只依終端機亂碼判定失敗；PowerShell／Inno Setup 的中文輸出可能因 code page 顯示錯誤。
- 沒有 `.git` 時不要自行填入假的 commit hash。
- `uiAccess=1` 需要受信任簽章；一般可分發安裝包應使用未簽章、`uiAccess=0`。
- 安裝完成後，已載入舊 TSF DLL 的應用程式需要重新開啟；Server 正在執行時也要留意安裝器的程序關閉流程。

## Verification

1. `cargo check --locked -p qingjian-windows-server -p qingjian-windows-tsf -p qingjian-windows-settings` 成功。
2. 四個 Windows release 產物存在，且時間晚於本次修改。
3. Inno Setup 輸出 `Successful compile`。
4. `target/installer/qingjian-<版本>-windows-x86_64-setup.exe` 存在。
5. 已產生並回報 SHA-256。
6. 搜尋 Windows／macOS 設定頁不再出現目標簡體 UI 詞彙，且 `介面位址`、`API 金鑰`、`設定檔`、`環境變數`、`測試連線` 等目標詞存在。
7. 若本 Skill 被更新，執行：

```powershell
python D:\OB\skills\skill-creator\scripts\quick_validate.py D:\OB\skills\qingjian-zh-hant-localization-packaging
python D:\OB\skills\obsidian-mcp-skill-router\scripts\rebuild_skills_index.py
```
