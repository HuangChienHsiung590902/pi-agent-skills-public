---
name: docker-mcp-toolkit
description: 管理 Docker Desktop 內建的 MCP Toolkit / Gateway（Claude Code 裡登記為 MCP_DOCKER）——docker mcp profile/secret CLI、profile 驗證失敗（缺 config/secret）的排查、把新 server 從 catalog 加進 profile。當使用者提到「MCP_DOCKER」「docker mcp」「MCP Toolkit」「Docker 的 MCP gateway 連不上/驗證失敗」時使用。
---

# Docker MCP Toolkit（MCP_DOCKER）

Docker Desktop 4.83+ 內建的功能：透過一個 gateway 行程 (`docker mcp gateway run --profile <id>`)
同時代理多個 MCP server（各自跑在獨立 container 裡），對 Claude Code 只曝露成
**一個** MCP server 連線（`MCP_DOCKER`），底下的工具（playwright_*、n8n_*...）
都是這個 gateway 動態列出來的。

## 目前現況（2026-07-26 整理）

- Claude Code 註冊：`claude mcp add MCP_DOCKER -s user -- docker mcp gateway run --profile hch`
- 使用的 profile：**`hch`**（原本還有一個 `cc` profile，因為裡面的 `line` server
  缺 `user_id` 設定一直驗證失敗，使用者選擇直接砍掉這個 profile，改用 `hch`）
- `hch` profile 目前啟用的 server：**playwright**、**n8n**、**fetch**、**obsidian**（共 59 個工具）
- 兩個需要金鑰的 server 都已設定好 secret：
  - `n8n.api_key`（n8n public API 的 JWT token）
  - `obsidian.api_key`（Obsidian Local REST API plugin 的 token，host 固定用
    `host.docker.internal`——**不要**改成 `127.0.0.1`，那是從 container 裡看
    「自己」，看不到跑在 Windows host 上的 Obsidian）
- 原本 `hch` profile 裡還有一個 `kubernetes` server 缺 `config_path`，因為使用者
  沒在用 k8s，已直接從 profile 移除（不是全域關掉，只是不在這個 profile 裡）

## 常用指令

```bash
docker mcp profile list                        # 列出所有 profile
docker mcp profile show <id>                   # 印出整份 profile（含每個 server 的完整 config schema，YAML 很長）
docker mcp profile server remove <id> <name>   # 把某個 server 從 profile 移除
docker mcp secret set <secret.name>            # 從 stdin 讀值設定 secret（存在本機 OS Keychain/docker-pass，不是明文檔案）
docker mcp secret ls                           # 只列出 secret 名稱，不會印出值
docker mcp profile remove <id>                 # 整個刪掉一個 profile
```

## 排查「Cannot activate profile 'X'. Validation failed for N server(s)」

```bash
timeout 15 docker mcp gateway run --profile <id> 2>&1
```

錯誤訊息會直接點名哪個 server 缺哪個 config 欄位，例如：
```
Server 'line':
  Missing/invalid config: user_id (missing)
```

兩種修法：
1. **不需要這個 server** → `docker mcp profile server remove <id> <name>` 直接移除
2. **需要，只是沒設定** → 先 `docker mcp profile show <id>` 找到該 server 的
   `config`/`secrets` 區塊，看欄位名稱對應到哪個 secret name，再用
   `docker mcp secret set <name>` 補上（如果是 secret 類型）；如果是純
   config（非 secret，像 `kubernetes` 的 `config_path`），目前這版 CLI 沒有
   直接的 `docker mcp profile config` 子指令可以在 script 裡設，需要另外查
   `docker mcp profile config --help` 的最新用法。

## 已知坑：第一次連線會逾時（不是真的壞掉）

`claude mcp list` 第一次顯示 `MCP_DOCKER ... ✘ Failed to connect — connection
timed out after 30000ms` 時，不代表設定錯誤——gateway 第一次啟動要拉 4 個
server 各自的 Docker 映像檔，可能超過 Claude Code 的 30 秒連線逾時。手動跑一次
`timeout 60 docker mcp gateway run --profile <id>` 讓映像檔快取好之後，
`claude mcp list` 通常就能在幾秒內顯示 `✔ Connected`。不要看到一次逾時就急著
改設定或刪掉 profile。

## 通用 MCP 除錯知識：行程死掉 ≠ 安裝壞掉

這條不限 Docker MCP，任何 MCP server 都適用（曾在排查 `codegraph` 與
其他使用本機檔案鎖的 server 時遇到）：

- `claude mcp list` 每次執行都是**獨立健康檢查**——它會重新嘗試連線，跟目前
  這個 Claude Code session 實際在用的連線是兩回事。如果某個 server 的
  MCP 子行程在 session 中途死掉/斷線，這個 session 裡對應的工具
  （`mcp__<name>__*`）就會消失，但 `claude mcp list` 獨立測試可能顯示
  「✔ Connected」（因為它自己重新啟動了一個新的測試連線）——**這不代表
  目前 session 裡的工具已經恢復**，需要重開 Claude Code session 才會重新
  建立一份會話內的連線。
- 反過來，如果某個 server 用的是「單一行程獨占某資源」的設計（例如
  PGLite 類型的本機資料庫檔案鎖），`claude mcp list` 的健康檢查行程會因為資源被目前
  session 的真正連線占用而顯示「✘ Failed to connect」——這種情況**才是
  誤報**，先確認：目前 session 是不是已經真的有這個工具可用（看
  ToolSearch 或直接試打一個該 server 的工具），如果有，這個「連不上」就
  可以忽略。
- 判斷原則：先看「這個 session 裡的工具還在不在」，再看 `claude mcp list`
  的結果，不要單看後者就下結論。

---

## Conformance Addendum

## When to Use
管理 Docker Desktop 內建的 MCP Toolkit / Gateway（Claude Code 裡登記為 MCP_DOCKER）——docker mcp profile/secret CLI、profile 驗證失敗（缺 config/secret）的排查、把新 server 從 catalog 加進 profile。當使用者提到「MCP_DOCKER」「docker mcp」「MCP Toolkit」「Docker 的 MCP gateway 連不上/驗證失敗」時使用。

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
