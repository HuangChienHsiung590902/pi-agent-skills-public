---
name: anchor-deck-wide-workspace
description: 當使用者用 Anchor Deck MCP 建立、匯入、修改或抱怨 HTML deck／簡報在 AI 協作預覽中「太窄、兩側灰色空白很大、工作區沒有用滿、頁面與工作框間距不對」時使用。先區分外層 AI 協作 iframe 工作區與內層 HTML 頁面；讓工作區自動填滿目前可視範圍，頁面只比工作框縮 5px，禁止把 960、1440、1920 等固定寬度當成解法。適用於 Windows Anchor Deck MCP、本機 127.0.0.1 工作區與 Pi 的 anchor_deck MCP 工具。
compatibility: 需要 Anchor Deck MCP；若要驗證外層 AI 協作工作區，需本機 Node.js 與 Playwright 可用。
---

# Anchor Deck 寬版工作區

## When to Use

在下列情況載入本 Skill：

- 使用者要用 Anchor Deck MCP 建立或改 HTML deck / 簡報頁面。
- 使用者說「頁面太窄」、「兩邊灰色空白」、「工作區沒用滿」、「工作框比頁面大」、「寬版」、「滿版」、「自動偵測寬度」。
- 使用者提供 Anchor Deck 畫面截圖，要求調整預覽工作區大小或頁面與外框的間距。
- 使用者要求後續 Anchor Deck 頁面都採用最大可用寬度，而非固定 `960px` / `1440px` / `1920px`。

此 Skill 處理的是 **Anchor Deck AI 協作模式的外層預覽工作區** 與內層 HTML 頁面的尺寸協作；不要把它混同於一般網頁 RWD 或單純 `.slide` 的 CSS。

## Inputs and Outputs

### Inputs

- 當前 Anchor Deck 工作區，由 `anchor_deck_deck_state` 確認。
- 使用者希望的工作區外圍邊界（預設 `8px`）與頁面相對於工作框的邊界（預設 `5px`）。
- 現有 HTML，或使用者要建立的新 deck 內容。

### Outputs

- 外層 AI 協作工作區使用目前可視範圍的最大寬度與高度，不保留左右 letterbox 空白。
- 頁面四邊距離工作區外框預設只有 `5px`，讓頁面最大化且仍看得到工作框。
- 所有尺寸由 `window.innerWidth` / `window.innerHeight` 動態計算，不依賴固定像素設計寬度。
- 寫入後的 Anchor Deck HTML 與實際驗證結果。

## 核心模型：先辨識兩層容器

不要先改 `.slide` 就結束。AI 協作模式有兩個不同層級：

```text
瀏覽器可視區
└─ 外層 AI 協作工作區（parent document）
   ├─ .codex-collab-frame-stage
   ├─ .codex-collab-frame-box
   └─ iframe.codex-collab-frame
      └─ 內層 HTML deck（iframe document）
         ├─ #deck
         └─ .slide
```

使用者看到的「兩側灰色空白」通常由外層 parent document 的協作預覽器造成。該層預設會以固定 `1440 × 810` 設計尺寸縮放置中。只改內層 `#deck`、`.slide` 或 `body`，無法消除外層灰色區。

## Procedure

### 1. 先確認目前工作區與目前 HTML

先呼叫：

```text
anchor_deck_deck_state
anchor_deck_get_html
```

確認 `deckRoot`、`indexPath` 與目前 HTML。不要根據先前對話猜工作區，因為使用者可能已經在 Anchor Deck UI 切換工作區。

### 2. 先判斷問題所在

依畫面或 DOM 判斷：

| 現象 | 根因 | 正確處理層級 |
| --- | --- | --- |
| 白色頁面內側有留白 | HTML 的 `body` / `#deck` / `.slide` CSS | iframe 內層 |
| 白色頁面已很大，但左右仍有大塊灰色區 | AI 協作預覽 iframe 被固定設計尺寸縮放 | parent 外層 |
| 工作框看不到 | 頁面和工作框同尺寸 | 將內層頁面縮小 `5px` |
| 工作框過大、頁面太小 | page inset 過大 | 預設縮為 `5px` |

### 3. 套用動態尺寸規則

禁止將下列數字當成寬版的固定解法：

```text
960px
1440px
1920px
810px
1080px
```

這些值只能出現在對 Anchor Deck 既有行為的說明，不能作為新頁面的固定尺寸。工作區尺寸應來自目前視窗：

