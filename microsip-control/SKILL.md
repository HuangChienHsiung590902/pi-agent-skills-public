---
name: microsip-control
description: 用 Python 包裝 MicroSIP（Windows SIP 軟體電話，C:\Users\HCH\AppData\Local\MicroSIP\MicroSIP.exe）內建的命令列自動化介面——撥號、接聽、掛斷、DTMF、轉接。當使用者提到「MicroSIP」「控制軟體電話」「MicroSIP API/wrapper」時使用。注意 PyPI 上的 microsip-api 和 GitHub 上多數「Microsip API」專案都是撞名（墨西哥 ERP 系統 Microsip，跟這個 SIP 軟體電話無關），不要被誤導。
---

# microsip-control

## 背景

MicroSIP 沒有真正的 REST/雙向 API，只有官方內建的**命令列參數**（單向下指令）跟**設定檔事件鉤子**（單向收事件，尚未實作，見下方）。網路上能搜到的「Microsip API」（PyPI `microsip-api`、GitHub `dtremp007/Microsip-API`）幾乎都是同名的墨西哥 ERP 系統「Microsip」的資料庫查詢工具，**不是**這個 SIP 軟體電話，不要誤用。

MicroSIP 是 single-instance 應用程式（`MicroSIP.ini` 裡 `singleMode=1`）：對已經開著的實例再次執行 `MicroSIP.exe <參數>`，指令會被送給那個既有視窗（不會另外開一個新分頁/新視窗），所以底下這些指令隨時呼叫都安全，不會製造重複程序。

## 快速參考

| 項目 | 值 |
|---|---|
| 執行檔路徑 | `C:\Users\HCH\AppData\Local\MicroSIP\MicroSIP.exe` |
| 設定檔（UTF-16 編碼） | `C:\Users\HCH\AppData\Roaming\MicroSIP\MicroSIP.ini` |
| Wrapper | `scripts/microsip.py` |

## 使用方式

當函式庫用：
```python
from microsip import MicroSIP
sip = MicroSIP()
sip.dial("201")
sip.answer()
sip.hangup_all()
sip.dtmf("123")
sip.transfer("202")
```

命令列直接呼叫：
```bash
python scripts/microsip.py status          # running / not running
python scripts/microsip.py dial 201
python scripts/microsip.py answer
python scripts/microsip.py hangup-all
python scripts/microsip.py hangup-incoming
python scripts/microsip.py hangup-calling
python scripts/microsip.py transfer 202
python scripts/microsip.py dtmf 123
python scripts/microsip.py minimize
python scripts/microsip.py exit             # 會關掉使用者的 MicroSIP，小心使用
python scripts/microsip.py reset --yes      # 重設設定，不可逆，小心使用
```

## 已驗證

`status` 指令已測試成功（正確偵測到執行中的 MicroSIP process，PID 7204）。其餘指令（`dial`/`answer`/`hangup-*`/`dtmf`/`transfer`）走同一條 `subprocess.Popen([exe, *args])` 路徑，參數字串直接對照官方文件（`microsip.org/help`），未對使用者當時開著的即時通話/視窗實際觸發，避免干擾正常使用——真的要撥測試電話時，建議先確認 MicroSIP 沒有在講真正的通話。

## 尚未實作：雙向事件（Call-in 方向）

MicroSIP.ini 裡有 `cmdOutgoingCall`/`cmdIncomingCall`/`cmdCallRing`/`cmdCallAnswer`/`cmdCallBusy`/`cmdCallStart`/`cmdCallEnd` 這幾個欄位，通話狀態變化時會自動執行外部程式（可傳入來電號碼參數），目前全是空字串 `""`，尚未設定。若要做「來電自動彈窗」這類完整 CTI 面板，需要：
1. 把這些欄位指向一支小程式（例如寫入一個 named pipe / 打一個 localhost HTTP endpoint）
2. **注意 `.ini` 是 UTF-16LE 編碼**，直接用純文字工具寫入容易寫壞檔案，Python 要用 `open(path, encoding="utf-16")` 讀寫

這部分目前只是規劃記錄，還沒有實作/測試過，之後要做再回來補。

---

## Conformance Addendum

## When to Use
用 Python 包裝 MicroSIP（Windows SIP 軟體電話，C:\Users\HCH\AppData\Local\MicroSIP\MicroSIP.exe）內建的命令列自動化介面——撥號、接聽、掛斷、DTMF、轉接。當使用者提到「MicroSIP」「控制軟體電話」「MicroSIP API/wrapper」時使用。注意 PyPI 上的 microsip-api 和 GitHub 上多數「Microsip API」專案都是撞名（墨西哥 ERP 系統 Microsip，跟這個 SIP 軟體電話無關），不要被誤導。

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
