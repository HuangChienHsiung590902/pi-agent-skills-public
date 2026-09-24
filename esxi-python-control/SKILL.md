---
name: esxi-python-control
description: 用 Python 管理本機網段 ESXi Host Client / pyVmomi / WinRM / SSH / VMware Tools Guest Operations API 維運流程。適用於連線到 ESXi 192.168.2.125、查看/修改 VM 設定、修正 Windows Server Guest OS 類型、檢查與啟用 Windows VM 的 OpenSSH Server、建立/整併/改名快照、查詢 host/datastore 狀態、找出未設定的實體磁碟並建立新 VMFS datastore、在網路完全被防火牆封鎖的 VM 上透過 Guest Operations API 執行程式與讀寫檔案、檢查 clone VM 是否忘了做 sysprep(重複 hostname/SID)、把 Windows Server 升級成 Domain Controller 並讓 client VM 加入網域、用 OU+GPO 管理網域內 client（登入橫幅/RDP/開機腳本/UTF-8/SSH 部署）、用 ISO+虛擬光碟機繞過 SMB 被擋的檔案傳輸、處理 VM 重開機後卡黑屏/Tools 不回應的復原流程。密碼不要寫入 skill，執行時向使用者索取或使用當次 session 已提供的憑證。
---

# ESXi Python Control Skill

## 固定環境

- ESXi Host Client: `https://192.168.2.125/ui/#/host`
- ESXi API Host: `192.168.2.125`
- 已驗證 ESXi 版本：`VMware ESXi 8.0.3 build-24022510`
- 主要 Windows Server VM：`SRV`
- `SRV` OS：`Microsoft Windows Server 2025 (64-bit)`
- **2026-08-15 升級成 Domain Controller**：`SRV` 已升級為 `lab.local`
  Forest 的第一台（目前也是唯一一台）DC。升級過程：改靜態 IP →
  `Install-WindowsFeature AD-Domain-Services` → `Install-ADDSForest
  -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns -Force`
  → 自動重開機。驗證方式：`Get-Service NTDS,DNS,Netlogon,ADWS,DFSR`
  應全部 `Running`；`Get-ADDomain` 應回 `DNSRoot=lab.local`。
  - FQDN：`SRV.lab.local`，NetBIOS 網域名稱：`LAB`
  - **靜態 IP：`192.168.2.200`/24**（原本是 DHCP 派發的 `192.168.2.83`，
    升級前已改成靜態；DC 不能用 DHCP IP，之後查/連線一律用
    `192.168.2.200`，不要再用 `.83`）
  - 閘道 `192.168.2.1`，DNS 指向自己（`192.168.2.200`）+ 備援
    `192.168.2.1`
  - `SRV` WinRM: `http://192.168.2.200:5985/wsman`
  - `SRV` SSH: `192.168.2.200:22`
  - DSRM（目錄服務還原模式）密碼由當次 session 產生，沒有記錄在這裡，
    需要時向使用者索取
- ESXi Host 名稱：`Lenovo`
- 實體磁碟 2 顆：
  - WD PC SN740 512GB NVMe → 建成 `datastore1`（VMFS，約 348.8 GB）
  - Crucial BX500 500GB SATA SSD → 建成 `datastore2`（VMFS，約 465.5 GB，2026-08-15 建立，建立時是全新空的，之前完全沒設定過）
- 其他已知 VM（Windows 11 Client，皆無 WinRM/RDP/SSH，只能靠 Guest Operations API 操作）：
  - `WIN11-1`：**靜態 IP `192.168.2.201`**（2026-08-15 從 DHCP 的
    `192.168.2.124` 改為靜態，對齊 SRV 的 `.200` 排序）。這是母片
    （原版），WIN11-2/WIN11-3 是從它複製出來的。**2026-08-15 更新**：
    原本判斷「母片不用 sysprep」，但先前對它跑過的 sysprep 其實只是
    卡住delay、沒有真的失敗——generalize 在 Windows Update 那個 hook
    （`wuaueng.dll`／`GeneralizeForImaging`）卡了將近 20 分鐘後**自己
    默默跑完**，過程中我們已經把 `C:\Windows\System32\Sysprep\unattend.xml`
    刪掉了，但 sysprep 在 generalize 階段會把它複製一份到
    `C:\Windows\Panther\`，specialize 階段讀的是那份副本，所以刪除
    System32 那份不影響結果。最後guest OS 電腦名稱先變成
    `WIN11`（沿用 unattend.xml 裡當時寫的舊名稱），後來手動用
    `Rename-Computer` 改成 `WIN11-1` 對齊 ESXi 庫存名稱。**現況**：
    ComputerName/SID 已經是獨立身分（跟 WIN11-2/WIN11-3 一樣，不再共用
    母片的舊身分），不用再擔心它跟其他機器衝突。**教訓**：VM 卡在
    Guest Ops 連不到／Tools 心跳灰掉時，先別急著斷定是新問題——可能是
    延遲很久的舊操作終於跑完，查 bootTime 有沒有變、guest hostname
    有沒有變，比直接假設「壞了」更準。
  - `WIN11-2`：**靜態 IP `192.168.2.202`**（原 DHCP `192.168.2.254`）。
    已於 2026-08-15 完成 sysprep generalize，ComputerName/SID 已跟母片
    脫鉤成獨立身分。
  - `WIN11-3`：**靜態 IP `192.168.2.203`**（原 DHCP `192.168.2.27`）。
    已於 2026-08-15 完成 sysprep generalize，ComputerName/SID 已跟母片
    脫鉤成獨立身分。
- **IP 配置總表（2026-08-15 全部改成靜態，統一排序）**：
  `SRV`=`.200`（DC）、`WIN11-1`=`.201`、`WIN11-2`=`.202`、
  `WIN11-3`=`.203`，全部 `/24`，閘道 `192.168.2.1`，DNS 都指向
  `192.168.2.200`（DC 自己）+ 備援 `192.168.2.1`。改 IP 後記得跑
  `ipconfig /registerdns` 讓 DC 上的 DNS A 記錄同步更新。
- **2026-08-15：WIN11-1/WIN11-2/WIN11-3 三台全部加入 `lab.local` 網域**
  （見下方「升級 Domain Controller / 讓 client 加入網域」章節），guest OS
  的 `Get-WmiObject Win32_ComputerSystem` 應回 `Domain=lab.local`、
  `PartOfDomain=True`，`vm.guest.hostName` 會多出 `.lab.local` 尾巴（例：
  `WIN11-1.lab.local`）。
- **⚠️ 2026-08-15 之後，這 3 台 client 的 Guest Ops 帳密已經不是
  `hch`/`<SSH_PASSWORD>` 了！** 本機 `hch` 帳號已被**停用**（`Disable-LocalUser`），
  改成強制只能用網域帳號登入。現在對這 3 台跑 Guest Ops，要用：
  - `LAB\Administrator` / `<ECP_PASSWORD>`（網域系統管理員，沿用 DC 升級前
    SRV 本機 Administrator 密碼，對所有網域成員機器都有本機管理員權限）
  - `LAB\hch` / `<SSH_PASSWORD>`（網域使用者帳號，已加進這 3 台各自的本機
    Administrators 群組，密碼特意設回跟以前本機帳號一樣的 `<SSH_PASSWORD>`——因為
    這個密碼不符合預設網域密碼原則的複雜度/長度要求，已經把
    `Set-ADDefaultDomainPasswordPolicy` 放寬成
    `ComplexityEnabled=$false`、`MinPasswordLength=4` 才能設成功；之後
    網域整體密碼強度都會比較弱，這是使用者明確要求的取捨）
  `SRV`（DC 本身）不受影響，繼續用 `Administrator`/`<ECP_PASSWORD>`。
- **已建立網域使用者**：`LAB\hch`（`CN=hch,CN=Users,DC=lab,DC=local`），
  `Enabled=True`、`PasswordNeverExpires=True`，已加入 WIN11-1/2/3 各自的
  本機 Administrators 群組。
- **OU / GPO 現況（2026-08-15 建立，非預設空白狀態）**：
  - OU `Workstations`（`OU=Workstations,DC=lab,DC=local`，
    `ProtectedFromAccidentalDeletion=$true`）——WIN11-1/2/3 三台電腦物件
    都在裡面，`SRV` 留在內建的 `Domain Controllers` OU。
  - GPO `Workstations - Test Policy`，連結在 `Workstations` OU，內容見下方
    「已知 GPO 設定內容」小節。
- **快照命名慣例**：這個環境目前用「加入網域的快照」這個中文名稱標記
  DC+3台 client 都穩定加入網域後的整體基準點（2026-08-15 建立在全部 4
  台 VM 上）；後續又建了「靜態 IP 完成的快照」標記全部改完靜態 IP 後的
  基準點。之後如果要建新的整體基準快照，跟使用者確認好要用的名稱，
  不要自己亂取英文技術名稱又不通知——這個環境明顯偏好中文、有語意的
  快照名稱。
- **Aipower 應用程式部署在 `WIN11-1`（`.201`）**：`D:\Aipower`（本機來源，
  Tomcat + 內嵌 MariaDB4j 的 Java web app）已用「最小安裝」方式（使用者
  明確要求「多餘的就不複製了」，跳過 JDK/管理工具/種子 SQL 等非必要檔案）
  複製到 guest 端 `C:\Aipower`，經 SYSTEM 排程工作 `AipowerService` 啟動。
  HTTP port `22821`、HTTPS port `22822`，登入帳密 `Administrator`/`<ECP_PASSWORD>`。
  **⚠️ 這個排程工作只是「現在啟動了」，沒有設開機自動觸發（trigger 不是
  `AtStartup`），VM 重開機後不會自動回來，要重跑一次 `Start-ScheduledTask`
  才會再啟動**——之後如果要做成開機自動啟動，比照本 skill 其他章節的
  `AtStartup` trigger 模式補上即可。
- **2026-08-15：全部 4 台 VM 系統字碼頁改成 UTF-8**（`HKLM:\SYSTEM\
  CurrentControlSet\Control\Nls\CodePage` 的 `ACP`/`OEMCP`/`MACCP` 改成
  `65001`），同時也寫進 `Workstations - Test Policy` GPO（讓未來新加入
  OU 的機器自動套用），改完 4 台都重開機生效過。
- **已解決：UTF-8 重開機後 `.201`/`.202`/`.203` RDP(3389) 一度從外部連不上**
  （`.200` 不受影響），原本懷疑是本機到 VM 路徑上中間路由器的 ARP cache
  沒刷新（本機對這幾個 IP 完全沒有 ARP entry，代表不同 L2 segment），
  單純等待沒用。**後來證實：把該台 VM 重開機（`vm.ResetVM_Task()`）一次
  就會恢復**——2026-08-15 在幫 WIN11-1、WIN11-3 個別重開機做別的驗證
  （SSH GPO 腳本測試）時，兩台重開機完成後外部 RDP 就都恢復了，不需要
  動路由器。合理推測是重開機時 NIC 重新走一次 DHCP/ARP announce 流程，
  順便把中間路由器的舊 ARP 紀錄刷新掉。**下次再遇到「重開機後某台 RDP
  連不上，但其他台正常」，先試著把那台單獨再重開機一次，通常就會自己
  好，不用去查 GPO/防火牆設定。**

> 不要把 ESXi 或 Windows 密碼寫進此 skill。需要時向使用者確認，或使用當次對話已提供的暫時憑證。

## Python 套件臨時安裝方式

避免污染全域 Python，使用 `%TEMP%` 目錄：

```bash
python -m pip install --quiet --target "$TEMP/pyvmomi-temp" pyvmomi
python -m pip install --quiet --target "$TEMP/pywinrm-temp" pywinrm requests-ntlm
python -m pip install --quiet --target "$TEMP/paramiko-temp" paramiko
```

執行時：

```bash
PYTHONPATH="$TEMP/pyvmomi-temp" python script.py
PYTHONPATH="$TEMP/pywinrm-temp" python script.py
PYTHONPATH="$TEMP/paramiko-temp" python script.py
```

> **坑**：若同一次 `PYTHONPATH` 要接多個目錄（如 `paramiko-temp;pyvmomi-temp`），直接寫 `"$TEMP/a;$TEMP/b"` 在 git-bash 下會被錯誤轉換成 `C:\tmp\a` 而不是真實 Windows temp 路徑（MSYS 路徑轉換對分號分隔的多段路徑處理有問題）。正確做法：先用 `cygpath -w "$TEMP"` 轉成真實 Windows 路徑字串，再自己拼分號：
> ```bash
> TEMP_WIN=$(cygpath -w "$TEMP")
> PYTHONPATH="${TEMP_WIN}\\paramiko-temp;${TEMP_WIN}\\pyvmomi-temp" python script.py
> ```

> **坑**：用 `cat > file.py << EOF`（**沒引號**的 heredoc）寫 Python 檔時，bash 會對內容做反斜線/變量展開，把 Python 字串裡的 `\\` 、`\n` 側包字元吹掉或變形（例如 `'D:\\ E:\\'` 變成 `'D:\ E:\'`，實際送進 console 的指令沒换行，因為 `\n` 已經不是真正的換行字元）。**沒把握就直接用 `Write` 工具寫檔，不要用未引號 heredoc**；若一定要用 heredoc，記得給 `'EOF'`（加引號）才不會被 shell 展開。

