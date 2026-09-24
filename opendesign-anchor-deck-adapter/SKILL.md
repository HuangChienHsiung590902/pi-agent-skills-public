---
name: opendesign-anchor-deck-adapter
description: 將 OpenDesign／Local Codex 產生的設計系統、HTML、CSS、元件與視覺規範安全套用到 Anchor Deck 本機 HTML deck；當使用者要求「把 OpenDesign 套到 Anchor Deck」、「把設計稿匯入 Anchor Deck」、「同步 OpenDesign 樣式到簡報」、「保留 Anchor Deck 頁面切換但換成 OpenDesign 風格」，或要把完整網頁拆成 Anchor Deck 多頁投影片時使用。即使使用者沒有明說 OpenDesign，只要任務同時涉及 OpenDesign artifact 與 Anchor Deck HTML 整合，也要使用本 Skill。
compatibility: 需要 Anchor Deck MCP；若需讀取 OpenDesign project/artifact，還需要可用的 OpenDesign MCP daemon。可在 Windows 本機 127.0.0.1 工作區執行。
---

# OpenDesign → Anchor Deck 整合

## When to Use

使用者要把 OpenDesign 的設計成果套用到 Anchor Deck 時載入本 Skill，包括：

- 把 OpenDesign／Local Codex 產生的 HTML、CSS、design tokens、元件或素材套進 Anchor Deck。
- 保留 Anchor Deck 的簡報切頁、編輯、匯出能力，但改用 OpenDesign 的視覺樣式。
- 將一個完整 OpenDesign 網頁拆成多個 Anchor Deck `section.slide`。
- 讓 OpenDesign 的設計系統成為 Anchor Deck 後續新增頁面的共同樣式。
- 使用者要求比較、同步、匯入、轉換或「不要破壞目前 deck 功能」的設計整合。

若只是在調整 Anchor Deck 工作區寬度、外層 iframe 或頁面左右留白，另外載入 `anchor-deck-wide-workspace`；本 Skill 負責設計成果的轉換與套用，不取代工作區尺寸 Skill。

## Inputs and Outputs

### Inputs

- 目前 Anchor Deck workspace，由 `anchor-deck_deck_state` 即時確認。
- 目前 Anchor Deck HTML，由 `anchor-deck_get_html` 取得。
- OpenDesign project/artifact、HTML、CSS、設計 token 或使用者提供的視覺規範。
- 使用者指定的保留項目：頁面數量、文字內容、互動、導航、工作區標籤與滿版規則。

### Outputs

- 更新後的 Anchor Deck `index.html`，保留可辨識的 deck 結構與 runtime。
- 明確記錄哪些 OpenDesign 樣式、元件與內容已套用。
- 實際驗證結果：目前 workspace、HTML 結構、頁面切換／瀏覽器畫面，以及仍存在的限制。
- 若 OpenDesign MCP 不可用：不要假稱已取得 artifact；改回報連線證據，並可在使用者提供檔案或樣式後繼續。

## 核心模型

兩個工具的責任不同：

```text
OpenDesign / Local Codex
  └─ 設計系統、視覺方向、HTML、CSS、元件、素材
       ↓ 轉換與保留結構
Anchor Deck
  └─ index.html、deck 導航、頁面編輯、預覽與匯出
```

Anchor Deck 的最低結構是：

```html
<main id="deck" class="deck" data-deck>
  <section class="slide" data-title="封面">
    <!-- 頁面內容 -->
  </section>
</main>
```

除非使用者明確要求重建整個 Anchor Deck runtime，否則不要刪除或改名：

- `#deck`
- `.deck`
- `.slide`
- `data-deck`
- `data-title`
- Anchor Deck 的 runtime script

## Procedure

### 1. 確認目前目標，不要猜檔案

先呼叫：

```text
anchor-deck_deck_state
anchor-deck_get_html
```

以當次 `deckRoot`、`indexPath` 與目前 HTML 為準。不要直接寫入對話中曾出現的舊路徑，也不要把 URL token 寫進 Skill、回覆或新檔案。

若使用者要求從 OpenDesign 讀取 artifact，先確認 OpenDesign MCP 狀態與目前 project context，再使用 `get_artifact`；不要只看 OpenDesign 桌面程式是否開啟就宣稱 MCP 可用。

### 2. 盤點 OpenDesign 輸入

