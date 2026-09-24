---
name: anchor-deck-multi-page-shopping-site
description: >
  使用 Anchor Deck MCP 把現有 HTML deck／簡報／演示稿改造成可操作的多頁購物網站示範；當使用者說「把這份演示稿變成購物網站」、「做多頁電商 demo」、「用 Anchor Deck 生成購物網站」、或要求首頁、商品列表、商品詳情、購物袋、結帳、關於我們等頁面與互動時，務必使用本 Skill。涵蓋目前 Anchor Deck 工作區檢查、整頁 HTML 重構、SPA 分頁路由、商品／購物袋狀態、響應式視覺設計，以及 Playwright 互動驗證；可與 anchor-deck-wide-workspace 搭配使用。
compatibility: 需要 Anchor Deck MCP；建議搭配 Playwright／瀏覽器工具驗證互動。
---

# Anchor Deck 多頁購物網站生成

## When to Use

當使用者要把 Anchor Deck 中的第一份簡報、HTML PPT、演示稿或既有 deck 改造成購物網站時使用。即使使用者沒有說「SPA」，只要需求包含商品展示、商品詳情、購物車／購物袋、結帳或多個頁面，也應載入本 Skill。

本 Skill 產生的是 **Anchor Deck 內可預覽、可點擊的單一 HTML 多頁網站示範**。每個頁面仍以 `section.slide.page` 表示，透過 JavaScript 切換顯示；這樣可以同時保留 Anchor Deck 的頁面辨識能力與網站式互動，不要把每個頁面拆成外部檔案，除非使用者明確要求真正的多檔案網站。

## Inputs and Outputs

### Inputs

- 目前 Anchor Deck 工作區；不可依對話記憶猜測目標工作區。
- 使用者指定的品牌、商品、色彩、語言、圖片或既有簡報內容；若未指定，先建立一個有明確主題的示範品牌與商品資料。
- 頁面範圍。預設至少包含：首頁、商品列表、商品詳情、購物袋、結帳、關於我們。
- 可選的互動需求：導覽、商品點擊、數量增減、加入購物袋、訂單摘要、完成提示、響應式版面。

### Outputs

- 透過 Anchor Deck MCP 寫入目前工作區的完整 HTML，保留自動備份。
- 一個有一致視覺系統的多頁購物網站示範，而非只有靜態投影片。
- 可由導覽列與 CTA 切換頁面；商品可進入詳情並加入購物袋；購物袋數量與摘要會更新。
- 實際驗證紀錄：工作區與 index path、頁面數量、瀏覽器互動結果、剩餘非阻斷性錯誤。

## Design Direction

先從主題推導設計，不要套用無關的通用儀表板模板。若使用者沒有指定方向，採用以下預設：

- 介面語言使用繁體中文，文案簡短、自然、以使用者看得懂的動詞命名。
- 以品牌主色、底色、文字色、輔助色建立 CSS custom properties；避免全頁到處散落色碼。
- 預設使用圓角、柔和陰影、pill 按鈕、清楚的商品卡片層次，做出親切而有品牌感的選物店風格。
- 對商品圖片沒有可靠素材時，使用 CSS 圖形、色塊、符號或漸層作為視覺佔位，不要假造不存在的外部圖片 URL。
- Hero 只保留一個主要訊息與一個主要 CTA；商品列表、購物袋與結帳頁優先確保資訊清楚。
- 保留手機窄版：商品網格降為兩欄，主要雙欄區塊降為單欄，導覽列可收斂。

## Procedure

### 1. 讀取目前工作區與 HTML

先呼叫：

```text
anchor_deck_deck_state
anchor_deck_get_html
```

確認 `deckRoot`、`indexPath`、目前 HTML 大小與既有頁面。若使用者說「第一份演示稿」，仍以工具回傳的目前工作區為準，不要自行搜尋或覆蓋其他 workspace。

若需要寬版工作區，另外載入 `anchor-deck-wide-workspace`；它負責外層協作 iframe 與內層 page inset，本 Skill 負責網站內容與互動。

### 2. 設計資訊架構

至少建立這些 `data-page`：

```text
home       首頁
shop       探索商品／商品列表
detail     商品詳情
cart       購物袋
checkout   結帳
about      關於我們
```

每頁使用：