## 連線 ESXi 並列 VM

```python
import ssl
from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim

si = SmartConnect(host='192.168.2.125', user='<ESXI_USER>', pwd='<ESXI_PASS>', sslContext=ssl._create_unverified_context())
try:
    about = si.content.about
    print(about.fullName)
    view = si.content.viewManager.CreateContainerView(si.content.rootFolder, [vim.VirtualMachine], True)
    for vm in view.view:
        print(vm.name, vm.runtime.powerState)
    view.Destroy()
finally:
    Disconnect(si)
```

## 查 Host / Datastore 整體狀況

```python
hview = si.content.viewManager.CreateContainerView(si.content.rootFolder, [vim.HostSystem], True)
for h in hview.view:
    print(h.name, h.runtime.connectionState, h.runtime.powerState)
    print('CPU MHz:', h.summary.quickStats.overallCpuUsage, 'Mem MB:', h.summary.quickStats.overallMemoryUsage)
hview.Destroy()

dsview = si.content.viewManager.CreateContainerView(si.content.rootFolder, [vim.Datastore], True)
for ds in dsview.view:
    free_gb = ds.summary.freeSpace / (1024**3)
    cap_gb = ds.summary.capacity / (1024**3)
    print(ds.name, f'{free_gb:.1f} GB free / {cap_gb:.1f} GB total')
dsview.Destroy()
```

查 VM 的 snapshot 樹、VMware Tools 狀態一併用這段（跟列 VM 同一輪跑掉，不用另外連線）：

```python
print(vm.guest.toolsStatus, vm.guest.toolsRunningStatus)  # toolsOk / guestToolsRunning 才代表 Tools 正常在跑
snap = vm.snapshot
if snap:
    def walk(nodes, depth=0):
        for n in nodes:
            print('  ' * depth + f'- {n.name} ({n.createTime})')
            if n.childSnapshotList:
                walk(n.childSnapshotList, depth + 1)
    walk(snap.rootSnapshotList)
```

## 查實體磁碟清單 / 找出還沒建 datastore 的磁碟

用 `configManager.storageSystem.storageDeviceInfo.scsiLun` 列出所有實體磁碟，再比對每個 datastore 的 `info.vmfs.extent[].diskName`，兩邊差集就是「還沒設定的磁碟」：

```python
storage_sys = h.configManager.storageSystem
disks = storage_sys.storageDeviceInfo.scsiLun

ds_disk_names = set()
for ds in h.datastore:
    if ds.info and hasattr(ds.info, 'vmfs') and ds.info.vmfs:
        for ext in ds.info.vmfs.extent:
            ds_disk_names.add(ext.diskName)

for d in disks:
    if d.deviceType != 'disk':
        continue
    cap_gb = (d.capacity.block * d.capacity.blockSize) / (1024**3)
    used = d.canonicalName in ds_disk_names
    print(d.canonicalName, d.displayName, f'{cap_gb:.1f} GB', 'used' if used else 'NOT USED / no datastore')
```

## 建立新的 VMFS Datastore（格式化未使用的磁碟）

⚠️ 這是格式化操作，會清掉目標磁碟上原本可能存在的任何分割/資料，且不可逆。動手前：
1. 先確認 `QueryAvailableDisksForVmfs()` 有列出該磁碟（代表 ESXi 認定它未被佔用）。
2. 用 `RetrieveDiskPartitionInfo([diskName])` 看一下有沒有既有分割紀錄，空的才安全。
3. 跟使用者明確確認要建立才動手。

```python
ds_sys = h.configManager.datastoreSystem

avail = ds_sys.QueryAvailableDisksForVmfs()
disk = next(d for d in avail if d.canonicalName == DISK_NAME)

options = ds_sys.QueryVmfsDatastoreCreateOptions(disk.devicePath)
spec = options[0].spec          # 預設就是用整顆磁碟
spec.vmfs.volumeName = 'datastore2'

new_ds = ds_sys.CreateVmfsDatastore(spec)
print(new_ds.name, new_ds.summary.capacity / (1024**3), 'GB')
```

已知案例：Crucial BX500 500GB SATA SSD 一直閒置沒建 datastore，2026-08-15 用上面流程建成 `datastore2`（465.5 GB 可用），過程順利，`QueryVmfsDatastoreCreateOptions` 回傳的第一個選項就是用整顆磁碟、不用額外指定 partition spec。

## 查 SRV VM 設定

重點欄位：

- `vm.config.guestId`
- `vm.config.guestFullName`
- `vm.config.hardware.numCPU`
- `vm.config.hardware.memoryMB`
- `vm.config.firmware`
- `vm.config.bootOptions.efiSecureBootEnabled`
- `vm.guest.ipAddress`
- disks / nics / cdroms
- `vm.runtime.consolidationNeeded`
- snapshot tree

已知目前健康設定：

```text
name = SRV
power = poweredOn
guestId = windows2022srvNext_64Guest
guestFullName = Microsoft Windows Server 2025 (64-bit)
numCPU = 4
memoryMB = 4096
firmware = efi
secureBoot = True
ip = 192.168.2.83
network = VM Network
nic = VMXNET3
mac = 00:0c:29:6a:7e:4c
disk = [datastore1] SRV/SRV.vmdk
snapshots = 0
consolidationNeeded = False
```

## 修正 Guest OS 類型不相符警告

ESXi UI 若出現：

```text
針對此虛擬機器設定的客體作業系統 ... 與目前正執行的客體 ... 不相符
```

例如 VM 設為 Windows Server 2022，但 guest 實際為 Windows Server 2025。

先查 ESXi 支援的 guest id：

```python
opt = vm.environmentBrowser.QueryConfigOption(None, vm.runtime.host)
for g in opt.guestOSDescriptor:
    if 'Windows Server 2025' in g.fullName:
        print(g.id, g.fullName)
```

本機 ESXi 對應：

```text
windows2022srvNext_64Guest | Microsoft Windows Server 2025 (64-bit)
windows2019srvNext_64Guest | Microsoft Windows Server 2022 (64-bit)
```

修改 Guest OS 類型需要 VM 關機，流程：

1. `vm.ShutdownGuest()` 優雅關機。
2. 等 `vm.runtime.powerState == poweredOff`。
3. `vm.ReconfigVM_Task(vim.vm.ConfigSpec(guestId='windows2022srvNext_64Guest'))`。
4. `vm.PowerOnVM_Task()`。
5. 等 VMware Tools / IP 回來。

不要在 VM 開機時硬改，會得到：

```text
vim.fault.InvalidPowerState: requestedState = poweredOff, existingState = poweredOn
```

## 快照管理

### 建立快照

建立快照前先確認是否需要記憶體快照。一般系統修復後快照建議：

```python
task = vm.CreateSnapshot_Task(
    name='after-ssh-enabled-YYYYMMDD-HHMM',
    description='OpenSSH Server repaired/enabled; verified SSH login.',
    memory=False,
    quiesce=False,
)
```

### 整併/刪除所有快照

此操作會移除所有 snapshot，無法再回復到舊時間點。必須先取得使用者明確確認。

```python
print(vm.runtime.consolidationNeeded)
task = vm.RemoveAllSnapshots_Task(consolidate=True)
# wait task
if vm.runtime.consolidationNeeded:
    task = vm.ConsolidateVMDisks_Task()
```

成功後應確認：

```text
snapshots = 0
consolidationNeeded = False
disk file = [datastore1] SRV/SRV.vmdk
```

若整併前看到 `SRV-000001.vmdk` / `SRV-000002.vmdk`，代表正在跑 snapshot delta disk，長期不建議。

## 透過 WinRM 控制 SRV

`SRV` 已知 WinRM HTTP `5985` 可用，使用 `pywinrm`：

```python
import winrm

s = winrm.Session(
    'http://192.168.2.83:5985/wsman',
    auth=('<WIN_USER>', '<WIN_PASS>'),
    transport='ntlm',
    server_cert_validation='ignore',
    operation_timeout_sec=20,
    read_timeout_sec=60,
)
r = s.run_ps('hostname; whoami; $PSVersionTable.PSVersion.ToString()')
print(r.status_code)
print(r.std_out.decode('utf-8', 'replace'))
print(r.std_err.decode('utf-8', 'replace'))
```

## 檢查 Windows OpenSSH Server

用 WinRM 執行：

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
Get-Service -Name sshd,ssh-agent -ErrorAction SilentlyContinue
Get-Command sshd.exe -ErrorAction SilentlyContinue
Test-Path C:\ProgramData\ssh\sshd_config
Get-NetTCPConnection -LocalPort 22 -State Listen -ErrorAction SilentlyContinue
Get-NetFirewallRule | Where-Object { $_.DisplayName -match 'OpenSSH|SSH' -or $_.Name -match 'OpenSSH|SSH' }
```

常見問題：

1. `OpenSSH.Server` 已安裝，但 `C:\ProgramData\ssh\sshd_config` 不存在。
2. `sshd.exe -t` 報：

```text
__PROGRAMDATA__\ssh/sshd_config: No such file or directory
```

3. host key 權限太寬，`sshd -D -e` 報：

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions for '__PROGRAMDATA__\ssh/ssh_host_*_key' are too open.
sshd: no hostkeys available -- exiting.
```

## 修復 Windows OpenSSH Server

### 1. 建立 sshd_config / host keys

