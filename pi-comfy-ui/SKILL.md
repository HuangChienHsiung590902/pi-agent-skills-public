---
name: pi-comfy-ui
description: >-
  `pi` coding agent 的 TUI 美化擴充套件 `pi-comfy-ui`（npm 套件，作者 adanft，MIT，
  https://github.com/adanft/pi-comfy-ui）：只負責輸入框（editor）背景色/側邊框樣式，以及
  互動面板（設定、模型選擇器、確認框、ask_user_question 等）背景色/側邊框樣式，不碰任何
  padding 設定與功能邏輯。本機已安裝於 `~/.pi/agent/npm/node_modules/pi-comfy-ui`
  （見 `~/.pi/agent/settings.json` 的 `packages`），原始碼副本在
  `~/Projects/pi-comfy-ui`（git clone，已對 origin push 過效能優化）。
  本機另外加了 upstream 沒有的「輸入框固定在終端機底部」功能（`extensions/bottom-anchor.ts`）。
  當使用者提到 pi-comfy-ui、pi 的輸入框/面板美化、輸入框置底/貼底、
  `customMessageBg`/`userMessageBg` 主題色、或要研究/修改/優化這個套件的原始碼、
  或更新後要還原本機自訂修改時使用。
---

# pi-comfy-ui — pi TUI 輸入框與互動面板美化擴充

## 這是什麼

`pi-comfy-ui`（https://github.com/adanft/pi-comfy-ui）是給 `pi` coding agent
（`@earendil-works/pi-coding-agent`）用的 TypeScript 擴充套件，純粹做**視覺樣式**，
不影響任何功能邏輯：

1. **輸入框（editor）**：背景塗上主題色 token `customMessageBg`，並把原本上下+左右的框線
   改成只留左右兩側的「側欄」（`┃`），視覺上更簡潔。
2. **互動面板**：設定選單、模型選擇器、確認框、select、`ask_user_question` 等彈出面板，
   背景塗上主題色 token `userMessageBg`，同樣把上下框線改成左右側欄樣式。
3. **完全不管 padding**：不讀/寫 `editorPaddingX`、`outputPad`，這兩個是 pi 原生設定，
   使用者要自己在 `~/.pi/agent/settings.json` 或 `.pi/settings.json` 調整。
   README 明確建議搭配值：`{"editorPaddingX": 1, "outputPad": 1}`。
4. 若偵測到已有其他擴充套件自訂了 editor（`ctx.ui.getEditorComponent()` 已被非
   pi-comfy-ui 的 factory 佔用），會自動避讓、不覆蓋，並跳出警告通知。

## 本機狀態

### 目前配置（已確認可用）

**直接跑 clone，不經過 npm 安裝。** 這是目前的正確狀態：

`~/.pi/agent/settings.json`（全域）：
```json
"packages": [
  "npm:omniroute-pi-ext-integration",
  "npm:pi-playwright",
  "npm:pi-mcp-adapter",
  "~/Projects/pi-comfy-ui"
]
```

- 已移除 `"npm:pi-comfy-ui"`（舊設定備份在 `settings.json.bak-<timestamp>`）
- 已刪除 `~/.pi/settings.json`
- **好處：改 clone 直接生效，不用同步到 `node_modules`，`pi update` 也不會覆蓋**

路徑寫 `~/...` 是安全的：pi 的 `normalizePath()` 預設 `expandTilde: true`
（`dist/utils/paths.js:47-53`）。

- **原始碼**：`~/Projects/pi-comfy-ui`（`git clone` 自
  `https://github.com/adanft/pi-comfy-ui.git`，`origin` 有 push 權限，fork/PR 尚未建立）。
  main 分支上有兩個本機 commit：效能優化 + bottom-anchor。
- npm 安裝版**已完全移除**（`pi uninstall npm:pi-comfy-ui`）：`node_modules/pi-comfy-ui`、
  備份目錄、`~/.pi/agent/npm/package.json` 的依賴項、lock 紀錄全清。
  現在 clone 是唯一來源。

### 為什麼不用 `pi install -l`

`pi install -l` 把設定寫進 `<cwd>/.pi/settings.json`，**專案設定是看 cwd 的**——
在家目錄執行就只有從家目錄開 pi 才生效，換任何其他目錄就完全沒有 comfy-ui。
寫全域 `packages` 才是任何目錄都生效。