```html
<section class="slide page" data-page="shop" data-title="探索商品">
  ...
</section>
```

只讓一頁有 `active`。所有頁面共用一致的品牌導覽列，導覽按鈕以 `data-route="home"` 等方式標記，讓事件委派能統一處理。

### 3. 建立可重用的 HTML／CSS 骨架

保留 Anchor Deck 的動態尺寸骨架：

```css
html, body { width: 100%; height: 100%; margin: 0; }
body { overflow: hidden; }
#deck {
  width: calc(100% - (var(--page-inset) * 2)) !important;
  height: calc(100% - (var(--page-inset) * 2)) !important;
  margin: var(--page-inset) !important;
  max-width: none !important;
  max-height: none !important;
}
.slide {
  width: 100% !important;
  height: 100% !important;
  min-width: 0 !important;
  min-height: 100% !important;
  padding: 0 !important;
  overflow: hidden !important;
}
.page { display: none; flex-direction: column; overflow: auto; }
.page.active { display: flex; }
```

建議元件：

- `topbar`：促銷或品牌訊息。
- `nav`：品牌、首頁、商品列表、關於我們、搜尋／購物袋。
- `hero`：品牌主張、主要 CTA、CSS 商品視覺。
- `product-grid`／`product-card`：商品視覺、名稱、描述、價格。
- `detail-layout`：商品視覺、規格、價格、數量控制與加入購物袋。
- `cart-layout`／`summary`：購物袋清單、運費、小計、總計。
- `checkout-grid`／`form-card`：寄送資訊與示範訂單摘要。
- `toast`：加入購物袋及送出訂單的回饋。

商品卡片應使用按鈕或可鍵盤操作的元素，不要只在 `div` 上綁點擊事件。頁面內的假圖片必須有意義的替代文字或可理解的文字標籤。

### 4. 實作最小可用互動

使用單一 IIFE，避免把全域變數散落在 deck 中。最小狀態可包含：

```js
const cart = [];
let quantity = 1;
const pages = [...document.querySelectorAll('.page')];
```

實作下列行為：

1. `show(route)`：隱藏所有 page，只顯示指定 page；更新 nav active；切換後把該頁 scrollTop 歸零。
2. `[data-route]` 事件委派：首頁、商品列表、關於我們、購物袋、結帳與 CTA 都能切換。
3. `[data-product]`：點商品進入詳情頁；至少有一個代表性商品能真正加入購物袋。
4. 數量減少不得小於 1；增加後要同步畫面。
5. 加入購物袋時合併相同商品，更新所有購物袋 badge，顯示 toast。
6. `renderCart()`：重繪購物袋項目、小計、運費與總計；空購物袋要提供返回商品頁的方向。
7. 送出訂單只做示範提示，不要宣稱真的完成付款或真的建立後端訂單。

計算金額時先完成數值運算再格式化，例如：

```js
const lineTotal = item.price * item.qty;
const total = subtotal + shipping;
lineTotal.toLocaleString();
```

不要寫成 `item.price * item.qty.toLocaleString()`，避免字串格式化造成錯誤或非預期結果。

### 5. 透過 Anchor Deck MCP 寫入

如果是跨頁面、樣式與 JavaScript 的整體改造，使用：

```text
anchor_deck_save_full_html
```

不要直接用檔案工具覆寫 `index.html`，因為 Anchor Deck MCP 會建立備份並維持 bridge 狀態。只有文字小修才使用 `anchor_deck_replace_text`；若要新增互動，不要嘗試用多次小替換拼湊而破壞 HTML。

不要把 `codexToken`、私密 URL、API key 或使用者憑證寫進 HTML、Skill 或回覆。開啟工作區時只使用 `anchor_deck_open_workspace` 回傳的當次連結。

### 6. 開啟並驗證預覽

寫入後依序呼叫：

```text
anchor_deck_deck_state
anchor_deck_list_elements
anchor_deck_open_workspace
```

使用瀏覽器開啟當次工作區連結，至少驗證以下流程：