```js
const width = Math.max(1, window.innerWidth - WORKSPACE_GAP * 2);
const height = Math.max(1, window.innerHeight - WORKSPACE_GAP * 2);
```

預設尺寸規則：

```js
const WORKSPACE_GAP = 8; // 瀏覽器可視區到工作框
const PAGE_INSET = 5;    // 工作框到白色頁面
```

### 4. 建立或重寫 HTML 時採用此結構

以下模板的重點是：

1. `#deck` 與 `.slide` 是內層頁面，使用工作區的 `100%`。
2. `#deck` 只縮小 `5px × 2`，保留可見工作框。
3. script 在 iframe 內存取同源 `window.parent`，直接調整外層協作預覽 iframe；這才會真正吃滿使用者看到的空間。

保留使用者的實際內容、配色與語意結構，只合併以下版面骨架：

```html
<style>
  :root {
    --workspace-gap: 8px;
    --page-inset: 5px;
    --outer-bg: #eef1f5;
    --page-radius: 20px;
  }

  * { box-sizing: border-box; }
  html, body { width: 100%; height: 100%; min-height: 100%; margin: 0; }
  body { padding: 0; overflow: hidden; background: var(--outer-bg); }

  #deck {
    width: calc(100% - (var(--page-inset) * 2));
    height: calc(100% - (var(--page-inset) * 2));
    max-width: none;
    max-height: none;
    margin: var(--page-inset);
  }

  .slide {
    width: 100%;
    height: 100%;
    min-width: 0;
    min-height: 0;
    max-width: none;
    max-height: none;
    overflow: hidden;
    border-radius: var(--page-radius);
  }
</style>
```

在 `</body>` 前加入下列 runtime。不要把 token、URL 或使用者私密資料寫入 HTML。

```html
<script>
(() => {
  const WORKSPACE_GAP = 8;

  function fitAnchorDeckWorkspace() {
    const root = document.documentElement;
    const innerWidth = Math.max(1, window.innerWidth - WORKSPACE_GAP * 2);
    const innerHeight = Math.max(1, window.innerHeight - WORKSPACE_GAP * 2);

    // 內層 Anchor Deck 固定畫布：改為動態可用尺寸，不讓它再次縮小。
    root.style.setProperty('--codex-fixed-deck-width', `${innerWidth}px`, 'important');
    root.style.setProperty('--codex-fixed-deck-height', `${innerHeight}px`, 'important');
    root.style.setProperty('--codex-fixed-deck-padding', `${WORKSPACE_GAP}px`, 'important');
    root.style.setProperty('--codex-fixed-deck-scale', '1', 'important');

    // AI 協作頁的真正外層容器在同源 parent document。
    if (window.parent === window) return;
    const host = window.parent;
    const box = host.document.querySelector('.codex-collab-frame-box');
    const frame = host.document.querySelector('.codex-collab-frame');
    if (!box || !frame) return;

    const width = Math.max(1, host.innerWidth - WORKSPACE_GAP * 2);
    const height = Math.max(1, host.innerHeight - WORKSPACE_GAP * 2);

    box.style.setProperty('position', 'fixed', 'important');
    box.style.setProperty('left', `${WORKSPACE_GAP}px`, 'important');
    box.style.setProperty('top', `${WORKSPACE_GAP}px`, 'important');
    box.style.setProperty('width', `${width}px`, 'important');
    box.style.setProperty('height', `${height}px`, 'important');
    box.style.setProperty('max-width', 'none', 'important');
    box.style.setProperty('max-height', 'none', 'important');
    box.style.setProperty('outline', '2px dashed rgba(17, 17, 17, .48)', 'important');

    frame.style.setProperty('width', `${width}px`, 'important');
    frame.style.setProperty('height', `${height}px`, 'important');
    frame.style.setProperty('max-width', 'none', 'important');
    frame.style.setProperty('max-height', 'none', 'important');
    frame.style.setProperty('transform', 'none', 'important');
  }

  fitAnchorDeckWorkspace();
  window.addEventListener('load', fitAnchorDeckWorkspace);
  window.addEventListener('resize', fitAnchorDeckWorkspace, { passive: true });
  [50, 200, 700, 1500].forEach((delay) => setTimeout(fitAnchorDeckWorkspace, delay));
})();
</script>
```

### 5. 透過 Anchor Deck MCP 寫入