```powershell
$sshDir = 'C:\ProgramData\ssh'
New-Item -ItemType Directory -Force -Path $sshDir | Out-Null
Copy-Item $env:WINDIR\System32\OpenSSH\sshd_config_default C:\ProgramData\ssh\sshd_config -Force
Add-Content C:\ProgramData\ssh\sshd_config "`nPasswordAuthentication yes`nPubkeyAuthentication yes`n"
& $env:WINDIR\System32\OpenSSH\ssh-keygen.exe -A
& $env:WINDIR\System32\OpenSSH\sshd.exe -t
```

### 2. 修正 host key ACL

Windows OpenSSH 對 private host key ACL 很嚴格。可將 private host key 限制到 `SYSTEM`：

```powershell
foreach($key in Get-ChildItem C:\ProgramData\ssh\ssh_host_*_key -File | Where-Object { $_.Name -notlike '*.pub' }){
  icacls $key.FullName /inheritance:r | Out-Null
  icacls $key.FullName /remove:g "Administrators" "BUILTIN\Administrators" "Authenticated Users" "Users" "Everyone" "$env:COMPUTERNAME\Administrator" 2>$null | Out-Null
  icacls $key.FullName /grant:r "SYSTEM:F" | Out-Null
  icacls $key.FullName /setowner "SYSTEM" | Out-Null
}
```

> 若 WinRM session 使用 Administrator，改成 SYSTEM-only 後讀取 ACL 可能顯示 `Access is denied`，但由 SYSTEM 執行的 sshd 可正常讀取。

### 3. 防火牆開 TCP 22

```powershell
if(-not (Get-NetFirewallRule -Name 'Allow-SSH-22-Any' -ErrorAction SilentlyContinue)){
  New-NetFirewallRule -Name 'Allow-SSH-22-Any' -DisplayName 'Allow SSH TCP 22 Any' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 22 -Profile Any | Out-Null
}else{
  Set-NetFirewallRule -Name 'Allow-SSH-22-Any' -Enabled True -Action Allow -Profile Any | Out-Null
}
```

### 4. 啟動方式注意事項

**這個問題不是只有 `SRV`/Server 2025 Evaluation 才有，Windows 11 client
（WIN11-1/2/3）親測也會發生（2026-08-15）**：`Get-Service sshd` 回報
`Status=Running`、`StartType=Automatic`，看起來一切正常，但實際上
`Get-Process sshd` 查不到任何行程、`Get-NetTCPConnection -LocalPort 22`
也是空的——**service 回報的狀態是假的/過期的，process 早就已經死掉，
沒有人在監聽 22 port**。這次是先前用 `Restart-Service sshd -Force`
重啟 native service 之後過一段時間才出現（不是重啟當下就死，是之後
才悄悄掛掉），使用者實測 `ssh hch@192.168.2.201` 直接被
`Connection closed by 192.168.2.201 port 22` 打斷，就是連到一個表面
Running、實際上行程已死的 sshd。**教訓：`Get-Service sshd` 的
`Status=Running` 不能當作「真的在監聽」的證據，要嘛額外查
`Get-NetTCPConnection -LocalPort 22`，要嘛乾脆不要依賴 native service，
一律改用下面這套 SYSTEM 排程工作跑前景 `sshd.exe -D` 的方式**（手動/
前景模式可正常 listen，而且排程工作掛了 Windows 排程本身有機制可以自動
重啟）。

已使用的 workaround：建立 `SYSTEM` Scheduled Task 跑 foreground sshd。

排程工作：

```text
TaskPath = \Custom\
TaskName = Run OpenSSH sshd foreground
```

腳本：

```powershell
C:\ProgramData\ssh\run_sshd_debug.ps1
```

內容：

```powershell
& C:\Windows\System32\OpenSSH\sshd.exe -D -e *>> C:\ProgramData\ssh\task_sshd.log
"EXIT=$LASTEXITCODE $(Get-Date)" | Out-File C:\ProgramData\ssh\task_sshd.log -Append
```

建立/啟動排程：

```powershell
$taskName='Run OpenSSH sshd foreground'
$taskPath='\Custom\'
$action=New-ScheduledTaskAction -Execute 'powershell.exe' -Argument '-NoProfile -ExecutionPolicy Bypass -File "C:\ProgramData\ssh\run_sshd_debug.ps1"'
$trigger=New-ScheduledTaskTrigger -AtStartup
$principal=New-ScheduledTaskPrincipal -UserId 'SYSTEM' -RunLevel Highest
# RestartCount/RestartInterval：如果 sshd.exe 這個前景行程意外中斷，讓
# 排程工作自動重跑，不用整個手動重建（見上方「啟動方式注意事項」的
# service 假 Running 陷阱，前景模式也要防它自己中斷）
$settings=New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit (New-TimeSpan -Days 0) -RestartCount 999 -RestartInterval (New-TimeSpan -Minutes 1)
Unregister-ScheduledTask -TaskName $taskName -TaskPath $taskPath -Confirm:$false -ErrorAction SilentlyContinue
Register-ScheduledTask -TaskName $taskName -TaskPath $taskPath -Action $action -Trigger $trigger -Principal $principal -Settings $settings | Out-Null
Start-ScheduledTask -TaskName $taskName -TaskPath $taskPath

# 徹底避免 native service 又悄悄搶起來造成埠衝突/狀態混亂，乾脆停用它
Stop-Service sshd -Force -ErrorAction SilentlyContinue
Set-Service sshd -StartupType Disabled -ErrorAction SilentlyContinue
```

驗證：

```powershell
Get-Process sshd
Get-NetTCPConnection -LocalPort 22 -State Listen
Get-Content C:\ProgramData\ssh\task_sshd.log -First 80
```

成功訊息：

```text
Server listening on :: port 22.
Server listening on 0.0.0.0 port 22.
```

### 5. 用 GPO/排程工作在 client 上裝 OpenSSH.Server 時，`Get-WindowsCapability -Online` 很容易卡很久

在 client（Windows 11）上要用 GPO 開機腳本 / SYSTEM 排程工作批次裝
`OpenSSH.Server` capability，**不要用**：

```powershell
$cap = Get-WindowsCapability -Online -Name "OpenSSH.Server*"   # 不要這樣查
Add-WindowsCapability -Online -Name $cap.Name
```

`Get-WindowsCapability -Online` 帶萬用字元查詢會去列舉完整的線上
capability 清單，親測**即使網路正常**（`Test-NetConnection 8.8.8.8:443`
/`www.microsoft.com:443` 都回 `True`），這個查詢也可能佔住 DISM/CBS 鎖
卡好幾分鐘不返回，且會擋住同一台機器上**其他獨立的** `Get-WindowsCapability
-Online` 查詢一起卡住。這不是「無網路」的錯誤訊息，是單純查詢很慢；親測
過最久卡了將近 6 分鐘（`t+340s` 才出現結果）才自己跑完，**不是真的無限
卡死，只是非常慢**，如果外層腳本/排程工作的 timeout 設太短（例如
1-2 分鐘）會被誤判成失敗而提前砍掉。

改用**已知確切的 capability 名稱**直接裝，跳過列舉查詢：

```powershell
$log = "C:\Windows\Temp\ssh_install_test_log.txt"
"START $(Get-Date)" | Out-File $log
try {
    Add-WindowsCapability -Online -Name "OpenSSH.Server~~~~0.0.1.0" -ErrorAction Stop
    "Add-WindowsCapability SUCCESS $(Get-Date)" | Out-File $log -Append
} catch {
    "Add-WindowsCapability FAILED: $_" | Out-File $log -Append
}
"sshd.exe present: $(Test-Path 'C:\Windows\System32\OpenSSH\sshd.exe')" | Out-File $log -Append
"END $(Get-Date)" | Out-File $log -Append
```

`OpenSSH.Server~~~~0.0.1.0` 是 Windows 11 24H2/25H2 目前這個版本的確切
capability 名稱（跟 `Get-WindowsCapability -Online | Where Name -like
'OpenSSH.Server*'` 查出來的 `Name` 欄位值一致，只是省掉了「先查一次
清單」這個慢步驟，直接把已知名稱寫死）。實測（2026-08-15，`WIN11-2`）：
包在 15 分鐘 timeout 的 SYSTEM 排程工作裡跑，約 6 分鐘後 log 出現
`Add-WindowsCapability SUCCESS`、`sshd.exe present: True`，確認裝好。

**執行環境要包在有足夠 timeout 的 SYSTEM 排程工作裡**（`New-ScheduledTaskSettingsSet
-ExecutionTimeLimit (New-TimeSpan -Minutes 15)` 這種等級，不要設太短），
而不是直接 `StartProgramInGuest` 同步等（Guest Ops 輪詢端通常等不了那麼
久）。

## 設定 Windows OpenSSH 免密碼登入（公鑰認證）

前提：目標帳號是本機/網域 **Administrators 群組成員**（這個環境目前都是
用 `Administrator`/`LAB\Administrator` 連線）。Windows OpenSSH 對「管理員
帳號」跟「一般帳號」的 authorized_keys 檔案位置**不一樣**，管理員一律
共用同一個系統層級檔案，不是每個帳號自己 `~/.ssh/authorized_keys`：

```powershell
$pubkey = "ssh-ed25519 AAAA... 你的公鑰"
$authFile = "C:\ProgramData\ssh\administrators_authorized_keys"
Set-Content -Path $authFile -Value $pubkey -Encoding ASCII   # 或 Add-Content 追加多把

# 這個檔案的 ACL 一樣很嚴格（跟 host key 同一套邏輯），只能給 SYSTEM 跟
# Administrators，繼承權限沒清掉的話 sshd 會直接拒絕使用這把 key
icacls $authFile /inheritance:r | Out-Null
icacls $authFile /grant "SYSTEM:F" | Out-Null
icacls $authFile /grant "Administrators:F" | Out-Null
icacls $authFile /setowner "Administrators" | Out-Null
```

`sshd_config` 要有 `PubkeyAuthentication yes`（本 skill 前面「修復 Windows
OpenSSH Server」步驟 1 已經預設帶上了，通常不用再加）。改完 ACL/config
要重啟 `sshd`：service 模式用 `Restart-Service sshd -Force`；如果這台是走
「service 常常起不來、改用 SYSTEM 排程工作跑前景 `sshd.exe -D`」那套
workaround（見前面章節），要改成先砍掉舊的 `sshd.exe` 行程、`Stop-
ScheduledTask` 再 `Start-ScheduledTask` 重新拉起。

已知案例（2026-08-15）：對 `SRV`/`WIN11-1`/`WIN11-2`/`WIN11-3` 四台都用
操作端本機既有的 `~/.ssh/id_ed25519` 公鑰，透過 Guest Ops 寫入
`administrators_authorized_keys`+修 ACL+重啟 `sshd`，接著用 `paramiko`
（`key_filename=...`、**不帶** `password`、`look_for_keys=False`）對四台
各自的 IP 用 `Administrator`/`LAB\Administrator` 登入，全部一次成功、
完全沒有密碼提示，`whoami` 回報身分正確（`lab\administrator`）。**這個
方式不會動到密碼登入本身**（`PasswordAuthentication yes` 還留著），是
單純多加一種登入方式，不是取代；如果之後要求「強制只能用金鑰、密碼登入
也要關掉」，才需要額外把 `sshd_config` 的 `PasswordAuthentication` 改成
`no` 再重啟 `sshd`。**`administrators_authorized_keys` 是共用檔案，只要
帳號是本機 Administrators 群組成員就適用**——這個環境的 `LAB\hch` 因為
先前被加進每台 client 的本機 Administrators 群組（見「讓 client 只能用
網域帳號登入」章節），不用額外幫它加一份 key，用同一把公鑰登入
`LAB\hch` 一樣成功。

**使用者從 Windows 用內建 `ssh.exe`（PowerShell）連網域帳號時的指令
寫法**：PowerShell 裡反斜線不是跳脫字元，`DOMAIN\user@host` 可以直接寫，
不用加引號、也不用寫成雙反斜線：

```powershell
ssh LAB\hch@192.168.2.201
```

如果想用 UPN 格式（`user@domain`），因為第二個 `@` 會跟 `user@host` 語法
的 `@` 混淆，要改用 `-l` 參數指定帳號：

```powershell
ssh -l hch@lab.local 192.168.2.201
```

兩種寫法等價，都能正確登入 `LAB\hch`。

**⚠️ 這個環境的原生 `sshd` Windows service 有「顯示 Running 但其實
process 已死、沒監聽 port」的隱性當機問題**（不限 SRV，WIN11 client 也
會發生，見下方「啟動方式注意事項」），**幫使用者設定完免密碼登入後，
務必額外確認走的是排程工作前景執行模式、且已經停用 native service**，
不要只看 `Get-Service sshd` 顯示 `Running` 就當作沒問題——這正是使用者
在這個環境親身遇到「金鑰設好了，但連線被 `Connection closed` 打斷」的
根本原因，不是金鑰或帳號設定錯誤。

## 用 Paramiko 驗證 SSH

```python
import paramiko

