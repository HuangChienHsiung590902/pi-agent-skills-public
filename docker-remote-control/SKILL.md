---
name: docker-remote-control
description: Manage Docker across this Windows machine and the remote GPU host at 10.145.119.19 — especially the custom Python Flet GUI in D:\Codes\docker-remote-control, docker context switching, backup/restore, migration, disk-space cleanup (builder/image prune), pushing images to Docker Hub (incl. Docker Hub username lookup and switching new repos from Private/Locked to Public via connect-chrome), SSH access via plink when password-only auth is required, and legacy Portainer notes. Use when the user asks to control remote Docker, check/clean up Docker disk usage, upload/push images to Docker Hub, make a Docker Hub repo public, open/fix the Docker control GUI, tune GPU selection for containers, switch docker context, back up/restore Docker data, migrate a compose stack, or manage Portainer. NOT about a specific application's business logic containers — this is the cross-host plumbing layer underneath them.
---

# Docker 本機 ↔ 遠端多主機統一管理

## 架構總覽

目前兩台遠端 Docker 主機透過 SSH 管理：
- GPU 主機：`10.145.119.19`，hostname `gigabyte`，SSH `hch@10.145.119.19`
- Jetson Edge 主機：`10.145.119.12`，hostname `Jetson`，SSH `hch@10.145.119.12`
- 本機 Windows Docker Desktop：ZeroTier IP `10.145.119.100`

`.19` 是 RTX GPU／Spark／ComfyUI 的主要主機；`.12` 是 Jetson ARM64、JetPack 4.6.7、CUDA 10.2、約 4 GB RAM 的 Edge 主機。Docker Dashboard 已可用 `serverId` 切換兩台。

四層工具疊在這個連線基礎上：

1. **CLI 層**：`docker context` 讓本機 `docker` 指令直接操作遠端（SSH 通道）
2. **備份/還原層**：跨兩台主機的 compose 專案/獨立容器/孤兒 volume 備份還原
3. **自製 GUI 層（優先）**：Python Flet app 在 `D:\Codes\docker-remote-control`，用 `docker-py` 的 `ssh://hch@10.145.119.19` 直接控制遠端 Docker，不依賴本機 Docker CLI/Desktop
4. **Portainer 層（舊方案）**：Portainer 裝在遠端 10.145.119.19，使用者已表示不想用付費功能；除非明確要求，不要再優先推薦 Portainer

```
本機 docker CLI ──(SSH context "remote")──▶ 10.145.119.19
10.145.119.19 Portainer:9000 ◀──(agent :9001, 10.145.119.100)── 本機
```

## 1. CLI Context 切換 + 備份/還原（統一選單）

**執行**：`D:\BIN\docker-remote-menu.bat`（雙擊或在終端機跑，不要直接跑 `.ps1`——見下方執行原則坑）

選單：
```
-- Context 切換 --
1. 切換到遠端 (hch@10.145.119.19)
2. 切回本機 (desktop-linux)
3. 顯示目前 context
4. 查看容器 (docker ps，依目前 context)
-- 備份 / 還原 (本機 + 遠端) --
5. 備份 (Backup)
6. 還原 (Restore)
7. 列出備份 (List backups)
-- 搬遷 (複製 compose 專案到另一台主機) --
8. 搬遷 (本機 <-> 遠端)
0. 離開 (Exit)
```

