---
name: twtsl-ccu-dict
description: 查詢「台灣手語線上辭典」（TSL@CCU，中正大學語言所建置，https://twtsl.ccu.edu.tw）取得官方手語示範影片、手形/位置資訊、動作文字說明。用在：要找某個中文詞彙對應的正確台灣手語動作、要下載官方手語示範影片（例如拿去做 MiniMax-H3 R2V motion transfer 的參考素材）、或要查手形/位置代碼反查詞彙時。
---

# 台灣手語線上辭典（TSL@CCU）API 查詢

2026-08-05 逆向工程網站前端 JS bundle 找到的內部 REST API。網站本身是 React SPA
（`https://twtsl.ccu.edu.tw`，vite 打包），畫面上的搜尋功能全部走這些 API，可以繞過瀏覽器直接
`curl` 查詢。

## 為什麼要查這個網站

這是中正大學語言所建置的**官方、免費、公開**台灣手語辭典，影片是真人手語老師的標準示範，畫質
乾淨（純色背景、固定機位），非常適合當 MiniMax-H3 Ref2VA 的 motion-transfer 參考影片來源
（見 skill `minimax-h3-comfyui` 的「R2V」章節）。相較於自己找 YouTube/抖音影片再裁切，這裡的
素材更乾淨、動作更標準、且有文字說明可以直接轉成 prompt。

## 已驗證可用的 API

Base URL: `https://twtsl.ccu.edu.tw/api/`

### 1. `manualSearch` — 關鍵字查詢中文詞彙（找 id）

```bash
curl "https://twtsl.ccu.edu.tw/api/manualSearch?name=吃飯&lang=zh&page=1&pageSize=10"
```
```json
{"Record":[{"id":649,"name":"吃飯_A"},{"id":965,"name":"吃飯_B"}],"Total":2}
```

- `name`：中文詞彙（URL encode）
- 若詞彙有多種打法會回傳多筆，`_A`/`_B`（同一詞彙不同打法，`_A`較常用）或 `_N`/`_S`（北部/南部方言）
- 支援異體字（「台灣」「臺灣」都查得到「台灣」詞條）

### 2. `querySearch` — 依 id 查詳細內容（手形/位置/描述/影片路徑）

```bash
curl "https://twtsl.ccu.edu.tw/api/querySearch?id=649&lang=zh"
```
```json
{
  "Record": [{
    "id": 649,
    "name": "吃飯_A",
    "location1": "602_palm",
    "lo1_handshape_number_1": "fiveFinger",
    "lo1_hs1": "503_挖a",
    "stroke": 6,
    "description": "一手在嘴巴前做扒飯狀，另一手接近下巴做拿碗狀。",
    "clip": "video/e/eat_rice"
  }],
  "Total": 1
}
```

- `clip` 欄位是影片相對路徑，**去掉副檔名**，實際檔案在 `clip + ".mp4"`（見下方步驟3）
- `location1`~`location5`、`lo1_handshape_number_1`/`lo1_hs1` 等最多到 `5`，代表複合位置/手形的詞彙
  （單手單位置的詞只填 `location1`/`lo1_*`，其餘為 `null`）
- `description` 是最實用的欄位，可以直接拿來當 R2V prompt 的動作描述基礎

### 3. 下載影片

```bash
curl -O "https://twtsl.ccu.edu.tw/video/e/eat_rice.mp4"
```
把 `querySearch` 回傳的 `clip` 路徑直接接上 `.mp4` 副檔名即可，不需要額外認證，`https://twtsl.ccu.edu.tw` + `/` + `clip` + `.mp4`。已實測 200 OK 直接下載成功。

**已知需前處理**：官方影片畫面右下角有「TSL@CCU」浮水印疊字，若要拿去做 MiniMax-H3 R2V 參考素材，
先用 `ffmpeg delogo` 濾鏡去除（浮水印大約在畫面下方 1/4 處，實際座標需針對每支影片微調，可用
`ffmpeg -vf fps=3 frame_%02d.png` 抽格後人工核對浮水印像素範圍）：
```bash
ffmpeg -y -i eat_rice.mp4 -vf "delogo=x=255:y=360:w=190:h=45:show=0" -c:v libx264 -crf 18 -c:a copy eat_rice_clean.mp4
```

### 4. `pinSearch` — 筆劃查詢（已驗證 `field=stroke` 可用）

```bash
curl "https://twtsl.ccu.edu.tw/api/pinSearch?field=stroke&value=6&lang=zh&page=1&pageSize=10"
```
回傳所有筆劃數符合的詞彙清單（`id`+`name`），沒有影片細節，需要再用 `querySearch` 查每個 id。
`field` 目前只驗證過 `stroke` 這個值有效；其他 `field` 值未測試。

### 5. `handSearch` — 依手形+位置反查詞彙（已驗證，用三位數字代碼）

```bash
curl "https://twtsl.ccu.edu.tw/api/handSearch?handShape=503&location=602&lang=zh&page=1&pageSize=100"
```

**重要**：`handShape`/`location` 吃的是**三位數字代碼**（例如 `503`、`602`），**不是**
`querySearch` 回傳裡看到的 `fiveFinger`/`602_palm` 這種字串（那些是給人看的分類標籤，不是 API
參數值）。已實測 `handShape=503&location=602` 能查到「吃飯_A」（id 649），對應
`querySearch` 回傳的 `lo1_hs1: "503_挖a"` 開頭數字。

