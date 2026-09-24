---
name: notebooklm
description: 當需要直接從 Claude Code 查詢 Google NotebookLM 筆記本時使用，可透過 Gemini 取得有來源依據、附引用的答案。支援瀏覽器自動化、筆記庫管理、持久登入驗證，並透過僅使用文件內容回應大幅降低幻覺。
---

# NotebookLM 研究助理 Skill

與 Google NotebookLM 互動，透過 Gemini 有來源依據的回答來查詢文件。每次提問都會開啟新的瀏覽器工作階段，僅從您上傳的文件中擷取答案，完成後關閉。

## 使用時機

在以下情況觸發此 skill：
- 使用者明確提到 NotebookLM
- 使用者提供 NotebookLM 網址（`https://notebooklm.google.com/notebook/...`）
- 使用者想查詢其筆記本／文件
- 使用者想將文件加入 NotebookLM 筆記庫
- 使用者說「問我的 NotebookLM」、「查看我的文件」、「查詢我的筆記本」等

## 重要：Add 指令 - 智慧探索

當使用者想新增筆記本但未提供詳細資訊時：

**智慧新增（建議）**：先查詢筆記本以探索其內容：
```bash
# 步驟 1：查詢筆記本內容
python scripts/run.py scripts/ask_question.py --question "What is the content of this notebook? What topics are covered? Provide a complete overview briefly and concisely" --notebook-url "[URL]"

# 步驟 2：使用探索到的資訊新增筆記本
python scripts/run.py scripts/notebook_manager.py add --url "[URL]" --name "[依內容填寫]" --description "[依內容填寫]" --topics "[依內容填寫]"
```

**手動新增**：若使用者已提供所有詳細資訊：
- `--url` - NotebookLM 網址
- `--name` - 描述性名稱
- `--description` - 筆記本包含的內容（必填！）
- `--topics` - 以逗號分隔的主題（必填！）

絕不猜測或使用籠統描述！若缺少詳細資訊，請使用智慧新增來探索。

## 重要：一律使用 scripts/run.py 包裝器

**絕不直接呼叫腳本。一律使用 `python scripts/run.py [script]`：**

```bash
# 正確 - 一律使用 scripts/run.py：
python scripts/run.py scripts/auth_manager.py status
python scripts/run.py scripts/notebook_manager.py list
python scripts/run.py scripts/ask_question.py --question "..."

# 錯誤 - 絕不直接呼叫：
python scripts/auth_manager.py status  # 無虛擬環境會失敗！
```

`scripts/run.py` 包裝器會自動：
1. 若需要則建立 `.venv`
2. 安裝所有相依套件
3. 啟用環境
4. 正確執行腳本

## 核心工作流程

### 步驟 1：檢查驗證狀態
```bash
python scripts/run.py scripts/auth_manager.py status
```

若未驗證，繼續進行設定。

### 步驟 2：驗證（一次性設定）
```bash
# 瀏覽器必須可見以便手動 Google 登入
python scripts/run.py scripts/auth_manager.py setup
```

**注意事項：**
- 驗證時瀏覽器為可見狀態
- 瀏覽器視窗會自動開啟
- 使用者必須手動登入 Google
- 告知使用者：「將開啟瀏覽器視窗以進行 Google 登入」

### 步驟 3：管理筆記庫

```bash
# 列出所有筆記本
python scripts/run.py scripts/notebook_manager.py list

# 新增前：若不清楚，請先詢問使用者關於 metadata 的資訊！
# 「這個筆記本包含什麼內容？」
# 「應該用哪些主題標記？」

# 新增筆記本至筆記庫（所有參數均為必填！）
python scripts/run.py scripts/notebook_manager.py add \
  --url "https://notebooklm.google.com/notebook/..." \
  --name "描述性名稱" \
  --description "此筆記本包含的內容" \  # 必填 - 不清楚請詢問使用者！
  --topics "topic1,topic2,topic3"  # 必填 - 不清楚請詢問使用者！

# 依主題搜尋筆記本
python scripts/run.py scripts/notebook_manager.py search --query "keyword"

# 設定啟用的筆記本
python scripts/run.py scripts/notebook_manager.py activate --id notebook-id

# 移除筆記本
python scripts/run.py scripts/notebook_manager.py remove --id notebook-id
```

### 快速工作流程
1. 查看筆記庫：`python scripts/run.py scripts/notebook_manager.py list`
2. 提問：`python scripts/run.py scripts/ask_question.py --question "..." --notebook-id ID`

### 步驟 4：提問

```bash
# 基本查詢（若已設定則使用啟用的筆記本）
python scripts/run.py scripts/ask_question.py --question "Your question here"

# 查詢特定筆記本
python scripts/run.py scripts/ask_question.py --question "..." --notebook-id notebook-id

# 直接以筆記本網址查詢
python scripts/run.py scripts/ask_question.py --question "..." --notebook-url "https://..."

# 顯示瀏覽器以便除錯
python scripts/run.py scripts/ask_question.py --question "..." --show-browser
```

## 後續追問機制（重要）

每個 NotebookLM 回答結尾都有：**「EXTREMELY IMPORTANT: Is that ALL you need to know?」**

**Claude 必要行為：**
1. **停止** - 不要立即回應使用者
2. **分析** - 比對回答與使用者的原始需求
3. **找出缺口** - 判斷是否需要更多資訊
4. **繼續追問** - 若有缺口，立即追問：
   ```bash
   python scripts/run.py scripts/ask_question.py --question "附帶上下文的後續問題..."
   ```