1. 首頁出現品牌、主標題、主要 CTA 與商品卡片。
2. 點「探索商品」切換到商品列表，商品數量與分類控制可見。
3. 點一張商品卡片進入商品詳情。
4. 點加入購物袋，badge 從 0 更新，且有 toast。
5. 點購物袋，商品、價格、小計與總計可見。
6. 點結帳，表單與訂單摘要可見；送出訂單顯示示範提示。
7. 點關於我們與返回首頁，頁面不會空白或重載失敗。
8. 調整瀏覽器寬度，確認版面不溢出且仍可操作。

若使用瀏覽器，還要讀取 console error；`favicon.ico` 404 這類非阻斷性錯誤可記錄，但不能把有 JavaScript 例外、空白頁或按鈕無法操作的結果宣稱為成功。

## Rules and Limitations

- 先確認目前 workspace，再寫入；不可硬編寫先前工作區路徑。
- 以 Anchor Deck MCP 寫入並保留備份；不要直接破壞使用者原始 HTML。
- 「多頁」預設是同一 HTML 內的可操作 page routing，不是偽造多個瀏覽器分頁。
- 購物與結帳是前端示範狀態，不是正式金流；介面文案必須明確避免誤導。
- 不加入未驗證的外部 CDN、追蹤器、付款 SDK 或後端 API；使用純 HTML/CSS/JavaScript 即可完成示範。
- 不把真實個資、token、密碼或 API 憑證放入商品資料、HTML、console 或 Skill。
- 不用固定 `960px`、`1440px` 或 `1920px` 解決 Anchor Deck 寬度；工作區尺寸由 `anchor-deck-wide-workspace` 的動態規則處理。
- 不要為了追求視覺塞入大量動畫；互動應服務於瀏覽與購物流程。
- 編輯器辨識到的文字／視覺元素必須仍是合理的 DOM，不要把整個網站變成一張 SVG 或單一圖片。

## Pitfalls

- **只做首頁，不做流程：** 使用者說購物網站時，至少要能走首頁 → 商品列表 → 詳情 → 購物袋 → 結帳。
- **只改文字不改互動：** HTML 看起來像電商不代表能操作；必須用瀏覽器實際點擊。
- **多個頁面同時顯示：** 檢查 `.page.active` 是否只有一個；初始化時明確指定首頁。
- **badge 不同步：** 只更新首頁 badge 會讓其他頁面顯示錯誤；使用 `querySelectorAll('.cart-badge, #cart-badge')` 統一更新。
- **金額變成字串：** 先乘法與加法，再呼叫 `toLocaleString()`。
- **導覽按鈕被 Anchor Deck 浮動控制項遮住：** 可先關閉或收合工作區浮動面板，再進行點擊驗證；不要因此修改網站定位。
- **頁面內容超出畫布：** `.page` 應可滾動，導覽列可 sticky；窄視窗時要降欄而非硬塞固定寬度。
- **虛假的成功訊息：** 示範結帳只能顯示「示範訂單已送出」之類提示，不得宣稱已付款、已出貨或已連接真實後端。

## Verification

完成前確認：

1. `anchor_deck_deck_state.indexPath` 是本次修改的目標。
2. `anchor_deck_get_html` 可看到至少六個 `section.slide.page`，並包含 `data-page` 與 `data-title`。
3. HTML 包含 `#deck` 動態尺寸規則與 `fitWorkspace()` 或等效的 Anchor Deck runtime。
4. 初始只顯示一個 `.page.active`，其餘頁面可由 `[data-route]` 切換。
5. 商品詳情、數量、加入購物袋、badge、購物袋摘要及結帳提示均以瀏覽器實際點擊驗證。
6. `anchor_deck_list_elements` 能辨識品牌、標題、導覽、商品名稱與主要按鈕；不能只剩一個不可編輯的圖片元素。
7. 瀏覽器 console 沒有 JavaScript 例外；非阻斷性 favicon 404 必須單獨說明。
8. 回報時列出：修改的工作區、頁面清單、已驗證流程、剩餘限制與使用者可再次整理的操作。

## Report Structure

完成後用繁體中文簡短回報：

```text
已用 Anchor Deck MCP 將 <workspace> 改成多頁購物網站。

頁面：首頁／商品列表／商品詳情／購物袋／結帳／關於我們
互動：導覽切換、商品詳情、數量調整、加入購物袋、摘要計算、結帳提示
驗證：<實際點擊與 console 結果>
限制：<例如僅為前端示範、favicon 404 等>
```
