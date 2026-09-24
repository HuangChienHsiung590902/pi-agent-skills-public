---
name: omniroute-embedded-services
description: >-
  Use OmniRoute's "內嵌服務" (Embedded Services) page — the four pluggable local routing engines CLIProxyAPI, 9Router, Mux, and Bifrost that OmniRoute spawns/manages as child processes. Covers how these engines nest under OmniRoute (each is a full standalone open-source gateway with its own dashboard/DB/provider config, not reimplemented), Bifrost specifically (npm @maximhq/bifrost, own admin UI on http://127.0.0.1:8080 loopback-only, own config.db/logs.db under ~/.omniroute/services/bifrost/, needs its own provider/API-key setup before /v1/models returns anything), the "Provider Exposure" toggle that merges an engine's discovered models into OmniRoute's own unified endpoint, and why a transient "Health probe did not succeed within 15000ms" red banner right after Start is a race that usually clears on refresh. Use when the user mentions 內嵌服務, embedded services, Bifrost, 9Router, Mux, CLIProxyAPI, or routing traffic through one of these engines.
---

# OmniRoute — 內嵌服務 (Embedded Services)

## 這頁在管什麼

`/dashboard/embedded-services`（內嵌服務）底下四個分頁 CLIProxyAPI / 9Router / Mux / Bifrost，是 OmniRoute **內嵌並代管生命週期**的四套獨立路由引擎（啟停、健康檢查、自動更新、log tail），不是 OmniRoute 自己刻的轉發邏輯。每一套本身都是完整的開源 LLM 閘道，各自有自己的 dashboard、資料庫、provider/key 設定——跟 OmniRoute 主介面的 provider 設定是**分開兩份資料**。所以 Bifrost 的介面看起來會很像 OmniRoute 本人（有 provider 管理、request log、統一端點……），因為它們是同一類工具的巢狀關係：

```
OmniRoute（外層管理/路由層，自己的 provider 設定 + 統一端點 :20128/v1）
 └─ 內嵌服務（四選一或並存，各自是完整閘道）
     ├─ CLIProxyAPI
     ├─ 9Router
     ├─ Mux
     └─ Bifrost（npm @maximhq/bifrost，自己的 provider 設定 + 端點 :8080/v1）
```

**兩條使用路線二選一**，不必兩邊都設：
1. 只用 OmniRoute 原生 provider 管理，不碰任何內嵌服務——多數情境走這條就夠。
2. 特別想用某個引擎的專屬功能（例如 Bifrost 的快取/路由策略），才進該引擎自己的後台加 provider/key，再回 OmniRoute 這頁把「Provider Exposure」開關打開，讓它探測到的 models 併入 OmniRoute 全站的 provider 選單與 `localhost:20128/v1` 統一端點。不開的話，該引擎只能用它自己的端點單獨呼叫，不會出現在 OmniRoute 裡。

## Bifrost 細節

- 套件：`@maximhq/bifrost`，安裝在 `~/.omniroute/services/bifrost/`（`config.db`、`logs.db` 是它自己的資料庫，跟 OmniRoute 主 `storage.sqlite` 無關）
- 預設埠：8080，**只能透過迴路位址（127.0.0.1）存取**，管理介面/OpenAI 相容端點都在這
  - Dashboard: `http://127.0.0.1:8080`
  - Models 探測: `http://127.0.0.1:8080/v1/models`（沒設 provider 前回傳 `{"data":[]}`，不是壞掉）
  - Health: `http://127.0.0.1:8080/health` → `{"components":{"db_pings":"ok"},"status":"ok"}`
- **要讓 Bifrost 真正轉發到什麼都不用打，先進 `127.0.0.1:8080` 後台加 provider/API key**，Bifrost 本身不吃 OmniRoute 既有的 provider 設定
- 加完 provider 後回到 OmniRoute 內嵌服務頁的 Bifrost 分頁，打開「**Provider Exposure**」開關（`providerExpose` 欄位），才會併入 OmniRoute 統一端點 `localhost:20128/v1`

## 「Health probe did not succeed within 15000ms」紅色警告

按 Start 之後立刻出現的這行紅字通常是**啟動當下的暫時性 race**：OmniRoute 的背景 supervisor 剛把子行程 spawn 起來，首次健康探測（`healthIntervalMs: 5000` 起跳）常常搶輸初始化時間。實測 `curl http://127.0.0.1:8080/health` 這時已經回 `ok`，代表服務本身沒事，重新整理頁面通常就會變綠。不要看到這個就急著 Stop/Restart 或懷疑安裝壞了——先直接 curl 對應埠的 health 端點確認再說。

## 相關

原生（非 Docker）OmniRoute 安裝、啟停、密碼重設、Provider Topology 幽靈節點排查，見 skill `omniroute-native`。

---

## Conformance Addendum

## When to Use
Use OmniRoute's "內嵌服務" (Embedded Services) page — the four pluggable local routing engines CLIProxyAPI, 9Router, Mux, and Bifrost that OmniRoute spawns/manages as child processes. Covers how these engines nest under OmniRoute (each is a full standalone open-source gateway with its own dashboard/DB/provider config, not reimplemented), Bifrost specifically (npm @maximhq/bifrost, own admin UI on http://127.0.0.1:8080 loopback-only, own config.db/logs.db under ~/.omniroute/services/bifrost/, needs its own provider/API-key setup before /v1/models returns anything), the "Provider Exposure" toggle that merges an engine's discovered models into OmniRoute's own unified endpoint, and why a transient "Health probe did not succeed within 15000ms" red banner right after Start is a race that usually clears on refresh. Use when the user mentions 內嵌服務, embedded services, Bifrost, 9Router, Mux, CLIProxyAPI, or routing traffic through one of these engines.

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
