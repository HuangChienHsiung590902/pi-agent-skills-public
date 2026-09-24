---
name: gcp-comfyui-gpu-deploy
version: 1.0.0
description: 在 GCP 上申請 GPU 配額、建立 GPU VM、安裝並執行 ComfyUI 的完整流程。使用者要求「繼續 GCP ComfyUI 部署」、「檢查配額」、「建立 GPU VM」、「裝 ComfyUI」時使用。
---

# GCP ComfyUI GPU 部署

## ⚡ 目前狀態速覽（2026-08-07 晚間更新）

**目前有一台運行中的 L4 VM**：`comfyui-h3-l4-a`（us-central1-a），是從舊VM（`comfyui-h3-l4`，us-central1-c，GPU缺貨後遷移）重建的。詳見下方新增章節「⚠️ MiniMax H3 在L4 VM上會反覆宕機（2026-08-07診斷）」了解已知的穩定性問題與繞過方式。

- **VM 名稱**：`comfyui-h3-l4-a`，**zone**：`us-central1-a`，機型 `g2-standard-8`（8 vCPU / 32GB RAM）+ `nvidia-l4`（24GB VRAM）
- **公網IP**：會在每次 stop/start 後改變，用 `gcloud compute instances describe comfyui-h3-l4-a --zone=us-central1-a --format="value(networkInterfaces[0].accessConfigs[0].natIP)"` 查詢
- **ZeroTier VPN 內網IP**：`10.145.119.181`（同樣可能隨VM重建而改變，需重新查）
- **⚠️ 重要：這台VM的 SSH連線非常容易在GPU全力運算時斷線甚至整台卡死**，根本原因是RAM不足（見下方新章節），**優先用公網IP + 標準ssh連線**（比ZeroTier VPN路徑更穩，但仍不是100%可靠解方）
- **這台VM上已裝好 MiniMax H3 模型**（Docker容器 `comfyui-h3`，image `comfyui-h3:local`，模型放在 `/home/HCH/ComfyUI/models/`，用 `docker run -v /home/HCH/ComfyUI:/home/HCH/ComfyUI` 掛載，容器有 `--restart unless-stopped`，VM重啟後容器會自動起來但里面GPU任務不會恢復，需重新提交）
- **舊的 A100 配額申請已遭拒**（2026-08-06申請，同日拒絕，要求編號 `f1d8236f043742df82`，區域`us-west4`），GCP對Free Trial帳號基本不核准A100，這是已知限制
- **Google Drive 備份**（帳號 `hch590902.gemini@gmail.com`）：`GCP-ComfyUI-Backup/` 資料夾仍保留舊VM（`comfyui-h3-l4`/`comfyui-gpu`系列）的61.2GB備份，可作為救援，但目前使用中的 `comfyui-h3-l4-a` 是直接沿用磁碟快照遷移過來的，模型已經在磁碟上不需要再從Drive下載

## 目的

延續先前已完成的 GCP 環境設定，在配額核准後接續：建立 GPU VM → 安裝 ComfyUI → 啟動服務 → 之後管理開關機省錢。

## 環境現況（已完成的部分，不用重做）

- **gcloud CLI**：已安裝於 Windows，路徑：
  `C:\Users\HCH\AppData\Local\Google\Cloud SDK\google-cloud-sdk\bin`
  在 bash 中每次要用需先：
  ```bash
  export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
  ```
- **GCP 帳號**：`0tmd0544@gmail.com`
- **Project ID**：`project-fbcea0c4-05a8-4472-b9d`（已設為 gcloud 預設 project）
- **Billing**：已連結，`billingEnabled: true`，帳戶 ID `019062-F4FEB4-30EE89`
- **免費試用額度**：約 $9,713（到期日 2026-11-05）
- **已啟用 API**：`compute.googleapis.com`
- **區域配額狀況**（`us-central1`）：T4/L4/V100/P4/P100/K80 各有 1 顆的區域配額；A100 是 0（若要用A100需另外申請）
- **卡關點1（已解決）**：全域總量配額 `GPUs (all regions)` 原本是 0，已於 2026-08-06 送出提升申請
  - 服務單編號：`#88c91698b0784339b7`
  - **已於同日核准**，`effectiveLimit` 變為 1（核准速度比預估的2個工作天快很多）
- **卡關點2（進行中，2026-08-06 發現）**：配額核准後，實際建立VM時遇到 `ZONE_RESOURCE_POOL_EXHAUSTED` / `ZONE_RESOURCE_POOL_EXHAUSTED_WITH_DETAILS`
  - 已測試過的組合全部缺貨：us-central1-a/b/c、us-east1-b/c/d、us-west1-a、us-west4-a、us-east4-a、asia-east1-a、asia-southeast1-a、asia-northeast1-a
  - 測試過的GPU型號：nvidia-l4、nvidia-tesla-t4
  - **這是 Google Free Trial 帳號的已知限制**：即使配額核准，Free Trial 帳號在實際硬體調度上的優先度較低，容易被擋。這跟帳號設定無關，是 Google 端容量分配問題。
  - 解決方向見下方「GPU缺貨排解」章節
- **gcloud-mcp**：已加入 `~/.pi/agent/mcp.json`（需要重開 pi session 才會生效載入）
- **Playwright MCP**：設定為連接外部 Chrome debug port 9222，若連不上需要：
  ```bash
  taskkill //F //IM chrome.exe
  "/c/Program Files/Google/Chrome/Application/chrome.exe" --remote-debugging-port=9222 --user-data-dir="C:\Users\HCH\chrome-debug-profile" &
  disown
  ```
  然後要重新登入 Google 帳號（新 profile 沒有登入狀態）。

## 步驟 1：檢查配額是否已核准

先檢查信箱有沒有 Google Cloud Support 的核准通知，或直接用 CLI 查詢目前配額值：

```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
gcloud alpha services quota list \
  --service=compute.googleapis.com \
  --consumer=projects/project-fbcea0c4-05a8-4472-b9d \
  --filter="metric:compute.googleapis.com/gpus_all_regions"
```

若 `effectiveLimit` 或配額值變成 1（而非 0），代表已核准，可以繼續下一步。

## 步驟 2：建立 GPU VM

確認 `us-central1-a` 有 L4 GPU 可用（先前已確認過），直接執行：

```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"

gcloud compute instances create comfyui-gpu \
  --project=project-fbcea0c4-05a8-4472-b9d \
  --zone=us-central1-a \
  --machine-type=g2-standard-8 \
  --accelerator="type=nvidia-l4,count=1" \
  --image-family=common-cu129-ubuntu-2204-nvidia-580 \
  --image-project=deeplearning-platform-release \
  --maintenance-policy=TERMINATE \
  --boot-disk-size=150GB \
  --boot-disk-type=pd-balanced \
  --metadata="install-nvidia-driver=True" \
  --tags=comfyui
```

若遇到 `Could not fetch resource: Quota 'GPUS_ALL_REGIONS' exceeded` 錯誤，代表配額還沒核准，需繼續等待。

若這個機型/GPU組合在 us-central1-a 缺貨（stockout），可以改嘗試 `us-central1-b` 或 `us-central1-c`（先前查詢過這三個zone都有L4）。

## 步驟 3：開防火牆規則，允許連 8188 port

```bash
gcloud compute firewall-rules create allow-comfyui \
  --allow=tcp:8188 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=comfyui \
  --project=project-fbcea0c4-05a8-4472-b9d
```

⚠️ 正式使用建議把 `--source-ranges` 改成自己的固定 IP，或改用 SSH Tunnel 更安全：
```bash
gcloud compute ssh comfyui-gpu --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d -- -L 8188:localhost:8188
```

## 步驟 4：SSH 進去安裝 ComfyUI

```bash
gcloud compute ssh comfyui-gpu --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d
```

進去後執行：

```bash
# 確認 GPU 驅動狀態
nvidia-smi

# 安裝 ComfyUI
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI

python3 -m venv venv
source venv/bin/activate

pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
pip install -r requirements.txt

# 啟動，監聽所有 IP 才能從外部連
python main.py --listen 0.0.0.0 --port 8188
```

建議背景執行並記錄 log，方便關掉終端機後還在跑：
```bash
nohup python main.py --listen 0.0.0.0 --port 8188 > comfyui.log 2>&1 &
```

## 步驟 5：取得外部 IP 並連線

```bash
gcloud compute instances describe comfyui-gpu \
  --zone=us-central1-a \
  --project=project-fbcea0c4-05a8-4472-b9d \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

瀏覽器開啟：`http://<外部IP>:8188`

## 步驟 6：日常開關機（省錢的關鍵）

```bash
# 用完馬上關閉（GPU計費停止，磁碟資料保留，儲存費很低）
gcloud compute instances stop comfyui-gpu --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d

# 下次要用
gcloud compute instances start comfyui-gpu --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d
```

⚠️ 千萬別忘記關機，GPU持續開著計費很快，養成用完立刻Stop的習慣。

若確定長期不用，才用 `delete` 徹底刪除（連磁碟儲存費都不用付，但下次要用要重新安裝）：
```bash
gcloud compute instances delete comfyui-gpu --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d
```

## ✅ 已成功建立VM（2026-08-06）

