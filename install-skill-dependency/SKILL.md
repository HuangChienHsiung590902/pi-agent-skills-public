---
name: install-skill-dependency
version: 1.0.0
description: 診斷並修復已安裝技能所需的缺失依賴項、執行檔或執行環境。當技能因缺少資源、找不到執行檔、執行時依賴項不可用而失敗時，或當使用者希望主動安裝所有技能依賴項時使用。
description_zh: 診斷並修復已安裝技能所需的缺失依賴、二進位檔案或執行時環境。當技能因缺少資源、二進位檔案未找到、執行時依賴不可用而失敗時，或當使用者希望主動安裝所有技能依賴項時使用。
---

# 安裝技能依賴項

掃描已安裝的技能、偵測缺失的依賴項，並在使用者授權後進行安裝。

## 工作流程

依序執行以下步驟：

### 第一步：偵測環境

1. **偵測作業系統類型**：判斷為 macOS、Linux 或 Windows。
2. **偵測是否在虛擬機器/沙盒中執行**：檢查以下指標：
   - `/root/.qoderwork` 是否存在（容器/虛擬機器中常見）
   - 環境變數，如 `CODESPACES`、`GITPOD_WORKSPACE_ID`、`REMOTE_CONTAINERS`
   - 在 Linux 環境中以 root 使用者執行
3. **調查現有工具鏈**：檢查常見工具的安裝狀態與版本：
   - 套件管理器：`brew`、`apt`、`yum`、`dnf`、`pacman`、`choco`、`winget`
   - 執行環境：`node`、`python`、`python3`、`ruby`、`java`、`go`、`rust`/`cargo`
   - 語言套件管理器：`npm`、`yarn`、`pnpm`、`pip`、`pip3`、`gem`、`mvn`、`gradle`
   - 其他常用工具：`git`、`curl`、`wget`、`jq`、`ffmpeg`、`imagemagick`、`pandoc`、`poppler`

可用時使用 `CheckRuntime` 工具進行執行環境偵測。其他工具請透過 Bash 使用 `which` 或 `command -v`。

向使用者呈現已偵測環境的摘要表格。

### 第二步：掃描技能依賴項

1. **找到技能目錄**：依序檢查以下路徑：
   - macOS / Linux：`~/.qoderwork/skills/`
   - Linux 虛擬機器 / 容器：`/root/.qoderwork/skills/`
   - Windows：`%USERPROFILE%\.qoderwork\skills\`
2. **解析每個技能**：對每個包含 `SKILL.md` 的子目錄：
   - 讀取 `SKILL.md` 內容
   - 從以下位置提取依賴項資訊：
     - 明確的依賴項宣告（例如：frontmatter 中的 `requires:`）
     - 參照 `pip install`、`npm install`、`brew install`、`apt install` 等的程式碼區塊
     - import 陳述式或工具呼叫（例如：`import pdfplumber`、`ffmpeg`、`pandoc`）
     - 技能中參照的腳本檔案（例如：`scripts/*.py`、`scripts/*.sh`）
     - 技能目錄中的 `requirements.txt`、`package.json` 或類似清單檔案
3. **與環境比對**：將提取的依賴項與第一步結果進行交叉比對。
4. **回報結果**：呈現包含以下欄位的表格：
   - 技能名稱
   - 所需依賴項
   - 目前狀態（已安裝 / 缺失 / 版本不符）
   - 建議的安裝指令

若所有依賴項均已滿足，通知使用者並停止。

### 第三步：規劃安裝

根據作業系統類型選擇適當的策略：

**macOS / Linux：**
- 大多數安裝可自主處理。
- **虛擬機器/沙盒環境**：可較自由地進行，但安裝前仍需詢問。
- **主機環境**：須謹慎。詢問授權前，清楚說明每個步驟的目的與影響。
- 系統級工具優先使用系統套件管理器（macOS 用 `brew`，Linux 用 `apt`/`yum`/`dnf`）。
- 函式庫依賴項使用語言專屬套件管理器（`pip`、`npm`）。
- 適時考慮使用虛擬環境（`venv`、`nvm`）以避免污染系統環境。

**Windows：**
- 由於環境差異較大，使用 `AskUserQuestion` 告知使用者需要安裝的內容並推薦安裝方式。讓使用者確認或手動執行。

### 第四步：優化下載來源

根據使用者的地區（從系統語言、時區推斷，或直接詢問），設定適當的鏡像：

| 工具      | 中國鏡像         | 指令                                                                          |
|-----------|-----------------|-------------------------------------------------------------------------------|
| npm       | 淘寶 registry   | `npm config set registry https://registry.npmmirror.com`                      |
| pip       | 清華鏡像         | `pip install -i https://pypi.tuna.tsinghua.edu.cn/simple`                     |
| Homebrew  | 中科大鏡像       | 設定 `HOMEBREW_BREW_GIT_REMOTE` 和 `HOMEBREW_CORE_GIT_REMOTE`                 |

僅在使用者位於有助益的地區時才套用鏡像設定。更改全域設定前請先詢問使用者。

### 第五步：執行安裝

**重要規則**：在執行任何安裝、升級或移除操作之前，必須使用 `AskUserQuestion` 取得使用者的明確授權。

向使用者呈現：
- 將要安裝的內容
- 將執行的確切指令
- 任何副作用或系統變更

獲得授權後：
1. 每次安裝一個依賴項
2. 移至下一個之前先驗證每次安裝是否成功
3. 回報每個依賴項的成功或失敗狀態

### 第六步：驗證與回報

所有安裝完成後：
1. 重新執行第二步的依賴項掃描
2. 確認所有依賴項均已滿足
3. 呈現最終摘要，顯示已安裝的內容及目前狀態

## 處理不確定情況

若在任何時刻對以下事項不確定：
- 某個依賴項是否真正必要
- 目前作業系統的正確套件名稱
- 安裝是否可能與現有軟體衝突

請使用 `AskUserQuestion` 向使用者尋求指引。詢問永遠優於做出錯誤假設。

## 互動流程範例

```
1. 環境偵測：
   OS: macOS 14.0（主機）
   Homebrew: 已安裝 (4.2.0)
   Node.js: 已安裝 (v20.11.0)
   Python: 已安裝 (3.12.1)
   pip: 已安裝 (24.0)
   ffmpeg: 未安裝
   pandoc: 未安裝

2. 技能依賴項掃描：
   ┌─────────────┬──────────────┬─────────┬─────────────────────┐
   │ 技能        │ 依賴項       │ 狀態    │ 安裝指令            │
   ├─────────────┼──────────────┼─────────┼─────────────────────┤
   │ pdf         │ pdfplumber   │ 缺失    │ pip install pdfplum… │
   │ pdf         │ poppler      │ 缺失    │ brew install poppler │
   │ docx        │ python-docx  │ 正常    │ -                   │
   │ pptx        │ python-pptx  │ 缺失    │ pip install python-… │
   │ xlsx        │ openpyxl     │ 正常    │ -                   │
   └─────────────┴──────────────┴─────────┴─────────────────────┘

3. [AskUserQuestion] 請求授權安裝缺失的依賴項
4. 執行已批准的安裝
5. 驗證所有依賴項均已滿足
```

---

## Conformance Addendum

## When to Use
診斷並修復已安裝技能所需的缺失依賴項、執行檔或執行環境。當技能因缺少資源、找不到執行檔、執行時依賴項不可用而失敗時，或當使用者希望主動安裝所有技能依賴項時使用。

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