client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(
    hostname='192.168.2.83',
    username='<WIN_USER>',
    password='<WIN_PASS>',
    timeout=15,
    auth_timeout=15,
    banner_timeout=15,
    look_for_keys=False,
    allow_agent=False,
)
stdin, stdout, stderr = client.exec_command('hostname && whoami && ver')
print(stdout.read().decode('utf-8', 'replace'))
print(stderr.read().decode('utf-8', 'replace'))
client.close()
```

已驗證成功輸出類似：

```text
SRV
srv\administrator
Microsoft Windows [Version 10.0.26100.33158]
```

## 用 VMware Tools Guest Operations API 操作「沒有網路管道」的 VM

當 VM 的 WinRM/RDP/SSH 全部連不上（防火牆擋住、或根本沒開），只要 `vm.guest.toolsRunningStatus == 'guestToolsRunning'`，就可以繞過 VM 網路，直接透過 ESXi 管理通道叫 guest 內的 VMware Tools 執行程式、讀寫檔案。連線用 ESXi 帳密（root），guest 操作用 guest OS 帳密（兩組憑證不同，各自傳入）。

適用情境判斷：先用 `port scan`（5985/3389/22）確認真的連不到，再改走這條路，不要一開始就跳過標準 WinRM/SSH。

### 基本連線骨架

```python
gom = si.content.guestOperationsManager
creds = vim.vm.guest.NamePasswordAuthentication(username='<GUEST_USER>', password='<GUEST_PASS>')
```

### ProcessManager：在 guest 內執行程式（含跑 PowerShell 取得輸出）

Guest Operations API 本身不會直接回傳 stdout，標準作法是：把輸出重導向到 guest 內一個暫存檔，等程式跑完，再用 FileManager 把那個檔案下載回來讀取。

```python
import base64, time

ps_script = 'Get-ItemProperty "HKLM:\\SYSTEM\\Setup" | Format-List *'
encoded = base64.b64encode(ps_script.encode('utf-16-le')).decode('ascii')
out_path = 'C:\\Windows\\Temp\\out.txt'

spec = vim.vm.guest.ProcessManager.ProgramSpec(
    programPath='C:\\Windows\\System32\\cmd.exe',
    arguments=f'/c powershell.exe -NoProfile -EncodedCommand {encoded} > "{out_path}" 2>&1',
)
pid = gom.processManager.StartProgramInGuest(vm, creds, spec)

for _ in range(30):
    time.sleep(1)
    procs = gom.processManager.ListProcessesInGuest(vm, creds, [pid])
    if procs and procs[0].endTime is not None:
        break

file_info = gom.fileManager.InitiateFileTransferFromGuest(vm, creds, out_path)
url = file_info.url.replace('*', ESXI_HOST)  # ESXi 回傳的 URL 用 * 當 host 佔位符，要換成實際 IP
resp = requests.get(url, verify=False, timeout=20)
print(resp.text)
```

> 用 `-EncodedCommand`（base64 UTF-16LE）包 PowerShell 指令，避免命令列引號跳脫問題。

`ListProcessesInGuest(vm, creds)` 不帶 pid 參數可列出 guest 內所有程序（pid / owner / cmdLine），`TerminateProcessInGuest(vm, creds, pid)` 可砍程序。

### FileManager：列目錄 / 上傳 / 下載 / 刪除

```python
res = gom.fileManager.ListFilesInGuest(vm, creds, 'C:\\')
for f in res.files:
    print(f.type, f.path)

tmp_path = gom.fileManager.CreateTemporaryFileInGuest(vm, creds, prefix='tmp_', suffix='.txt')

payload = b'hello'
put_url = gom.fileManager.InitiateFileTransferToGuest(
    vm, creds, tmp_path,
    vim.vm.guest.FileManager.FileAttributes(),
    len(payload), True,
).replace('*', ESXI_HOST)
requests.put(put_url, data=payload, verify=False, timeout=20)

get_url = gom.fileManager.InitiateFileTransferFromGuest(vm, creds, tmp_path).url.replace('*', ESXI_HOST)
print(requests.get(get_url, verify=False, timeout=20).text)

gom.fileManager.DeleteFileInGuest(vm, creds, tmp_path)
```

`MakeDirectoryInGuest` / `DeleteDirectoryInGuest` / `MoveFileInGuest` 用法同理。

限制：
- 沒有互動式 shell，每次都是「丟程式執行 → 輪詢結束 → 視需要撈輸出檔」的一次性模式。
- 檔案傳輸有大小限制，適合設定檔/腳本/log，不適合傳大檔。
- 需要 VMware Tools 正常運作與有效 guest 帳密；跟 VM 網路防火牆設定完全無關。

## 用 ISO + 虛擬光碟機傳大量檔案（本機 SMB client 被擋時的繞路方法）

`InitiateFileTransferToGuest`（Guest Ops）適合傳單一小檔（設定檔/腳本），
不適合傳一整個資料夾的應用程式（例如 `D:\Aipower` 這種內含 JRE+Tomcat+
一堆 jar 的目錄）。原本想法是掛 admin share（`net use` 連 `<VM_IP>` 的
`C` 槽 admin share）直接用 SMB 複製，但親測 `net use` 對這個環境**任何**
目標（連 DC 也一樣）都回 `System error 67`——先用 SRV 當控制組測過，
排除是單一 VM
的問題，判斷是**本機（操作端）自己的 SMB client 被防火牆/EDR 擋住**，
不是 VM 端的問題，沒有繼續往這個方向修，改走下面這條路：

### 做法：把整個資料夾包成 ISO，透過 ESXi datastore 上傳，再掛成 VM 的虛擬光碟機

1. **本機端**（Windows）用 `IMAPI2FS.MsftFileSystemImage` COM 物件把來源
   資料夾打包成 `.iso`：
   ```powershell
   $image = New-Object -ComObject IMAPI2FS.MsftFileSystemImage
   $image.FileSystemsToCreate = 4   # ISO9660 + Joliet + UDF 視需要調整
   $image.Root.AddTree($sourcePath, $true)   # 第二參數一定要 $true
   $result = $image.CreateResultImage()
   ```
   **`AddTree($path, $false)` 會把子資料夾「攤平合併」進去**，如果來源
   底下有多個子資料夾各自都有同名子目錄（例如 `apache-tomcat\bin` 跟
   `jre\bin`），會直接報 `'bin' 名稱已存在` 衝突。**一定要用
   `AddTree($path, $true)`** 保留子資料夾原本的階層/名稱。
2. **`$result.ImageStream` 是原生 COM `IStream`，不是 IDispatch**，
   PowerShell 預設的動態 COM 呼叫（`$stream.Read(...)`、`$stream.Length`）
   對這種介面會靜默失敗或回傳垃圾值/0，讀不出正確內容。要用內嵌 C#
   helper 明確轉型成 `System.Runtime.InteropServices.ComTypes.IStream`
   才能正常呼叫：
   ```powershell
   Add-Type -TypeDefinition @"
   using System;
   using System.IO;
   using System.Runtime.InteropServices.ComTypes;
   public static class IsoHelper {
       public static void SaveStream(object comStream, string path) {
           IStream stream = (IStream)comStream;
           STATSTG stat; stream.Stat(out stat, 0);
           long size = stat.cbSize;
           using (FileStream fs = new FileStream(path, FileMode.Create)) {
               byte[] buffer = new byte[65536];
               IntPtr bytesReadPtr = Marshal.AllocHGlobal(sizeof(int));
               long remaining = size;
               while (remaining > 0) {
                   stream.Read(buffer, buffer.Length, bytesReadPtr);
                   int bytesRead = Marshal.ReadInt32(bytesReadPtr);
                   if (bytesRead <= 0) break;
                   fs.Write(buffer, 0, bytesRead);
                   remaining -= bytesRead;
               }
               Marshal.FreeHGlobal(bytesReadPtr);
           }
       }
   }
   "@
   [IsoHelper]::SaveStream($result.ImageStream, "C:\path\to\output.iso")
   ```
3. **上傳到 ESXi datastore**（HTTP PUT，Basic Auth，跟截圖下載同一套端點
   風格）：
   ```powershell
   Invoke-WebRequest -Uri "https://<esxi>/folder/<VM資料夾>/app.iso?dsName=datastore1" `
       -Method Put -InFile "C:\path\to\output.iso" -Headers @{ Authorization = "Basic <base64 root:pwd>" } -SkipCertificateCheck
   ```
4. **VM 開機狀態下**直接把虛擬光碟機的 backing 換成這個 ISO（不用關機，
   跟裝 VMware Tools 那次要關機新增 CD-ROM 裝置不一樣，這裡是已經有
   CD-ROM 裝置、只是換它指向的 ISO 檔）：
   ```python
   cdrom = next(d for d in vm.config.hardware.device if isinstance(d, vim.vm.device.VirtualCdrom))
   cdrom.backing = vim.vm.device.VirtualCdrom.IsoBackingInfo(fileName='[datastore1] <VM資料夾>/app.iso')
   cdrom.connectable = vim.vm.device.VirtualDevice.ConnectInfo(connected=True, startConnected=True, allowGuestControl=True)
   spec = vim.vm.ConfigSpec(deviceChange=[vim.vm.device.VirtualDeviceSpec(
       operation=vim.vm.device.VirtualDeviceSpec.Operation.edit, device=cdrom)])
   vm.ReconfigVM_Task(spec)
   ```
5. **guest 端用 `robocopy` 從掛載的光碟機代號複製到本機磁碟**（比
   Guest Ops 檔案傳輸快很多，因為走的是虛擬硬體而不是一個個檔案輪詢）：
   ```powershell
   robocopy D:\ C:\Aipower /E
   ```
   **`robocopy` 的退出碼 `1` 代表「成功複製了檔案」，不是錯誤**——不要
   用 `$LASTEXITCODE -eq 0` 判斷成功，`robocopy` 的 exit code 是位元旗標，
   `0`/`1` 都算正常結束，`>=8` 才是真的有錯誤。
6. **⚠️ 從 ISO9660 複製出來的檔案會帶著唯讀（ReadOnly）屬性**，`robocopy`
   照抄過去，NTFS 上的檔案也會是唯讀。這對純靜態檔案沒差，但如果應用
   程式會在執行期寫入這些檔案（例如這次案例中 Aipower 內嵌的 MariaDB4j
   要寫 DDL log），會直接失敗：
   ```text
   DDL_LOG: Failed to create ddl log file
   Process exited with an error: 1
   ```
   修法：複製完之後多跑一次
   ```powershell
   attrib -R "C:\Aipower\*.*" /S /D
   ```
   拿掉唯讀屬性，再重啟服務即可。