小改動優先用 `replace_text`、`set_attribute` 或 `replace_outer_html`；需要同時調整樣式與 runtime 時使用：

```text
anchor_deck_save_full_html
```

所有寫入工具會建立 Anchor Deck 備份；不要直接手動覆寫工作區的檔案來取代 MCP 儲存流程。

### 6. 驗證：先驗證 DOM 尺寸，再請使用者看畫面

呼叫：

```text
anchor_deck_deck_state
anchor_deck_get_html
```

若可使用 Node.js + Playwright，建立暫存檢查腳本於：

```text
C:\Users\HCH\AppData\Local\Temp\
```

以 headless Chrome 載入外層 AI 協作 URL，讀取：

```js
const frame = document.querySelector('.codex-collab-frame');
const box = document.querySelector('.codex-collab-frame-box');
const rect = frame.getBoundingClientRect();
```

成功條件：

- 外層 `.codex-collab-frame` 的 `x`、`y` 接近 `8px`。
- 外層 iframe 寬高接近 `window.innerWidth - 16` 與 `window.innerHeight - 16`。
- iframe 的 `transform` 為 `none`，不是 `scale(...)`。
- 內層 `#deck` 與 `.slide` 只比內層工作框各小 `10px`（左右各 `5px`）。
- 若開啟或導航任何瀏覽器頁面，依使用者偏好主動讀取並回報畫面／DOM 結果，不能只完成導航。

最後請使用者強制重新整理：

```text
Ctrl + Shift + R
```

並請他確認：工作區外框仍可見、白色頁面幾乎填滿工作區，兩者間距約 `5px`。

## 新建工作區預設模板與既有頁面

這兩件事必須分開處理：

### 新建工作區

新建空白 deck 的模板由已安裝 Anchor Deck MCP 的執行檔內建產生，不會讀取 `C:\Users\HCH\Downloads\for-ai.md`。要讓所有新工作區寬版，必須修改 MCP 內建模板中的：

```css
#deck { width: 960px; margin: 48px auto; }
.slide { width: 960px; min-height: 540px; }
```

改成動態寬版骨架：

```css
#deck { width: 100%; margin: 0; max-width: none; }
.slide { width: 100%; min-height: 100%; max-width: none; }
```

若完整 server 原始碼不存在，才可對已安裝 EXE 做可還原的 binary patch；必須先備份，並實際重啟與驗證。這不是重新編譯 source build，回報時要說清楚。

### 既有工作區

修改新建模板不會改變已存在的 `index.html`。既有工作區必須重新讀取目前 HTML，透過 `anchor_deck_save_full_html` 或等效 MCP 寫入寬版 CSS/runtime，並再次驗證。

### `for-ai.md`

`C:\Users\HCH\Downloads\for-ai.md` 是 AI 交接快照，不是 Anchor Deck 的新建模板設定。它若含有 `width: 960px`、`margin: 48px auto` 或 `data-design-width="1440"`，可影響 AI 後續輸出的 HTML，但單獨修改它不會改變 Anchor Deck 產品預設，也不會自動寫回目前 workspace。

## 原始碼修改：何時才需要

一般 deck 的尺寸問題先依上述 HTML runtime 修正。若使用者要永久改變 Anchor Deck 產品的預設協作工作區行為，才修改 MCP host／server 的完整原始碼；不要誤以為只有核心 editor repo 一定包含 MCP server。

已觀察到的產品行為：外層協作預覽預設會以固定設計尺寸建立：

```js
var designWidth = 1440;
var designHeight = 810;
```

並使用：

```text
.codex-collab-frame-stage
.codex-collab-frame-box
iframe.codex-collab-frame
```

永久修正應改為讓外層 server 產生的協作頁面採用 `window.innerWidth` / `window.innerHeight`，而非固定 `1440 × 810` 後做 `transform: scale(...)`。修改前必須：

1. 找到與已安裝 Anchor Deck MCP 版本完全對應的 server / desktop source。
2. 建立 Git 分支或可還原備份。
3. 修改外層協作頁 HTML 的 `fitCodexCollabFrame()`。
4. 建置新的 Windows 安裝包或可執行檔。
5. 安裝、重啟 MCP host，並以上述 Playwright DOM 檢查驗證。

`D:\html-deck-editor-main` 是核心編輯器來源，但未必包含 Windows MCP server 的完整發行來源；不要只修改它就宣稱已改到正在執行的 `Anchor Deck MCP.exe`。

