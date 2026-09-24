---
name: wsl-docker-engine
description: 在 Windows 上管理「不安裝 Docker Desktop、直接使用 WSL 2 Ubuntu 內 Docker Engine」的本機環境。當使用者提到 WSL Docker、純 Docker Engine、Ubuntu 內安裝 Docker、Docker daemon、WSL 中的容器、MySQL Docker 容器、VS Code 連不到 WSL Docker、或看到「Failed to connect. Is WSLC installed?」時，務必使用本 Skill。涵蓋安裝、systemd、docker 群組、容器與 volume、MySQL、VS Code WSL 整合及診斷；不要把它和 Docker Desktop、Podman 或遠端 Docker 混用。
compatibility: Windows 11/10 with WSL 2 and Ubuntu; commands must be run in the indicated Windows PowerShell or Ubuntu WSL context.
---

# WSL 2 Ubuntu 純 Docker Engine

## When to Use

使用者要求以下任一事項時載入本 Skill：

- 安裝或檢查 WSL 2 Ubuntu 內的 Docker Engine。
- 明確要求「不要 Docker Desktop」或「純 WSL Docker」。
- 管理本機 WSL Docker 的 daemon、容器、映像、volume 或 Compose。
- 在 WSL Docker 裡部署 MySQL 或其他服務。
- VS Code Docker/Containers 面板無法連到 WSL Docker，尤其是 `Failed to connect. Is WSLC installed?`。

本 Skill 的目標架構是：

```text
Windows
└── WSL 2
    └── Ubuntu
        └── systemd
            └── Docker Engine / dockerd
                └── containers and volumes
```

不要安裝或建議安裝 Docker Desktop，除非使用者明確改變需求。

## 已驗證的本機基線

這台機器目前已實測具備：

- 發行版：`Ubuntu`
- WSL：版本 `2`
- PID 1：`systemd`
- Docker Engine：`29.8.1`
- `docker.service`：`active`
- Docker socket：`/var/run/docker.sock`
- 使用者：已加入 `docker` 群組
- MySQL 容器：`mysql-server`
- MySQL image：`mysql:8.4`
- MySQL volume：`mysql-data`
- MySQL 對外連接埠：`3306`
- VS Code 已安裝 WSL、Docker 與 Container Tools 擴充功能

版本與容器狀態可能會變動；每次執行前以現場檢查結果為準，不要盲信這段基線。

## Windows 與 Ubuntu 的界線

### Windows PowerShell

只在 PowerShell 執行 WSL 控制指令：

```powershell
wsl -l -v
wsl --status
wsl --shutdown
wsl -d Ubuntu
```

### Ubuntu WSL

以下 Linux 指令必須在 Ubuntu 終端機執行，而不是 `PS C:\Users\...>`：

```bash
sudo apt update
docker ps
systemctl status docker
```

如果使用 agent 執行跨環境命令，明確使用 `wsl.exe -d Ubuntu -- bash -lc '...'`，並在回報中說明實際執行環境。

## Inputs and Outputs

### Inputs

- Windows WSL 狀態與 Ubuntu 發行版名稱。
- Ubuntu 內的 systemd、Docker Engine、Docker context、socket、使用者群組與服務狀態。
- 容器、映像、volume、port、Compose 或 MySQL 的需求。
- VS Code 錯誤訊息與目前是否以 WSL Remote 開啟。

### Outputs

- 不安裝 Docker Desktop 的 WSL 2 Ubuntu Docker Engine 設定或修復結果。
- 容器、volume、port 與 VS Code WSL 整合的實際狀態。
- 可重現的驗證命令、資料保留說明與尚未驗證的限制。

## Procedure

### 1. 先檢查環境，不要重複安裝

在 PowerShell：

```powershell
wsl -l -v
wsl --status
```

在 Ubuntu：

```bash
ps -p 1 -o comm=
command -v docker || true
docker --version || true
systemctl is-active docker || true
docker info 2>&1 || true
```

判定條件：

- `VERSION` 應為 `2`。
- PID 1 應為 `systemd`，才能使用 `systemctl` 管理 Docker。
- `docker info` 必須能連到 daemon；只有 `docker --version` 成功不代表 daemon 正常。

### 2. 安裝 WSL 2 與 Ubuntu

只有在檢查確認 WSL/Ubuntu 不存在時，才請使用者以系統管理員 PowerShell 執行：

```powershell
wsl --install -d Ubuntu
```

這是 Windows 系統變更，通常需要重新開機並在第一次啟動 Ubuntu 時建立 Linux 使用者。安裝後確認：

```powershell
wsl -l -v
```

如果 Ubuntu 不是 WSL 2：

```powershell
wsl --set-version Ubuntu 2
```

