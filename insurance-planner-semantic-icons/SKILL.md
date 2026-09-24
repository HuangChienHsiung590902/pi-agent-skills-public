---
name: insurance-planner-semantic-icons
description: 當保險規劃網站需要重新生成、整理、挑選或整合保險分類圖示時使用；透過遠端 ComfyUI 生成與分類意義相符、不是單純抽象符號的 10 組語意化圖示，放大網站顯示並完成部署驗證。
---

# 保險規劃網站語意化圖示

## 任務目標

為保險規劃網站建立一套視覺一致、尺寸足夠、且能直接表達保險分類含義的圖示。圖示不能只是重複的盾牌、勾選或幾何符號；每個主分類都應有對應的主體與輔助意象，讓使用者在左側分類選單與右側規劃卡片中能快速理解用途。

本 Skill 涵蓋：

- 使用現有遠端 ComfyUI 生成 10 個保險分類專屬圖示。
- 下載、檢查、重新命名及整理 PNG 資產。
- 將圖示套用到左側分類選單與右側分類卡片。
- 在網站保留候選圖示展示區，讓使用者比較與挑選。
- 將左側及右側圖示放大，但維持頁面緊湊布局。
- 重新建置 Docker 網站、驗證資產 URL、檢查舊網站未被修改。

## 適用時機

使用者提出以下需求時使用：

- 「圖示太小」或要求放大保險分類圖示。
- 「圖示要符合意思」或不接受單純盾牌、勾選、抽象符號。
- 重新生成壽險、醫療險、意外險等分類圖示。
- 使用 ComfyUI 為保險網站生成多個圖示版本。
- 整理候選圖示、加入網站展示區或將選定圖示整合到網站。
- 比較不同圖示風格，並驗證圖示可在部署網站直接查看。

網站維護本身仍遵守 [`insurance-planner-web`](../insurance-planner-web/SKILL.md)；本 Skill 專注於語意化圖示的生成與整合。

## 輸入與輸出

### 輸入

- 要重新生成、整理、挑選或整合的保險分類圖示需求。
- 現有遠端 ComfyUI（`http://10.145.119.19:8190/`）與保險規劃網站（`http://10.145.119.12:18084/`）。

### 輸出

- 10 張語意化分類 PNG，以及網站內的固定分類 mapping。
- 部署後的資產 URL、尺寸與 `18083` 未被修改的驗證結果。

## 固定環境與路徑

- ComfyUI：`http://10.145.119.19:8190/`
- ComfyUI 輸出主機目錄：`/home/hch/ComfyUI/output-h3/`
- 已確認可用模型：
  - `/home/hch/ComfyUI/models/diffusion_models/z_image_turbo_bf16.safetensors`
  - `/home/hch/ComfyUI/models/text_encoders/qwen_3_4b.safetensors`
  - `/home/hch/ComfyUI/models/vae/ae.safetensors`
- 網站主機：`hch@10.145.119.12`
- 網站專案：`/home/hch/docker-webs/insurance-planner/`
- 網站首頁：`/home/hch/docker-webs/insurance-planner/site/index.html`
- 正式圖示目錄：`/home/hch/docker-webs/insurance-planner/site/assets/icons/`
- 網站網址：`http://10.145.119.12:18084/`
- 原有網站 `18083` 必須保留且不可修改。

本工作不修改或重啟 ComfyUI 設定、不下載已存在的模型，也不修改 `18083`。

## 圖示分類與語意對應

固定使用以下 10 個分類與檔名；生成時不得讓 10 個分類循環共用少數幾張無關圖示：

| 分類 | 檔名 | 必須表達的主體 |
|---|---|---|
| 壽險 | `01-life-family-shield.png` | 家庭人物與家庭守護 |
| 醫療險 | `02-medical-doctor-care.png` | 醫師照護病患、醫療元素 |
| 意外險 | `03-accident-safety-helmet.png` | 安全帽、人物與意外防護 |
| 癌症險 | `04-cancer-cell-care.png` | 癌症細胞、醫療照護或癌症關懷 |
| 重大疾病險 | `05-critical-heart-care.png` | 心臟、心電圖或重要疾病守護 |
| 長照險 | `06-caregiver-elderly.png` | 照護者陪伴長者、長期照護 |
| 失能險 | `07-disability-support.png` | 輪椅／助行器與生活扶助 |
| 旅平險 | `08-travel-airplane-luggage.png` | 飛機、行李箱與旅途保障 |
| 車險 | `09-car-collision-shield.png` | 汽車與防撞／交通安全 |
| 火險／住宅險 | `10-home-fire-protection.png` | 房屋、火焰及居家防護 |