## UI 工作區按鈕與說明面板

「MCP 工作區」選單可移動、位置持久化、說明隱藏的完整流程已獨立成 `anchor-deck-movable-workspace-panel` skill（含 bundled build/install/CDP 腳本）。本節只保留寬版 skill 需要的原則；實際操作請改用該 skill。

使用者若要改右上角 `MCP 工作區` 按鈕，先分開處理「外觀」、「行為」與「面板內容」：

- 預設應保留矩形按鈕；不要把可讀的工作區面板做成圓形或橢圓。
- 若使用者要移動按鈕，應只讓按鈕本身可拖曳，面板開啟後仍保持矩形、可捲動、可閱讀。
- 拖曳事件必須使用事件委派或 `MutationObserver`，因為 `workspace-ui.js` 會在啟動後動態建立按鈕；頁面初始載入時直接 `querySelector` 可能找不到按鈕。
- 拖曳時要阻止同一次 pointer 操作誤觸 click；拖曳結束後清除臨時旗標。
- 「Codex / Claude / WorkBuddy」client chips 與「選擇元素」說明可以用精準 CSS 隱藏；不要為了隱藏說明而替換整個 `ensureButton()` 函式。
- 功能球、拖曳與說明隱藏是獨立需求；使用者若只要求拖曳，不要擅自改成圓形。
- 拖曳位置必須持久化：寫入 `localStorage["anchor-deck-ui-pos"]`（`{"b":[x,y],"p":[x,y]}`，`b` 為按鈕、`p` 為面板），拖曳結束才寫入；頁面載入與面板開啟時讀取並套用，讓重新整理與重啟後位置還原。
- 改 EXE 前先做 CLI `version` 和獨立短暫啟動 smoke test；啟動失敗立即回復上一個可工作的備份。

## Rules and Limitations

- 先改外層協作 iframe，再改內層頁面；不要反覆只調整 `.slide`。
- 頁面預設與工作框保留 `5px`；工作框與瀏覽器可視區預設保留 `8px`。
- 不可用固定 `960px` / `1440px` / `1920px` 作為「最大寬版」的最終解法。
- 只有使用者明確要特定投影片比例／輸出尺寸時，才能使用固定設計尺寸。
- `window.parent.document` 僅在同源 `127.0.0.1` 的 Anchor Deck 協作 iframe 可用；若改成跨網域嵌入，必須改用 postMessage 與 host 端支援。
- 寫入 deck 必須優先透過 Anchor Deck MCP 工具，保留自動備份。
- 任何改動已安裝的 `.exe`、重建安裝包或替換產品程式前，要先備份並向使用者說明會影響目前安裝。
- Bun compiled EXE 的 binary patch 必須維持檔案長度，且不得直接假設替換整個 bundled JavaScript 函式是安全的；優先使用等長 CSS 或既有 script 空間，並保留可回復的原始 EXE。

## Pitfalls

- **把頁面變寬，不等於把工作區變寬。** 截圖若灰邊在白色頁面外，根因多半在 outer collaboration frame。
- **只設 `--codex-fixed-deck-width` 不夠。** 外層 collaboration page 還可能用 `iframe { transform: scale(...) }` 將它再次縮回。
- **直接把工作框和頁面都滿版會看不到外框。** 使用者要能看見工作框時，保留 `5px` 的內層 page inset。
- **不要宣稱成功卻只看文字。** 要檢查 outer iframe 的 DOM rect 或實際截圖。
- **目前工作區可能隨時被使用者切換。** 每次寫入前重新讀取 `anchor_deck_deck_state`。
- **不要把 MCP token 寫進回覆、Skill 或提交內容。** 本機 URL 若含 token，僅在工具呼叫內使用。

## Verification

1. `anchor_deck_deck_state` 回傳的 `indexPath` 與本次寫入目標一致。
2. `anchor_deck_get_html` 確認含有動態 `fitAnchorDeckWorkspace()` 或等效實作。
3. 外層 iframe DOM 的可視矩形填滿工作區，四周約 `8px`。
4. 頁面相對內層工作框僅保留 `5px` 邊界。
5. 調整瀏覽器寬度後，外層 iframe 與頁面皆重新計算，不回退成固定 `1440 × 810`。
6. 使用者重新整理後確認畫面符合預期。