不要把 Windows 的「Sudo」功能當成 Linux `sudo`，也不要在 PowerShell 執行 `sudo apt`。

### 3. 啟用 systemd

在 Ubuntu 編輯 `/etc/wsl.conf`：

```bash
sudo nano /etc/wsl.conf
```

加入或合併，不要重複建立 `[boot]`：

```ini
[boot]
systemd=true
```

回到 PowerShell 套用：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

驗證：

```bash
ps -p 1 -o comm=
```

預期是 `systemd`。

### 4. 使用 Docker 官方 Ubuntu APT 儲存庫

在 Ubuntu 執行：

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

建立來源時使用正確的 URL 與路徑：

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"${UBUNTU_CODENAME:-$VERSION_CODENAME}\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

重要：不要使用 `https://docker.com` 當 APT 來源，也不要使用錯誤的 `/etc/apt/sources.list.p/`。

安裝 Engine：

```bash
sudo apt-get update
sudo apt-get install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### 5. 啟動 daemon 與設定權限

```bash
sudo systemctl enable --now docker
systemctl is-active docker
sudo docker info
```

若要免 `sudo`：

```bash
sudo usermod -aG docker "$USER"
```

加入群組後必須重新登入 WSL；穩妥做法是在 PowerShell 執行：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

再驗證：

```bash
id -nG
docker info
docker run --rm hello-world
```

`docker` 群組具有近似 root 的權限。不可把它描述成沒有安全影響的單純便利設定。

### 6. 部署 MySQL 的安全基準做法

先檢查名稱、port 與 volume，避免覆蓋既有資料：

```bash
docker ps -a
docker volume ls
ss -ltn | grep ':3306 ' || true
```

標準的本機開發用 MySQL 方案：

```bash
docker volume create mysql-data
docker pull mysql:8.4
```

產生密碼時不要把密碼寫入 Skill、命令歷史或回覆：

```bash
umask 077
printf 'MYSQL_ROOT_PASSWORD=%s\n' "$(openssl rand -hex 16)" > ~/.mysql-server-credentials
```

啟動前讀取該檔案，然後建立容器：

```bash
. ~/.mysql-server-credentials
docker run -d \
  --name mysql-server \
  --restart unless-stopped \
  -e "MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD" \
  -p 3306:3306 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.4
```

檢查初始化完成：

```bash
docker ps --filter name=mysql-server
docker logs --tail 80 mysql-server
docker exec -e MYSQL_PWD="$MYSQL_ROOT_PASSWORD" \
  mysql-server mysqladmin ping -uroot --silent
docker exec -e MYSQL_PWD="$MYSQL_ROOT_PASSWORD" \
  mysql-server mysql -uroot -e 'SELECT VERSION(), @@port;'
```

資料放在 `mysql-data` volume；刪除容器不等於刪除資料，但 `docker volume rm mysql-data` 會破壞資料，執行前必須取得明確確認。

### 7. VS Code 連線 WSL Docker

推薦工作方式不是讓 Windows VS Code 猜測 WSLC，而是使用 WSL Remote：

1. Windows VS Code 安裝：
   - `ms-vscode-remote.remote-wsl`
   - `ms-azuretools.vscode-docker`
   - `ms-azuretools.vscode-containers`
2. `Ctrl+Shift+P` → `WSL: New Window` → 選 `Ubuntu`。
3. 確認左下角顯示 `WSL: Ubuntu`。
4. 在 VS Code 的 WSL 終端機執行：

```bash
docker ps
```

也可以在 Ubuntu 專案目錄執行：

```bash
code .
```

若 Container Tools 曾被設定成 WSLC，將 Windows VS Code 使用者設定中的：

```json
"containers.containerClient": "com.microsoft.visualstudio.containers.wslc"
```

改為：

```json
"containers.containerClient": "com.microsoft.visualstudio.containers.docker"
```

修改後完整重啟 VS Code。VS Code extension 的設定名稱與版本可能改變，若設定不再存在，優先採用 `WSL: New Window`，不要暴露 Docker TCP API。

不要為了讓 VS Code 連線而開啟未加密的：

```text
tcp://0.0.0.0:2375
```

這可能讓可連到該 port 的人取得近似 root 的 Docker 控制權。

## Troubleshooting

### `sudo apt` 顯示 Windows Sudo 已停用

這表示指令是在 PowerShell 執行。先執行：

```powershell
wsl -d Ubuntu
```

看到類似 `user@host:~$` 後才執行 `sudo apt`。

### `docker --version` 成功但 `docker ps` permission denied

檢查：

```bash
id -nG
ls -l /var/run/docker.sock
```

若使用者不在 `docker` 群組，加入後重啟 WSL。不要把 Docker socket chmod 成 `666`。

### `Cannot connect to the Docker daemon`

```bash
systemctl is-active docker
systemctl --no-pager --full status docker
docker context ls
printf 'DOCKER_HOST=%s\n' "${DOCKER_HOST-}"
```

在本機純 WSL Engine 的預設 context 應使用：

```text
unix:///var/run/docker.sock
```

必要時：

```bash
unset DOCKER_HOST
docker context use default
sudo systemctl restart docker
```

### VS Code 顯示 `Failed to connect. Is WSLC installed?`

這通常是 Container Tools 選到 WSLC，而不是 WSL Ubuntu 裡的 Docker daemon。先安裝/確認 WSL extension，接著以 `WSL: New Window` 進入 Ubuntu，再確認 VS Code WSL 終端機中的 `docker ps`。不要因此安裝 Docker Desktop。

### WSL 重開後 Docker 沒有啟動

確認 `/etc/wsl.conf` 的 `systemd=true`、`docker.service` 是 enabled，並用：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

再在 Ubuntu：

```bash
systemctl is-enabled docker
systemctl is-active docker
```

### MySQL port 3306 被占用

```bash
ss -ltn | grep ':3306 ' || true
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