- **VM 名稱**：`comfyui-gpu`
- **區域**：`us-west4-b`（當us-central1系列全部缺貨時，經重試至第11個區域才成功）
- **機型**：`n1-standard-8` + `nvidia-tesla-t4` x1
- **外部IP**：`34.125.217.98`（注意：重新啟動後 IP 可能改変，重新查詢：
  ```bash
  gcloud compute instances describe comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
  ```
  ）
- 之後所有 `--zone` 參數都要用 `us-west4-b`，不是一開始計劃的 `us-central1-a`

## ⚠️ MiniMax-H3 需求變更（2026-08-06）

使用者要求安装 minimax-h3-comfyui 工作流（詳見 `minimax-h3-comfyui` skill）。確認現有 `comfyui-gpu`（T4，15GB VRAM，29GB RAM）**硬體不夠**：
- MiniMax-H3 需要雙卡方案（參考另一台伺服器雙RTX 4060Ti 16GB，54GB RAM）才能分層卸載避免OOM，單卡T4連CLIP權重(15.7GB)都塞不進去
- **決定改用大顯存單卡（A100）**，不用MultiGPU分片方案，直接用原生Loader節點，更简単也更快

### 已執行的前提作業
1. **已 Stop（不是刪除）現有 T4 VM**（`comfyui-gpu`，`us-west4-b`），保留資料以後可以回來用：
   ```bash
   gcloud compute instances stop comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d
   ```
2. **確認 A100 區域配額均為0**（us-central1、us-west4、asia-east1都是0），必需網頁申請。CLI直接調整會回 `COMMON_QUOTA_CONSUMER_OVERRIDE_TOO_HIGH`（同 GPUS_ALL_REGIONS 一樣遭限）
3. **已於 2026-08-06 送出 A100 配額提升申請，但已於同日遭拒（2026-08-07確認）**：
   - **要求編號**：`f1d8236f043742df82`（於 GCP Console「配額」→「提高要求」分頁確認，非之前記錄的 `b30e29c1-9d21-4cd0-a6c4-3b7fe8baf27b`）
   - **區域**：`us-west4`（與現有T4 VM相同區域，方便之後重用防火牆/網路設定）
   - **配額項目**：`NVIDIA A100 GPUs`（注意不是 `Committed NVIDIA A100 GPUs`，那是不同機制）
   - **從 0 → 1**
   - **狀態：已遭拒**（提交與回應時間都是 2026/8/6 下午3:10，幾乎是自動即時拒絕，頁面沒有列出拒絕原因）
   - 結論：Free Trial 帳號基本上無法核准 A100 等高階GPU配額，這是 Google 端已知限制，不用再等待審核

### A100路線已死路，下一步請從下方三個備案擇一

1. **升級為完整付費帳號**（GCP Console 首頁有「啟用完整帳號」按鈕），升級後不會馬上扣款，$9,713免費額度依然有效，可能提高A100核准機率，升級後可重新走「提高要求」流程再申請一次
2. **放棄A100，改用T4/L4方案**：復原已停機的 `comfyui-gpu`（T4×1），搭配 `minimax-h3-comfyui` skill 中的 ComfyUI-MultiGPU 分層卸載方案（已預先安裝好備用）
3. **改用其他雲端GPU平台**（RunPod / Vast.ai），這些平台的A100/H100等高階卡可直接租用，不受GCP Free Trial限制

（以下A100建VM步驟保留供日後帳號升級或配額核准後參考，目前不可執行）

查詢是否核准（若走升級帳號路線，重新申請後可用此指令確認）：
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
gcloud compute regions describe us-west4 --project=project-fbcea0c4-05a8-4472-b9d --format="json(quotas)" | grep -B2 -A2 '"metric": "NVIDIA_A100_GPUS"'
```
⚠️ 不要用 `gcloud alpha services quota list`，實測發現其 `overrideValue` 欄位具誤導性（`-1`不代表核准），上面這個 `gcloud compute regions describe` 才是可靠的真實核准值來源。

核准後建立VM（A100一般搭配 `a2-highgpu-1g` 機型，40GB顯存，支持CUDA旧版本需訽確認image-family相容）：
```bash
gcloud compute instances create comfyui-h3-gpu \
  --project=project-fbcea0c4-05a8-4472-b9d \
  --zone=us-west4-a \
  --machine-type=a2-highgpu-1g \
  --accelerator="type=nvidia-tesla-a100,count=1" \
  --image-family=common-cu129-ubuntu-2204-nvidia-580 \
  --image-project=deeplearning-platform-release \
  --maintenance-policy=TERMINATE \
  --boot-disk-size=250GB \
  --boot-disk-type=pd-balanced \
  --metadata="install-nvidia-driver=True" \
  --tags=comfyui-h3
```
注意 `a2-highgpu-1g` 只有特定區域/zone有庫存，若缺貨參考下方「GPU缺貨排解」的重試方式，並把 asia-east1-a/b/c 也加入重試清單（見下方。）。

建立完 VM 後，直接参考 `minimax-h3-comfyui` skill 裡「雲端部署建議SOP」章節的「若顯存夠大」路徑：不用ComfyUI-MultiGPU，直接用原生 `UNETLoader`/`CLIPLoader`/`VAELoader` 節點。

## GPU缺貨排解（Free Trial帳號常見）

若遇到 `ZONE_RESOURCE_POOL_EXHAUSTED` 或 `ZONE_RESOURCE_POOL_EXHAUSTED_WITH_DETAILS`，代表配額沒問題，但實際硬體被占滿。可能解決方式：

1. **不斷重試（最簡单，但需要耐心）**：庫存會動態變化，過一段時間重試可能就成功。可寬一個輪詢 script：
   ```bash
   export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
   ZONES=(us-central1-a us-central1-b us-central1-c us-central1-f us-east1-b us-east1-c us-east1-d us-west1-a us-west1-b us-west4-a us-west4-b us-east4-a us-east4-b asia-east1-a asia-east1-b asia-east1-c asia-southeast1-a asia-southeast1-b)
   # asia-east1-a/b/c 是臺灣彰化資料中心，離使用者最近、網路延遲最低，若要建一般（非 MiniMax-H3）ComfyUI VM 可优先嘗試這裡（已確認T4/L4區域配額都是1，同us-central1）
   for z in "${ZONES[@]}"; do
     echo "=== 嘗試 $z ==="
     gcloud compute instances create comfyui-gpu \
       --project=project-fbcea0c4-05a8-4472-b9d \
       --zone=$z \
       --machine-type=n1-standard-8 \
       --accelerator="type=nvidia-tesla-t4,count=1" \
       --image-family=common-cu129-ubuntu-2204-nvidia-580 \
       --image-project=deeplearning-platform-release \
       --maintenance-policy=TERMINATE \
       --boot-disk-size=150GB \
       --boot-disk-type=pd-balanced \
       --metadata="install-nvidia-driver=True" \
       --tags=comfyui 2>&1 | tail -5
     if [ $? -eq 0 ]; then echo "成功！區域：$z"; break; fi
     sleep 5
   done
   ```
   建議間隔幾小時後再執行一次，或用 cron/已登入session每小時跑一次。

2. **改用 Spot VM（可能庫存比較寬鬆，但会被隨時回收，不適合需要常駐的場合）**：
   ```bash
   gcloud compute instances create comfyui-gpu \
     --project=project-fbcea0c4-05a8-4472-b9d \
     --zone=us-central1-a \
     --machine-type=g2-standard-8 \
     --accelerator="type=nvidia-l4,count=1" \
     --image-family=common-cu129-ubuntu-2204-nvidia-580 \
     --image-project=deeplearning-platform-release \
     --provisioning-model=SPOT \
     --instance-termination-action=STOP \
     --boot-disk-size=150GB \
     --boot-disk-type=pd-balanced \
     --metadata="install-nvidia-driver=True" \
     --tags=comfyui
   ```

3. **升級為完整帳號（最有效，但需要主動操作）**：
   GCP Console 首頁有「啟用完整帳號」按鈕（先前截圖看到過），升級後不再是 Free Trial，可能會提高 GPU 実體調度的優先度。不会马上扭錢，免費額度依然有效。

4. **改用其他雲端平台**：若 GCP 持續無法取得GPU，可考慮回到之前討論過的 RunPod / Vast.ai，那邊專門做GPU出租，庫存比較不會被Free Trial限制影響。

## ✅ ComfyUI 已安装並運行中（2026-08-06）

- **安装位置**：`~/ComfyUI`（venv在 `~/ComfyUI/venv`）
- **啟動方式**：背景執行 `nohup python main.py --listen 0.0.0.0 --port 8188 > ~/comfyui.log 2>&1 &`
- **連線網址（已改用ZeroTier VPN，見下方新章節）**：`http://10.145.119.9:8188`
- **已需修復的小問題**：影像預裝 `python3-venv` 不完整，需先 `sudo apt install -y python3-venv` 才能建 venv

## ✅ 已改用 ZeroTier VPN連線（不再公開暴露 port，2026-08-06）

因為公開开放8188 port觸發了 Google Cloud 的敏感操作安全通知信，改用 ZeroTier 建立VPN，不再對公開網路暴露 port。

