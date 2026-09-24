---
name: "esxi-vmware-python-control"
description: "用 Python (pyVmomi/WinRM/SSH/Guest Ops API) 連線並管理本機網段 ESXi 主機 192.168.2.125（VM、快照、datastore、Guest OS 操作等）"
version: 1
created: "2026-08-15"
updated: "2026-08-15"
---
## When to Use
當使用者要求連線/操作/查詢 ESXi 主機 192.168.2.125（VMware ESXi 8.0.3）或其上的 VM（SRV/WIN11-1/WIN11-2/WIN11-3）時使用，包括：查看或修改 VM 設定、開關機、快照管理、datastore/磁碟管理、Guest OS 內部操作（WinRM/SSH/Guest Operations API）、sysprep、網域相關操作。

## Procedure
1. 完整、最新的操作細節與踩坑記錄都在 D:\.system\.claude\skills\esxi-python-control\SKILL.md（1300+ 行，含固定環境資訊、Python 套件安裝方式、連線範例程式碼、快照管理、Thick→Thin轉換、Sysprep、Domain Controller/GPO 等完整章節），操作前務必先讀取該檔案取得最新細節，不要只憑這裡的摘要行動。
2. 基本連線骨架（pyVmomi）：SmartConnect(host='192.168.2.125', user=<ESXI_USER>, pwd=<ESXI_PASS>, sslContext=ssl._create_unverified_context())，用 viewManager.CreateContainerView 搭配 vim.VirtualMachine 列出所有 VM。
3. 已知環境：ESXi Host 名稱 Lenovo，datastore1(~348.8GB)/datastore2(~465.5GB)；VM 固定 IP：SRV(DC)=192.168.2.200、WIN11-1=.201、WIN11-2=.202、WIN11-3=.203，網域 lab.local(NetBIOS LAB)。
4. 密碼一律不寫入 skill/程式碼，執行時向使用者索取或使用當次 session 已提供的憑證。
5. Python 套件（pyvmomi/pywinrm/paramiko）用 pip install --target 裝到 %TEMP% 暫存目錄再用 PYTHONPATH 指定，避免污染全域環境（詳見原始 SKILL.md 的『Python 套件臨時安裝方式』章節，含 git-bash 下 PYTHONPATH 分號路徑轉換的坑）。

## Pitfalls
- standalone ESXi（無 vCenter）的 RelocateVM_Task/CloneVM_Task 會被授權擋掉，需改用 SSH + vmkfstools 或手動 CreateVM_Task 組裝。
- VM 開機狀態下不能改 guestId 或新增 CD-ROM 裝置，會報 InvalidPowerState，需先關機。
- 沒有 VMware Tools 時只能靠 CreateScreenshot_Task+PutUsbScanCodes 盲操作；裝好 Tools 後改用 Guest Operations API 更可靠（截圖裝完 Tools 後可能全黑，不代表當機）。
- 詳細踩坑清單（sysprep 地雷、robocopy 唯讀屬性、ARP cache RDP連不上等）務必查閱原始 SKILL.md，此處僅摘要不完整。

## Verification
1. python -c "import pyVim, pyVmomi" 或用 pip install --target 暫存目錄後 PYTHONPATH 執行不報 ModuleNotFoundError
2. SmartConnect 連線成功後 si.RetrieveContent().about.fullName 應回傳 'VMware ESXi 8.0.3 build-24022510'
3. 列出的 VM 清單應包含 SRV/WIN11-1/WIN11-2/WIN11-3 且電源狀態符合預期

---

## Conformance Addendum

## Inputs and Outputs
- **Input:** the user request, relevant paths/configuration, and any current error or runtime evidence.
- **Output:** the requested result plus a concise record of actual changes and verification evidence.

## Rules and Limitations
- Resolve relative paths from this Skill directory, not from an unspecified working directory.
- Treat recorded paths, versions, hosts, and UI details as potentially stale; current system evidence takes precedence.
- Do not expose credentials or perform destructive changes without explicit authorization.
