---
name: ecp-toolset
description: Chainsea ECP (aipower, C:\com\chainsea) 內建維運工具集 chainsea\tool\*.bat 的用途與互動流程說明——資料源設定/測試/加密 (dsconfig/dstest/dsenc)、多語系檢查 (i18n-check)、授權檢查 (license-check)、資料匯出匯入 (export/import)。當使用者要設定或測試資料庫連線、加密 datasource.xml 密碼、檢查 i18n 缺漏、確認授權到期日、或要匯出/匯入資料時使用。
---

# ECP 維運工具集 (`chainsea\tool\*.bat`)

`chainsea\tool` 下的每個 `.bat` 都只是用內建 JRE 呼叫 `lib/*` classpath 裡的一個 Quicksilver toolset class，
沒有另外的編譯/建置步驟。**必須先 `cd chainsea\tool` 再執行**，因為 bat 內用的是相對路徑
（`..\jre\bin\java`、`lib/*`）。設定檔在 `tool\config\`（`instance.xml`、`server.xml`、`export.xml`、
`log4j2.xml`、`license-check-config.xml`）。

## 工具總覽

| Script | Java class | 用途 | 互動方式 |
|---|---|---|---|
| `dsconfig.bat` | `...datasource.DsConfig` | 檢視/新增/編輯/刪除/測試資料源設定 | **互動選單**（`1` 檢視 default、`A` 新增、`T` 測試全部、`S` 存檔離開、`Q` 不存檔離開） |
| `dstest.bat` | `...datasource.DsTest` | 測試目前設定的資料源連線是否正常 | 直接執行，非互動，印出結果後結束 |
| `dsenc.bat` | `...datasource.DsEncrypt` | 加密 `datasource.xml` 裡的欄位（url/user/password） | 需帶 `-f` 參數，例如 `dsenc -f default.password`；**會直接改寫設定檔**，不帶參數只會印用法 |
| `i18n-check.bat` | `...i18n.I18nCheck` | 掃描檔案與資料庫，找出缺漏/多餘的多語系文字資源 | 直接執行，唯讀，結果寫到 `tool\log\i18n-check-file.log` 與 `i18n-check-db.log` |
| `license-check.bat` | `...license.LicenseCheck` | 印出目前授權（到期日、使用人數/IP限制） | 直接執行，唯讀 |
| `export.bat` | `...migrate.DsExport` | 依 `config/export.xml` 設定匯出資料 | **互動**：先問要用哪個 export 設定檔（預設 `config/export.xml`），再往下走 |
| `import.bat` | `...migrate.DsImport` | 將匯出的資料匯入指定資料源 | **互動**：先問資料源設定檔路徑（相對於 `apache-tomcat/extension/aipower`），再問資料目錄（相對於 `tool/data`）——**會寫入資料庫，正式執行前務必確認方向和目標環境** |

## 已驗證的行為（實測，2026-07-07）

- `dsconfig.bat` 目前 default 資料源：`com.jeedsoft.marialocal.MariaLocalDriver`，
  URL `jdbc:marialocal:${root}/mariadb/data/default`，user `root`，密碼空白，pool 3~100。
- `dstest.bat` 連線正常：`MariaDB 10.11.5-MariaDB`。
- `license-check.bat` 印出的到期日是 `2025-01-01`——**已過期**，若遇到授權相關的功能限制/警告先檢查這裡。
- `dsenc.bat` 不帶 `-f` 只會印用法（`Missing required option: f`），不會誤改檔案；真的要加密某欄位才會動到 `datasource.xml`。
- `export.bat` / `import.bat` 在還沒輸入完所有互動問題前不會真的動資料；只按 Enter 接受預設值後遇到 EOF 會安全中止（不留殘檔、不寫 DB）。

## 常見用法

```bat
cd C:\com\chainsea\tool
dstest.bat                     REM 快速確認 DB 連線活著
dsconfig.bat                   REM 互動修改資料源設定（選 1 檢視，B 返回，S 存檔）
license-check.bat              REM 檢查授權是否過期
i18n-check.bat                 REM 找多語系缺漏（結果在 tool\log\）
dsenc -f default.password      REM 加密 datasource.xml 裡 default 的密碼欄位
```

## 注意事項

- `import.bat` 會實際寫入資料庫，且路徑是相對於 `apache-tomcat/extension/aipower` 和 `tool/data` 算的，
  執行前務必確認來源資料目錄與目標資料源是對的，避免覆蓋到正式資料。
- `apache-tomcat/extension/aipower/config/datasource.xml` 才是 App（server.bat 啟動的 webapp）實際使用的資料源設定；
  `tool\config` 底下的設定檔是給這些維運工具用的，兩者不是同一份，改的時候不要搞混。

---

## Conformance Addendum

## When to Use
Chainsea ECP (aipower, C:\com\chainsea) 內建維運工具集 chainsea\tool\*.bat 的用途與互動流程說明——資料源設定/測試/加密 (dsconfig/dstest/dsenc)、多語系檢查 (i18n-check)、授權檢查 (license-check)、資料匯出匯入 (export/import)。當使用者要設定或測試資料庫連線、加密 datasource.xml 密碼、檢查 i18n 缺漏、確認授權到期日、或要匯出/匯入資料時使用。

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

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