不要未確認就停止或刪除其他服務；若只是要改外部 port，使用例如 `-p 3307:3306`，並同步更新應用程式連線設定。

## Rules and Limitations

- 這是本機 WSL 2 Ubuntu 的 Docker Engine 流程，不是 Docker Desktop、Podman 或遠端 Docker。
- 任何 `docker rm -f`、`docker volume rm`、`docker system prune`、資料目錄刪除或重建資料庫，先確認目標與備份；沒有明確授權不可做破壞性清理。
- 不得把任何密碼、token 或 credential 寫入 Skill、Git、公開輸出或固定命令列範例。
- 不要將 Docker daemon 暴露到未加密 TCP port；優先使用 Unix socket、WSL Remote 或 SSH。
- 專案與 bind mount 若重視 I/O，優先放在 WSL Linux 檔案系統，例如 `~/projects`，不要預設放在 `/mnt/c`。
- 執行前檢查目前容器、port、volume 與 Docker context；不要因為 Skill 中的舊基線而覆蓋現況。

## Pitfalls

- 把 `sudo apt` 貼到 PowerShell。
- 把 `https://docker.com` 或 `/etc/apt/sources.list.p/` 當成官方 APT 來源。
- 只看 `docker --version` 就宣稱 daemon 正常，沒有測 `docker info` 或 `hello-world`。
- 使用 `chmod 666 /var/run/docker.sock` 解決權限問題。
- 為了 VS Code 連線安裝 Docker Desktop，或把 WSLC 誤認為 Ubuntu 裡的 Docker Engine。
- 用 `docker volume rm` 清除 MySQL 資料，或在未備份前 `docker system prune --volumes`。
- 在回覆中顯示 MySQL root 密碼；只告知 credential 檔案位置並要求使用者在本機讀取。

## Verification

每次完成安裝或修復至少驗證：

```powershell
wsl -l -v
```

```bash
ps -p 1 -o comm=
systemctl is-active docker
docker --version
docker info --format 'Server={{.ServerVersion}} Containers={{.Containers}} Images={{.Images}}'
docker run --rm hello-world
docker ps -a
```

若包含 MySQL，再驗證：

```bash
docker exec -e MYSQL_PWD="$MYSQL_ROOT_PASSWORD" \
  mysql-server mysqladmin ping -uroot --silent
```

若包含 VS Code，確認：

- VS Code 左下角為 `WSL: Ubuntu`。
- VS Code WSL 終端機的 `docker ps` 可列出容器。
- Containers 面板不再顯示 WSLC 連線錯誤。

回報時清楚區分：已修改的檔案/設定、實際執行的驗證、以及尚未驗證的部分。

## Conformance Addendum

### Inputs and Outputs

- **Input:** WSL/Ubuntu 狀態、Docker daemon/context、容器/volume/port 需求、VS Code 錯誤訊息。
- **Output:** 最小且可回復的設定或修復、驗證結果、涉及的容器與資料保留說明。

### Procedure

1. 先確認 Windows/Ubuntu 執行環境與現況。
2. 依需求採用 WSL 2 Ubuntu Docker Engine 流程。
3. 避免 Docker Desktop、未加密 API 與未授權破壞性操作。
4. 用 Docker daemon、容器服務與使用者端工具三層驗證。

### Rules and Limitations

- 以現場狀態優先於 Skill 中記錄的版本與容器清單。
- 不暴露 credential，不把密碼寫入 Skill。
- 破壞性操作必須先取得明確授權並說明資料影響。