- Context 定義存在 `docker context ls` 裡（`remote` 這個 context 已建立，指向 `ssh://hch@10.145.119.19`），腳本啟動時若不存在會自動建立
- 備份範圍判斷**不掃資料夾，直接問 Docker 引擎**：`docker compose ls -a` 抓所有 compose 專案，沒被任何 compose 標記的容器歸進「未納入任何 compose 專案的容器」，沒被任何容器掛載的 volume 歸進「孤兒 volume」——所以不管本機/遠端上有什麼新東西，都不會漏備份
- 備份存放：`D:\Docker BAK\backups\<專案名稱>\<timestamp>\`，每份都有 `manifest.json` 記錄實際內容，還原完全照它執行
- 遠端備份的實際搬運邏輯在 `D:\Docker BAK\remote_backup.py` / `remote_restore.py`（部署在遠端 `/home/hch/`），這支選單只負責選單 UI + SSH 觸發 + scp 搬回本機
- **選項 5/6/7 進入時會自動先 `docker context use desktop-linux`**：因為備份腳本判斷「本機」資源是用裸的 `docker` 指令，若使用者用選項 1 切到 remote 後忘記切回來，本機清單會誤判成遠端內容，所以強制校正，不用手動處理

腳本原始碼：`D:\BIN\docker-remote-menu.ps1`（合併自舊版 `D:\Docker BAK\backup-restore.ps1`，那份 `.bat/.ps1` 還留著當備用，功能一致）

### 選項 8：搬遷（複製一個 compose 專案到另一台主機）

**只支援 compose 專案**，standalone 容器缺乏原始 `docker run` 參數（port/env/network），manifest 沒記錄，無法安全重建，選了會直接告知不支援。

流程：① 呼叫既有的備份邏輯（選項 5 同一套函式）備份來源 → ② scp compose 目錄（含 build context）+ 備份出來的 image/volume tar 到目標主機 → ③ 目標主機 `docker load` image、把 volume tar 解壓還原進去 → ④ 用 `docker compose -f ... -p <同名 project>` 在目標主機啟動，顯式指定 `-p` 讓 volume 命名跟備份時記錄的對上。

語意是**複製不是搬移**：來源主機的容器備份完會照常重啟，不會被刪除或停止，跑完之後要不要處理來源（留著/停掉/移除）由使用者自己決定，不要自作主張清掉來源。

已用 `mikopbx` 本機→遠端實測驗證成功（2026-07-19）：image 正確載入、volume 資料（138MB storage + conf 設定）確實還原、遠端容器正常開機顯示 Web 介面資訊。過程中抓到並修掉「坑六」（見下方）。

## 2. Portainer（圖形化雙主機管理）

- 網址：**http://10.145.119.19:9000**（HTTP，避開自簽憑證問題；`https://10.145.119.19:9443` 也通但 CDP/Playwright 接管時會被憑證檔擋下，人工瀏覽器才建議走 https）
- 帳號：`admin` / 密碼 `<ECP_PASSWORD>###`
- 已設定兩個 Environment：
  - **`local`**（endpoint id `3`）：10.145.119.19 自己，直連 `/var/run/docker.sock`
  - **`local-docker-desktop`**（endpoint id `4`）：這台 Windows 機器，透過 Agent 連 `10.145.119.100:9001`
- 常用網址（`{id}` 換成 3 或 4）：
  - 容器列表：`http://10.145.119.19:9000/#!/{id}/docker/containers`
  - 某容器 logs：容器列表裡點 Logs 連結，或 `.../docker/containers/<container-id>/logs`

### 本機 agent 容器

```powershell
docker run -d -p 9001:9001 --name portainer_agent --restart=always `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v /var/lib/docker/volumes:/var/lib/docker/volumes `
  -v /:/host `
  portainer/agent:latest
```

`-v /:/host` 是後補的（第一版沒加），讓 Portainer 的主機檔案瀏覽等進階功能可用。重建這個容器不會影響 Portainer 那邊已建立的 environment 連線紀錄（IP:port 沒變就自動重連）。

## 已知坑

### 坑一：PowerShell 執行原則擋住 .ps1
系統預設停用指令碼執行，直接 `D:\BIN\docker-remote-menu.ps1` 會報 `UnauthorizedAccess`。永遠透過 `.bat` 包裝（`powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0docker-remote-menu.ps1"`）啟動，不要建議使用者改全域執行原則（那是持久性安全變更）。

### 坑二：Docker Desktop 應用程式本身**永遠**看不到遠端內容
不管 `docker context use remote` 切了沒有，Docker Desktop 的 GUI Dashboard 只會顯示本機 `desktop-linux` 引擎的容器——這是 Docker 官方軟體的已知限制（多個未解決 GitHub issue），沒有設定能繞過。使用者要圖形化看遠端內容，一律導去 **Portainer**，不要嘗試在 Docker Desktop app 裡找設定。