（參考 `core/package-manager.js:680-694`：專案與全域 packages 會合併，同一套件兩邊都有
時由 `dedupePackages()` 去重，但**不同來源形式（npm: vs 本地路徑）不會被視為同一套件**，
會載入兩次。）

## 更新後驗證（最常用）

現行配置直接跑 clone，`pi update` 不會碰到它，**一般不需要還原**。
真正會弄丟修改的是：`git pull` 衝突、重新 clone、或誤改回 `npm:pi-comfy-ui`。

```bash
# 驗證本機修改都還在（不寫檔）
bash ~/.claude/skills/pi-comfy-ui/scripts/restore-local-patches.sh --check

# 拉 upstream 新版並重放本機 commit
bash ~/.claude/skills/pi-comfy-ui/scripts/restore-local-patches.sh --rebase
```

腳本會：驗證 8 個本機修改的標記字串 → 跑 typecheck + test →（非 `--check` 時）
備份並同步到 `node_modules`。

| 參數 | 行為 | 現行配置下是否需要 |
|---|---|---|
| `--check` | 只驗證，不寫任何檔案 | ✅ 常用 |
| `--rebase` | `git pull --rebase` 後驗證 + 同步 | ✅ 拉新版時用 |
| （無） | 驗證 + 同步到 `node_modules` | ❌ 現在沒 npm 安裝版，會報錯退出 |

可用環境變數覆寫路徑：`PI_COMFY_SRC`、`PI_COMFY_DEST`。

**改完必須完整重開 pi**（`/reload` 不夠，因為改的是 `session_start` 路徑）。

若腳本報 `MISSING:`，代表某項本機修改真的掉了，照下面「還原步驟」處理。

## 專案結構

```
extensions/
  comfy-ui.ts                  入口，session_start 時掛 editor + patch 面板渲染
  bottom-anchor.ts             【本機新增】輸入框置底：在 editor 上方補空行

  editor-input.ts              PanelEditor：繼承官方 CustomEditor，畫側欄+背景
  ansi.ts                      ANSI 處理：stripAnsi、createBackgroundPainter
  interactive-panel-render.ts  patchPanelRender：patch TUI/Container 的 addChild、showOverlay
  component-render-patch.ts    對已知元件 class 的 prototype.render 做 monkey-patch
  known-pi-panels.ts           列出哪些 Pi 匯出的元件 class 可套樣式（EXPORTED_STYLABLE_COMPONENTS）
                                + 哪些「未匯出」元件靠 call-stack 字串比對辨識
  chat-panel-patches.ts        InteractiveMode 特定方法（changelog/hotkeys/reload 面板）的包裝
  ask-user-question-panel.ts   處理第三方擴充 `@juicesharp/rpiv-ask-user-question` 的自訂 UI
  panel-painter.ts             核心繪製：偵測動態框線、填色、框線轉側欄
  panel-render-state.ts        用 Symbol.for(...) 存 patch state，避免重複套用
  panel-render-types.ts        共用型別
tests/
  content-padding-contract.test.mjs   守門測試：runtime 程式碼禁止再出現舊的 padding 相關識別字
  interactive-panel-render.test.mjs   面板渲染行為測試（最大，482 行）
  panel-editor-padding.test.mjs       PanelEditor 的 render/padding 行為測試
```

## 實作技巧（值得記住的關鍵設計）

因為 Pi 目前**沒有官方的「面板渲染」擴充 API**，作者用「已知路徑 monkey-patch」補足：

- **正規路徑**：`editor-input.ts` 透過官方公開 API `ctx.ui.setEditorComponent()`，
  繼承 `CustomEditor` 重寫 `render()`，這塊是唯一走官方擴充點的部分。
- **已知匯出元件**：`known-pi-panels.ts` 的 `EXPORTED_STYLABLE_COMPONENTS`
  列出一批可以直接 import 到的元件 class（`ModelSelectorComponent`、
  `SettingsSelectorComponent`、`ThemeSelectorComponent` 等），對這些 class 的
  `prototype.render` 直接 patch。
- **未匯出的內部元件**：如 `ScopedModelsSelectorComponent`、`TrustSelectorComponent`，
  無法 import，改用 `new Error().stack` 字串比對呼叫路徑（例如 stack 含
  `showModelsSelector`）來判斷是否要套樣式，屬於 hacky 但有防呆
  （`shouldStyleKnownPiCoreComponent`）。