## 檢查 Clone VM 是否忘了做 Sysprep（重複 hostname / SID）

Windows Server 2022+ / Server 2025 / Windows 11 已經沒有沿用舊版 `HKLM:\SYSTEM\Setup\State` 的 `ImageState` 機制（查詢會得到路徑不存在），不能再用「有沒有 ImageState」判斷 sysprep 有沒有跑完。改用下面兩個更可靠的訊號：

```powershell
$setup = Get-ItemProperty 'HKLM:\SYSTEM\Setup' -ErrorAction SilentlyContinue
$cn = (Get-WmiObject Win32_ComputerSystem).Name
$adminSid = (Get-CimInstance Win32_UserAccount -Filter "Name='Administrator'" -ErrorAction SilentlyContinue).SID
"ComputerName=$cn"
"AdminSID=$adminSid"
"OOBEInProgress=$($setup.OOBEInProgress)"
"SystemSetupInProgress=$($setup.SystemSetupInProgress)"
"CloneTag=$($setup.CloneTag)"
"RespecializeCmdLine=$($setup.RespecializeCmdLine)"
```

- `OOBEInProgress=0` 且 `SystemSetupInProgress=0` 只代表「開機精靈跑完了」，**不代表**真的做過 sysprep generalize。
- `CloneTag` / `RespecializeCmdLine`（`Sysprep\sysprep.exe /respecialize /quiet`）存在，代表這台曾被 VM clone 偵測機制標記過，但不保證 respecialize 真的成功執行。
- **真正判斷有沒有做好 sysprep 的關鍵：跨多台比對 `ComputerName` 跟 `AdminSID`。** 如果多台 VM 這兩個值完全一樣，就是直接複製母片、guest 內部從沒 generalize 過，會有 hostname/SID 衝突風險（同網段 NetBIOS 衝突、加網域直接炸、SMB 互搶名稱）。

已知案例（2026-08-15 查到）：`WIN11` / `WIN11-2` / `WIN11-3` 三台 ComputerName 全部是 `DESKTOP-PBEER7E`、Administrator SID 全部是 `S-1-5-21-1958869348-3828327283-2317817556-500`，只有 ESXi 層級的 `MachineUUID` 不同 → 確認是沒做 sysprep 的裸複製，需要在其中至少 2 台重新執行 `sysprep /generalize /oobe /reboot`（**這是會改變 guest 狀態並重開機的操作，動手前要跟使用者確認**）。`SRV` 因為只有一台不會有衝突問題，即使它也帶有 `CloneTag`。

這三台網路服務全擋，只能靠上面的 Guest Operations API 執行 PowerShell 查詢，查不到就代表根本連不到，不是帳密錯。

## VM 磁碟 Thick → Thin 轉換（省空間，不需要 vCenter）

獨立（非 vCenter 管理）ESXi 主機的 `RelocateVM_Task`／`CloneVM_Task` 常會被 Free/Essentials 授權擋掉：

```text
(vmodl.fault.NotSupported) { msg = 'The operation is not supported on the object.' }
```

遇到這個錯誤，改用 **SSH + `vmkfstools`** 在檔案層直接轉換，不走 vSphere API 的 migrate/clone：

1. 先啟用 SSH（standalone ESXi 預設關閉）：
   ```python
   host = content.rootFolder.childEntity[0].hostFolder.childEntity[0].host[0]
   ss = host.configManager.serviceSystem
   ss.StartService(id='TSM-SSH')
   ```
2. VM 必須先關機（`vm.PowerOffVM_Task()`），檔案才不會被鎖住（`Failed to lock the file (16392)`）。
3. 用 paramiko SSH 進 ESXi，執行：
   ```bash
   vmkfstools -i /vmfs/volumes/datastore1/WIN11/WIN11.vmdk -d thin /vmfs/volumes/datastore1/WIN11/WIN11-thin.vmdk
   ```
   這個轉換會自動偵測並跳過全零區塊，只保留實際寫入資料（親測 100GB thick eager-zeroed 磁碟，內容其實只有 20多GB 資料，轉完只佔 22-25GB）。
4. 用 `mv` 把新舊檔案互換名稱（原檔改成 backup 名稱，新 thin 檔案改成原檔名），並用 `sed` 把 `.vmdk` descriptor 裡的 extent 檔名指向改過來：
   ```bash
   mv WIN11.vmdk WIN11-thick-backup.vmdk
   mv WIN11-flat.vmdk WIN11-thick-backup-flat.vmdk
   mv WIN11-thin.vmdk WIN11.vmdk
   mv WIN11-thin-flat.vmdk WIN11-flat.vmdk
   sed -i 's/WIN11-thin-flat.vmdk/WIN11-flat.vmdk/' WIN11.vmdk
   ```
5. **開機驗證再刪備份**：用 `vm.PowerOnVM_Task()` 開機，`vm.CreateScreenshot_Task()` 截圖確認能正常進到鎖定畫面／桌面（見下方截圖驗證段落），確認沒問題才 `rm` 掉 thick 備份釋放空間。絕對不要沒驗證就刪舊檔。
6. 用 `vm.Reload()` 才會讓 pyVmomi 抓到新的 `backing.thinProvisioned=True`（不 reload 會顯示舊的快取值）。

用 `du -h` 而非 `ls -la` 才能看到 thin 磁碟的實際佔用（`ls -la` 對 sparse 檔案顯示的是邏輯大小，會誤判成沒省到空間）。

## 用磁碟複製建立新 VM（不用 CloneVM_Task）

同樣因為 standalone ESXi 擋掉 `CloneVM_Task`，改用「SSH 複製 vmdk + `CreateVM_Task` 手動組裝」：

1. 先讀原 VM 完整 `config.hardware.device` 列表（含 SCSI controller 型別、disk 的 controllerKey/unitNumber/capacityInKB、NIC 型別與 network 名稱、`firmware`、`bootOptions.efiSecureBootEnabled`、`config.version`），複製時要照抄。
2. SSH 用 `vmkfstools -i 來源.vmdk -d thin 目的資料夾/目的.vmdk` 複製磁碟（連同步驟一的 thin 轉換一起做）。
3. 用 `vm_folder.CreateVM_Task()` 建立新 VM，`deviceChange` 裡新增 SCSI controller（例如 `VirtualLsiLogicSASController`）+ 磁碟（`VirtualDisk`，backing 指向剛剛複製好的 vmdk，**不要設 `fileOperation`**，設了會被當成「要新建磁碟」而不是「掛載既有磁碟」）+ 網卡。
4. 新機開機後身分（電腦名稱/SID）會跟原機一模一樣，是正常的——要換身分得靠 sysprep（見下方章節）。

## 沒有 VMware Tools 時的「盲操作」主控台自動化（PutUsbScanCodes）

Guest 裡沒裝 VMware Tools 時，唯一能跟主控台互動的方式是：

- `vm.CreateScreenshot_Task()` → 回傳 `[datastore1] VM/xxx.png`，用 `https://<esxi>/folder/VM/xxx.png?dcPath=ha-datacenter&dsName=datastore1`（Basic Auth root 帳密）下載成圖片來看。
- `vm.PutUsbScanCodes(spec)` 送 USB HID 鍵盤事件（**沒有滑鼠 API**，只能鍵盤）。事件編碼：`usbHidCode = (hid_usage_id << 16) | 0x07`，這個 `0x07` 是社群樣本裡驗證過會同時觸發 press+release 的魔術值。Shift 等修飾鍵透過 `vim.UsbScanCodeSpecKeyEvent.modifiers`（`vim.UsbScanCodeSpecModifierType`，`leftShift`/`leftGui`/`leftControl`/`leftAlt`）設定，`leftGui=True` 就是 Win 鍵（例如 Win+R、Win+X）。
- Windows 鎖定畫面：只有一個帳號時，送一個 `ENTER` 就會直接跳到密碼欄且已經 focus，可以直接接著打密碼再 `ENTER`。
- **UAC 對話框預設焦點常常不是「是」**（親測是「否」），要先送 `LEFT` 再 `ENTER`，不能無腦按 `ENTER`。
- 這招連 Win+X 選單的英數快捷鍵（如按 `a` 選「終端機(系統管理員)(A)」）有時會沒反應（原因未查清，懷疑是選單接收焦點的時序問題），**改用 Win+R 開「執行」對話框打 `cmd` 更穩定可靠**。
- `wmic` 在新版 Windows 11 已被移除（`'wmic' 不是內部或外部命令`），查光碟機代號等資訊改用 `powershell -Command "Get-CimInstance Win32_CDROMDrive | Select Drive,VolumeName,MediaLoaded"`。
- **`Get-CimInstance` 回報的 `VolumeName` 可能是過期快取**（親眼看過顯示 `DVD_ROM` 但實際 `dir D:\` 進去看到的是正確的 `VMware Tools` 磁碟內容）——這類資訊不確定時，直接 `dir D:\` 看真實內容比較準。

## 用主控台自動化靜默安裝 VMware Tools

1. VM 關機時才能新增 CD-ROM 裝置（`ReconfigVM_Task` 開機中會報 `InvalidPowerState`）；沒有 CD-ROM 裝置時 `vm.MountToolsInstaller()` 會報 `VmToolsUpgradeFault` / `msg.gui.toolsNoDevice`。所以順序是：關機 → 加 CD-ROM（IDE controller，`RemotePassthroughBackingInfo`，`connectable.connected=True`）→ 開機 → `vm.MountToolsInstaller()`。
2. 光碟掛載後用主控台鍵盤開 `cmd`，執行：
   ```text
   D:\setup64.exe /S /v"/qn REBOOT=R"
   ```
   會跳 UAC（見上方「LEFT 再 ENTER」），確認後靜默安裝，通常不用重開機、幾秒到一分鐘內 `vm.guest.toolsRunningStatus` 就會變成 `guestToolsRunning`。
3. **如果同一台 VM 之前已經默默啟動過一次安裝**（例如螢幕沒截到、以為沒反應而重打指令），第二次執行會跳出「您確定要取消 VMware Tools 安裝嗎？」對話框，選「否(N)」讓它繼續，不要選「是」取消。
4. **裝完 VMware Tools 後，`CreateScreenshot_Task()` 截圖可能整張變全黑**，即使 VM 本身完全正常（`toolsRunningStatus=guestToolsRunning`、有 IP）。這是 Tools 換了 SVGA 3D 顯示驅動後，ESXi 這個舊式截圧 API 讀不到新驅動的畫面緩衝區，**不代表 VM 掛掉或黑屏當機**——之後要確認狀態改用 Guest Operations API（見下一節），不要再依賴截圖判斷。

## VMware Tools 裝好後改用 Guest Operations API（比截圖+盲打鍵盤可靠很多）

```python
auth = vim.vm.guest.NamePasswordAuthentication(username='<guest_user>', password='<guest_pass>')
pm = si.content.guestOperationsManager.processManager
fm = si.content.guestOperationsManager.fileManager

spec = vim.vm.guest.ProcessManager.ProgramSpec()
spec.programPath = 'C:\\Windows\\System32\\cmd.exe'
spec.arguments = '/c whoami > C:\\out.txt 2>&1'
pid = pm.StartProgramInGuest(vm, auth, spec)

# 輪詢結果
procs = pm.ListProcessesInGuest(vm, auth, [pid])
exit_code = procs[0].exitCode  # None 代表還沒結束

