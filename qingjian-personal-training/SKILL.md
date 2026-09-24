---
name: qingjian-personal-training
description: 當使用者要看清箋／Qingjian Windows 輸入法已累積多少可訓練文字、開始個人訓練、解釋狀態條「訓練 N」，或部署這個訓練計數與個人模型時使用。涵蓋已確認上屏計數、私密與撤銷資料排除、狀態條右鍵與設定頁開始訓練、personal-lm.tsv，以及未簽名 Server 必須用 QINGJIAN_UIACCESS=0 才能啟動。
---

# 清箋個人訓練資料與開始訓練

## When to Use

使用者提到以下任一事項時載入本 Skill，並同時載入 `qingjian-ime-development`：

- 狀態條、輸入法切換按鈕上的「訓練 N」或可訓練資料數量。
- 如何開始訓練、個人模型、`personal-lm.tsv`、輸入日誌能否拿來訓練。
- 關閉或清空輸入日誌後數量沒有變 0。
- 私密輸入、未上屏組句、英文或原樣直輸是否算進訓練資料。
- 只改了 Server 後輸入法不能用。

不要把這件事做成上傳資料、雲端微調，或改動隨包整句模型。個人 LoRA 仍是 `docs/plan/todo.md` 的未完成項目；目前殼內訓練的是本機字元二元模型。

## Procedure

1. 先確認使用者要的是數量提示、開始訓練，還是部署。預設只處理大千注音路徑。
2. 可訓練數量的定義固定為 `input-log.jsonl` 中仍有效的已確認中文上屏：
   - 計入：`Word`、`Cloud`、`Sentence`、`CloudSentence`，且文字非空。
   - 不計入：按鍵事件、尚未確認的組句、英文、原樣直輸、翻譯、快捷結果。
   - `Retract` 會移除被撤銷的 commit。
   - 私密輸入由 `MutedLogger` 在寫入前丟棄，因此不會進入日誌，也不能被計入。
   - 關閉輸入日誌後狀態條必須顯示 0，不能繼續把舊檔案算進去。
3. 狀態條顯示位置是切換按鈕右側的獨立格子，文字為 `訓練 N`。該格本身不攔截點擊。相關程式：
   - `apps/windows/server/src/dispatch/status/view.rs`
   - `apps/windows/server/src/ui/status/mod.rs`
   - `crates/qingjian-learning/src/input_log.rs`
4. 開始訓練的使用者入口：
   - 懸浮狀態條右鍵「開始個人訓練」。
   - 設定程式「高階」頁的「開始訓練」。
5. 訓練讀取 `%APPDATA%\Qingjian\input-log.jsonl`，寫入 `%APPDATA%\Qingjian\personal-lm.tsv`。段與段之間以 `break` 切開；組句外直打的標點可接在同一段。實作在 `crates/qingjian-learning/src/personal_lm.rs`。
6. Server 約每秒看一次 `personal-lm.tsv`。空模型不改變選字。有資料時，只以 0.15 的權重把個人偏好疊到既有整句重排上，且先扣掉同批平均分，避免蓋過詞庫與隨包模型。實作在 `apps/windows/server/src/dispatch/rescore/personal.rs`。
7. 只改 Server 時，未簽名的快速部署必須關掉 uiAccess，否則 Windows 不會啟動它，輸入法會把按鍵吃掉：

```powershell
Set-Location D:\qingjian-main
$env:QINGJIAN_UIACCESS = '0'
cargo build --release --locked -p qingjian-windows-server -p qingjian-windows-settings
Stop-Process -Name qingjian-server -Force -ErrorAction SilentlyContinue
Start-Sleep -Seconds 2
Copy-Item D:\qingjian-main\target\release\qingjian-server.exe 'C:\Program Files\Qingjian\qingjian-server.exe' -Force
Copy-Item D:\qingjian-main\target\release\qingjian-settings.exe 'C:\Program Files\Qingjian\qingjian-settings.exe' -Force
Start-Process 'C:\Program Files\Qingjian\qingjian-server.exe' -WorkingDirectory 'C:\Program Files\Qingjian'
```

8. 部署後比對兩個 exe 的 SHA-256，確認新 PID，並看 `%LOCALAPPDATA%\Qingjian\logs\server.*.log` 是否出現管道監聽與「個人模型已訓練／已載入」。

## Pitfalls

- 狀態條上的「訓練 N」只是可訓練筆數，不是已經訓練完成。必須由使用者從右鍵或設定頁開始訓練。
- 不要把個人訓練說成 LoRA 或隨包 `.qjm` 已更新。那些權重沒有被改寫。
- `QINGJIAN_UIACCESS` 預設為開。未簽名又帶 uiAccess 的 Server 會完全起不來；TSF 若宿主不是 Medium 完整性，也不會代為拉起。
- 複製 exe 前要先停掉 `qingjian-server` 並等待檔案解鎖。複製失敗後又立刻 `Start-Process`，會把舊程式重新啟動。
- 設定頁按鈕在 `qingjian-settings.exe`。只換 Server 時，右鍵可以訓練，但設定頁不會出現新按鈕。
- 關閉輸入日誌只讓計數歸零，不會刪除既有 `personal-lm.tsv`。若要停用已訓練偏好，需另外處理該檔案，不要擅自刪除使用者資料。

## Verification

1. `cargo test -p qingjian-learning --lib personal_lm` 確認撤銷與原樣直輸不進入訓練，且見過的接續分數較高。
2. `cargo test -p qingjian-windows-server --lib personal` 確認空模型不改分數、訓練後偏好已見文字。
3. `cargo test -p qingjian-windows-server --test engine_loop status_bar` 確認狀態條文字含 `訓練 N`，清空與關閉日誌後為 0。
4. 安裝檔與 `target\release` 的 SHA-256 一致，`qingjian-server` 有新 PID，日誌有 `命名管道監聽中`。
5. 右鍵狀態條可見「開始個人訓練」；設定「高階」頁可見「開始訓練」。訓練後 `%APPDATA%\Qingjian\personal-lm.tsv` 存在，Server 日誌出現已載入或已訓練。
