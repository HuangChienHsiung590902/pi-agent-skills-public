---
name: ecp-icon-patch
description: 替換 aipower ECP 系統圖示（phosphor 風格，透明背景）。用在：圖示模糊需更換、重新套用 phosphor 風格、從備份回滾。操作對象為 aipower-module-base 和 quicksilver-module-main 兩個 jar 內的 META-INF/resources/ecp/image/ 路徑。
---

# ECP 圖示替換（Phosphor 風格）

## 背景

`aipower-module-base-7.3.12.5.jar` 和 `quicksilver-module-main-7.2.2.jar` 內的原始圖示大量為 16×16 px 低解析點陣圖，且白色背景不透明。2026-06-23 已用 Phosphor Icons 全面替換（彩色、透明背景），共替換 aipower 378 個、quicksilver 351 個 entry。

## 關鍵路徑

| 用途 | 路徑 |
|------|------|
| 兩個 JAR 所在目錄 | `C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib\` |
| 工作目錄（生成腳本） | `C:\Users\HCH\fluent-icon-work\` |
| 生成腳本 | `C:\Users\HCH\fluent-icon-work\generate-phosphor-assets.js` |
| 生成結果 | `C:\Users\HCH\fluent-icon-work\phosphor-generated\` |
| 穩定備份（已知可用的乾淨版本） | `*.jar.20260623_095844.bak`（同目錄） |
| 套用腳本 | `C:\Users\HCH\apply_phosphor_icons_safe.ps1`（Codex sandbox 生成，不保證保留，需重建） |

## 圖示對應表（generate-phosphor-assets.js）

| 語意 key | Phosphor 圖示 | 顏色 |
|----------|--------------|------|
| check | check-circle | `#16a34a` |
| cross | x-circle | `#dc2626` |
| delete | trash | `#dc2626` |
| workbench | clipboard-text | `#8b5cf6` |
| phone | phone-call | `#0284c7` |
| chat | chat-circle-dots | `#16a34a` |
| unit | cube | `#2563eb` |
| page | file-text | `#4f46e5` |
| export | export | `#f59e0b` |
| menu | list-bullets | `#64748b` |
| settings | gear-six | `#7c3aed` |
| report | chart-bar | `#0ea5e9` |
| contact | address-book | `#0d9488` |
| user | user-circle | `#06b6d4` |
| group | users | `#06b6d4` |
| office | buildings | `#64748b` |
| table | table | `#2563eb` |
| folder | folder | `#ca8a04` |
| mail | envelope | `#0284c7` |
| database | database | `#2563eb` |
| calendar | calendar | `#0284c7` |
| edit | pencil-simple | `#7c3aed` |
| save | floppy-disk | `#2563eb` |
| search | magnifying-glass | `#64748b` |
| print | printer | `#64748b` |

生成尺寸：`10, 12, 13, 14, 15, 16, 17, 20, 23, 24, 32, 48, 64` px，每個 key 輸出 `.svg` + 各尺寸 `.png` + 各尺寸 `.gif`。

## 安全排除規則（apply 腳本必須遵守）

**不替換**以下語意的 entry：
- logo、背景（background）
- 寬圖（>2:1 寬高比）或大圖（>128px）
- emoji、sticker、fromDevice、loading
- 第三方套件圖（Facebook、CKEditor sprite、note 等）

**只替換**：功能性小圖示（≤64px，1:1 或接近正方形）

## 操作流程

### 前置：確認 Tomcat / CbmLite 已停

jar 被 Java process 佔用時無法覆蓋。先停：

```powershell
# 找到鎖住 jar 的 process
Get-Process java -ErrorAction SilentlyContinue | ForEach-Object { "$($_.Id) $($_.Path)" }

# 停掉 Tomcat（透過 shutdown.bat 較優雅，或直接 Stop-Process）
& 'C:\com\chainsea\apache-tomcat\bin\shutdown.bat'
# CbmLiteServer 無 shutdown script，直接 kill
Stop-Process -Name java -Force  # 注意：會殺所有 java，確認沒其他重要 process
```