# 抓檔案內容（把裡面的輸出讀回來）
info = fm.InitiateFileTransferFromGuest(vm, auth, 'C:\\out.txt')
# info.url 裡的主機名是 '*'，要換成 ESXi 真實 IP 才能用 requests 下載：
url = info.url.replace('*', '<esxi_ip>')
```

寫檔案（例如寫 unattend.xml）沒有簡單的單步 API，繞過 `InitiateFileTransferToGuest` 多步驟流程的實用做法：把內容 base64 編碼後，用一條 PowerShell 指令在 guest 端解碼寫檔（`[IO.File]::WriteAllBytes(...)`）。

**已知坑**：
- `ListProcessesInGuest` 不帶 pid 參數可以列出「全部」目前追蹤到的行程（包含系統行程），可以用來確認某個指令是否真的還在跑、或找出卡死的殭屍行程。
- Guest Ops 偶爾會短暫回 `GuestOperationsUnavailable`（`The guest operations agent could not be contacted`），常見於系統正在重開機/服務正在重啟，通常十幾秒到幾分鐘會恢復，不代表永久失敗，但也可能代表卡住了——要交叉比對 `vm.guest.hostName` 有沒有變化來判斷。
- `TerminateProcessInGuest` 對別的帳號啟動的行程（例如系統本身/其他 session）會報 `GuestPermissionDenied`，只能砍自己認證帳號啟動的行程。

## 關於這段經驗的執行环境註記

以上 sysprep 自動化卡住的部分，是由 **Codex**（透過 pi coding agent harness 執行）嘗試並最終沒做完。現場查到的 `PI_MODEL` 環境變量顯示 `claude-sonnet-5`，但實際上執行的是 Codex（可能經過 OmniRoute 等路由層轉發，環境變量未必反映真實模型）。**這代表上面記錄的卡住/失敗是這次執行者的實作結果，不代表其他模型（如 Claude）或其他自動化方法一定也做不到**——接手繼續做的人/AI 不要因為看到這段就预設這條路行不通，可以尝試不同模型或直接改用 vCenter/PowerCLI Customization Specification 路徑。

## Sysprep 已知地雷（親測 Windows 11 24H2/25H2 中文版）

1. **最常見卡點**：`Microsoft.LanguageExperiencePackzh-TW` 這類語言套件是「只裝給目前使用者」而非「所有使用者佈建」，sysprep `/generalize` 驗證階段會直接失敗：
   ```text
   SYSPRP Package Microsoft.LanguageExperiencePackzh-TW_... was installed for a user, but not provisioned for all users.
   SYSPRP Failed to remove apps for the current user: 0x80073cf2.
   ```
   這個錯誤幾乎是**瞬間發生**（log 時間戳只差 0 秒），**跟卡住不動是兩回事**，一定要去看 `C:\Windows\System32\Sysprep\Panther\setupact.log` 的 tail 才看得出來，光靠 `hostName`/`toolsRunningStatus` 完全看不出這個錯誤。
   修法：`Get-AppxPackage -AllUsers *LanguageExperiencePack* | Remove-AppxPackage -Package $_.PackageFullName -AllUsers` 移除後再重跑。
2. **驗證失敗後 sysprep.exe 會卡死變殭屍行程，不會自己結束**：因為失敗後它想彈一個訊息框，但用 `StartProgramInGuest` 這種非互動方式啟動的行程沒有辦法把 UI 顯示到互動桌面上，導致行程永遠卡在等待畫面渲染，`exitCode` 永遠是 `None`。這會誤導人以為「還在跑、快好了」，其實已經死了。判斷方式：`ListProcessesInGuest` 全列表裡看到 `sysprep.exe`，且對照 `setupact.log` 的時間戳早就不再更新——就是卡死了，要用 `TerminateProcessInGuest` 主動砍掉再重跑，不要傻等。
3. 修完 Appx 問題重跑前，記得先把卡死的 `sysprep.exe` 和它的 `cmd.exe` 父行程都砍乾淨，否則新的 sysprep 可能因為全域 mutex 被舊行程佔住而立刻失敗或悄悄不動作。
4. Sysprep 過程中 Guest Ops 斷線（`GuestOperationsUnavailable`）是正常現象（服務/網路重新設定），但如果斷線超過 3-5 分鐘、`hostName` 還是沒變，代表沒有真的在跑 generalize+reboot，通常又是上面第 2 點的殭屍行程問題。
5. 這整套「螢幕截圖＋盲打鍵盤＋輪詢 Guest Ops」自動化 sysprep 流程非常耗時間、容易卡在肉眼看不出來的地方（尤其黑屏之後完全瞎），**如果時間有限，更建議用 vCenter/PowerCLI 的 Customization Specification 做 sysprep**，或至少準備好「隨時能砍掉重練」的心理準備，不要對著同一個卡住的行程死等。

## 升級 Domain Controller / 讓 client 加入網域

### 升級成第一台 DC（新建 Forest）

前置：需要跟使用者確認 FQDN 網域名稱（例：`lab.local`）、要不要改用固定
的靜態 IP（DC 不能用 DHCP IP）。這兩件事**不要自己決定**，一定要問。

1. 先改靜態 IP（用 Guest Operations 跑 PowerShell，記得先查現有
   gateway/prefix 再改，改完 DNS 先指向自己 + 備援用原本的 gateway）：
   ```powershell
   Remove-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress "<舊IP>" -Confirm:$false
   Set-NetIPInterface -InterfaceAlias "Ethernet0" -Dhcp Disabled
   New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress "<新靜態IP>" -PrefixLength 24 -DefaultGateway "<gateway>"
   Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses ("<新靜態IP>","<gateway>")
   ```
2. 產生一組 DSRM（目錄服務還原模式）密碼（本機用 `secrets` 產生，不要用
   使用者既有密碼），跟使用者回報一次，不要寫進 skill/repo。
3. 角色安裝 + 升級 Forest，用 **SYSTEM 排程工作**觸發（見下方「重開機後
   卡黑屏」章節說明為什麼一定要走這條，不要直接 `StartProgramInGuest`
   跑會觸發重開機的指令）：
   ```powershell
   Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
   Import-Module ADDSDeployment
   $secPwd = ConvertTo-SecureString "<DSRM密碼>" -AsPlainText -Force
   Install-ADDSForest -DomainName "<FQDN>" -DomainNetbiosName "<NETBIOS>" `
       -InstallDns -SafeModeAdministratorPassword $secPwd -Force `
       -NoRebootOnCompletion:$false -ErrorAction Stop
   ```
4. 驗證（`Install-ADDSForest` 完成後會自動重開機，等 Tools 回來）：
   ```powershell
   Get-Service NTDS,DNS,Netlogon,ADWS,DFSR   # 全部應該是 Running
   Get-ADDomain | Format-List NetBIOSName,DNSRoot,DomainMode
   Get-ADDomainController -Filter * | Format-List Name,HostName,IPv4Address,OperationMasterRoles
   Get-DnsServerZone   # 應該看到 <FQDN> 跟 _msdcs.<FQDN> 等標準區域
   ```
5. 升級完成後 DC 本機的 `Administrator` 帳號自動變成網域的
   `<NETBIOS>\Administrator`，密碼延用原本的本機密碼，不用另外設定。

**已知坑**：`ProgramSpec` 的 `envVariables` 參數如果有帶值，會**整個取代**
guest 行程的環境變數而不是合併，導致 `cmd.exe` 連 PATH 都沒有、報
`'powershell.exe' is not recognized`。**不要用 `envVariables` 傳密碼這類
敏感值**，改成在 Python 端用字串 `.replace()` 把密碼直接內嵌進要送進
guest 的 PowerShell 腳本內容裡（寫進 guest 端一個暫存 `.ps1` 檔案，執行完
可以再清掉），跟本 skill 其他章節「用 base64 EncodedCommand 包 PowerShell」
的模式一致。

### 讓 client VM 加入網域

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses ("<DC的IP>","<原本的gateway>")
$secPwd = ConvertTo-SecureString "<domain admin密碼>" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("<NETBIOS>\Administrator", $secPwd)
Add-Computer -DomainName "<FQDN>" -Credential $cred -Restart -Force
```

**DNS 一定要先指向 DC 才能加入**，不然 `Add-Computer` 會因為找不到網域
（無法解析 SRV 記錄）直接失敗。跟升 DC 一樣，用 SYSTEM 排程工作觸發（會
自動重開機）。驗證：

```powershell
$cs = Get-WmiObject Win32_ComputerSystem
"Domain=$($cs.Domain)"; "PartOfDomain=$($cs.PartOfDomain)"
```

或直接看 `vm.guest.hostName` 有沒有變成 `<電腦名稱>.<FQDN>`。

## 用 OU + GPO 管理 client（不只是加入網域而已）

單純 `Add-Computer` 加入網域，client 只會受 AD 自動產生的 Default Domain
Policy 影響（密碼原則），**不會有任何額外管控**。要真的「管」client，要:

### 1. 建 OU、把電腦移進去

GPO 只能連結到 **OU / 網域根 / Site**，不能連到預設的 `CN=Computers`
容器（`Add-Computer` 預設把電腦丟在這裡）。流程：

```powershell
New-ADOrganizationalUnit -Name "Workstations" -Path "DC=lab,DC=local" -ProtectedFromAccidentalDeletion $true
foreach ($name in @("WIN11-1","WIN11-2","WIN11-3")) {
    $comp = Get-ADComputer -Identity $name
    Move-ADObject -Identity $comp.DistinguishedName -TargetPath "OU=Workstations,DC=lab,DC=local"
}
```

`ProtectedFromAccidentalDeletion` 開著，之後如果真的要刪這個 OU，得先
`Set-ADOrganizationalUnit -Identity <OU> -ProtectedFromAccidentalDeletion $false`
才刪得掉。

### 2. 建 GPO、連結到 OU、寫入登錄檔式設定

```powershell
New-GPO -Name "Workstations - Test Policy" -Comment "說明文字"
New-GPLink -Name "Workstations - Test Policy" -Target "OU=Workstations,DC=lab,DC=local" -LinkEnabled Yes

# 寫入一個 Computer Configuration 的登錄檔設定（HKLM 開頭 = 電腦設定，
# HKCU 開頭 = 使用者設定）
Set-GPRegistryValue -Name "Workstations - Test Policy" `
    -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System" `
    -ValueName "LegalNoticeCaption" -Type String -Value "文字"