將輸入分類為：

1. **Design tokens**：色彩、字體、間距、圓角、陰影、斷點。
2. **全域 CSS**：`body`、背景、版面容器、排版尺度。
3. **元件 CSS/HTML**：按鈕、卡片、標籤、圖表、導覽。
4. **頁面內容**：標題、段落、圖片、SVG、互動。
5. **資源與 script**：圖片、字體、動畫或第三方依賴。

先判斷是「只換樣式」還是「新增／重排頁面」。優先採小範圍整合，因為保留既有內容與 runtime 比完全覆蓋更不容易破壞 Anchor Deck。

### 3. 建立轉換對照

| OpenDesign 輸入 | Anchor Deck 套用位置 | 注意事項 |
| --- | --- | --- |
| `:root` tokens | Anchor Deck `<style>` 的 `:root` | 避免重複命名；用現有 token 映射 |
| `body`／全域背景 | `html, body` 與 workspace／deck CSS | 不要讓 body margin 重新造成外圍空白 |
| 導覽或頁首 | 每個 `.slide` 內部 | 不要取代 Anchor Deck 外層導航 |
| OpenDesign page | 一個或多個 `.slide` | 每頁要有 `data-title` |
| 卡片／按鈕／圖表 | 對應 class 或新增 scoped class | 避免通用 `button`、`p` 影響 runtime UI |
| 圖片／SVG | 可用相對路徑的 assets 或 inline SVG | 不寫入私人絕對路徑，檢查 MIME 與載入狀態 |
| OpenDesign JS | 僅移植頁面必要互動 | 不覆蓋 Anchor Deck runtime，不重複綁定導航 |

### 4. 套用 CSS 時使用 scoped、可回復的方式

先把 OpenDesign token 正規化到 Anchor Deck：

```css
:root {
  --od-bg: #030913;
  --od-page: #07111f;
  --od-text: #f4f7fb;
  --od-muted: #aab5c5;
  --od-primary: #6ef0dc;
  --od-secondary: #74b9ff;
  --od-accent: #9d8cff;
  --od-page-radius: 28px;
  --od-card-radius: 24px;
}
```

若目前 deck 已有同義 token，優先映射既有名稱，不要同時維護兩套互相衝突的值。頁面元件使用明確 class，例如 `.od-card`、`.od-hero`、`.od-metric`；不要把 OpenDesign 的 `*`、`button`、`p` 等廣泛 selector 原封不動貼入，避免影響工作區工具列、編輯器或其他頁面。

### 5. 把完整 OpenDesign 網頁拆成 deck pages

若 OpenDesign 輸出的是普通網頁：

```html
<body>
  <header>...</header>
  <main>...</main>
</body>
```

轉換為：

```html
<main id="deck" class="deck" data-deck>
  <section class="slide" data-title="首頁">
    <header>...</header>
    <main>...</main>
  </section>
  <section class="slide" data-title="服務介紹">
    ...
  </section>
</main>
```

每頁應避免依賴上一頁的 layout state；頁面要能單獨顯示、縮放與截圖。若內容超過一頁，先依語意拆分，不要單純把整個長網頁塞入一張 slide。

### 6. 保留 Anchor Deck runtime

整合完成後確認既有 runtime 仍存在，例如：

```html
<script src="runtime/deck-stage.js?..." data-html-deck-editor-runtime></script>
<script src="runtime/html-deck-editor.js?..." data-html-deck-editor-runtime></script>
```

實際 query string 由當前工作區提供；不要在 skill 或回答中複製 token。不要移除 `data-html-deck-editor-runtime` 標記，也不要以 OpenDesign router 取代 Anchor Deck 的頁面導航，除非使用者明確要求。

### 7. 寫入前做結構檢查，然後透過 Anchor Deck MCP 儲存

寫入前至少確認：

- `#deck` 仍存在且只保留一個主要 deck root。
- 每一頁都是 `.slide` 並有 `data-title`。
- OpenDesign 的 CSS 沒有把頁面固定回 `960px`、`1440px` 等造成寬度衝突的尺寸。
- 工作區／deck／page 的滿版規則若要調整，依 `anchor-deck-wide-workspace` 的動態尺寸流程處理。
- 所有素材路徑可在目前 workspace 內解析。
- 沒有把憑證、token、cookie 或本機私人絕對路徑寫入輸出。