### 坑三：Portainer admin 初始化的 setup token 會逾時
`docker run` 剛啟動的 Portainer 第一次開 `/#!/init/admin` 需要 `docker logs portainer 2>&1 | grep setup_token` 撈 token 填入表單。**這個 token 預設約 5 分鐘就過期**（`redirect-reason: AdminInitTimeout`），如果中途去問使用者密碼之類的操作拖時間，送出時會失敗。解法：`docker restart portainer` 重置計時器、重新撈 token、立刻填表送出，不要拖。

### 坑四：`docker system prune -a --volumes -f` 不一定真的清 volume
清空遠端所有 Docker 資源時，`prune --volumes` 的輸出如果沒有出現 `Deleted Volumes:` 這一段，代表 volume 沒被清到（曾實測留下 5 個孤兒 volume）。要徹底清空需要額外補一行：
```bash
docker volume rm $(docker volume ls -q)
```

### 坑五：Portainer Agent 新增環境時預設選項是 Edge Agent，不是傳統 Agent
精靈頁面預設勾 "Edge Agent Standard"（Recommended），但如果已經用 `docker run portainer/agent` 部署的是傳統 Agent，必須手動展開「More options」選 **Agent**（不是 API、不是 Socket），Environment URL 填 `<ip>:9001`，才會用剛部署的那個容器連線,不需要重新產生指令碼再部署一次 Edge Agent。