## 圖像生成規格

### 正向 Prompt 結構

每張圖至少包含：

1. 乾淨的扁平向量／編輯式插畫風格。
2. 一個明確的分類主體。
3. 一個與主體有關的輔助意象。
4. 方形、置中、適合縮小到網站選單的構圖。
5. 清楚邊緣、低雜訊、柔和背景與一致的 UI 插畫風格。
6. 與分類固定顏色相容，但不可犧牲語意辨識度。

建議共用描述：

```text
A polished semantic insurance category icon for a modern web menu,
flat vector editorial illustration, one centered recognizable subject
with a small supporting detail, compact square composition, clean crisp
edges, premium friendly UI illustration, no text, no letters, no words,
no logo, no watermark, no border, no extra objects, not an abstract
generic symbol, not only a shield.
```

### 負向 Prompt

至少排除：

```text
text, letters, words, typography, watermark, logo,
generic abstract symbol, plain shield only, multiple unrelated objects,
photorealistic, clutter, blurry, distorted, cropped, asymmetry
```

不得生成文字、字母、品牌 Logo、浮水印或多個不相關主體。若輸出仍只剩抽象盾牌，必須只重生該分類，不要直接採用。

## 執行流程

### 1. 確認目前環境

先唯讀檢查：

```bash
curl -fsS http://10.145.119.19:8190/system_stats
curl -fsS http://10.145.119.19:8190/queue
ssh hch@10.145.119.19 "docker ps --filter name='^/comfyui-h3$' --format '{{.Names}}|{{.Status}}|{{.Ports}}'"
```

若 ComfyUI 不可用、模型遺失或 GPU 狀態異常，先停止，不要修改 ComfyUI 設定；另依相關 ComfyUI Skill 處理。

### 2. 建立專用暫存目錄

Windows 測試與下載暫存統一放在專案外，例如：

```text
C:\Users\HCH\AppData\Local\Temp\pi-work\insurance-planner-semantic-icons-<task-id>\
```

生成的 JSON、下載 PNG、HTTP 回應與抽出的測試 JavaScript 都放在該目錄，不得放入網站專案或 `D:\OB\skills`。

可使用本 Skill 的腳本建立 10 個 ComfyUI workflow：

```bash
python D:/OB/skills/insurance-planner-semantic-icons/scripts/build_comfyui_workflows.py \
  --output-dir <temporary-directory>/workflows
```

腳本只產生 JSON，不會自動送出遠端任務，避免未經確認就啟動 GPU 工作。

### 3. 送出與監控

將 workflow 透過 `POST /prompt` 送到 ComfyUI。每次只送一個任務，確認前一個任務的 `history/<prompt_id>` 為 `success`、輸出檔存在且容器仍正常後，再送下一個。

成功回應必須包含：

```json
{"prompt_id":"...","node_errors":{}}
```

以以下 API 監控：

```bash
curl -fsS http://10.145.119.19:8190/queue
curl -fsS http://10.145.119.19:8190/history/<prompt_id>
```

若 `node_errors` 非空、任務為 `error` 或輸出不符合語意，記錄問題並只重試該分類。不得連續大量重送任務。

### 4. 下載與檢查

從 ComfyUI `/view` API 或遠端輸出目錄下載結果到暫存目錄，統一命名為表格中的正式候選檔名。每張圖必須檢查：

- PNG 格式可讀。
- 尺寸為 `512×512` 或更大且為正方形。
- 主體置中、未被裁切、縮小後仍可辨識。
- 內容與分類語意相符。
- 沒有文字、浮水印、Logo 或明顯生成瑕疵。
- 10 張圖的整體風格一致，但分類顏色與主體可區分。

### 5. 整合網站

正式資產放入：

```text
/home/hch/docker-webs/insurance-planner/site/assets/icons/
```

`site/index.html` 必須：

- 以 10 個分類 ID 一一對應圖示，不可用 6 張圖循環套用。
- 左側主分類使用對應圖示。
- 右側分類卡片使用對應圖示；若右側由分類資料渲染，必須沿用相同 mapping。
- 保留「ComfyUI 圖示候選」展示區，顯示分類名稱與圖示，供使用者比較。
- 點選候選圖示可以標示目前選取或預覽，但未經使用者明確選定，不應暗中替換另一套正式 mapping。
- 圖示使用 `object-fit: cover` 或 `contain` 時，必須先確認不會裁切主體。
- 左側及右側主要分類圖示建議至少 `44×44px`；不可因縮小而失去分類意義。
- 子項目維持文字為主，不塞入過大的重複圖示。
- 不改變既有拖曳、防重複、欄位、刪除確認、`localStorage` 與分類配色功能。