- **掛載時機攔截**：patch `TUI.prototype.addChild`、`showOverlay`、
  pi-tui 的 `Container.prototype.addChild`，在元件被掛進畫面時順便套 patch。
- **重複套用防呆**：所有 patch 一律用 `Symbol.for("pi-comfy-ui.xxx")` 當旗標，
  `hasOwnProperty` 檢查避免重複 wrap。
- **明確的不作為承諾**：README/測試（`content-padding-contract.test.mjs`）都白紙黑字
  禁止 runtime 程式碼碰 `patchTuiRender`、`resolvePaddingX`、`__piComfyUi` 等字樣，
  確保它只做樣式、不碰渲染寬度和根層 render。

## 本機新增功能：輸入框固定在底部（bottom-anchor）

**upstream 沒有這個功能**，是本機加的。commit：`feat: anchor editor to bottom of terminal`。

### 問題

Pi 的 TUI 是「內容流」渲染：`TUI extends Container`，`render()` 把所有 children
依序 render 後串接。children 順序是 header → chat → status → widgetAbove →
**editorContainer** → widgetBelow → footer。對話短的時候總行數 < 終端高度，
輸入框就浮在畫面中間。

### 作法

`extensions/bottom-anchor.ts` 導出 `anchorEditorToBottom(tui, editor)`：包住 `render`，
找出含有 editor 的 child index，分成「上半」和「下半」，若總行數 < `terminal.rows`
就在中間塞 `height - total` 行空白。內容超過螢幕高度時完全不介入。

### 三個踩過的坑（很重要，改這塊必看）

**1. 不能 patch `TUI.prototype.render`**

第一版 patch prototype，**測試全過但實際完全沒作用**。原因在 Pi 自己原始碼的註解：

```js
// interactive-mode.js
// Use duck typing since instanceof fails across jiti module boundaries
```

擴充經由 jiti 載入，`import { TUI }` 拿到的 class 跟 Pi 內部實際 new 的**不是同一個
模組實例**，patch 到的是一個沒人用的 class。→ 改成 patch **factory 收到的 tui 實例**。

（同理：寫測試時若用 `require("@earendil-works/pi-tui")` 拿 Container、卻用 `jiti` 載入
受測模組，兩邊也不是同一個實例。現行測試改成自己建最小 stub，不依賴 class 識別。）

**2. editor 會被換掉，旗標不能只是 `true`**

`setCustomEditorComponent()` 會 `editorContainer.clear()` 再 `addChild(newEditor)`。
若旗標存 `true`、closure 抓著舊 editor，重新呼叫時會直接 return，而舊 editor 已不在
樹上 → `anchorIndex === -1` → 靜默失效。這在**全域 + 專案兩份同時安裝**時必發。
→ 旗標改存 `{ editor }` state 物件，重複呼叫改成 retarget。

另外 editor 是在 factory return **之後**才被 `addChild` 進 container，所以要用
`queueMicrotask` 延後套用，否則找不到。

**3. 千萬別用 `record.render.bind(tui)` 抓「原始 render」——`tui` 可能是 Proxy**

2026-08 生產環境撞到 `RangeError: Maximum call stack size exceeded`，堆疊在
`renderWithBottomAnchoredEditor` 與 pi 內部 `interactive-mode.js` 的一個 `Proxy` trap
之間無穷互相呼叫。根因：`anchorEditorToBottom(tui, editor)` 收到的 `tui` 不一定是
真正的 `TuiMainScreen` 實例，可能是 pi 的 `createInteractiveTuiReference(getTui)`
包出來的**穩定引用 Proxy**（session 換 renderer 時用來保持外部持有的引用不失效）。

這個 Proxy 的 `get` trap 對函式屬性回傳的**不是**真正的函式引用，而是一個「每次呼叫都
重新 `Reflect.get(getTui(), property)` 現查」的包裝函式：

```js
// interactive-mode.js: createInteractiveTuiReference
return (...args) => {
    const tui = getTui();
    const method = Reflect.get(tui, property, tui);   // 每次呼叫都重新查找！
    return Reflect.apply(method, tui, args);
};
```