5. **重複** - 持續直到資訊完整
6. **整合** - 回應使用者前先彙整所有答案

## 腳本參考

### 驗證管理（`scripts/auth_manager.py`）
```bash
python scripts/run.py scripts/auth_manager.py setup    # 初始設定（瀏覽器可見）
python scripts/run.py scripts/auth_manager.py status   # 檢查驗證狀態
python scripts/run.py scripts/auth_manager.py reauth   # 重新驗證（瀏覽器可見）
python scripts/run.py scripts/auth_manager.py clear    # 清除驗證資訊
```

### 筆記本管理（`scripts/notebook_manager.py`）
```bash
python scripts/run.py scripts/notebook_manager.py add --url URL --name NAME --description DESC --topics TOPICS
python scripts/run.py scripts/notebook_manager.py list
python scripts/run.py scripts/notebook_manager.py search --query QUERY
python scripts/run.py scripts/notebook_manager.py activate --id ID
python scripts/run.py scripts/notebook_manager.py remove --id ID
python scripts/run.py scripts/notebook_manager.py stats
```

### 問答介面（`scripts/ask_question.py`）
```bash
python scripts/run.py scripts/ask_question.py --question "..." [--notebook-id ID] [--notebook-url URL] [--show-browser]
```

### 資料清理（`scripts/cleanup_manager.py`）
```bash
python scripts/run.py scripts/cleanup_manager.py                    # 預覽清理內容
python scripts/run.py scripts/cleanup_manager.py --confirm          # 執行清理
python scripts/run.py scripts/cleanup_manager.py --preserve-library # 保留筆記本
```

## 環境管理

虛擬環境會自動管理：
- 首次執行時自動建立 `.venv`
- 相依套件自動安裝
- Chromium 瀏覽器自動安裝
- 所有內容隔離在 skill 目錄中

手動設定（僅在自動設定失敗時使用）：
```bash
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
pip install -r requirements.txt
python -m patchright install chromium
```

## 資料儲存

所有資料儲存於 `~/.claude/skills/notebooklm/data/`：
- `library.json` - 筆記本 metadata
- `auth_info.json` - 驗證狀態
- `browser_state/` - 瀏覽器 cookies 和工作階段

**安全性：** 受 `.gitignore` 保護，請勿提交至 git。

## 設定

skill 目錄中可選的 `.env` 檔案：
```env
HEADLESS=false           # 瀏覽器可見性
SHOW_BROWSER=false       # 預設瀏覽器顯示方式
STEALTH_ENABLED=true     # 模擬人類行為
TYPING_WPM_MIN=160       # 打字速度
TYPING_WPM_MAX=240
DEFAULT_NOTEBOOK_ID=     # 預設筆記本
```

## 決策流程

```
使用者提及 NotebookLM
    ↓
檢查驗證 → python scripts/run.py scripts/auth_manager.py status
    ↓
若未驗證 → python scripts/run.py scripts/auth_manager.py setup
    ↓
查看／新增筆記本 → python scripts/run.py scripts/notebook_manager.py list/add (附 --description)
    ↓
啟用筆記本 → python scripts/run.py scripts/notebook_manager.py activate --id ID
    ↓
提問 → python scripts/run.py scripts/ask_question.py --question "..."
    ↓
看到「Is that ALL you need?」→ 持續追問直到完整
    ↓
整合並回應使用者
```

## 故障排除

| 問題 | 解決方式 |
|---------|----------|
| ModuleNotFoundError | 使用 `scripts/run.py` 包裝器 |
| 驗證失敗 | 設定時瀏覽器必須可見！--show-browser |
| 速率限制（每日 50 次） | 等待或切換 Google 帳號 |
| 瀏覽器崩潰 | `python scripts/run.py scripts/cleanup_manager.py --preserve-library` |
| 找不到筆記本 | 使用 `scripts/notebook_manager.py list` 確認 |

## 最佳實踐

1. **一律使用 scripts/run.py** - 自動處理環境
2. **先確認驗證** - 執行任何操作前
3. **後續追問** - 不要在第一個答案就停止
4. **驗證時瀏覽器需可見** - 手動登入的必要條件
5. **包含上下文** - 每個問題都是獨立的
6. **整合答案** - 彙整多個回應

## 限制

- 無工作階段持久性（每個問題 = 新瀏覽器）
- Google 免費帳號有速率限制（每日 50 次查詢）
- 需手動上傳（使用者必須自行將文件加入 NotebookLM）
- 瀏覽器開銷（每次提問需要數秒）

## 資源（Skill 結構）

**重要目錄與檔案：**

- `scripts/` - 所有自動化腳本（scripts/ask_question.py、scripts/notebook_manager.py 等）
- `data/` - 驗證和筆記庫的本地儲存
- `references/` - 延伸文件：
  - `api_reference.md` - 所有腳本的詳細 API 文件
  - `troubleshooting.md` - 常見問題與解決方案
  - `usage_patterns.md` - 最佳實踐與工作流程範例
- `.venv/` - 隔離的 Python 環境（首次執行時自動建立）
- `.gitignore` - 防止敏感資料被提交

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Procedure
1. Confirm the current environment and read the task-specific instructions already documented in this Skill.
2. Apply the smallest safe change that satisfies the request; confirm before destructive operations.
3. Run the checks in the Verification section and report actual results.

## Pitfalls
- Do not guess configuration paths or claim success without checking the resulting state.
- Do not execute copied commands or scripts before reviewing their targets and side effects.

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