### 1. 從穩定備份還原（可選，回滾或重置用）

```powershell
$lib = 'C:\com\chainsea\apache-tomcat\webapps\aipower\WEB-INF\lib'
Copy-Item "$lib\aipower-module-base-7.3.12.5.jar.20260623_095844.bak" `
          "$lib\aipower-module-base-7.3.12.5.jar" -Force
Copy-Item "$lib\quicksilver-module-main-7.2.2.jar.20260623_095844.bak" `
          "$lib\quicksilver-module-main-7.2.2.jar" -Force
```

### 2. 重新生成 Phosphor 資產

```powershell
cd 'C:\Users\HCH\fluent-icon-work'
node generate-phosphor-assets.js
# 輸出：generated 25 phosphor icons（到 phosphor-generated\）
```

依賴：`npm install`（需 `sharp`、`@phosphor-icons/core`）已在 `fluent-icon-work\node_modules\`。

### 3. 套用到 JAR（apply_phosphor_icons_safe.ps1 核心邏輯）

腳本邏輯（如需重建）：
1. 備份兩個 jar（`*.jar.before_phosphor_<timestamp>.bak`）
2. 讀取 `phosphor-generated\` 下所有 key 對應的 PNG/GIF/SVG
3. 開啟 jar（ZIP API：`[System.IO.Compression.ZipFile]`）
4. 遍歷 jar 內 entry，符合安全條件者依 key 語意 + 尺寸匹配，覆寫 entry bytes
5. 儲存 jar

```powershell
# 直接執行已存在的腳本（如果 Codex sandbox 保留了它）
& 'C:\Users\HCH\apply_phosphor_icons_safe.ps1'
# 預期輸出：
# generated 25 phosphor icons
# UPDATED aipower-module-base-7.3.12.5.jar 378
# UPDATED quicksilver-module-main-7.2.2.jar 351
```

### 4. 清快取並重啟

```powershell
$tomcat = 'C:\com\chainsea\apache-tomcat'
# 清 Tomcat 快取
Remove-Item "$tomcat\work\Catalina\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$tomcat\temp\*" -Recurse -Force -ErrorAction SilentlyContinue

# 重啟（背景執行）
Start-Process cmd.exe -ArgumentList '/c','server.bat' -WorkingDirectory 'C:\com\chainsea' -WindowStyle Hidden
```

### 5. 驗證

等 Tomcat 啟動（約 8–15 秒），開：
```
http://127.0.0.1:22821/aipower/Qs.MainFrame.page?cacheBust=<timestamp>
```

## 坑

1. **jar 被鎖住**：一定要先停 Tomcat（PID 掃 java.exe + Bootstrap）和 CbmLiteServer（PID 掃 CbmLiteServer）再動 jar。
2. **GIF 透明問題**：GDI+ 存 GIF 會吃掉 alpha，必須用自定 GIF89a encoder 或 `sharp` 的 `.gif()` 輸出。`sharp` 方式已在 `generate-phosphor-assets.js` 實作。
3. **PNG 路徑 ≠ SVG 路徑**：jar 內 `.png` / `.gif` entry 不能直接換成 SVG bytes（MIME 問題），只能換同格式。`.svg` entry 可直接換 SVG。
4. **`095844` 是唯一已知乾淨備份**：這個時間戳的 bak 是原廠圖示，其餘 bak 都是各次嘗試的中間態。
5. **Edge 快取**：重啟後要清 Edge 快取或加 cacheBust 參數，否則舊圖示會從瀏覽器快取讀取。

---

## Conformance Addendum

## When to Use
替換 aipower ECP 系統圖示（phosphor 風格，透明背景）。用在：圖示模糊需更換、重新套用 phosphor 風格、從備份回滾。操作對象為 aipower-module-base 和 quicksilver-module-main 兩個 jar 內的 META-INF/resources/ecp/image/ 路徑。

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

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