```

`Set-GPRegistryValue` 每呼叫一次，`(Get-GPO -Name ...).ComputerVersion`
會 +1，可以用這個數字確認設定真的有寫進去。

### 3. 驗證要在 client 端主動 `gpupdate /force`，不要傻等背景排程

Windows 預設背景自動套用 GPO 的間隔是**每 90 分鐘 + 0~30 分鐘隨機延遲**
（DC 本身是每 5 分鐘）。要立即驗證改動有沒有生效，直接在 client 上跑：

```powershell
gpupdate /force /target:computer
gpresult /r /scope:computer | Select-String "Applied Group Policy Objects|<GPO名稱>"
```

要把背景自動套用間隔改快（例如接近即時），一樣用 `Set-GPRegistryValue`
寫進同一個 GPO：

```powershell
Set-GPRegistryValue -Name $gpoName -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" -ValueName "GroupPolicyRefreshTime" -Type DWord -Value 0
Set-GPRegistryValue -Name $gpoName -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" -ValueName "GroupPolicyRefreshTimeOffset" -Type DWord -Value 0
```

`GroupPolicyRefreshTime=0` 分鐘 → 大約每 7 秒檢查一次（微軟官方行為，不是
筆誤）。⚠️ 這對小型測試環境沒問題，機器一多會增加 DC 負擔，正式環境不要
照抄，記得跟使用者說清楚這個取捨。

### 4. 已知 GPO 設定內容（`Workstations - Test Policy`，2026-08-15）

| 設定 | Key/Value | 效果 |
|---|---|---|
| 登入橫幅 | `HKLM\...\Policies\System\LegalNoticeCaption`/`LegalNoticeText` | 登入前彈出公告 |
| 允許 RDP | `HKLM\System\CurrentControlSet\Control\Terminal Server\fDenyTSConnections=0` | 允許遠端桌面連線 |
| RDP 防火牆開通 | `HKLM\SOFTWARE\Policies\Microsoft\WindowsFirewall\DomainProfile\GloballyOpenPorts\List\3389:TCP` = `3389:TCP:*:enabled:Remote Desktop` | 這是舊式（legacy）Windows Firewall GPO 開 port 的寫法，不會出現在 `Get-NetFirewallRule -DisplayGroup "Remote Desktop"` 這種新式規則清單裡，**要驗證真的開了，直接從外部對 3389 做 TCP port scan，不要只看 `Get-NetFirewallRule`** |
| 背景更新間隔 | `GroupPolicyRefreshTime=0`、`GroupPolicyRefreshTimeOffset=0` | ~7 秒自動套用 |
| 開機自動裝 Node.js | Computer Startup Script（見下方） | 見下方獨立小節 |

### 5. GPO 開機/登入指令碼（Startup/Logon Script）—— 沒有現成 cmdlet，要手動組三件事

`GroupPolicy` 模組**沒有** `Set-GPStartupScript` 這種 cmdlet。要手動做三件事，
**少做任何一件，腳本放對位置也不會被觸發、而且不會報任何錯誤**（最容易
踩雷的地方）：

```powershell
$gpo = Get-GPO -Name "Workstations - Test Policy"
$guid = $gpo.Id.ToString("B").ToUpper()   # 轉成 {GUID} 格式
$sysvolBase = "\\lab.local\SysVol\lab.local\Policies\$guid\Machine\Scripts"
$startupDir = "$sysvolBase\Startup"

# 1) 腳本本體放到 SYSVOL
New-Item -ItemType Directory -Path $startupDir -Force | Out-Null
Set-Content -Path "$startupDir\MyScript.ps1" -Value $scriptContent -Encoding UTF8

# 2) 寫 psscripts.ini 登記這個腳本（PowerShell 腳本用 psscripts.ini，
#    傳統 .bat/.vbs 用 scripts.ini，兩者格式一樣，檔名不同）
$iniContent = "[Startup]`r`n0CmdLine=MyScript.ps1`r`n0Parameters=`r`n"
Set-Content -Path "$sysvolBase\psscripts.ini" -Value $iniContent -Encoding Unicode   # 要 Unicode(UTF-16LE) 編碼

# 3)（最容易漏的一步）在 GPO 的 AD 物件上註冊 Scripts 這個
#    client-side extension（CSE）的 GUID pair，不然 GPO 引擎根本不知道
#    要去讀 psscripts.ini
$gpoDN = "CN=$guid,CN=Policies,CN=System,DC=lab,DC=local"
$scriptsCSE = "[{42B5FAAE-6536-11D2-AE5A-0000F87571E3}{40B6664F-4972-11D1-A7CA-0000F87571E3}]"
$existing = (Get-ADObject -Identity $gpoDN -Properties gPCMachineExtensionNames).gPCMachineExtensionNames
if ($existing -notlike "*42B5FAAE-6536-11D2-AE5A-0000F87571E3*") {
    Set-ADObject -Identity $gpoDN -Replace @{gPCMachineExtensionNames="$existing$scriptsCSE"}
}
```

**`gPCMachineExtensionNames` 是用 `[{GUID1}{GUID2}][{GUID3}{GUID4}]...`
這種一組一組方括號串接的格式，一定要用「讀出現有值再字串附加」，不要
直接覆蓋整個值**——這個屬性同時記錄了 Registry CSE（`Set-GPRegistryValue`
自己會維護的那組）跟 Scripts CSE，覆蓋掉會讓其他設定失效。

**驗證方式（不要只看檔案有沒有寫進去，要實際重開機驗證有跑）**：對一台
client 跑 `vm.ResetVM_Task()`，等它開機回來，去讀腳本自己寫的 log 檔
（例：`C:\Windows\Temp\xxx_log.txt`），確認真的執行過。已知案例：這樣建的
「開機自動裝 Node.js LTS」腳本（先查 `C:\Program Files\nodejs\node.exe`
存不存在，不存在才下載官方 MSI 用 `msiexec /quiet` 裝），三台各自重開機
後都在 30~40 秒內完成下載+安裝，`node -v` 回報 `v24.19.0`，全程無需人工
介入。

#### 同一個 GPO 掛第二支開機腳本（多腳本註冊格式 + 未解的「第二支不觸發」bug）

`psscripts.ini` 的 `[Startup]` 區段可以用數字前綴掛多支腳本，格式是
`0CmdLine=`/`0Parameters=`、`1CmdLine=`/`1Parameters=` 依序往下編號：

```ini
[Startup]
0CmdLine=NodeInstall.ps1
0Parameters=
1CmdLine=SshSetup.ps1
1Parameters=
```

⚠️ **這個格式本身是對的**（第一支 `NodeInstall.ps1` 掛成 `0`、後來加的
第二支 `SshSetup.ps1` 掛成 `1`，這樣寫沒有語法錯誤），但親測**只有 `0`
號腳本在實際重開機時被觸發（`node_install_log.txt` 時間戳跟重開機時間
吻合），`1` 號腳本完全沒有執行過的痕跡（`ssh_setup_log.txt` 從未出現，
連腳本開頭寫的第一行 `START` log 都沒有，代表根本沒被啟動，不是卡住）**。

**根本原因（2026-08-15 已確認並修好）**：Windows Group Policy 的 Scripts
CSE（client-side extension）在 client 端會把「這個 GPO 要跑哪些開機腳本」
快取進登錄檔：

```text
HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Startup\<GPO序號>\<腳本序號>
```

親測診斷時查到 `Startup\0` 下面**只有一個 `0` 子機碼**（對應
`NodeInstall.ps1`），完全沒有代表 `SshSetup.ps1` 的 `1` 子機碼——即使
SYSVOL 上的 `psscripts.ini` 內容明明已經正確寫了 `0CmdLine=`/`1CmdLine=`
兩行。**這個快取只有在 GPO 版本號真的變動時才會重新掃描 `psscripts.ini`
重建**。我們是直接用 `Set-Content` 改 SYSVOL 上的 `psscripts.ini` 檔案，
沒有透過任何會讓 `gpt.ini` 的 `[General] Version=` 往上跳、AD 上
`gPCMachineVersionNumber` 也同步跳號的正規 GPO 動作（`New-GPO`／
`Set-GPRegistryValue`／`New-GPLink` 這些 cmdlet 才會觸發版本號遞增），
所以 client 端一直以為「這個 GPO 沒變過」，continues 用舊的、只有一支
腳本的快取，永遠不會注意到多了第二支。

**修法**：在 DC 上對同一個 GPO 隨便重跑一次 `Set-GPRegistryValue`（哪怕
是覆寫一個已經設過的既有值，只要有呼叫這個 cmdlet 就會讓
`ComputerVersion`／`gpt.ini` 的 `Version=` +1），逼 client 端下次
`gpupdate`/開機處理時偵測到版本變了、強制重新讀取並重建 Scripts 快取：

```powershell
Import-Module GroupPolicy
Set-GPRegistryValue -Name "Workstations - Test Policy" `
    -Key "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" `
    -ValueName "GroupPolicyRefreshTime" -Type DWord -Value 0 | Out-Null
```

親測（2026-08-15，WIN11-2）：`gpt.ini` 的 `Version=` 從 `10` 跳到 `11`
後，重開機一次，`ssh_setup_log.txt` 就正常出現、`sshd` 服務變成
`Running`、TCP 22 開始監聽（IPv4+IPv6 都有）。**結論：以後只要用「直接
改 SYSVOL 檔案」的方式幫同一個既有 GPO 追加/修改開機腳本，事後一定要
補一次任意的 `Set-GPRegistryValue`（或其他會動到 GPO 版本號的正規操作）
才會讓 client 真的重新讀取新的 `psscripts.ini`，光改檔案本身不會生效**，
這比「多腳本註冊格式有問題」或「Get-WindowsCapability 卡住」更接近真正
的根因——後兩個問題在這次案例裡也同時存在，但即使先解決了它們，沒補
版本號跳動這一步，腳本還是不會被觸發。

## VM 重開機後卡黑屏 / Tools 不回應（親測會反覆發生，有明確處理順序）

這批 Windows 11 client VM（尤其剛做完 sysprep/改名/加入網域，觸發過
`-Restart` 的那次重開機）**很容易在重開機後卡在黑屏、`toolsRunningStatus`
一直是 `guestToolsNotRunning`**，不是偶發一次，同一台可能連續發生好幾次。

### 判斷是不是真的卡死（不要一看到 Tools 掉線就緊張）

1. 先查 `vm.runtime.bootTime` 有沒有變 —— 如果沒變，代表根本沒重開機，
   可能是延遲很久之前觸發的操作終於跑完（見上面 WIN11-1 sysprep 延遲
   完成的案例），過陣子自己會回來，不用馬上動手。
2. 如果 bootTime 有變（真的重開機了）但 Tools 一直不回來，用截圖 + CPU
   用量交叉確認是不是真卡死：
   ```python
   print(vm.summary.quickStats.overallCpuUsage, 'MHz')  # 個位數~十幾 MHz 且截圖全黑 = 真的卡死
   ```
   截圖端點：`https://<esxi>/screen?id=<vm._moId>`（Basic Auth root 帳密），
   螢幕全黑 + CPU 個位數 MHz + 這個狀態持續超過 1-2 分鐘沒變化，才算真的
   卡住，不要截一張黑圖就急著動手，重開機過程本來就會經過幾秒黑屏。
3. 想看更底層的證據，直接抓 ESXi 端的 `vmware.log`（不需要 Guest Ops，
   VM 掛了也讀得到）：
   ```bash
   curl -s -k -u root:<esxi密碼> "https://<esxi>/folder/<VM資料夾名>/vmware.log?dsName=datastore1"
   ```
   注意 VM 資料夾名可能跟目前的 VM 顯示名稱不一樣（改名只改 ESXi
   inventory name，不會動磁碟上的資料夾，例：VM 叫 `WIN11-1` 但資料夾
   還是 `WIN11`）。log 裡如果只剩下每分鐘固定跳出的
   `USB: New set of 1 USB devices` 訊息、完全沒有其他開機進度紀錄，就是
   真的卡死的鐵證（那個 USB 訊息只是背景輪詢，不代表 OS 有在動）。

### 修復順序：先軟後硬，不要一次就上重手

1. **`vm.ResetVM_Task()`**（軟重置，等同虛擬主機上按下 reset 鍵）——大多
   數情況（WIN11-2、WIN11-3 每次都是）這樣就會正常回來。
