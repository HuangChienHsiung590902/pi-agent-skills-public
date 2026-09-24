---
name: skills-github-publish
description: >
  把 D:\OB\skills 技能庫發布到 GitHub：同步正式 private repo
  （HuangChienHsiung590902/pi-agent-skills），並為 CC Switch 等「匿名下載」的
  skill manager 建立/更新 public mirror repo
  （HuangChienHsiung590902/pi-agent-skills-public），發布前自動消毒憑證。
  當使用者說「把 skills 上傳/發布/同步到 GitHub」、「更新 public mirror」、
  「CC Switch 識別到 0 個技能」、「技能儲存庫抓不到/下載失敗」、或想把技能庫
  放進公開 repo 又怕洩憑證時使用。不用於：匯入單一外部 skill 進本機庫
  （用 pi-install-skill-import）、或庫內結構稽核整理
  （用 skills-library-conformance-maintenance）。
---

# Skills GitHub Publish & Public Mirror

## When to Use

- 使用者要求把 `D:\OB\skills` 上傳、發布、同步到 GitHub。
- 要建立或更新 public mirror repo，供 CC Switch 等 skill manager 匿名抓取。
- CC Switch「管理技能儲存庫」顯示「識別到 0 個技能」，或 log 出現
  `DOWNLOAD_FAILED / 404`。
- 任何「把技能庫內容放進公開 repo」的需求，需要先過憑證消毒。

不用於：

- 從 GitHub 匯入單一外部 skill 到本機庫（用 `pi-install-skill-import`）。
- 庫內結構稽核、章節修復、索引一致性整理（用
  `skills-library-conformance-maintenance`）。

## Inputs and Outputs

### Inputs

- 來源庫路徑，預設 `D:\OB\skills`。
- 正式 private repo：`HuangChienHsiung590902/pi-agent-skills`（branch `main`）。
- Mirror repo：`HuangChienHsiung590902/pi-agent-skills-public`（branch `main`），
  本機工作目錄預設 `D:\Github\pi-agent-skills-public`。
- 是否 commit／push、是否 `--force` 接管既有 mirror 目錄。

### Outputs

- Private repo 同步結果（commit hash、ahead/behind）。
- Mirror 目錄重建結果與 push 結果。
- 匿名驗證報告：HTTP 狀態、SKILL.md 數量、有無 root SKILL.md、殘留憑證清單。
- 實際消毒的檔案數與 placeholder 對照（只回報數量與檔名，不回報真值）。

## Procedure

### Phase 0：狀態確認

```powershell
git -c safe.directory=D:/OB/skills -C D:/OB/skills status --short --branch
git -c safe.directory=D:/OB/skills -C D:/OB/skills rev-list --left-right --count origin/main...HEAD
```

`D:/OB/skills` 的 owner SID 與目前使用者不同，所有 git 指令都要帶
`-c safe.directory=D:/OB/skills`，否則直接 fatal。

### Phase 1：同步正式 private repo

1. 先掃高可信憑證 pattern（用 python re 或 `rg --pcre2`，rg 預設不支援
   lookaround）：`gh[pousr]_…`、`github_pat_…`、邊界後的 `sk-…`、`AIza…`、
   `xox[baprs]-…`、`-----BEGIN … PRIVATE KEY-----`、
   `line.channelSecret/channelAccessToken=…`、`Bearer <長 token>`。
2. 發現真值憑證時：先備份到庫外（例如 `D:/OB/skill-secrets-backup-<ts>/`），
   再從追蹤移除（`git rm --cached`）＋加進 `.gitignore`，或把文件內真值改成
   環境變數/placeholder。**本流程不重寫歷史**；已進歷史的憑證一律視為已洩，
   提醒使用者輪換。
3. `git add` 目標範圍 → commit → `fetch origin main` → `rebase origin/main`
   （遠端可能有其他機器推的 commit，直接 push 會被拒）→ `push origin main`。

### Phase 2：重建 public mirror

```powershell
python D:\OB\skills\skills-github-publish\scripts\build_public_mirror.py `
  --source D:/OB/skills --dest D:/Github/pi-agent-skills-public `
  --commit --push
```

腳本行為（不要手動複製檔案取代它）：

- 只複製每個頂層 skill 目錄的 `SKILL.md`，加上 `SKILLS_INDEX.md` 與
  `_consolidation/` 六個索引檔；**不複製** `scripts/`、`references/`、
  `assets/`、內部 Java/SQL、部署設定。