原本的寫法 `const baseRender = record.render.bind(tui)` 若 `record` 是這個 Proxy，抓到的
正是這個「動態查找」包裝函式，`.bind()` 對它毫無保護作用。當 patch 接著執行
`record.render = renderWithBottomAnchoredEditor`（透過 Proxy 的 `set` trap 寫到底層真實
renderer 實例上）之後，`baseRender` 每次被呼叫時，內部重新查到的 `tui.render` 已經是
**自己**的 wrapper，於是 `baseRender()` 等於呼叫自己 → 無穷遞迴，直到爆堆疊。第一次
加「重入旗標」防呆完全沒用，因為旗標保護的是 wrapper 本體，觸發時仍然會呼叫這個
會自我遞迴的 `baseRender`，只是把症狀延後而已。

**正確修法**：不要透過（可能被代理的）實例抓原始方法，改從 **prototype** 拿：

```ts
const proto = Object.getPrototypeOf(record) as { render?: RenderLines } | null;
const protoRender = proto?.render;
if (typeof protoRender !== "function") return false;
const baseRender = protoRender.bind(record);
```

Proxy 的 `getPrototypeOf` trap 會轉發到真正的底層 renderer，所以 `protoRender` 是真正
穩定、不受動態查找影響的 `TuiBase.prototype.render`（或同層級）方法，之後即使
`record.render` 被換成我們自己的 wrapper 也不影響它。重入旗標保留當作額外防線
（防的是「這個 wrapper 之後又被別的擴充套一層」的情境，不是本案根因）。

`tests/bottom-anchor.test.mjs` 新增了兩個 regression test，用完整還原的
`createInteractiveTuiReference` Proxy 結構（同樣的 `get`/`set`/`has`/`getPrototypeOf`
trap）測試：一個走「有東西可 anchor」分支，一個走「editor 是第一個 child、直接
fallback 呼叫 `baseRender`」分支——**第二個才是真正會觸發這個 bug 的分支**，第一版
只測第一種分支時完全測不出問題（因為那個分支不會呼叫到 `baseRender`）。改動這塊
時務必連這兩個 Proxy regression test 一起跑，不能只看「有 anchor 效果」就當過關。

### 關閉

`PI_COMFY_BOTTOM_EDITOR=0`。

### 注意

這是目前唇一 patch root render 的地方，README 原本寫「不 patch root TUI render」，
已一併更新該段。若要發佈到 npm 算行為變更，建議升 minor 版本。

## 本機已做的優化（perf，2026 年整理）

以下 4 個檔案的修改**只影響效能，不改變任何輸出/行為**，已通過
`npm run typecheck` + `npm test`（3 個測試檔全過）：

| 檔案 | 問題 | 修法 |
|---|---|---|
| `extensions/ansi.ts` | `stripAnsi()` 每次呼叫都在函式體內建立新的正則字面量，每行文字每幀都重新編譯 regex | 正則提到模組層級常數 `ANSI_ESCAPE_PATTERN` |
| `extensions/editor-input.ts` | `isHorizontalEditorBorder()` 同樣每次 render 都重建正則 | 提到模組層級 `HORIZONTAL_EDITOR_BORDER_PATTERN` |
| `extensions/panel-painter.ts` | `isDynamicBorder()` 同上，且逐行掃描面板內容時呼叫 | 提到模組層級 `DYNAMIC_BORDER_PATTERN` |
| `extensions/known-pi-panels.ts` | `isKnownExportedComponent()` 在元件掛載熱路徑（`patchMountedComponent`）用 `Array.some()` 線性掃描 16 個 class（O(16)） | 額外建 `EXPORTED_STYLABLE_COMPONENT_SET`（`Set`），改 `.has()` 做 O(1) 查找；原陣列保留供 `for...of` 遍歷用 |

**安全性依據**：帶 `g` flag 的正則用在 `.replace()` 時，規範保證每次呼叫前重置
`lastIndex`，模組層級共用不會有跨呼叫殘留 state 的問題；其餘兩個正則本就無狀態
（只用 `.test()`）。

若要重現這批優化，改動位置精確如下（可直接 diff 對照）：