- **ZeroTier Network ID**：`08752e18b15e0bce`（網路名稱 `AI3`，使用者既有網路，非新建）
- **VM上的 ZeroTier 裝置 ID**：`26c169aca6`
- **VM 取得的 ZeroTier IP**：`10.145.119.9`（固定在該網段，不隨公網IP改変而改変，比公網IP更穩定）
- **已删除公開防火牆規則** `allow-comfyui`（原本允訿 `0.0.0.0/0` 任何人連 8188）

### 安装/加入網路的指令（日後如果重裝或新建其他VM可參考）
```bash
# 安装 ZeroTier
gcloud compute ssh comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d --command="curl -s https://install.zerotier.com | sudo bash"

# 加入網路
gcloud compute ssh comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d --command="sudo zerotier-cli join 08752e18b15e0bce"

# 加入後需到 https://my.zerotier.com/network/08752e18b15e0bce 手動授權新裝置（Members列表勾選Auth），才会從ACCESS_DENIED轉為OK並分配IP

# 確認IP已分配
gcloud compute ssh comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d --command="sudo zerotier-cli listnetworks"
```

使用者自己的電腦也必需已加入且被授權到同一個 ZeroTier 網路（`08752e18b15e0bce`）才能相互連通。

### 日常操控指令

重新連上握SSH檢查/重啟ComfyUI：
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
gcloud compute ssh comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d --command="tail -30 ~/comfyui.log"
```

重啟ComfyUI服務（若當掉或VM重啟後）：
```bash
gcloud compute ssh comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d --command="cd ~/ComfyUI && source venv/bin/activate && nohup python main.py --listen 0.0.0.0 --port 8188 > ~/comfyui.log 2>&1 & sleep 5; tail -10 ~/comfyui.log"
```
⚠️ **重要陷坑**：不要先執行 `pkill -f 'python main.py'` 再啟動！`pkill` 找不到結果時回傳非0 exit code，會導致 `gcloud compute ssh` 整個命令回報 exit code 128、且表面上看起來像是失敗並無任何輸出（實際上後面的命令可能已成功執行，只是 gcloud 回報這個錯誤碼讓人以為失敗）。若確定服務没在跑、想先清將舊進程，改用 `pgrep`（回傳值正常）確認沒有舊進程在跑，或直接用 `ss -tlnp | grep 8188` 確認port是否已有人占用，再決定要不要重啟。

重新查詢外部IP（重啟VM後可能改変，但現在主要用ZeroTier IP `10.145.119.9` 不受重啟影響，這個主要備接見公網IP用）：
```bash
gcloud compute instances describe comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

### 日常開關機省錢（重要！）
```bash
# 用完關機
 gcloud compute instances stop comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d

# 下次要用开機（開完需重新執行上面「重啟ComfyUI服務」指令，因為stop不会保留背景程式）
gcloud compute instances start comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d
```

## ✅ 直接 SSH 連線方式（不透過 gcloud ssh，2026-08-06）

除了 `gcloud compute ssh` 之外，已確認可以直接用一般 SSH client + 金鑰連線，透過 ZeroTier IP，速度更快、不受 gcloud 那個 `pkill` exit code 128 的問題影響。

- **金鑰位置（OpenSSH格式，給 ssh/WinSCP用）**：`C:\Users\HCH\.ssh\google_compute_engine`
- **金鑰位置（PuTTY格式.ppk，給 WinSCP/PuTTY用）**：`C:\Users\HCH\.ssh\google_compute_engine.ppk`
- **使用者名稱**：`HCH`（注意大小寫，Linux帳號區分大小寫，WinSCP有時會自動轉小寫要手動修正）
- **VM上已有對應公鑰**在 `~/.ssh/authorized_keys`（gcloud compute ssh 第一次執行時自動建立並上傳）

### Git Bash / WSL 直接連線指令
```bash
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@10.145.119.9
```

### PowerShell 連線指令（注意路徑格式不同，不能用 `/c/...` 這種MSYS路徑）
```powershell
ssh -i C:\Users\HCH\.ssh\google_compute_engine HCH@10.145.119.9
```

### 金鑰權限問題（Windows OpenSSH 對權限較嚴格）
若PowerShell出現 `Permission denied (publickey)` 或 `not accessible`，先修正金鑰檔權限，只留自己帳號可存取：
```powershell
icacls C:\Users\HCH\.ssh\google_compute_engine /inheritance:r
icacls C:\Users\HCH\.ssh\google_compute_engine /grant:r 'HCH:F'
```

### WinSCP 連線設定
- 檔案協定：SFTP
- 主機名稱：`10.145.119.9`（需先連上ZeroTier網路）
- 連接埠：22
- 使用者名稱：`HCH`（**務必確認是大寫**，WinSCP偶爾會自動變小寫導致連線被拒）
- 進階 → SSH → Authentication → 私鑰檔案：選擇 `C:\Users\HCH\.ssh\google_compute_engine.ppk`

## ✅ VM 時區已設為台灣時間（2026-08-06）
```bash
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@10.145.119.9 "sudo timedatectl set-timezone Asia/Taipei"
```
目前VM時區：`Asia/Taipei`（CST, UTC+8），log時間戳都已是台灣時間。

## ✅ 硬碟已擴充到300GB（2026-08-06，為了裝MiniMax-H3模型）

原本150GB不夠裝MiniMax-H3所有模型檔案，已擴充磁碟並在VM內同步擴展檔案系統：
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
# 1. GCP端擴充磁碟大小
gcloud compute disks resize comfyui-gpu --zone=us-west4-b --project=project-fbcea0c4-05a8-4472-b9d --size=300GB --quiet

