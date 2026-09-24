---
name: self-evolving
description: 當 AI 犯錯、使用者糾正 AI、發現更好的方法，或想要檢視學習記錄時使用。⚠️ 跟 `self-improve`（重量級自主程式碼演化引擎，含 git worktree/tournament selection/benchmark 迴圈）是完全不同的兩個 skill，名字很像但這個只是輕量 YAML 筆記記錄系統，不執行任何程式碼修改。
---

# 自我演化 Skill

> 這是 opencode 導向的輕量「學習記錄」筆記系統（讀寫 YAML 條目），不會自動修改程式碼。若要找的是會自動改程式碼、跑 benchmark、git worktree 迭代的引擎，那是 `self-improve` skill，不是這個。名字沒改是因為儲存路徑與 `@self-evolving` 指令語法在 opencode 側是綁死的字面值，改這邊的 skill 名字會跟外部系統對不起來，所以用這則提醒取代改名。

## 概覽
將結構化知識（錯誤、優化、工作流程）儲存為 YAML 條目，以便在未來的 session 中擷取。學習記錄存在於外部檔案中，而非模型權重內。

## 使用時機
- 發生 AI 錯誤/失敗（編譯、執行時、指令）
- 使用者糾正（「不對」、「那是錯的」、「其實應該...」）
- AI 在實作了次佳方案後發現更好的做法
- 任務無法按照指定方式完成
- 使用者輸入：`@self-evolving review`、`@self-evolving suggest`、`@self-evolving stats`

## 知識類型

| 類型 | 目錄 | 說明 |
|------|------|------|
| error | errors/ | 錯誤、邏輯問題、不良模式 |
| optimization | optimizations/ | 發現的更佳做法 |
| workflow | workflows/ | 流程改進 |

## 條目格式

```yaml
type: error|optimization|workflow   # 必填
category: string           # 例如 typescript, git, python（必填）
severity: low|medium|high|critical  # 必填
tags: [tag1, tag2]        # 用於模糊擷取（必填）
pattern: string            # 觸發此條目的情境（必填）
trigger:                  # 觸發此條目的條件清單
  - string
solution: string           # 解決/優化方式（必填）
example:                   # 非必填但建議提供
  before: string
  after: string
context: string           # 額外背景資訊（非必填）
project: global|<name>     # 必填
source: auto|manual       # 必填
created_at: ISO8601       # 自動產生
updated_at: ISO8601       # 合併時自動更新
similar_to: [id]        # 相似條目的 ID（自動管理，通常為空 `[]`）
```

## 指令

| 指令 | 用途 |
|------|------|
| `@self-evolving remember <topic>` | 記錄新的學習內容 |
| `@self-evolving review [topic]` | 擷取相關學習記錄 |
| `@self-evolving suggest` | 取得優化建議 |
| `@self-evolving stats` | 顯示知識庫統計資料 |

## 儲存位置

- 全域：`~/.opencode/self-evolving/global-knowledge/`
- 專案：`[project]/.opencode-knowledge/`
- 專案範圍的條目在相同 pattern 下會覆蓋全域條目

## 去重複機制

- 相似度 = Jaccard(tags) × 0.6 + Levenshtein(pattern) × 0.4
- 相似度 > 0.8：與現有條目合併
- 相似度 < 0.3：建立新條目
- 0.3-0.8：建立新條目並連結 similar_to

## 擷取流程

1. 從目前脈絡中提取關鍵字
2. 先搜尋全域知識庫，再搜尋專案
3. 依相關性 + 新近度 + 嚴重程度排名
4. 回傳前 5 個最符合的結果

**注意（v1）：** 擷取由 AI 讀取這些 YAML 檔案並對其進行推理來執行。不需要獨立的搜尋工具——AI 利用其語言理解能力將目前脈絡與儲存條目進行比對。未來版本可能會加入相似度計算工具以實現更精確的比對。

---

## Conformance Addendum

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