小改動使用 `replace_text`、`set_attribute` 或 `replace_outer_html`；跨 CSS、HTML 與多頁的協調修改使用 `anchor-deck_save_full_html`。優先使用 MCP 寫入，讓系統建立備份；不要繞過 MCP 直接覆蓋使用者工作區檔案。

### 8. 實際驗證

寫入後再次呼叫：

```text
anchor-deck_deck_state
anchor-deck_get_html
```

檢查：

- `indexPath` 是本次預期目標。
- HTML 含有 `#deck`、`.slide`、`data-title` 與 Anchor Deck runtime。
- 所有預期頁面仍存在，頁面數量與標題正確。
- OpenDesign 的 token／元件 class 已出現在輸出中。
- 瀏覽器畫面沒有空白頁、水平溢出或樣式全失效。
- 需要時切換至少一個頁面，確認 navigation 沒被 OpenDesign script 破壞。
- 若有工作區外框或 deck 邊界需求，另外確認外層 workspace 與內層 page 的間距，不把 iframe 外層問題誤判成 OpenDesign CSS 問題。

若可用瀏覽器工具，截圖或讀取 DOM rect 作為證據；不要只因 `save_full_html` 成功就宣稱視覺整合完成。

## Rules and Limitations

- OpenDesign MCP 與 Anchor Deck MCP 是兩個不同系統；不能假設會自動同步。
- OpenDesign daemon 無法連線時，不要猜 project、artifact 或 design system 內容；先回報實際連線錯誤，必要時請使用者提供匯出的 HTML/CSS。
- Anchor Deck 的 `#deck`、`.slide` 與 runtime 是整合邊界；先保留，再逐步套用 OpenDesign 樣式。
- 不要把完整 OpenDesign app shell、React/Vite router 或 build-time import 原封不動貼進 Anchor Deck，除非已確認能在目前本機 workspace 執行；通常應轉成靜態 HTML/CSS/必要的輕量互動。
- 不要把第三方 CDN、遠端字體或圖片當成必然可用；若需要離線展示，提供 fallback 或將資源放入 workspace。
- 不要使用 OpenDesign 產生的固定畫布尺寸覆蓋 Anchor Deck 的滿版／動態尺寸規則。
- 任何會刪除原有頁面、runtime、互動或整個 workspace 的操作，先向使用者確認。
- 寫入前保留 Anchor Deck 自動備份；不要修改 Anchor Deck MCP 安裝 binary 來解決單一 deck 的樣式問題。

## Pitfalls

- **OpenDesign 能讀到不等於已套用。** 必須在 Anchor Deck HTML 中看到實際 token/class，並以瀏覽器驗證。
- **完整網頁不能直接當 deck。** 必須拆成 `section.slide`，否則 Anchor Deck 可能無法切頁或編輯。
- **全域 selector 互相污染。** OpenDesign 的 `button`、`h1`、`p`、`section` 規則可能改壞工作區 UI；改用 scoped class。
- **只換 CSS 不代表互動仍正常。** 導覽、編輯與匯出 runtime 要獨立驗證。
- **固定寬度造成兩側空白。** OpenDesign 輸出的 `width: 1440px`、`max-width` 或 `margin: auto` 可能重新製造 letterbox；工作區問題使用 `anchor-deck-wide-workspace`。
- **OpenDesign desktop 視窗開啟不等於 MCP daemon 可用。** 要以 MCP 工具或實際 API／status 證據判斷。
- **不要在回覆或 Skill 寫出 URL token。** 只在工具呼叫中使用當次 URL。

## Verification

完成前必須回報：

1. 使用的 Anchor Deck `deckRoot`／`indexPath`（可遮蔽 token；檔案路徑可回報）。
2. OpenDesign 輸入來源：實際讀到的 project/artifact，或明確說明因 daemon 不可用而改用使用者提供的檔案／規範。
3. 實際修改：tokens、CSS、元件、頁面或素材各自修改了什麼。
4. Anchor Deck 結構檢查結果：`#deck`、`.slide`、`data-title`、runtime 是否保留。
5. 瀏覽器／DOM／截圖驗證結果，以及尚未解決的限制。

若只是完成設計規劃、尚未寫入 Anchor Deck，必須清楚標示「尚未套用」，不能把設計建議當成實際修改。