# 2. VM內部擴展檔案系統（disk變大後檔案系統不會自動變大，必須手動做這步）
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@10.145.119.9 "sudo growpart /dev/sda 1 && sudo resize2fs /dev/sda1"
```
目前：291GB可用空間（df -h /），MiniMax-H3模型檔案裝完後還剩約209GB。

## ✅ MiniMax-H3 已在 T4 VM 上完成「安裝」（但硬體不足無法執行，2026-08-06）

使用者確認「可以直接安裝，跑不動先不執行」，已完整安裝好，等A100配額核准後把這些檔案複製過去或在新VM上重新安裝即可（也可以考慮直接把整個磁碟image化後在新VM沿用）。

### 已完成項目
1. **ComfyUI-MultiGPU custom node 已安裝**（雖然最終決定用A100不需要MultiGPU分片，但已裝起來備用）：
   ```bash
   ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@10.145.119.9 "cd ~/ComfyUI/custom_nodes && git clone https://github.com/pollockjj/ComfyUI-MultiGPU.git"
   ```
   確認已成功載入的方式：重啟ComfyUI後查看log應該要出現 `[MultiGPU] Registration complete...` 等字樣。

2. **5個必要模型檔案已下載完成**（共約60GB，注意：不要用 `hf download Comfy-Org/MiniMax-H3` 不指定檔名，那樣會把整個倉庫近200GB都抓下來，要精確指定檔案清單）：
   ```bash
   ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@10.145.119.9 "cd ~/ComfyUI && source venv/bin/activate && hf download Comfy-Org/MiniMax-H3 diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors vae/minimax_h3_video_vae_fp16.safetensors vae/minimax_h3_audio_vae_fp32.safetensors --local-dir /tmp/h3_download"
   ```
   下載完後搬到正確位置：
   ```bash
   ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@10.145.119.9 "
   mv /tmp/h3_download/diffusion_models/*.safetensors ~/ComfyUI/models/diffusion_models/
   mv /tmp/h3_download/text_encoders/*.safetensors ~/ComfyUI/models/text_encoders/
   mv /tmp/h3_download/vae/*.safetensors ~/ComfyUI/models/vae/
   rm -rf /tmp/h3_download
   "
   ```

### 已就位的檔案清單
| 目錄 | 檔案 | 大小 |
|---|---|---|
| `diffusion_models/` | `minimax_h3_fl2va_pruned_int8_convrot.safetensors` | 20G |
| `diffusion_models/` | `minimax_h3_ref2va_pruned_int8_convrot.safetensors` | 20G |
| `text_encoders/` | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | 15G |
| `vae/` | `minimax_h3_video_vae_fp16.safetensors` | 4.9G |
| `vae/` | `minimax_h3_audio_vae_fp32.safetensors` | 578M |

### 尚未執行的部分
- 尚未在 ComfyUI 網頁介面上實際建立/測試 MiniMax-H3 workflow（因T4只有15GB VRAM不夠跑，需等A100）
- 尚未用原生 UNETLoader/CLIPLoader/VAELoader 節點連好workflow（等A100 VM建立後再進行，詳見 `minimax-h3-comfyui` skill）

## ✅ A100 配額狀態（已確認結果，2026-08-07）

**不再需要持續追蹤/等待審核**，已在 GCP Console「配額」→「提高要求」記錄頁直接確認實際狀態：**要求編號 `f1d8236f043742df82`（區域 `us-west4`）已於 2026/8/6 下午3:10 送出並於同一分鐘內遭拒**，頁面未提供拒絕原因。

若日後需要重新確認配額狀態（例如升級完整帳號後重新申請），有兩種方式，**建議優先用方式2（網頁「提高要求」記錄頁）**，比CLI更能直接看到「已遭拒/已核准/審核中」的明確狀態，而CLI這種方式只能看到最終limit數字，看不到是否已遭拒：

**方式1（CLI查limit數字，適合快速確認是否已核准）**：
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
for region in us-west4 us-central1 asia-east1 us-west1 us-east1 us-east4 europe-west4; do
  gcloud compute regions describe $region --project=project-fbcea0c4-05a8-4472-b9d --format="json(quotas)" > /tmp/${region}-quota.json 2>&1
  echo "=== $region ==="
  grep -B2 -A2 '"metric": "NVIDIA_A100_GPUS"' /tmp/${region}-quota.json
done
```
注意：不要用 `grep -A1 ... | grep limit` 這種寫法，JSON區塊內 `limit` 欄位順序不一定在metric下一行，會抜不到正確值（實測確認過），改用 `-B2 -A2` 包含前後文再用結果中的 `limit` 行比對。

**方式2（網頁確認真實審核狀態，包括是否遭拒）**：前往 https://console.cloud.google.com/iam-admin/quotas/qirs?project=project-fbcea0c4-05a8-4472-b9d （「配額」頁面「提高要求」分頁），可直接看到每筆申請的狀態欄位（已遭拒/已核准/審核中）。

## 常見問題排查

### Playwright MCP 連不上（ECONNREFUSED 127.0.0.1:9222）
表示外部 Chrome debug 沒開，重啟：
```bash
taskkill //F //IM chrome.exe
"/c/Program Files/Google/Chrome/Application/chrome.exe" --remote-debugging-port=9222 --user-data-dir="C:\Users\HCH\chrome-debug-profile" &
disown
```
之後可能要重新登入 Google 帳號（新開的 profile 是空的，會跳出QR code驗證挑戰，需要手動操作手機或選擇其他登入方式，這步驟AI無法自動化，需要使用者本人配合）。

### 查詢 GCP 配額狀態優先用 CLI，不要依賴 Playwright UI（2026-08-06經驗）

Playwright 連線非常不穩定，常發生 `Frame has been detached`、`Target page, context or browser has been closed` 等錯誤，需要不斷重啟Chrome、重新登入，非常耗時。**查詢配額是否核准優先用下面這個CLI指令**，比UI更快、更可靠：
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
gcloud compute regions describe <region> --project=project-fbcea0c4-05a8-4472-b9d --format="json(quotas)" | grep -B2 -A2 '"metric": "NVIDIA_A100_GPUS"'
```
只有當這個CLI查不到或需要看審核進度細節時，才需要回到網頁UI。

### 篩選 GCP 配額頁面時篩不出結果（完整驗證過的流程，2026-08-06）

GCP配額頁面的篩選UI很不直覺，且playwright對它的click常性進入timeout（可能因為floating overlay擋住）。步驟：

1. 用 getByRole('combobox', {name: '篩選條件'}) 定位輸入框，必要時加 click({force:true}) 才能點擊成功（直接click經常timeout）
2. 點擊後會彈出屬性選單（服務/名稱/類型/值...）。直接用 page.keyboard.type() 打字也可以，不一定要先點屬性——系統會自動切到「關鍵字」模式全欄搜尋，結果會包含名稱/類型包含該字串的所有項目（例如搜 NVIDIA A100 GPUs 會同時包含普通版/Committed版/Preemptible版）
3. 不要試圖從下拉清單點擊選項來精確篩選（很容易click失敗或選錯），改用：直接按 ArrowDown 即可套用全文搜尋篩選，雖不精確但能縮小結果範圍，後面再用表格內文字過濾
4. 篩後結果可能還混在一大堆裡，改用網頁內 page.locator('[role=row]').all() 迭代每一行 innerText()，用字串包含/排除條件找到真正想要的那一行（例如 t.includes('us-west4') 且 t.includes('NVIDIA A100 GPUs') 且不包含'Committed'）
5. 頁面預設只顯示50行，想要的行可能在後面頁：直接將每頁顯示數改到200（左下角有個 mat-select 選器），比按下一頁更可靠
6. 找到目標行後，勾選checkbox的正確元素不是 input[type=checkbox]，是 mat-pseudo-checkbox，要先 scrollIntoViewIfNeeded() 再 click({force:true})
7. 勾選後頁面上方會出現「已選取1項配額」+「編輯配額」按鈕，點擊後彈出側邊面板，依序填「新值」、「要求說明」（至少幾句英文描述用途）、「完成」、「下一步」、「提交要求」，這一連串按鈕都可以直接用文字匹配找到，比前面篩選步驟穩定很多
3. 用 `pressSequentially`（逐字輸入，非一次性fill）打關鍵字，才能觸發清單篩選
4. 從彈出的清單中點擊正確選項（不要直接按 Enter，因為輸入框跟清單值可能不同步）

### CLI 直接調配額失敗
`gcloud alpha services quota update` 對這類全域安全限制配額（如 GPUS_ALL_REGIONS）通常會回 `COMMON_QUOTA_CONSUMER_OVERRIDE_TOO_HIGH` 錯誤，必須走網頁「提高要求」流程，無法用CLI繞過。

## ✅ 改用 Docker 化執行 ComfyUI（2026-08-06）

原本裸機直接跑 `python main.py` 的方式，已改為用 Docker 容器執行，優點是VM重啟後容器會自動重啟，不用手動SSH進去重新起 process。

### 關鍵設計決定（重要，避免踩坑）
**Docker image 不包含ComfyUI本身與模型**，只包含系統環境（Ubuntu 22.04 + Python 3.10 + ffmpeg等）。ComfyUI程式碼、venv、所有模型檔案都是透過 volume mount 方式持續存在host上，容器只是包住執行環境。好處：
- 重建image不需重新下載60GB+模型
- 方便備份分離處理（系統環境vs資料）

### ❗ 必須知道的踩坑：venv路徑必須與host一致
現有 venv 是用 `python3 -m venv` 建立，venv內的 `activate` script 會將絕對路徑（例如 `/home/HCH/ComfyUI/venv/bin`）寫死進 `$PATH`。若容器內掛載路徑與host不一致（例如掛到 `/workspace/ComfyUI`），activate會執行成功但 `$PATH` 裡的路徑指向不存在的位置，導致 `python: command not found`。

**解法**：容器內掛載路徑要與host完全一致：
```bash
docker run -d --name comfyui \
  --gpus all \
  --restart unless-stopped \
  -p 8188:8188 \
  -v /home/HCH/ComfyUI:/home/HCH/ComfyUI \
  -w /home/HCH/ComfyUI \
  comfyui:latest \
  bash -c 'source venv/bin/activate && python main.py --listen 0.0.0.0 --port 8188'
```

### Dockerfile（完整內容，可直接重用）
```dockerfile
FROM nvidia/cuda:12.1.0-cudnn8-runtime-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3.10 python3.10-venv python3-pip git wget curl ffmpeg libgl1 libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace

# ComfyUI 原始碼與模型會透過 volume mount 進來，不 COPY 進 image，
# 這樣重建 image 不用重新下載/複製 60GB+ 的模型檔案。

EXPOSE 8188

CMD ["bash", "-c", "cd /workspace/ComfyUI && source venv/bin/activate && python main.py --listen 0.0.0.0 --port 8188"]
```
⚠️ 注意上面Dockerfile裡的CMD用了 `/workspace/ComfyUI`路徑，但實際 `docker run` 時改用host相同路徑掛載並用 `-w` + 自定義命令覆蓋了CMD（見上面run指令）。日後若重建Dockerfile，建議直接把WORKDIR改成與host一致的路徑（例如 `/home/HCH/ComfyUI`）避免困擾。

### 安裝 Docker + nvidia-container-toolkit（新VM需重新執行）
```bash
# Deep Learning VM image 通常已內建 nvidia-container-toolkit，只缺Docker本身
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "
curl -fsSL https://get.docker.com -o /tmp/get-docker.sh
sudo sh /tmp/get-docker.sh
sudo usermod -aG docker HCH
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
"

# 驗證GPU可用
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "sudo docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi"
```

### 常用管理指令
```bash
# 查看log
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "sudo docker logs -f comfyui"

# 重啟容器
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "sudo docker restart comfyui"

# 停止容器
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "sudo docker stop comfyui"
```

## ✅ VM已徹底刪除並完整備份至 Google Drive（2026-08-06）

使用者確認所有資料已安全備份後，決定徹底刪除VM以停止所有計費（不同於Stop，Delete連磁碟都不在，不再有任何持續費用）。

### 備份使用工具：rclone
- Google Drive 帳號：`hch590902.gemini@gmail.com`
- rclone 設定檔：`~/.config/rclone/rclone.conf`（VM上）及 `C:\Users\HCH\rclone\rclone.conf`（本機備份一份，日後重新授權可直接上傳到新VM不用重新OAuth）
- **rclone日後在新VM上重裝時**，直接把 `C:\Users\HCH\rclone\rclone.conf`（已包含OAuth token）scp上去即可，不用重新登入授權：
  ```bash
  scp -i /c/Users/HCH/.ssh/google_compute_engine /c/Users/HCH/rclone/rclone.conf HCH@<VM_IP>:~/rclone.conf
  ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "mkdir -p ~/.config/rclone && mv ~/rclone.conf ~/.config/rclone/rclone.conf"
  ```
  若token過期，重新執行本機 `rclone.exe config reconnect gdrive: --config /c/Users/HCH/rclone/rclone.conf` 即可重新授權。

### ❗ 重要教訓：不要備份 venv（會慢到惱人）
第一次嘗試用 `rclone copy ~/ComfyUI gdrive:...` 整個後發現速度慢到離譜（ETA顯示數週），原因是 `venv` 資料夾內有 **53,580個小檔案**（Python套件），Google Drive API逐一上傳每個檔案都要一次API請求，非常慢。而 models 資料夾只有41個檔案（大部分是60GB的大檔），實測速度可達 **60-200 MiB/s**。

**結論**：venv不需要備份，日後用 `pip install -r requirements.txt` 重建即可，只需備份 `models/`（模型）、`custom_nodes/`（外掛節點，雖然這次實際上這個確認還未單獨備份，日後用git clone重裝即可）、核心程式碼（git管理，可直接git clone重建不用備份）。

### 實際執行的備份指令
```bash
# VM上安裝 rclone
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "curl -s https://rclone.org/install.sh | sudo bash"

# 上傳模型（排除venv，只備份models資料夾）
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "rclone --config ~/.config/rclone/rclone.conf copy /home/HCH/ComfyUI/models gdrive:GCP-ComfyUI-Backup/ComfyUI/models --progress --transfers 8 --drive-chunk-size 128M"

# 上傳Docker image（先在VM上docker save壓縮，再用rclone上傳，比從本機上傳快很多）
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "sudo docker save comfyui:latest | gzip > /home/HCH/comfyui-image.tar.gz"
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "rclone --config ~/.config/rclone/rclone.conf copy /home/HCH/comfyui-image.tar.gz gdrive:GCP-ComfyUI-Backup/ --progress"
```

### 備份完成後的 Google Drive 內容（確認過，總計61.2GB）
```
gdrive:GCP-ComfyUI-Backup/
├── comfyui-image.tar.gz          (2.13GB, Docker系統環境)
└── ComfyUI/
    ├── models/
    │   ├── diffusion_models/minimax_h3_fl2va_pruned_int8_convrot.safetensors  (20GB)
    │   ├── diffusion_models/minimax_h3_ref2va_pruned_int8_convrot.safetensors  (20GB)
    │   ├── text_encoders/qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors          (15GB)
    │   ├── vae/minimax_h3_video_vae_fp16.safetensors                           (4.9GB)
    │   └── vae/minimax_h3_audio_vae_fp32.safetensors                           (578MB)
    └── （其他ComfyUI根目錄下的程式碼檔案，在第一次嘗試全量上傳時已部分上傳，但custom_nodes目錄未確認完整性，建議日後重建時直接git clone ComfyUI-MultiGPU重裝更可靠）
```

### 如何從備份重建VM（下次接續的完整流程）
1. 建新VM（參考前面「建立 GPU VM」章節，A100核准後用 `a2-highgpu-1g` + `nvidia-tesla-a100`）
2. 安裝 Docker + nvidia-container-toolkit（上方步驟）
3. 安裝rclone、把 `C:\Users\HCH\rclone\rclone.conf` scp上傳新VM從日後重新授權
4. 從Google Drive下載回來：
   ```bash
   rclone --config ~/.config/rclone/rclone.conf copy gdrive:GCP-ComfyUI-Backup/ComfyUI/models /home/HCH/ComfyUI/models --progress
   rclone --config ~/.config/rclone/rclone.conf copy gdrive:GCP-ComfyUI-Backup/comfyui-image.tar.gz /home/HCH/
   ```
5. 還原 Docker image：`docker load < ~/comfyui-image.tar.gz`
6. 重新 `git clone` ComfyUI 本體 + ComfyUI-MultiGPU custom node（參考文件前面「步驟4：SSH 進去安裝 ComfyUI」章節）
7. 重建 venv 並安裝依賴（`python3 -m venv venv && source venv/bin/activate && pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121 && pip install -r requirements.txt`）
8. 把下載回來的 `models/` 放回 `~/ComfyUI/models/`
9. 重新加入 ZeroTier 網路：`sudo zerotier-cli join 08752e18b15e0bce`（需到網頁手動授權新裝置）
10. 啟動 Docker 容器（參考上方「Docker化執行ComfyUI」章節的 `docker run` 指令）

## MiniMax H3 在 L4 VM 上會反覆宕機（2026-08-07 診斷與解法，重要！）

### 症狀
跑 MiniMax H3（UNETLoader+CLIPLoader 載入後跑 KSampler）時，當 GPU VRAM 用量爬升到 20~22GB 附近、運算進行到一半，SSH 連線（不管走 ZeroTier VPN 還是公網IP）會斷線，實際上整台機器對新連線失去回應（serial console log 也停止產生新內容），但 GCP 控制台顯示 instance 仍然是綠色 RUNNING（不是真的 stop 了，只是對新 TCP 連線無回應）。

### 根本原因：RAM 不足，不是純 CPU 問題
- `g2-standard-8`（8 vCPU，只有32GB RAM）相對於 MiniMax H3 模型總大小（UNet 20GB + CLIP 15GB + VAE 共約5.5GB，總計~40GB）過小，GPU VRAM（L4只有24GB）塞不下全部模型，必須靠 CPU RAM 做 offloading
- 這台機沒有設定 swap（`free -h` 顯示 `Swap: 0B`），當實體RAM被模型+offloading瞬間塞滿時，沒有緩衝空間，讓記憶體回收壓力極高，連 sshd 要配置點記憶體處理新連線都做不到，症狀就像「當機」
- 實測驗證：換成 `g2-standard-16`（16 vCPU、62GB RAM，同一張L4 GPU不變）後，即使系統RAM用到 **41GB**（比原本32GB上限還高），SSH仍然**全程穩定不斷線**，問題徹底解決

### 解法：升級機型到 g2-standard-16（必須先stop才能改機型）
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"

# 1. 先stop（若機器已卡死對stop指令本身也可能逾時但實際上會生效，用describe確認STOPPING/TERMINATED就好，不用管timeout）
gcloud compute instances stop comfyui-h3-l4-a --zone=us-central1-a

# 2. 確認已完全停止
gcloud compute instances describe comfyui-h3-l4-a --zone=us-central1-a --format="value(status)"
# 要看到 TERMINATED 才繼續

# 3. 改機型
gcloud compute instances set-machine-type comfyui-h3-l4-a --zone=us-central1-a --machine-type=g2-standard-16

# 4. 重新開機
gcloud compute instances start comfyui-h3-l4-a --zone=us-central1-a
```
開機後公網IP會改變，用 `gcloud compute instances describe comfyui-h3-l4-a --zone=us-central1-a --format="value(networkInterfaces[0].accessConfigs[0].natIP)"` 重新查。Docker容器會自動起來（有設 `--restart unless-stopped`）。

g2系列規格對照（vCPU/RAM/GPU數量是搭套的）：
| 機型 | vCPU | RAM | GPU數量 |
|---|---|---|---|
| g2-standard-8 | 8 | 32GB | 1 |
| g2-standard-16 | 16 | 64GB（實際可見約62Gi）| 1 |
| g2-standard-32 | 32 | 128GB | 1 |

注意：改機型GPU數量不變（仍為1張L4），不會碰到GPU配額上限問題，只是vCPU/RAM費用會提高。

### 升級RAM後還會碰到的下一個問題：GPU VRAM (24GB) 不足
即使RAM夠大，如果解析度/幀數設得太高（實測試過 704x512, length=73 會OOM），依然會失敗，但這次是**乾淨報錯 `torch.OutOfMemoryError`**，不會拖垮整台機器。實測確認可行的安全參數：
- **解析度 512x384**（不要用704x512）
- **length=73**（~3秒@24fps，符合LINE動態貼圖規定的1-4秒）
- **steps=10**
這組參數實測穩定，約**4.5分鐘**跑完一支影片（包含模型載入時間，模型若已快取cached會更快）。

### 監控技巧：遇到SSH斷線不要恐慌 reset
1. **先用瀏覽器接管已開Chrome（port 9222）去 GCP Console 確認instance顯示綠色RUNNING還是其他狀態**，不要兩項都不確認就直接reset（reset會白白中斷正在跑的運算）
2. **優先用公網IP + 標準ssh**（不是`gcloud compute ssh`、也不是ZeroTier VPN內網IP）監控，實測發現比走ZeroTier穩得多（但升RAM後其實也不再斷線了，這點比較重要）：
   ```bash
   ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<公網IP>
   ```
3. 用 `gcloud compute instances get-serial-port-output comfyui-h3-l4-a --zone=us-central1-a` 看系統log最後一筆時間戳，若超過5分鐘都沒新log且SSH連不上才判斷真的卡死，這時候才 reset
4. `queue action:list`（comfyui-mcp的queue工具）查 running/pending 來確認運算是否已完成，別恢復性地根據斷線就假設失敗

## ⏱️ 如何用短片生成模型做出「長時間影片」（例如8分鐘）而不狂燒GPU（2026-08-07新增）

### 問題背景
MiniMax H3（及大多數影片生成模型）單次生成有實務上的長度上限——在這台VM安全參數下，一次穩定生成約是 **length=73（~3秒@24fps）**。若使用者想要的成品長度遠超過這個上限（例如8分鐘 = 480秒 = 約160個3秒片段），**不要**用「逐段生成160次再硬拼接」的做法，原因：
- GPU運算量等比放大（160段 × 每段約4.5-7分鐘 = 12小時以上連續運算），費用與風險都大幅增加
- 每段都是模型獨立取樣生成的結果，段與段之間的收尾姿勢無法保證一致，硬接會造成**明顯跳格/不連貫**的觀感，就算花了大錢也做不出「流暢的長影片」的效果

### 正確做法：只生成1（或少數幾個）循環單元，靠 FFmpeg 拼接湊時長
這是業界常見手法（許多迴圈式背景動畫、遊戲loop、MV都這樣做），步驟：

1. **只生成一小段（例如3秒/73幀）的動作片段**，prompt中可加入「loopable seamless dance loop animation」之類字眼提示模型讓首尾動作相近
2. **抓取生成結果的首尾幀比對相似度**（用 ffmpeg 擷取 `select=eq(n\,0)` 與 `select=eq(n\,72)` 存成png比對），如果首尾動作差異大（例如本次案例：開頭是手部特寫比讚、結尾是雙臂張開站姿），不能直接首尾相接，否則循環時會有明顯跳動
3. **用 FFmpeg `xfade` 濾鏡做交叉淡化**，把同一支影片的片尾平滑融合回片頭，做出真正無縫的「循環單元」影片：
   ```bash
   ffmpeg -i clip.mp4 -i clip.mp4 \
     -filter_complex "[1:v]trim=0:0.5,setpts=PTS-STARTPTS[v1];[0:v][v1]xfade=transition=fade:duration=0.5:offset=2.54[out]" \
     -map "[out]" -r 24 seamless_loop_unit.mp4 -y
   ```
   其中 `offset` 設為「原片長度 - xfade的duration」（例如原片3.04秒、duration=0.5秒，offset=2.54）
4. **用 FFmpeg concat demuxer 把這個循環單元重複貼N次，湊到目標總長度**，這步驟只需幾秒鐘，不吃GPU：
   ```bash
   # 產生重複清單（例如8分鐘目標，單元約3.04秒，需要約158次）
   for i in $(seq 1 158); do echo "file 'seamless_loop_unit.mp4'" >> loop_list.txt; done
   ffmpeg -f concat -safe 0 -i loop_list.txt -c copy dance_8min_full.mp4 -y
   ```
   `-c copy` 直接串流複製不重新編碼，速度極快（實測158段8分鐘影片，複製拼接不到1秒完成）。

### 這個做法的核心價值
- **GPU成本大幅降低**：8分鐘影片只需要生成1-2段素材（約10-15分鐘GPU運算），而非160段（12小時+）
- **視覺流暢度更好**：因為循環單元本身經過xfade處理無縫銜接，重複播放不會有生硬的跳動感；而逐段硬生成拼接則幾乎必然有跳格問題
- **如果想要「不單調」**：可以額外生成2-3種不同的循環單元（例如不同舞步/表情），在concat清單裡穿插排列不同素材檔名，讓長影片有變化但依然只需生成少數幾段

### 相關踩坑經驗：全身入鏡
若原始角色素材只有半身特寫（無下半身視覺資訊），MiniMax H3 在 Image-to-Video 生成時，只要 prompt 明確要求「wide shot, full body view, feet visible」等字眼，**AI是有能力自己「想像」補全出合理的下半身穿著與腿部比例的**，不需要另外準備全身立繪素材。但需注意：
- **steps=10 時容易出現中間畫格雜訊/色彩過曝閃爍**（實測第3、5幀出現明顯橘紅色雜訊糊圖），把 `steps` 提高到 **18** 可顯著改善此問題，代價是運算時間拉長（約從4.5分鐘增加到7分鐘/段），但仍在可接受範圍，換來的畫質穩定性提升是值得的

## ⚡⚡⚡ 最新狀態速覽（2026-08-12 晚間更新，LINE動態貼圖專案進行中，請優先看這裡）

### 目前進度：10段核心動作素材正在佇列生成中，SSH斷線（但VM未真正靛機）
- **workflow已完全驗證可行**：自己用原生ComfyUI節點（不依賴comfyui-easy-use、kjnodes等未安裝的套件）組了一份精簡版 image-to-video workflow API JSON，第一段（`01_wave`）實測成功產出 `stickers/01_wave_00001_.mp4`
- **已提交剩余9段到佇列**（`02_thankyou` ~ `10_dance`），使用ComfyUI自帶佇列機制（`/prompt` API連續提交多次，會自動依序執行，不需要等上一個完成才送下一個）
- **提交後約SSH連線在長時間監控中又斷線了**（在 `sleep 480` 等待期間發生），但 `gcloud compute instances describe ... --format="value(status)"`（不透過SSH，只查控制平面）確認VM**仍然是RUNNING**，不是真的崩溃/停機，符合先前已知的「g2-standard-8 RAM不足導SSH斷線但机器本身沒死」症狀。

### ⚠️ 新教訓：遇到SSH斷線不要恢性追加`sleep`重試SSH，改用控制平面API確認VM狀態
當SSH連線失敗/逾時，**第一步應該是用不需要SSH的方式確認VM是否真的靛機**：
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
# 不透過SSH，直接查控制平面狀態，快且穩定
timeout 20 gcloud compute instances describe comfyui-h3-l4-a --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d --format="value(status)"
```
若返回 `RUNNING`，代表GPU可能正在全力運算中（符合已知的RAM不足斷線模式），**不要病急亂投重新stop/start或reset**，而是：
1. 先用別的方式確認運算是否完成（例如隨後重新嘗試SSH、或等待更長時間後重試）
2. 若確定需要重新連線，優先用公網IP而非ZeroTier（先前實測經驗比較穩），且不要在同一個bash指令裡用`sleep N`堆叠後再接SSH，改用分次短暂確認，避免一次等太久導致整個指令被使用者中斷
3. 終極確認手段：`gcloud compute instances get-serial-port-output comfyui-h3-l4-a --zone=us-central1-a` 看系統log最後一筆時間戳是否遠遠落後，才判斷真的卵死

### ⚠️⚠️ 同一次任務中重複發生多次SSH斷線（2026-08-12，提交10段佇列後至少發生2次）
提交10段生成任務後，SSH在監控過程中**重複斷線好幾次**（不只發生一次），每次都確認VM控制平面狀態仍是RUNNING。**這不是偶發事件，而是 g2-standard-8（32GB RAM）跑MiniMax H3這種大模型長時間連續運算時的常態**，整個10段佇列需要執行近70分鐘，這段期間SSH逐漸失效極將是常態而非例外。

**改進後的正確应對方式**：
1. **不要在同一個 `Bash` 呼叫裡用 `sleep 60`以上的長時間等待再接SSH**，這樣做很容易被使用者中斷（看起來像卡住/无回應），改用**分次短暂查詢**（每次timeout 20秒內），若連不上就先回報現況、不要自己熊一直sleep
2. 確認SSH連不上時，优先用 `timeout 20 gcloud compute instances describe ... --format="value(status)"` 確認VM控制平面狀態（不需要SSH，不會被卡住），若顯示RUNNING就向使用者說明「VM沒真的卵死，只是SSH層面連不上，GPU可能仍在全力運算」，不要讓使用者誤以為服務卡死
3. **不要因為SSH連不上就存疑運算失敗而想reset/stop**，這樣會白白中斷正在進行的長時間GPU運算，浪費已經跑到一半的時間與金錢
4. 建議接下來監控節奏：每隔3-5分鐘才回報一次狀態（而非每分鐘都試SSH），避免頻繁嘗試反而更易觸發斷線、也避免使用者視重複回報為噴雜訊

### 下次接續時的檢查清單（LINE貼圖專案）
1. 確認VM狀態（用上面的timeout describe指令，不要直接SSH）
2. 若RUNNING，嘗試重新SSH連線（可能只是暫時性這時間點連不上，過幾分鐘再試），確認GPU運算是否仍在進行中
3. 連上後查 `curl -s http://localhost:8188/queue`，確認佇列進度（running/pending數量）
4. 查 `find /home/HCH/ComfyUI/output/stickers -type f -name '*.mp4'` 確認已完成幾段
5. 全邃完成後，進行後製（去背/裁切320x270/抽格≤十20格/轉APNG≤300KB）產出40張貼圖

### 建立好的 MiniMax H3 GGUF workflow 節點結構（可重用，已實測成功）
使用原生節點（不依賴額外custom node），流程：
```
LoadImage(1) → MiniMaxH3ImageToVideo(6, 需clip+vae+prompt+w/h/length+first_frame)
  → conditioning輸出進BasicGuider(9)，latent輸出進SamplerCustomAdvanced(12)
UnetLoaderGGUF(2) → MiniMaxH3SigmaShift(7, shift_video=12.0, shift_audio=3.0) → 這個model同時進BasicGuider和BasicScheduler(11)
CLIPLoaderGGUF(3, type="minimax") → 進MiniMaxH3ImageToVideo的clip
VAELoader(4, video_vae) → 進MiniMaxH3ImageToVideo的vae，也進VAEDecode(14)
VAELoader(5, audio_vae) → 進VAEDecodeAudio(15)
RandomNoise(8) + KSamplerSelect(10, res_multistep) + BasicScheduler(11, simple, steps) → 進SamplerCustomAdvanced(12)
SamplerCustomAdvanced(12) 輸出聯合AV latent → LTXVSeparateAVLatent(13) 拆成 video_latent + audio_latent
  → VAEDecode(14, 用video_vae) → images
  → VAEDecodeAudio(15, 用audio_vae) → audio
CreateVideo(16, images+audio, fps=24) → SaveVideo(17, filename_prefix)
```
關鍵點：`MiniMaxH3ImageToVideo` 輸出的是聯合AV latent（影音+音訊一起），**必須用`LTXVSeparateAVLatent`拆开**才能分別用`VAEDecode`和`VAEDecodeAudio`解碼（這個節點雖然名字帶LTXV，但說明中明記支援任何AV模型包括MiniMax H3）。

完整Python建構script已存在VM的 `/tmp/build_wf.py`，用法：`python3 build_wf.py "<prompt text>" <seed> "<filename_prefix>"`，輸出可直接POST到 `http://localhost:8188/prompt`。安全參數預設：512x384, length=73, steps=18（已写死width/height=512x384，steps=18）。

❗ **重要**：`/tmp` 不在磁碟上永久化（VM重啟就會消失），完整script備份如下，需要時直接用`cat > /tmp/build_wf.py << 'PYEOF' ... PYEOF`重建：

```python
import json, sys

def make_workflow(prompt_text, seed, filename_prefix, width=512, height=384, length=73, steps=18):
    wf = {
        "1": {"class_type": "LoadImage", "inputs": {"image": "79405.jpg"}},
        "2": {"class_type": "UnetLoaderGGUF", "inputs": {"unet_name": "MiniMax-H3-FL2VA-Q3_K_M.gguf"}},
        "3": {"class_type": "CLIPLoaderGGUF", "inputs": {"clip_name": "qwen3vl-32B-MiniMax-H3-Q2_K.gguf", "type": "minimax"}},
        "4": {"class_type": "VAELoader", "inputs": {"vae_name": "minimax_h3_video_vae_fp16.safetensors"}},
        "5": {"class_type": "VAELoader", "inputs": {"vae_name": "minimax_h3_audio_vae_fp32.safetensors"}},
        "6": {"class_type": "MiniMaxH3ImageToVideo", "inputs": {
            "clip": ["3", 0], "vae": ["4", 0], "prompt": prompt_text,
            "width": width, "height": height, "length": length,
            "first_frame": ["1", 0]
        }},
        "7": {"class_type": "MiniMaxH3SigmaShift", "inputs": {"model": ["2", 0], "shift_video": 12.0, "shift_audio": 3.0}},
        "8": {"class_type": "RandomNoise", "inputs": {"noise_seed": seed}},
        "9": {"class_type": "BasicGuider", "inputs": {"model": ["7", 0], "conditioning": ["6", 0]}},
        "10": {"class_type": "KSamplerSelect", "inputs": {"sampler_name": "res_multistep"}},
        "11": {"class_type": "BasicScheduler", "inputs": {"model": ["7", 0], "scheduler": "simple", "steps": steps, "denoise": 1.0}},
        "12": {"class_type": "SamplerCustomAdvanced", "inputs": {
            "noise": ["8", 0], "guider": ["9", 0], "sampler": ["10", 0],
            "sigmas": ["11", 0], "latent_image": ["6", 1]
        }},
        "13": {"class_type": "LTXVSeparateAVLatent", "inputs": {"av_latent": ["12", 0]}},
        "14": {"class_type": "VAEDecode", "inputs": {"samples": ["13", 0], "vae": ["4", 0]}},
        "15": {"class_type": "VAEDecodeAudio", "inputs": {"samples": ["13", 1], "vae": ["5", 0]}},
        "16": {"class_type": "CreateVideo", "inputs": {"images": ["14", 0], "audio": ["15", 0], "fps": 24}},
        "17": {"class_type": "SaveVideo", "inputs": {"video": ["16", 0], "filename_prefix": filename_prefix, "format": "auto", "codec": "auto"}}
    }
    return wf

if __name__ == '__main__':
    prompt_text = sys.argv[1]
    seed = int(sys.argv[2])
    prefix = sys.argv[3]
    wf = make_workflow(prompt_text, seed, prefix)
    print(json.dumps({"prompt": wf}))
```

使用方式：先上傳照片到ComfyUI input（`curl -s -X POST http://localhost:8188/upload/image -F 'image=@/tmp/79405.jpg' -F 'overwrite=true'`），再執行 `python3 build_wf.py "<prompt>" <seed> "stickers/<name>" > wf.json && curl -s -X POST http://localhost:8188/prompt -H 'Content-Type: application/json' -d @wf.json`。可以連續提交多個，ComfyUI佇列會自動依序執行。

## ⚡⚡ 最新狀態速覽（2026-08-12 更新）

### VM 曾經被誤刪，改用「磁碟優先、VM隨時可重建」的心態管理
2026-08-12 發現：先前紀錄運行中的 VM `comfyui-h3-l4-a` 其實已經不存在了（instance not found），但它的**開機磁碟還在**（zonal disk，200GB，us-central1-a，狀態READY）。這代表 VM instance 本身很容易在某些情境下消失（可能是先前手動delete、或是被GCP自動處理），但只要磁碟還在，資料（ComfyUI+模型+Docker image）都還在，**隨時可以重新掛載磁碟建立新VM**，不用重新下載任何模型。

**教訓**：管理這個環境的心態應該是「磁碟是資產，VM是消耗品」，VM隨時可能要重建，重建前優先確認磁碟是否還在：
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
gcloud compute instances list --project=project-fbcea0c4-05a8-4472-b9d
gcloud compute disks list --project=project-fbcea0c4-05a8-4472-b9d
gcloud compute snapshots list --project=project-fbcea0c4-05a8-4472-b9d
```

### 用既有磁碟重建VM的正確指令（不是全新create，是掛載既有disk開機）
```bash
gcloud compute instances create comfyui-h3-l4-a \
  --project=project-fbcea0c4-05a8-4472-b9d \
  --zone=us-central1-a \
  --machine-type=g2-standard-8 \
  --accelerator="type=nvidia-l4,count=1" \
  --maintenance-policy=TERMINATE \
  --disk="name=comfyui-h3-l4-a,boot=yes,auto-delete=no" \
  --tags=comfyui-h3
```
注意：**不要**加 `--image-family`/`--image-project`（那是建全新開機磁碟用的），改用 `--disk="name=...,boot=yes,auto-delete=no"` 掛載既有磁碟當開機碟，`auto-delete=no` 確保刪除VM時磁碟不會被連坐刪除。

### ⚠️ 新踩坑：Zonal Disk 無法跨 zone 掛載，跨zone搬移必須先拍快照
磁碟是 zone-locked 資源（例如 `comfyui-h3-l4-a` 只能在 `us-central1-a` 用），如果該 zone GPU缺貨、想換到別的zone試，**不能直接把磁碟掛到別zone的VM**，必須：
```bash
# 1. 對現有磁碟拍快照（快照是全域資源，不受zone限制）
gcloud compute disks snapshot comfyui-h3-l4-a --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d --snapshot-names=comfyui-h3-l4-a-migrate

# 2. 用快照在目標zone建立新磁碟
gcloud compute disks create comfyui-h3-l4-b --zone=us-central1-b --project=project-fbcea0c4-05a8-4472-b9d --source-snapshot=comfyui-h3-l4-a-migrate --type=pd-balanced

# 3. 用新磁碟在目標zone建VM（同上面「重建VM」指令，換--zone和--disk的name）
```

**⚠️ 重要副作用：這樣做等於把資料複製了一份，SSD空間配額會double！**

### ⚠️⚠️ 新踩坑：SSD_TOTAL_GB 配額 500GB/region，很容易被「重複磁碟」頂爆
實測案例：原本1顆200GB磁碟（`comfyui-h3-l4-a`），為了跨zone嘗試又從快照複製了一顆200GB（`comfyui-h3-l4-c`），兩顆同時存在 = 400GB，逼近 `us-central1` region 的 `SSD_TOTAL_GB` 上限500GB，只剩100GB餘裕。

**檢查目前SSD用量的方法**（比 `gcloud alpha services quota list` 更可靠，這個alpha指令在有的環境下會因缺少alpha元件、且沒有系統管理員權限而失敗）：
```bash
gcloud compute regions describe us-central1 --project=project-fbcea0c4-05a8-4472-b9d --format="json(quotas)" > /tmp/quota.json
grep -B1 -A2 '"metric": "SSD_TOTAL_GB"' /tmp/quota.json
grep -B1 -A2 '"metric": "NVIDIA_L4_GPUS"' /tmp/quota.json
```

**教訓**：每次為了「試試看哪個zone有貨」而從快照複製磁碟，一旦確定不用了要**立刻刪除**，不要留著，否則配額會被迅速吃光：
```bash
gcloud compute disks delete comfyui-h3-l4-c --zone=us-central1-c --project=project-fbcea0c4-05a8-4472-b9d --quiet
```

### ⚠️⚠️⚠️ 新踩坑：GPU缺貨狀況可以在數十秒到數分鐘內劇烈變動，且GCP錯誤訊息裡的「建議zone」常常也是錯的/過時的
2026-08-12實測：短短十幾分鐘內，`us-central1-a`、`-b`、`-c` 三個zone的 `g2-standard-8`/`g2-standard-16` + L4 GPU可用性像骰子一樣random跳動，GCP錯誤訊息裡明明說「試試 us-central1-b 現在有貨」，切過去馬上又說「試試 us-central1-a/-c」，繞了一圈全部lock不到。**不要相信單次錯誤訊息裡的zonesAvailable建議，那只是那一瞬間的快照，幾秒後就可能不準了。**

**唯一有效對策：無腦重試迴圈**，在同一個zone、同一個機型上，每隔8-15秒重試一次，撐過幾輪（實測第5輪左右常會成功）：
```bash
export PATH="$PATH:/c/Users/HCH/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin"
for i in $(seq 1 15); do
  echo "===== 第 $i 次嘗試 ====="
  gcloud compute instances start comfyui-h3-l4-a --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d 2>&1 | tail -3
  STATUS=$(gcloud compute instances describe comfyui-h3-l4-a --zone=us-central1-a --project=project-fbcea0c4-05a8-4472-b9d --format="value(status)" 2>&1)
  echo "目前狀態: $STATUS"
  if [ "$STATUS" == "RUNNING" ]; then break; fi
  sleep 10
done
```
若這個「無腦重試」策略連續15輪以上都失敗（代表這段時間該zone/機型組合是真的全面缺貨、不是短暫波動），**應直接放棄等待，改用當前能成功的最低規格重試（例如從 g2-standard-16 降到 g2-standard-8），先求VM能開機，之後再擇機升級規格**，不要無限期空等，浪費使用者時間。

### ⚠️ 新提醒：stop → set-machine-type → start 這套「升級RAM」流程，改機型後的start一樣會撞到GPU缺貨
先前技能文件的既有章節提到「rebuild機型從g2-standard-8升到g2-standard-16可解決RAM不足斷線問題」，但要注意：**改完機型後的 `start` 指令，一樣要重新搶GPU資源**，不是「改機型」本身失敗，是「開機」這個動作在搶资源。若升級規格的start屢屢碰到缺貨，考慮的優先順序：
1. 無腦重試（同上）
2. 若使用者時間有限，先退回目前能成功開機的規格（哪怕RAM較小、之後跑重負載要更小心監控），讓服務先能用，之後找空檔再升級
3. 不要「改機型」跟「找貨」這兩件事同時卡在一起賭運氣，可以先確定「這個規格能開機」，開機成功後才考慮要不要為了升RAM再賭一次stop/start（因為每一次stop/start都是重新排隊搶資源，有風險把本來能跑的VM搞到又停機搶不回來）

### Docker Image 備份到 Docker Hub（2026-08-12 開始執行，取代/補充 Google Drive 備份）
目的：讓 ComfyUI 執行環境（`comfyui-h3:local` image，含Python環境、ComfyUI本體、custom nodes如ComfyUI-GGUF）不用綁定在特定的GCP磁碟上，之後在任何有docker+GPU的地方（包括別的雲端平台如RunPod/Vast.ai）都能直接 `docker pull` 使用，降低對這顆磁碟/這個zone的依賴。

**前提**：VM必須是RUNNING狀態，才能對容器內的image做 `docker save`。若VM因GPU缺貨開不了機，這個備份動作會被卡住，須先想辦法讓VM開機（見上面的重試策略）。

**步驟（VM開機成功後執行）**：
```bash
# 1. VM上把運行中的容器image打包（docker commit，如果image有因臨時安裝套件如gguf而變動要先commit進image，不然只是docker save舊image不含新裝的套件）
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "sudo docker commit comfyui-h3 comfyui-h3:latest"

# 2. 登入Docker Hub（若尚未登入，需要使用者提供帳密或token）
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "sudo docker login"

# 3. Tag並推送
ssh -i /c/Users/HCH/.ssh/google_compute_engine HCH@<VM_IP> "sudo docker tag comfyui-h3:latest <dockerhub帳號>/comfyui-h3:latest && sudo docker push <dockerhub帳號>/comfyui-h3:latest"
```
⚠️ 注意：**這個image不含模型檔案**（模型是透過volume mount `/home/HCH/ComfyUI` 掛進去的，不在image內），所以push上去的image只有系統環境+ComfyUI本體+custom nodes，體積應該是GB等級而非數十GB。模型仍需另外用 rclone/Google Drive 或重新下載的方式取得（見前面「已完整備份至Google Drive」章節）。

### ✅ 已完成（2026-08-12）
- Docker Hub repo：**`hch590902/comfyui-h3:latest`**（帳號 `hch590902`，登入方式目前是帳號密碼，**建議後續改用 Access Token**）
- 內容：ComfyUI本體 + custom nodes（含ComfyUI-GGUF及其`gguf`依賴套件，已經 `docker commit` 進去image）
- 大小：12GB
- Digest：`sha256:24247c8800d4abdc67f7c492ab0bcc61178b64ebc6a34dad8d07bede40cc798f`
- ⚠️ 不含模型檔（volume mount進來的，不在image內），重建時仍需別外從磁碟快照或Google Drive取得模型
- 日後重建VM時可直接 `docker pull hch590902/comfyui-h3:latest` 不用重新clone repo/裝 ComfyUI-GGUF

**重要提醒**：使用者曾在對話中直接貼出 Docker Hub 帳號密碼明文，**日後應提醒使用者去 Docker Hub 網站更改密碼**，並改用 Access Token（Account Settings → Security 建立）來登入，方便日後撤銷而不影響帳號本身。

## LINE 動態貼圖製作專案（2026-08-12 新增）

### 使用者需求
以真人小孩照片（`D:\79405.jpg`，半身照，粉紅上衣+牛仔吊帶褲）為基礎，用 MiniMax H3 image-to-video 製作 LINE 動態貼圖一組40張。

### LINE 動態貼圖官方規格（重要，後製時要符合）
- 尺寸：320×270 px
- 格式：APNG（透明背景）
- 動畫：最多 **20 個畫格**，總長度 **最多 4 秒**
- 檔案大小：**每張貼圖必須 ≤ 300KB**
- 一組貼圖需要 8/16/24/32/40 張其中一種數量

MiniMax H3 生成的是短影片素材（mp4），必須經過後製管線才能變成合格的LINE貼圖：
1. 生成短片（image-to-video，約3秒/73幀，512x384，見前面「安全參數」章節）
2. 去背處理（可用 ComfyUI 的 remove_background / BiRefNet 節點，或FFmpeg+其他去背工具）成透明背景
3. 裁切/縮放成 320×270
4. 抽格：把24fps/73幀的素材降到20格以內（例如每3-4幀取1幀）
5. 轉檔成 APNG，並確認壓縮後單檔 ≤300KB（可能需要調色深度/幀數進一步壓縮）

### 已確認的執行方案：「方案A」省GPU成本策略
40張全部各自獨立生成不同動作需要約 **40段 × 7分鐘 ≈ 4.7小時GPU連續運算**，成本過高。改採：
- **只生成10種核心動作素材**（GPU運算約 10×7分鐘=70分鐘），核心動作清單：你好揮手、謝謝鞠躬、大笑、愛心比心、生氣鼓臉頰、驚訝、害羞臉紅、比YA、哭哭、跳舞
- **其餘30張用FFmpeg後製變化擴充**（不吃GPU，速度快）：對同一段素材做變速、局部裁切（例如只取臉部特寫段落 vs 全身動作段落）、調色/濾鏡微調等，做出有差異但仍是「動態」的版本
- 不加中文字（純動作表情，之後如需要可再疊字）

### 執行狀態（2026-08-12 中斷點，下次接續時從這裡開始）
**尚未開始生成**，卡在VM因GPU持續缺貨開不了機（見上面「最新狀態速覽」章節）。使用者已經決定「先不追求RAM升級到g2-standard-16，用現在能搶到的規格就好」，所以下次VM一旦成功開機（不論是g2-standard-8或16），**應直接開始跑這10段核心動作素材的生成**，不用再糾結要不要升RAM——已有前例證明g2-standard-8搭配安全參數（512x384, length=73, steps=18）大部分時候可行，只要密切監控即可，不用為了追求穩定性而在GPU缺貨時持續空等升級。

下次接續時的檢查清單：
1. 確認VM狀態（RUNNING與否）
2. 若TERMINATED，先確認磁碟還在（見上面「VM曾經被誤刪」章節），用磁碟重建VM，機型不用堅持16，8能開就先用8
3. VM開機成功後，**先完成Docker Hub備份**（上面章節），再開始跑10段素材生成
4. 10段素材生成完成後，進行FFmpeg後製管線（去背/裁切/抽格/轉APNG）產出40張最終貼圖

## 目的（原有章節，見下方）

---

## Conformance Addendum

## When to Use
在 GCP 上申請 GPU 配額、建立 GPU VM、安裝並執行 ComfyUI 的完整流程。使用者要求「繼續 GCP ComfyUI 部署」、「檢查配額」、「建立 GPU VM」、「裝 ComfyUI」時使用。

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