已知的三位數字代碼分段範圍（從網站前端 bundle 裡的手形/位置縮圖檔名逆向出來，共 66 組手形代碼、
數十組位置代碼，**具體每個數字對應哪個手形/位置圖片尚未逐一核對**，只驗證了 `503`+`602` 這組）：

| 數字開頭 | 推測分類 |
|---|---|
| 101–109 | 一隻手指的各種變體 |
| 201–219 | 兩隻手指的各種變體 |
| 301–317 | 三隻手指的各種變體 |
| 401–406 | 四隻手指的各種變體 |
| 501–512 | 五隻手指的各種變體 |
| 601–606 | 位置代碼（601手指、602掌心、603手指側邊、604手背、605手、606手腕，對照使用說明頁的「位置列表」圖） |

**待補完**：位置列表圖裡還有「身前」「背部」「身體兩側」「下手臂」「上手臂」「手臂」「手肘」等大分類
位置，對應的三位數字代碼還沒有實測確認（可能落在 `101–4xx` 區間，需要用已知詞彙反查驗證）。

### 6. `group` — 查多義詞/同義詞群組（發現但未深入測試）

```bash
curl "https://twtsl.ccu.edu.tw/api/group?id=<id>&lang=zh"
```
對應官網說明的「多義詞會標示數字，例如查詢『偉大_A』顯示『1 有名_S』『2 偉大_A』」這個功能，
用途待確認，尚未實測回傳格式。

### 7. `sentence` — 查例句（發現但未測試）

```bash
curl "https://twtsl.ccu.edu.tw/api/sentence?id=<id>&lang=zh"
```
對應官網「部分詞條有例句」的功能，回傳格式未測試。

### 8. `showAll`（發現但呼叫方式不明）

前端呼叫簽名是 `{case, query, lang, page, pageSize}`，但直接帶常見猜測值（`keyword`等）都回
`{"error":"Invalid case"}`。`case` 參數的正確枚舉值尚未破解，**這個端點暫時當作未知，優先用
`manualSearch`/`pinSearch`/`handSearch` 這三個已驗證好的端點**。

## 標準工作流程：查詞彙 → 下載乾淨影片 → 送進 MiniMax-H3 R2V

```bash
# 1. 關鍵字查 id（可能有多筆，通常選 _A 版本，較常用打法）
curl "https://twtsl.ccu.edu.tw/api/manualSearch?name=<中文詞彙的URL編碼>&lang=zh&page=1&pageSize=10"

# 2. 用 id 查詳細內容，記下 clip 路徑和 description
curl "https://twtsl.ccu.edu.tw/api/querySearch?id=<id>&lang=zh"

# 3. 下載影片
curl -O "https://twtsl.ccu.edu.tw/<clip路徑>.mp4"

# 4. 抽幾張關鍵格確認浮水印座標，用 delogo 去除
ffmpeg -y -i <下載的mp4> -vf "fps=3" frame_%02d.png   # 人工核對浮水印範圍
ffmpeg -y -i <下載的mp4> -vf "delogo=x=..:y=..:w=..:h=..:show=0" -c:v libx264 -crf 18 -c:a copy <clean.mp4>

# 5. scp 去除浮水印後的影片 + 人物參考照片到 ComfyUI 伺服器，組 Ref2VA prompt
#    （完整 R2V API 呼叫範本、DisTorch2 分卡設定、prompt 六段式格式見 skill minimax-h3-comfyui）
```

Prompt 撰寫時，`querySearch` 回傳的 `description`（例如「一手在嘴巴前做扒飯狀，另一手接近下巴做
拿碗狀」）可以直接當作 `detailed_description` 段落裡動作描述的核心依據，翻成英文並套進官方
Full-Reference Mode 六段式格式（`subject_definitions`/`summary`/`retention_analysis`/
`detailed_description`/`overall_soundscape`/`non_diegetic_music`）。

## 實測案例記錄

2026-08-05：查詢「吃飯」→ id 649（`吃飯_A`）→ `clip: video/e/eat_rice` → 下載
`eat_rice.mp4`（640×480, 3.4秒, 575KB）→ 內容為穿酒紅色上衣的示範者做「一手在嘴前扒飯狀、另一
手接近下巴做拿碗狀」的動作，跟使用者原本提供的另一支同詞條影片（穿橘棕色襯衫的年長示範者，
可能是 `_B` 版本或舊版影片）內容一致。已用使用者提供的版本成功跑過 MiniMax-H3 R2V 生成
（見 skill `minimax-h3-comfyui`）。

## 相關 skill

- `minimax-h3-comfyui` — 拿這裡查到的手語影片當 Ref2VA 參考素材，做角色套用手語動作的影片生成

---

## Conformance Addendum

## When to Use
查詢「台灣手語線上辭典」（TSL@CCU，中正大學語言所建置，https://twtsl.ccu.edu.tw）取得官方手語示範影片、手形/位置資訊、動作文字說明。用在：要找某個中文詞彙對應的正確台灣手語動作、要下載官方手語示範影片（例如拿去做 MiniMax-H3 R2V motion transfer 的參考素材）、或要查手形/位置代碼反查詞彙時。

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