### 6. 部署

修改前先抓回正式遠端 `index.html` 與必要設定到專用暫存目錄，讀取現況後再修改。完成後：

```bash
scp <index.html> hch@10.145.119.12:/home/hch/docker-webs/insurance-planner/site/index.html
scp <icon-files> hch@10.145.119.12:/home/hch/docker-webs/insurance-planner/site/assets/icons/
ssh hch@10.145.119.12 "cd /home/hch/docker-webs/insurance-planner && docker-compose -p insurance-planner up -d --build"
ssh hch@10.145.119.12 "docker exec insurance-planner-web nginx -t"
```

### 7. 清理

確認沒有仍在執行的 ComfyUI 任務或下載程序後，清理專用暫存目錄。若使用者要求保留候選圖示或報告，才保留並明確回報路徑。

## 規則與限制

- 不把抽象盾牌、勾選或幾何形狀當成所有分類的正式圖示。
- 不用少數圖示循環冒充 10 個分類的專屬圖示。
- 一次只送一個 ComfyUI 任務；長時間或大量批次前需確認 queue 與 GPU 狀態。
- 不修改 ComfyUI 設定、模型、Docker compose 或重啟服務，除非使用者另行授權。
- 不修改 `18083` 既有網站。
- 不上傳 Token、密碼、SSH 私鑰、瀏覽器資料、`localStorage`、Log 或 Cache。
- 不將測試輸出留在網站專案、Skill 目錄或目前工作目錄。
- 不新增後端、資料庫或前端依賴套件；網站仍維持靜態 Docker／Nginx。
- 不因圖示整合而恢復大型 padding、固定大型 `min-height` 或破壞現有緊湊版面。
- 不把圖示候選展示誤描述為已由使用者選定；候選展示與正式採用要分開。
- 圖示生成是視覺資產產製，不代表正式保險建議、報價或金融意見。

## 常見陷阱

- **只有盾牌沒有語意**：重新補充具體人物、醫療、交通、房屋等主體；不要只改顏色。
- **圖示太小或被裁切**：CSS 使用固定容器與 `object-fit` 後，要實際檢查 44px 顯示效果。
- **10 個分類錯用同一張圖**：用固定 ID mapping 與檔名檢查，禁止以陣列短循環取代分類對應。
- **生成任務送太快**：GPU 工作應等待上一個 prompt 完成；遇到 queue 不空不要再堆任務。
- **圖片快取舊版本**：網站 Nginx 應維持 no-cache；檢視時使用新的 `?v=` 參數並以 HTTP 重新確認。
- **只做 JavaScript 語法檢查**：語法通過不代表圖片 URL、尺寸、容器或實際頁面可用，必須加做 HTTP 與資產檢查。
- **把 ComfyUI H3 Skill 當成所有圖片流程**：本流程使用現有 Z-Image Turbo 圖片模型；影片生成與 H3 環境管理仍使用 `minimax-h3-comfyui`。

## 驗證方式

完成前必須具備以下證據：

1. ComfyUI 10 個任務各自回報 `status_str=success`，或明確記錄失敗分類與重試結果。
2. 10 個正式候選檔案存在，且每張是可讀的正方形 PNG，建議尺寸 `512×512`。
3. `site/index.html` 包含 10 個唯一檔名與分類 mapping，不是少數圖示循環。
4. 從 HTML 抽出 `<script>` 到暫存目錄後，`node --check` 通過。
5. 重新建置後 `insurance-planner-web` 狀態為 `Up`。
6. `docker exec insurance-planner-web nginx -t` 成功。
7. `http://10.145.119.12:18084/` 回應 `200 OK`，且頁面包含候選展示區。
8. 10 個 `/assets/icons/*.png` URL 均回應 `200`、`Content-Type: image/png`，且內容尺寸正確。
9. 左側與右側圖示容器實際至少約 `44×44px`，且主體不被裁切。
10. `http://10.145.119.12:18083/` 仍回應 `200 OK`。
11. GitHub 若需同步，確認 Private Repository `main` 的新 commit 與 10 個圖示檔案存在。
12. 測試與下載暫存目錄已清理，並回報實際使用的暫存路徑。