### 坑六：PowerShell 呼叫 `ssh` 時，巢狀雙引號會被錯誤轉譯（2026-07-19 實測踩到）
在 PowerShell 裡組一個要送給 `ssh` 執行的遠端指令字串，如果裡面又包一層 `sh -c "..."` 且用 backtick 跳脫（例如 `` ssh $h "docker run ... alpine sh -c `"cd /data && tar xzf ...`"" ``），Windows 上這個巢狀雙引號會被 PowerShell 轉成 argv 時解析錯誤——**指令會「看起來」正常執行、docker 也會回應成功，但實際上遠端收到的指令跟你以為的不一樣**，例如 `tar` 會回報 `Cannot open: No such file or directory`，即使檔案明明就在該路徑上（用 Bash 直接下同一條指令反而完全正常，因為 Bash 的引號展開規則不同，這是 PowerShell 特有的坑）。

**判斷方法**：同一條邏輯指令，改用 Bash 工具（而不是 PowerShell）透過 ssh 執行一次，如果 Bash 能成功而 PowerShell 失敗，就是這個問題，不是資料或路徑本身有錯。

**修法（優先順序）**：
1. 能不用 `sh -c` 就不要用——例如 tar 解壓縮不需要 `cd && tar xzf`，直接用 `tar xzf archive.tar.gz -C /target/dir` 的 `-C` 參數換目錄，完全不需要 shell，也就沒有巢狀引號問題（`docker-remote-menu.ps1` 的備份/還原/搬遷函式都已經統一改用這個寫法）。
2. 真的需要 shell 語法（例如 `rm -rf` 配合 glob、`&&` 串多條指令）沒辦法用參數繞開時，把內層的雙引號改成單引號（`sh -c 'rm -rf /data/* ...; true'`）——PowerShell 的雙引號字串裡，單引號是字面字元不需要跳脫，不會有這個轉譯問題。
3. 避免 backtick 跳脫的巢狀雙引號寫法（`` `"..."` ``），這是這個坑的根源寫法，日後新增任何「PowerShell → ssh → 遠端 shell」的指令都要避開。

### 坑七：Git Bash (MSYS) 會把 `docker run -v` 參數裡容器端的路徑也當成 Unix 路徑轉換（2026-07-27 實測踩到）
在 Bash 工具（Git Bash/MSYS）裡執行 `docker run --rm -v <vol>:/data -v "$STAGE:/backup" alpine tar czf /backup/xxx.tar.gz -C /data .`，MSYS 的自動路徑轉換會把 `-v` 參數字串裡「看起來像絕對 Unix 路徑」的部分（連容器端的 `/backup`、`/data`）都當成本機路徑轉換，導致 tar 收到的目的地變成類似 `C:/Program Files/Git/backup/xxx.tar.gz` 這種亂碼路徑，報 `No such file or directory`，即使 volume 名稱和邏輯完全正確。

**判斷方法**：錯誤訊息裡出現 `C:/Program Files/Git/...` 這種明顯不該存在的路徑片段，就是這個坑，不是 volume/路徑本身寫錯。

**修法**：在該次 `docker run` 呼叫前加 `export MSYS_NO_PATHCONV=1`（或該行前綴 `MSYS_NO_PATHCONV=1 docker run ...`），關閉 Git Bash 的自動路徑轉換，`-v` 參數的容器端路徑就會照字面傳給 Docker。本機端路徑（如 `$STAGE`）在 MSYS 下本來就要用 `/c/Users/...` 形式，不受這個開關影響。

### 坑八：搬遷 compose 專案到 Linux 遠端時，Windows 專屬的 bind mount 路徑會讓 `docker compose up` 整個失敗（2026-07-27 實測踩到）
本機（Windows）的 `docker-compose.yml` 如果有 `C:/Users/...:/container/path` 這種 Windows 絕對路徑 bind mount（例如把某個 Windows 資料夾唯讀掛進容器），照搬到 Linux 遠端主機後，`docker compose -p <project> up -d` 會直接報 `invalid volume specification: 'C:/Users/...:/container/path'` 並讓同一個 compose 檔裡其他 service 的容器也連帶建立失敗（compose 是整批處理，一個 service 掛了會擋住後面的）。

**判斷方法**：`docker compose up` 報錯訊息裡出現 `C:/...` 或 `C:\...` 這種 Windows 磁碟機代號路徑，就是這個坑。

**修法**：搬遷前檢查來源 `docker-compose.yml` 有沒有 `C:/` 或 `C:\` 開頭的 bind mount，搬到遠端前用 `sed -i '/C:\/Users\/.../d'` 這類方式把該行拿掉（遠端 Linux 主機本來就沒有對應目錄，拿掉不影響其他 service，只是掛了那個 mount 的 service 會讀不到那份 Windows 端內容）。若該掛載內容其實是遠端也需要的資源，考慮改成先 `scp` 複製一份到遠端再改路徑指向遠端本地目錄，而不是留著 Windows 路徑字面值。

## 備份範圍的固有限制

備份只涵蓋 image + named volume，**bind mount 不會被備份**（腳本執行時會列出偵測到的 bind mount 提醒使用者自行處理）。純環境變數/密鑰檔（例如某專案的 `.env`，可能含 API key、密碼）也不在容器/volume 內，一律不含在 docker 備份範圍內，需要的話要另外手動備份該檔案本身。

## 3. 清理磁碟空間（build cache / dangling images / 閒置 images）

**適用情境**：使用者說「Docker 佔了太多空間」「幫我清一下 Docker」時，在遠端主機（10.145.119.19）上依風險由低到高分階段清理，不要一開始就 `docker system prune -a --volumes -f`（那是坑四提過的全清，太粗暴）。

連線方式：見下方「SSH 連線注意事項」（密碼登入需用 `plink`，純 `ssh`/`sshpass` 在這個環境會失敗）。

分階段清理（由安全到需確認）：

1. **`docker builder prune -a -f`** — 清空 build cache。100% 安全，純建構暫存，不影響任何容器/映像。曾實測釋放 54GB。
2. **`docker image prune -f`** — 只清 dangling images（`<none>` tag 的殘留層）。100% 安全。
3. **`docker image prune -a -f`** — 清除「完全沒有被任何容器（含已停止）引用」的映像。**這一步不會動到任何已停止容器仍引用的映像**，所以不會誤刪之後可能要重啟的服務用的映像，相對安全，但動手前最好先跑一次 `docker ps -a --format '{{.Image}}'` 讓使用者看一下目前有哪些映像正被容器引用、哪些沒有。
4. **不要自動執行**、需要額外確認的部分：已停止的容器本身（`docker container prune`）、named volume（`docker volume prune`）——這些可能是使用者還要用的服務或資料，清理前要明確列出清單問過使用者（呼應下方「操作這台機器上 Docker 時的安全原則」）。

清理前後用 `docker system df` 和 `df -h /` 對照，跟使用者報告實際釋放了多少空間。

## 4. 推送映像到 Docker Hub（`docker push`）

**適用情境**：使用者要把遠端主機上自建/修改過的映像備份或分享到 Docker Hub。

### 步驟

1. **登入**：`echo '<password>' | docker login -u <email或帳號> --password-stdin`。用 email 登入沒問題，但 **push 的 repo 命名空間一定要用 Docker Hub 的 username，不是 email，也不一定等於網頁右上角顯示的 Display Name**（曾實測使用者提供的「HuangChienHsiung590902」其實是 Display Name，真正 username 是 `hch590902`）。

2. **確認真正的 username**（登入後 `~/.docker/config.json` 只存 auth token，看不出 username）：
   ```bash
   curl -s -H 'Content-Type: application/json' -X POST \
     -d '{"username": "<email>", "password": "<password>"}' \
     https://hub.docker.com/v2/users/login/
   ```
   回傳 JWT 的 payload 裡 `username` 欄位（base64 decode 中間那段，或直接用 `jq`/線上工具解）就是真正拿來 tag 的 namespace。**push 失敗訊息是 `push access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed`，這個症狀就是 namespace 用錯，不是密碼或權限問題**，先照這一步查真正 username 再重 tag。

3. **Tag**：`docker tag <local_image> <username>/<repo>:<tag>`，每個要上傳的映像都要重新 tag 一次。

4. **背景批次 push（大量/大型映像必用）**：映像動輒數十 GB，單次 `docker push` 可能要跑 10-20 分鐘以上，SSH 前景執行容易因逾時或連線中斷而失敗。做法：在遠端寫一支 shell script，用 `nohup setsid ... &` + `disown` 丟到背景執行，依序（建議由小到大）push 每個映像，每個映像的輸出各自寫入獨立 log 檔，最後寫一個 `_finished.flag` 標記全部完成：
   ```bash
   cat > /home/hch/push_all.sh << 'EOF'
   #!/bin/bash
   LOGDIR=/home/hch/push_logs
   IMAGES=(small1:latest small2:latest big1:latest)
   for img in "${IMAGES[@]}"; do
     name=$(echo $img | tr '/:' '__')
     echo "=== START $img at $(date) ===" >> $LOGDIR/${name}.log
     docker push <username>/$img >> $LOGDIR/${name}.log 2>&1
     echo "=== END $img at $(date) exit=$? ===" >> $LOGDIR/${name}.log
   done
   echo ALL_DONE > $LOGDIR/_finished.flag
   EOF
   chmod +x /home/hch/push_all.sh
   nohup setsid /home/hch/push_all.sh > /home/hch/push_logs/master.log 2>&1 < /dev/null &
   disown
   ```
   之後每隔 1-5 分鐘（依映像大小拉長間隔）用 `tail -N` 各 log 檔確認進度，用 `grep '=== (START|END)' *.log` 快速總覽全部映像的完成狀態，用 `ps aux | grep 'docker push'` 確認目前是哪一個還在跑。全部結束的判斷標準是每個 log 都出現 `exit=0` 的 `END` 行，且 `_finished.flag` 存在。

   有些大型官方基底映像（如 `ollama/ollama`、`vllm/vllm-openai`）若底層 layer 跟 Docker Hub 上已存在的公開映像完全相同，push 時會顯示 `Mounted from <原始repo>` 直接引用既有層，幾秒內就完成，不代表真的重新上傳了那麼多資料——這是正常現象，不是卡住或失敗。

### 免費帳號的私有 repo 數量限制（重要坑）

Docker Hub 免費方案通常只允許 **1 個私有（Private）repo**。用 `docker push` 建立的新 repo 預設是 **Private**，如果一次 push 多個新映像，超過額度的 repo 會被 Docker Hub **鎖定（Locked）**——映像資料還在、push 也顯示成功，但網頁上會顯示：

> "The number of private repositories in your account exceeds the limit of your current subscription. Upgrade to a paid tier to unlock this repository."

**這不是 push 失敗**，是 push 成功後才在 Docker Hub 帳號層級被鎖定，push 端不會有任何錯誤訊息，必須用瀏覽器登入 Docker Hub 網頁才能發現。

### 批次改成 Public 解鎖（需要 connect-chrome）

用 `connect-chrome` skill 接管已開啟的 Chrome，逐一到每個 repo 的 Settings 頁面操作：

1. 導覽到 `https://hub.docker.com/repository/docker/<username>/<repo>/settings`
2. 找到 Visibility settings 區塊的 **"Make public"** 按鈕並點擊
3. 彈出確認 dialog，**必須在 textbox 打字輸入 repo 名稱本身**（例如 repo 是 `comfyui` 就要打 `comfyui`）才能啟用第二個 "Make public" 確認按鈕
4. 點擊 dialog 裡的確認按鈕完成
5. 用 `browser_find` 搜尋 "This repository is public" 或觀察 "Using N of 1 private repositories" 的 N 遞減來確認成功