```ts
// ansi.ts
const ANSI_ESCAPE_PATTERN = /\u001b\[[0-?]*[ -/]*[@-~]/g;
export function stripAnsi(text: string): string {
  return text.replace(ANSI_ESCAPE_PATTERN, "");
}

// editor-input.ts
const HORIZONTAL_EDITOR_BORDER_PATTERN = /^[─ ↑↓0-9more]+$/;

// panel-painter.ts
const DYNAMIC_BORDER_PATTERN = /^─+$/;

// known-pi-panels.ts
const EXPORTED_STYLABLE_COMPONENT_SET = new Set(EXPORTED_STYLABLE_COMPONENTS);
export function isKnownExportedComponent(component: Component): boolean {
  return EXPORTED_STYLABLE_COMPONENT_SET.has(
    component.constructor as (typeof EXPORTED_STYLABLE_COMPONENTS)[number],
  );
}
```

**沒有動的部分**：`component-render-patch.ts` 的 `patchMountedComponent` 已經用
短路求值先判斷便宜的 `needsPiCoreStack()`，只有極少數情況才呼叫昂貴的
`new Error().stack`，設計已經不錯，不需要改。

## 還原步驟（腳本報 MISSING 或情況複雜時）

### 先搞清楚哪一份在生效

這是最常見的混淆來源。確認指令：
```bash
grep -A6 packages ~/.pi/agent/settings.json   # 全域
cat ~/.pi/settings.json 2>/dev/null           # 專案（應該不存在）
```

正確狀態：全域含 `"~/Projects/pi-comfy-ui"`，**不**含 `"npm:pi-comfy-ui"`，
且 `~/.pi/settings.json` 不存在。

開着 pi 時起始畫面的 `[Extensions]` 列也會顯示實際載入的來源。

常見錯誤狀態：

| 症狀 | 原因 | 修法 |
|---|---|---|
| 改 clone 沒反應 | 全域回到 `npm:pi-comfy-ui`，跑的是 `node_modules` | 把 `packages` 改回 `"~/Projects/pi-comfy-ui"` |
| 換目錄就沒樣式 | 設定跟著 cwd 跑到 `<cwd>/.pi/settings.json` | 改寫全域 `packages` |
| 置底失效、樣式重疊 | 全域+專案兩份同時載入 | 拉掉一份（不同來源形式不會被 dedupe） |

### 完整還原流程

```bash
cd ~/Projects/pi-comfy-ui

# 1. 本機修改都有 commit 嗎？
git log --oneline -5
git status --short

# 2. 若 clone 被覆蓋/重新 clone，從 commit 撿回來
git log --oneline --all | grep -i "anchor editor to bottom"
git cherry-pick <sha>

# 3. 拉 upstream 新版並重放本機 commit
bash ~/.claude/skills/pi-comfy-ui/scripts/restore-local-patches.sh --rebase
```

### 還原設定檔（若 packages 被改回 npm 版）

```bash
node -e '
const fs=require("fs"),p=process.env.HOME+"/.pi/agent/settings.json";
const s=JSON.parse(fs.readFileSync(p,"utf8"));
s.packages=s.packages.filter(x=>x!=="npm:pi-comfy-ui");
if(!s.packages.includes("~/Projects/pi-comfy-ui")) s.packages.push("~/Projects/pi-comfy-ui");
fs.writeFileSync(p,JSON.stringify(s,null,2)+"\n");
console.log(s.packages);
'
rm -f ~/.pi/settings.json   # 確保沒有專案層覆寫
```

舊設定備份：`ls ~/.pi/agent/settings.json.bak-*`

### 本機修改清單（還原時逐項核對）

| 項目 | 檔案 | 標記字串 |
|---|---|---|
| 輸入框置底 | `extensions/bottom-anchor.ts`（新檔） | `anchorEditorToBottom` |
| 置底接線 | `extensions/comfy-ui.ts` | `PI_COMFY_BOTTOM_EDITOR` |
| 置底：baseRender 改從 prototype 拿（避免 Proxy 重入爆堆疊） | `extensions/bottom-anchor.ts` | `protoRender` |
| 置底測試 | `tests/bottom-anchor.test.mjs`（新檔） | — |
| 測試註冊 | `package.json` 的 `scripts.test` | `bottom-anchor.test.mjs` |
| perf: regex 提升 | `extensions/ansi.ts` | `ANSI_ESCAPE_PATTERN` |
| perf: regex 提升 | `extensions/editor-input.ts` | `HORIZONTAL_EDITOR_BORDER_PATTERN` |
| perf: regex 提升 | `extensions/panel-painter.ts` | `DYNAMIC_BORDER_PATTERN` |
| perf: Set 查找 | `extensions/known-pi-panels.ts` | `EXPORTED_STYLABLE_COMPONENT_SET` |