- **不複製**來源庫根目錄的 `SKILL.md`（見 Pitfalls）。
- 對所有複製出的文字檔跑消毒：已知洩漏真值清單＋高可信 pattern →
  placeholder（`<DB_PASSWORD>`、`<ECP_PASSWORD>`、`<SSH_PASSWORD>`、
  `<BEARER_TOKEN>` 等）。
- 殘留檢查：任何已知真值或高可信 pattern 仍存在就 exit 2，不會 commit。
- 寫入 `README.md`、`.gitignore`、`.mirror-marker`（防誤刪保護）。
- dest 目錄非空且無 `.mirror-marker` 時拒絕執行，需 `--force` 接管一次。

### Phase 3：匿名驗證

```powershell
python D:\OB\skills\skills-github-publish\scripts\verify_public_mirror.py
```

腳本以**完全不帶憑證**的方式下載
`https://github.com/<repo>/archive/refs/heads/<branch>.zip`（與 CC Switch 相同
路徑），檢查：HTTP 200、每個 skill 資料夾各有一個 `SKILL.md`、沒有 root
`SKILL.md`、壓縮包內無殘留憑證。exit 0 才算通過。

### Phase 4：用戶端設定（CC Switch）

- CC Switch 只能吃 public repo：刪除指向 private repo 的條目，改加
  `HuangChienHsiung590902/pi-agent-skills-public`（branch `main`）。
- 診斷依據：`C:\Users\<user>\.cc-switch\logs\cc-switch.log` 裡的
  `获取仓库 … 技能失败: {"code":"DOWNLOAD_FAILED","context":{"status":"404"}}`
  ＝匿名下載被拒＝repo 是 private。CC Switch 原始碼
  （`src-tauri/src/services/skill.rs`）沒有任何 GitHub token 設定入口。

## Rules and Limitations

- Mirror 只允許 `SKILL.md` 與索引檔；任何可執行腳本、內部原始碼、設定檔進
  public repo 都視為事故。
- Mirror 與正式庫內容**不同步**（mirror 是消毒後的子集）；每次正式庫更新後
  必須重跑 Phase 2，不能只 push 正式庫。
- 真值憑證不得寫進本 skill 的 `SKILL.md`；腳本內的已知真值清單僅用於
  「替換掉它們」，且腳本只存在於 private repo 與本機。
- 破壞性操作（清空 mirror 目錄內容）只由腳本在 `.mirror-marker` 或 `--force`
  保護下執行，且保留 `.git`。
- 不重寫 private repo 歷史；憑證一旦進歷史就提醒輪換，不用 filter-repo。

## Pitfalls

- CC Switch 用匿名 HTTP 抓 `archive/refs/heads/<branch>.zip`，private repo 永遠
  404，重試幾次都一樣；不要懷疑 repo URL 或 branch 填錯。
- GitHub API 未帶 token 回 404 不代表 repo 不存在，可能只是 private。
- Mirror 裡若放根目錄 `SKILL.md`，CC Switch 的 `scan_dir_recursive` 會在第一個
  含 `SKILL.md` 的目錄停下，整庫變成「1 個技能」。
- `rg` 預設不支援 lookaround；`sk-` 這類 pattern 會誤中 `ask-user` 內 substring，
  邊界匹配改用 `rg --pcre2` 或 python re。
- 其他 process 可能隨時在 `D:\OB\skills` 新增頂層 skill 目錄；範圍以索引重建
  結果為準，不要假設目錄清單固定。
- 內網 IP（`10.145.119.x`、`192.168.x`）與主機名是資訊不是憑證，但公開可見；
  需要更乾淨時另跑一輪 IP 遮罩，不要混進憑證消毒清單。
- `D:/OB/skills` 的 git 指令漏掉 `-c safe.directory` 會直接 fatal，容易被誤判成
  repo 壞掉。

## Verification

1. `verify_public_mirror.py` exit 0：匿名 zip HTTP 200、SKILL.md 數量等於來源庫
   頂層 skill 目錄數、root SKILL.md 為 0、殘留憑證為 0。
2. Private repo 與 mirror 的 `git status` 皆 clean，`ahead/behind` 為 `0 0`。
3. CC Switch 刪除舊條目、新增 mirror 後顯示正確技能數（非 0）。
4. 回報：執行時間、技能數量、消毒檔案數、是否遇到驗證或 403/429。