**坑**：Playwright 快照裡同一頁常常會出現兩個名字都叫 "Make public" 的按鈕（頁面上的按鈕 + dialog 裡的確認按鈕），務必用 `browser_find` 重新抓最新的 ref 再點擊，不要憑舊的 snapshot ref 亂點，容易點錯（曾誤點到 Cancel）。

最後回到 `https://hub.docker.com/repositories/<username>` 列表頁，用 `browser_snapshot` 一次性確認所有目標 repo 的 Visibility 欄位都已變成 `Public` 且不再有 `Locked` 字樣。

## SSH 連線注意事項（密碼登入）

這台主機的 SSH 若只有密碼、沒有已授權的金鑰，**純 `ssh` 指令（含 Bash 工具的 `sshpass`）在此環境常會失敗**（`Permission denied (publickey,password)`，即使密碼正確），原因是 Bash 工具的環境下 `ssh`/`sshpass` 無法正確餵密碼給互動式 prompt（`read_passphrase: can't open /dev/tty`），連 `-tt` 強制分配 pty 也一樣。

**穩定作法：改用 `plink`（PuTTY 套件）**，本機路徑通常在 `/c/Program Files (x86)/PuTTY/plink`：
```bash
# 第一次連線需先接受 host key（batch 模式下 plink 預設會拒絕未知 host key，要顯式帶入 fingerprint）：
plink -ssh -batch -pw '<password>' hch@10.145.119.19 "echo test"
# 若報 host key 未快取，錯誤訊息裡會附上 fingerprint，複製後帶入：
plink -ssh -batch -pw '<password>' -hostkey "ssh-ed25519 255 SHA256:xxxx" hch@10.145.119.19 "command"
```
之後同一個 session 內所有遠端指令都用這個 `plink -hostkey ...` 前綴執行即可穩定運作。

## 操作這台機器上 Docker 時的安全原則

- 任何「刪光」「全清空」等不可逆操作（`docker system prune -a --volumes`、`docker rm -f $(docker ps -aq)` 之類），**動手前一定要跟使用者確認範圍**（容器？image？volume？全部？），並主動提醒可以先用選單的備份功能（選項 5）備份一份回本機再刪——即使使用者最後選擇不備份直接刪，也要先問過。
- 遠端主機是共用的多服務環境（曾同時跑 GitLab、aipower、mikopbx、ASR、one-api 等），清空前列出目前跑什麼給使用者看過，不要默默執行。

---

## Conformance Addendum

## When to Use
Manage Docker across this Windows machine and remote hosts `10.145.119.19` and `10.145.119.12` — especially the custom GUI, multi-host Docker selection, Docker context switching, backup/restore, migration, disk-space cleanup, SSH access, GPU selection, service compatibility, Spark/ComfyUI on `.19`, and Jetson Docker/GPU behavior on `.12`. The Jetson host is not a drop-in RTX server and Spark-X2.5-4B should stay on `.19` or Windows Intel Arc.

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