腳本會自動檢查這些標記字串。新增本機修改時，記得同步更新上表與
`scripts/restore-local-patches.sh` 的 `check_marker` 清單。

### 回滾

下段只適用於「曾改回 npm 安裝模式並跑過無參數同步」的情況。
現行配置直接跑 clone，回滾就是 `git` 操作（`git reset` / `git checkout`）。

腳本同步前會備份。要還原到同步前：
```bash
ls -d ~/.pi/agent/npm/node_modules/pi-comfy-ui.bak-*   # 找備份
rm -rf ~/.pi/agent/npm/node_modules/pi-comfy-ui
mv ~/.pi/agent/npm/node_modules/pi-comfy-ui.bak-<stamp> \
   ~/.pi/agent/npm/node_modules/pi-comfy-ui
```

備份會累積，定期清：`rm -rf ~/.pi/agent/npm/node_modules/pi-comfy-ui.bak-*`

## 本機開發/測試環境注意事項（Windows + Git Bash）

- **裝依賴請用 `npm install`，不要用 `bun install`**：本機測試過 `bun install` 裝出來的
  `@earendil-works/pi-ai` 版本會缺少 `getOAuthApiKey`/`getOAuthProviders` 等具名 export，
  導致 `node tests/interactive-panel-render.test.mjs` 直接 `SyntaxError` 掛掉；
  改用 `bun test` 執行測試檔本身則會讓 bun runtime **segfault**（`panic(main thread)`）。
  用 `npm install` 裝出的 peer deps（`@earendil-works/pi-coding-agent` /
  `@earendil-works/pi-tui` / `pi-ai` 皆鎖 `0.80.x`）搭配 `node` 執行 `npm test` /
  `npm run typecheck` 才是乾淨可重現的組合。
- 驗證指令（在 `~/Projects/pi-comfy-ui` 下）：
  ```bash
  rm -rf node_modules bun.lock package-lock.json   # 若曾用 bun 裝過，先清乾淨
  npm install --no-audit --no-fund
  npm run typecheck   # tsc --noEmit
  npm test             # 依序跑 3 個 tests/*.test.mjs
  ```
- `tsconfig.json` 用 `NodeNext` module/resolution + `strict: true`，測試檔用
  `jiti` 直接 import `.ts` 原始碼，不需要額外編譯步驟。

## 若要提 PR 回 upstream

`~/Projects/pi-comfy-ui` 的 `origin` 就是 `adanft/pi-comfy-ui`，如果本機帳號沒有
直接 push 權限，改用 `gh repo fork` 或在 GitHub 上手動 fork 後改 remote，再開 PR；
提交前務必確保：
1. `npm run typecheck` 和 `npm test` 全過
2. 沒有觸碰 `content-padding-contract.test.mjs` 列出的禁用識別字清單
3. commit message 遵循倉庫既有風格（例：`perf: hoist hot-path regexes and use Set lookup for known components`）

---

## Conformance Addendum

## When to Use
`pi` coding agent 的 TUI 美化擴充套件 `pi-comfy-ui`（npm 套件，作者 adanft，MIT， https://github.com/adanft/pi-comfy-ui）：只負責輸入框（editor）背景色/側邊框樣式，以及 互動面板（設定、模型選擇器、確認框、ask_user_question 等）背景色/側邊框樣式，不碰任何 padding 設定與功能邏輯。本機已安裝於 `~/.pi/agent/npm/node_modules/pi-comfy-ui` （見 `~/.pi/agent/settings.json` 的 `packages`），原始碼副本在 `~/Projects/pi-comfy-ui`（git clone，已對 origin push 過效能優化）。 本機另外加了 upstream 沒有的「輸入框固定在終端機底部」功能（`extensions/bottom-anchor.ts`）。 當使用者提到 pi-comfy-ui、pi 的輸入框/面板美化、輸入框置底/貼底、 `customMessageBg`/`userMessageBg` 主題色、或要研究/修改/優化這個套件的原始碼、 或更新後要還原本機自訂修改時使用。

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