2. 如果 reset 後還是同樣的黑屏+CPU 個位數 MHz 模式（WIN11-1 這台親測發生
   過，reset 沒用），才升級成**完全斷電再開機**：
   ```python
   task = vm.PowerOffVM_Task()   # 等 task 完成，power state 變 poweredOff
   time.sleep(5)
   task = vm.PowerOnVM_Task()
   ```
   這比 `ResetVM_Task` 更徹底（會重跑完整 BIOS POST，不只是重置
   CPU/裝置狀態），實測解決了 `ResetVM_Task` 解不了的卡死。
3. 兩者都做完動作前**先跟使用者確認**要不要動手（這是會中斷 guest 內部
   任何未儲存操作的動作），除非使用者已經明確表示「這台反正卡死了，
   幫我修」。

### 觸發「會導致重開機」的操作時，一律走 SYSTEM 排程工作，不要直接跑

`Rename-Computer -Restart`、`Add-Computer -Restart`、
`Install-ADDSForest`（自動重開機）這類指令，一律照本 skill 其他章節示範
的「寫一個 `.ps1` 到 guest 暫存目錄 → 註冊 SYSTEM 權限、`Highest` RunLevel
的 `ScheduledTask` → `Start-ScheduledTask`」模式包起來執行，不要用
`StartProgramInGuest` 直接跑：
- `hch`（或其他一般 Administrator 群組帳號）透過 Guest Ops 拿到的是 UAC
  過濾後的標準權杖，不是完整管理員權限，這幾個指令需要真正的
  elevated/SYSTEM 權限才能可靠執行。
- 排程工作模式順便解決了「這個指令本身會觸發重開機、導致
  `StartProgramInGuest` 呼叫端連線中斷」的問題——工作是背景獨立跑的，
  重開機不會影響它有沒有執行成功，之後可以再連進去讀寫在暫存檔的
  log（例如 `C:\Windows\Temp\xxx_log.txt`）確認結果。

## 關閉睡眠/休眠（避免「喚不起」）

Windows 11 client 預設會依電源配置逾時進入待命/睡眠，VM 環境下「喚醒」不
一定可靠，容易跟上面那個「重開機後卡黑屏」的症狀混在一起分不清楚是哪個
原因。直接用 `powercfg` 關掉，比透過 GPO 電源管理設定（那些設定要對到一
堆 Power Scheme GUID，錯了不會報錯只是悄悄沒生效）可靠：

```powershell
powercfg /change standby-timeout-ac 0
powercfg /change standby-timeout-dc 0
powercfg /change monitor-timeout-ac 0
powercfg /change monitor-timeout-dc 0
powercfg /change hibernate-timeout-ac 0
powercfg /change hibernate-timeout-dc 0
powercfg /hibernate off
```

驗證用 `powercfg /query SCHEME_CURRENT SUB_SLEEP STANDBYIDLE`，看
「目前的 AC/DC 電源設定索引」是不是 `0x00000000`。**這台環境是中文
語系，`powercfg` 輸出是中文，不要用英文字串（`Current AC` 之類）做
`Select-String` 比對，會抓不到任何東西**（指令本身有跑成功，只是驗證
腳本語言比對失敗，容易誤判成沒生效）。這個設定是直接寫進系統電源配置，
立即生效、不用重開機、不會因重開機被還原。

## 讓 client 只能用網域帳號登入（停用本機帳號）

流程：① 在 DC 建網域使用者 ② 加進每台 client 的本機 Administrators 群組
③ 停用每台 client 的本機帳號。**本機內建 `Administrator` 帳號本身通常已
經是停用狀態（預設），不用特別去動它，留著當意外情況的救援管道**（網域
`Domain Admins` 群組本來就會自動變成每台網域成員的本機管理員，這是 AD
標準行為，不用手動加）。

```powershell
# 在 DC 上
Import-Module ActiveDirectory
$secPwd = ConvertTo-SecureString "<密碼>" -AsPlainText -Force
New-ADUser -Name "hch" -SamAccountName "hch" -UserPrincipalName "hch@lab.local" `
    -AccountPassword $secPwd -Enabled $true -PasswordNeverExpires $true

# 在每台 client 上（domain admin 帳密執行）
Add-LocalGroupMember -Group "Administrators" -Member "LAB\hch"
Disable-LocalUser -Name "hch"    # 停用同名的本機帳號
```

**⚠️ 致命陷阱，親身踩過**：如果原本是用本機 `hch`/`<SSH_PASSWORD>` 這組帳密透過
Guest Ops 連線，然後在同一個 PowerShell 腳本裡執行
`Disable-LocalUser -Name "hch"`，**這個指令會停用你自己正在用來認證
Guest Ops 的那個帳號**。`StartProgramInGuest` 本身因為用的是啟動當下還
有效的帳密所以會成功發動，但後續的 `ListProcessesInGuest`／
`InitiateFileTransferFromGuest` 輪詢呼叫，每一次都要重新驗證帳密——帳號
已經被停用，馬上收到
`vim.fault.InvalidGuestLogin: Failed to authenticate with the guest
operating system using the supplied credentials`，把自己鎖在 Guest Ops
外面，讀不到腳本執行結果（但腳本本身在 guest 端已經跑完了，只是拿不到
回傳輸出）。**解法：改用另一組沒被這次操作動到的帳密（例如
`LAB\Administrator`）重新連線確認結果**，不要以為連不上代表操作失敗了。
以後只要腳本內容涉及「停用/砍掉自己正在用的認證帳號」，一律要用**另一
組帳密**去發起這次呼叫，不要自己砍自己腳下的梯子。

**密碼原則放寬（使用者明確要求「密碼就是要跟以前本機帳號一樣簡單」時）**：

```powershell
Set-ADDefaultDomainPasswordPolicy -Identity "lab.local" -ComplexityEnabled $false -MinPasswordLength 4
```

這會放寬整個網域的預設密碼原則（不只是單一帳號），之後所有網域帳號的
密碼強度要求都會變低——這是全域性的取捨，要跟使用者說清楚再動手，不要
自己默默做掉。

## 改變預設輸入法（IME）

輸入法預設值是**使用者層級（HKCU）**設定，不是電腦層級。如果 client 平常
是用**本機帳號**登入（不是網域帳號），透過 GPO 的「使用者設定」
（`HKCU\...`）幾乎不會生效——GPO 的 User Configuration 預設只套用到「網域
使用者」登入時的 session，對本機帳號登入完全不起作用（除非另外設定
Loopback 處理，讓電腦所在的 OU 直接控制任何登入者的使用者設定，那個更
複雜，這個環境目前沒有用）。**要嘛用該帳號的身份直接跑，要嘛設 Loopback
處理**，不要以為 GPO 寫了 HKCU 設定就一定會套用到本機帳號登入。

實測有效、直接用該帳號身份（Guest Ops 認證用那個帳號的帳密）跑：

```powershell
$list = Get-WinUserLanguageList
$en = $list | Where-Object { $_.LanguageTag -eq 'en-US' }
if ($en) {
    $list.Remove($en) | Out-Null
    $list.Insert(0, $en)          # 搬到第一個 = 變成預設
} else {
    $newEn = New-WinUserLanguageList -Language 'en-US'
    $list.Insert(0, $newEn[0])
}
Set-WinUserLanguageList -LanguageList $list -Force
Set-WinDefaultInputMethodOverride -InputTip "0409:00000409"   # 英文(美國)-US 鍵盤
```

用 `Get-WinUserLanguageList` 查驗，`LanguageTag` 清單裡第一個就是登入後
的預設語言/輸入法，順序後面才是 `zh-Hant-TW` 之類的其他語言。改完會有
「若 Windows 顯示語言已變更，下次登入才會生效」的警告訊息，是正常的，
不代表失敗。

**⚠️ 已踩過的坑：改完之後又「變回中文」，通常是因為換了登入帳號。**
`HKCU` 設定綁的是「當時執行指令那個帳號的使用者設定檔（profile）」。
親身案例：先用**本機** `hch` 帳號把預設輸入法改成英文 → 之後把本機 `hch`
帳號停用、改成用**網域** `LAB\hch` 登入 → 網域帳號第一次登入時 Windows
會建立一份全新的使用者設定檔（跟本機 `hch` 的 profile 完全無關），沿用
系統預設語言（`zh-Hant-TW`），看起來就像「設定失效了」。**解法：認證
帳密要跟著改，用 Guest Ops 連線時要用當下實際登入用的那個帳號身份重跑
一次**（例：本機帳號停用後改用 `LAB\hch` 重跑 `Set-WinUserLanguageList`），
不是同一個指令跑一次就永久對所有未來的登入帳號都有效。

## 改快照名稱（RenameSnapshot）

```python
snap_obj = snapshot_tree_node.snapshot   # vim.vm.Snapshot 物件
snap_obj.RenameSnapshot(name='新名稱')
```

**`RenameSnapshot` 是同步呼叫，不回傳 Task 物件**（回傳 `None`）——不要
對它的回傳值呼叫 `wait_task()`，會得到 `AttributeError: 'NoneType' object
has no attribute 'info'`。呼叫完就已經生效了，直接接著做下一步即可。

## 安全注意

- ESXi 快照建立/刪除/整併、VM 關機/重開機都屬於會改變狀態或造成停機的操作，執行前需向使用者確認。
- 刪除/整併快照會失去回復點，必須取得明確確認。
- 不要把密碼、token、私鑰寫進 skill 或 repo。
- 修 SSH 時若要改防火牆、排程工作、服務啟動類型，要回報實際改動。
- 對 clone VM 執行 `sysprep /generalize` 會重開機並清空 guest 身分（新 SID/hostname），無法復原，執行前需向使用者確認。
- 建立 VMFS datastore（`CreateVmfsDatastore`）會格式化目標磁碟、清掉既有分割與資料，不可逆，執行前需向使用者確認，且建議先跑 `QueryAvailableDisksForVmfs` / `RetrieveDiskPartitionInfo` 確認磁碟真的是空的。
- 升級 DC（`Install-ADDSForest`）、改靜態 IP、讓 client 加入網域
  （`Add-Computer`）都會改變網路身分/重開機，動手前跟使用者確認網域名稱、
  要不要換 IP 這些關鍵參數（不要自己決定），DSRM 密碼產生後要回報給
  使用者、不要只寫在 guest 端的暫存檔裡就沒了。
- VM 卡黑屏後的 `PowerOffVM_Task`+`PowerOnVM_Task`（完全斷電重開）比
  `ResetVM_Task` 更重手，會讓沒儲存的 guest 內部狀態全部消失，一樣要
  先確認過（除非使用者已授權「這台卡死了直接修」）。

---

## Conformance Addendum

## When to Use
用 Python 管理本機網段 ESXi Host Client / pyVmomi / WinRM / SSH / VMware Tools Guest Operations API 維運流程。適用於連線到 ESXi 192.168.2.125、查看/修改 VM 設定、修正 Windows Server Guest OS 類型、檢查與啟用 Windows VM 的 OpenSSH Server、建立/整併/改名快照、查詢 host/datastore 狀態、找出未設定的實體磁碟並建立新 VMFS datastore、在網路完全被防火牆封鎖的 VM 上透過 Guest Operations API 執行程式與讀寫檔案、檢查 clone VM 是否忘了做 sysprep(重複 hostname/SID)、把 Windows Server 升級成 Domain Controller 並讓 client VM 加入網域、用 OU+GPO 管理網域內 client（登入橫幅/RDP/開機腳本/UTF-8/SSH 部署）、用 ISO+虛擬光碟機繞過 SMB 被擋的檔案傳輸、處理 VM 重開機後卡黑屏/Tools 不回應的復原流程。密碼不要寫入 skill，執行時向使用者索取或使用當次 session 已提供的憑證。

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
