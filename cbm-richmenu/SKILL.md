---
name: cbm-richmenu
description: >-
  LINE Rich Menu 母子選單完整管理——(1) 從零設計/建立/重建主選單與子選單、Flex 九宮格功能選單、圖片產生上傳、
  別名管理 (2) 切換 cbm-lite LINE channel（換 token/secret）並將既有 rich menu「完整複製搬遷」到新 channel
  （換帳號測試、配額用完換備用帳號、新 channel 上線）。合併自原本的 `cbm-richmenu-build` 與
  `cbm-richmenu-channel-migrate` 兩個 skill。
triggers:
  - rich menu
  - richmenu
  - 圖文選單
  - 功能選單
  - 選單設計
  - 換 channel
  - 換帳號
  - line 配額
  - 複製選單
argument-hint: "[rebuild|status|add-page|migrate-channel]"
---

# cbm-richmenu Skill（原名 cbm-richmenu-build + cbm-richmenu-channel-migrate）

LINE 圖文選單（Rich Menu）母子選單完整設定。設定腳本在
`C:\Lab2\chainsea\tool\setup-richmenu.ps1`，Java 端整合在
`CbmLiteServer.java`（`linkRichMenu`, `lineGridMenu`）。

> 兩種需求都在這份 skill：**Part A** 是從零設計/建立/重建（同一個 channel 內操作）；
> **Part B** 是把既有選單「完整複製搬遷」到另一個 LINE channel（換帳號/配額用完/新上線）。

---

## Part A — 從零設計/建立

### 選單結構

```
主選單（底部常駐，1×3，1200×405）
┌──────────┬──────────┬──────────┐
│  常見問題 │  真人客服 │ 更多選項 ›│  更多選項 → richmenuswitch → 子選單1
└──────────┴──────────┴──────────┘

子選單1（2×3，1200×810）
┌──────────┬──────────┬──────────┐
│  ◀ 返回  │  結束服務 │  服務時間 │  結束服務 → richmenuswitch(data=end-agent) → 主選單 + Java endAgentMode
├──────────┼──────────┼──────────┤
│  功能選單 │  帳號問題 │  繼續 ›  │  功能選單 → 送出「選單」→ Flex 九宮格彈出
└──────────┴──────────┴──────────┘  繼續 ›   → richmenuswitch → 子選單2

子選單2（2×3，1200×810）
┌──────────┬──────────┬──────────┐
│  ◀ 上頁  │  退款申請 │  會員中心 │  會員中心 → uri → https://liff.line.me/2010442070-e4uFbN3W (LIFF)
├──────────┼──────────┼──────────┤
│  問題回報 │  聯絡資訊 │  回主頁  │  回主頁 → richmenuswitch → 主選單
└──────────┴──────────┴──────────┘
```

> LIFF 入口（會員中心，子選單2 右上格）用 `type="uri"` action，指向 LIFF URL；
> 對應 cbm-lite 的 `/liff` 頁（見 cbm-lite skill 的 LIFF 段）。Endpoint 走
> `hch.james-huang.org/liff`（Cloudflare path 分流到 12621）。

真人客服模式還有一個 `cbm.lite.line.richMenuAgentId`（「主選單(客服中)」，同 1×3 版面，中間格紅底
「結束服務」），進入真人客服時自動切換過去，讓使用者一眼看出目前是否在真人客服模式。建立/重建腳本：
`C:\Lab2\chainsea\tool\setup-richmenu-agent.ps1`（只建這張）。若連同主選單/子選單一起重建，
用 `setup-richmenu.ps1`（完整版，`$propsPath`/`$outPath` 均已指向 `C:\Lab2\...`）。

### 別名對照

| 別名 | 對應選單 | 用途 |
|------|---------|------|
| `cbm-main-menu` | 主選單 | `linkRichMenu(userId,"cbm.lite.line.richMenuId")` |
| `cbm-sub-menu` | 子選單1 | `linkRichMenu(userId,"cbm.lite.line.richMenuSubId")` |
| `cbm-sub-menu-2` | 子選單2 | 純靠 richmenuswitch 切換，Java 不直接 link |

別名是 `richmenuswitch` 動作必須的；圖片上傳完**才能**建別名（上傳前建會失敗）。

### Properties（cbm-lite.properties）

```properties
cbm.lite.line.richMenuId=richmenu-<主選單ID>
cbm.lite.line.richMenuSubId=richmenu-<子選單1ID>
cbm.lite.line.richMenuSub2Id=richmenu-<子選單2ID>
```

`setup-richmenu.ps1` 執行完會自動更新這三個值。

### Java 整合點

#### 模式切換時自動連結選單

```java
// 進入真人客服 → 子選單1
switchToAgentMode()  →  linkRichMenu(userId, "cbm.lite.line.richMenuSubId")

// 結束真人客服 → 主選單
endAgentMode()       →  linkRichMenu(userId, "cbm.lite.line.richMenuId")
agentClose()（客服台關閉）→ linkRichMenu(userId, "cbm.lite.line.richMenuId")
```

#### postback 事件處理（結束服務按鈕）

子選單1 的「結束服務」使用 `richmenuswitch`（data=`end-agent`），LINE 同時：
1. 立刻切換 rich menu 回主選單（視覺即時）
2. 發 postback 事件到 webhook

`lineWebhook()` 中處理：
```java
if ("postback".equals(type)) {
    String data = event.optJSONObject("postback").optString("data","");
    if ("end-agent".equals(data)) {
        JSONArray msgs = endAgentMode(userId, getUserState(userId));
        if (!replyToken.isEmpty() && msgs.length() > 0) replyLine(replyToken, msgs);
    }
    continue;
}
```

#### 加入好友（follow 事件）

```java
linkRichMenu(userId, "cbm.lite.line.richMenuId");  // 顯示主選單
```

#### Flex 九宮格（`lineGridMenu`）

觸發詞：`lineAnswer()` 中 `isGridMenuRequest(lower)` 命中（輸入「選單」/「功能選單」/「menu」/「所有選項」/「全部功能」）
→ `lineGridMenu("功能選單", CS_GRID_ITEMS)` 回傳 Flex bubble

子選單1「功能選單」格子的 action 是 `message: text="選單"`，點擊即觸發。

新增格子只需在 `CS_GRID_ITEMS` 陣列加一行 `{"emoji","標籤","送出文字"}`，每 3 個自動成一列，顏色循環（藍→綠→橘→紫）。

### LINE API 尺寸規格

| 格式 | 寬×高 | 每格大小 |
|------|-------|---------|
| 1×3（主選單） | 1200×405 | 400×405 |
| 2×3（子選單） | 1200×810 | 400×405 |

圖片上傳用 `api-data.line.me`（不是 `api.line.me`）：
```
POST https://api-data.line.me/v2/bot/richmenu/{menuId}/content
Content-Type: image/png
```

### 完整重建（從頭）

```powershell
C:\Lab2\chainsea\tool\setup-richmenu.ps1
```

腳本會自動：清除舊選單/別名 → 建三個選單 → 產生圖片 → 上傳 → 建別名 → 套用給所有用戶 → 更新 properties。

### 新增子選單第三頁

1. 在 `setup-richmenu.ps1` 新增 `$sub3Id` 區塊（仿照 sub2）
2. 子選單2 最後一格改成 `richmenuswitch` 指向 `cbm-sub-menu-3`
3. 子選單3 第一格 `richmenuswitch` 指回 `cbm-sub-menu-2`（data="prev-page"）
4. 新增別名 `cbm-sub-menu-3`
5. 最多可建到第 17 頁（LINE 每頻道上限 20 個 Rich Menu，已佔 3 個）

### 查詢現有選單狀態

```powershell
$token = (Get-Content "C:\Lab2\chainsea\apache-tomcat\extension\aipower\config\cbm-lite.properties" | Select-String "channelAccessToken=").ToString().Split("=",2)[1].Trim()
$h = @{ "Authorization" = "Bearer $token" }
# 列出所有選單
(Invoke-RestMethod "https://api.line.me/v2/bot/richmenu/list" -Headers $h).richmenus | Select-Object richMenuId,name,@{n="size";e={"$($_.size.width)x$($_.size.height)"}}
# 列出別名
Invoke-RestMethod "https://api.line.me/v2/bot/richmenu/alias/list" -Headers $h
```

### Part A 注意事項

- `richmenuswitch` 的來源和目標選單**都必須有別名**，否則 LINE 不允許建立動作
- `POST /v2/bot/user/all/richmenu` 只套用給**已存在的用戶**，新用戶靠 follow 事件的 `linkRichMenu` 覆蓋
- 刪除選單前必須先刪對應別名，否則選單刪不掉（外鍵關係）
- Properties 改完不需重新編譯，但需重啟 cbm-lite（Java 啟動時讀入）；Rich Menu 變更是即時的不需重啟

---

## Part B — 換 LINE Channel（完整複製搬遷）

### 前置條件

- cbm-lite 在 `C:\Lab2\cbm-lite\` 跑著（port 12621）
- 有舊 channel 的 long-lived access token（下載原始圖片用）
- 有新 channel 的 Channel Secret 與 Channel Access Token

### Step 1：更新 cbm-lite 設定

編輯 `C:\Lab2\cbm-lite\config\cbm-lite.properties`：

```properties
cbm.lite.line.channelSecret=<新 channel secret>
cbm.lite.line.channelAccessToken=<新 long-lived token>
```

熱重載（不需重啟）：

```powershell
Invoke-RestMethod -Uri "http://127.0.0.1:12621/admin/reload" -Method POST
```

### Step 2：確認新 channel Webhook 設定

LINE Developers Console → Messaging API：
- **Webhook URL**：`https://gateway.james-huang.org/gateway`
- **Use webhook**：開啟（綠色）

### Step 3：下載舊 channel 所有 Rich Menu 結構與圖片

> ⚠️ **圖片下載必須用 `api-data.line.me`，不是 `api.line.me`** — 用錯會 404。

```powershell
$oldToken = "<舊 channel access token>"
$oh = @{ Authorization = "Bearer $oldToken" }

# 列出全部選單
$list = Invoke-RestMethod -Uri "https://api.line.me/v2/bot/richmenu/list" -Headers $oh
$list.richmenus | ForEach-Object { Write-Host "$($_.richMenuId) - $($_.name) $($_.size.width)x$($_.size.height)" }

# 下載每張圖片
foreach ($rm in $list.richmenus) {
    $out = "$env:TEMP\rm_$($rm.richMenuId).png"
    Invoke-RestMethod -Uri "https://api-data.line.me/v2/bot/richmenu/$($rm.richMenuId)/content" -Headers $oh -OutFile $out
    Write-Host "下載 $($rm.name): $((Get-Item $out).Length) bytes"
}

# 列出所有 alias
$oldAliases = Invoke-RestMethod -Uri "https://api.line.me/v2/bot/richmenu/alias/list" -Headers $oh
$oldAliases.aliases | ForEach-Object { Write-Host "alias $($_.richMenuAliasId) → $($_.richMenuId)" }

# 查預設 rich menu
$oldDefault = Invoke-RestMethod -Uri "https://api.line.me/v2/bot/user/all/richmenu" -Headers $oh
Write-Host "舊預設: $($oldDefault.richMenuId)"
```

### Step 4：清除新 channel 現有選單

> ⚠️ **LINE 不允許覆蓋已上傳的圖片**，必須刪除整個選單重建。

```powershell
$newToken = "<新 channel access token>"
$nh = @{ Authorization = "Bearer $newToken" }

# 刪 alias
foreach ($a in ($oldAliases.aliases | ForEach-Object { $_.richMenuAliasId })) {
    try { Invoke-RestMethod -Uri "https://api.line.me/v2/bot/richmenu/alias/$a" -Method DELETE -Headers $nh | Out-Null; Write-Host "刪 alias $a" } catch {}
}

# 刪選單
$existing = Invoke-RestMethod -Uri "https://api.line.me/v2/bot/richmenu/list" -Headers $nh
foreach ($rm in $existing.richmenus) {
    Invoke-RestMethod -Uri "https://api.line.me/v2/bot/richmenu/$($rm.richMenuId)" -Method DELETE -Headers $nh | Out-Null
    Write-Host "刪 $($rm.richMenuId)"
}
```

### Step 5：重建選單、上傳原始圖片

```powershell
$newIds = @{}
foreach ($rm in $list.richmenus) {
    # 複製結構（移除 richMenuId）
    $def = Invoke-RestMethod -Uri "https://api.line.me/v2/bot/richmenu/$($rm.richMenuId)" -Headers $oh
    $def.PSObject.Properties.Remove('richMenuId')
    $body = $def | ConvertTo-Json -Depth 10 -Compress

    # 建立
    $created = Invoke-RestMethod -Uri "https://api.line.me/v2/bot/richmenu" -Method POST -Headers ($nh+@{'Content-Type'='application/json'}) -Body $body
    $newId = $created.richMenuId
    $newIds[$rm.richMenuId] = $newId
    Write-Host "建立 $($rm.name): $newId"

    # 上傳原始圖片
    $bytes = [System.IO.File]::ReadAllBytes("$env:TEMP\rm_$($rm.richMenuId).png")
    Invoke-RestMethod -Uri "https://api-data.line.me/v2/bot/richmenu/$newId/content" -Method POST -Headers ($nh+@{'Content-Type'='image/png'}) -Body $bytes | Out-Null
    Write-Host "  ✓ 圖片上傳完成"
}
```

### Step 6：重建 Alias + 設定預設

> ⚠️ **Alias 不會跟著選單複製**，`richmenuswitch` 動作靠 alias 名稱，不重建按鈕切換就會失效。

```powershell
# 重建 alias
foreach ($alias in $oldAliases.aliases) {
    $newMenuId = $newIds[$alias.richMenuId]
    $b = "{`"richMenuAliasId`":`"$($alias.richMenuAliasId)`",`"richMenuId`":`"$newMenuId`"}"
    Invoke-RestMethod -Uri "https://api.line.me/v2/bot/richmenu/alias" -Method POST -Headers ($nh+@{'Content-Type'='application/json'}) -Body $b | Out-Null
    Write-Host "alias $($alias.richMenuAliasId) → $newMenuId"
}

# 設預設（同舊 channel）
$newDefaultId = $newIds[$oldDefault.richMenuId]
Invoke-RestMethod -Uri "https://api.line.me/v2/bot/user/all/richmenu/$newDefaultId" -Method POST -Headers $nh | Out-Null
Write-Host "預設: $newDefaultId"

# 輸出新 ID 供填 config
Write-Host "`n--- 填入 cbm-lite.properties ---"
$newIds.GetEnumerator() | ForEach-Object { Write-Host "$($_.Key) → $($_.Value)" }
```

### Step 7：更新 cbm-lite.properties + Reload

對照 Step 6 輸出，更新三個 rich menu ID：

```properties
cbm.lite.line.richMenuId=<新主選單 ID>
cbm.lite.line.richMenuSubId=<新客服模式選單 ID>
cbm.lite.line.richMenuSub2Id=<新子選單 ID>
```

```powershell
Invoke-RestMethod -Uri "http://127.0.0.1:12621/admin/reload" -Method POST
```

### 成功驗證

- LINE 對話底部顯示主選單，外觀與舊 channel 完全相同
- 按「真人客服」→ 切換為客服模式選單（紅色）
- 按「結束服務」→ 切換回主選單
- 子選單展開 / 返回正常

### Part B 已知坑

| 坑 | 說明 |
|----|------|
| 圖片 404 | 下載用 `api-data.line.me`，不是 `api.line.me` |
| 圖片無法覆蓋 | 必須刪除選單重建，不能直接覆寫 |
| Alias 不複製 | `richmenuswitch` 的 alias 要手動重建，否則切換失效 |
| Alias 數量要全抄 | HCH channel 有 **3 個** alias（`cbm-main-menu`、`cbm-sub-menu`、`cbm-sub-menu-2`），漏掉任何一個都會讓對應頁的「回上一頁」失效。Step 3 的腳本會自動遍歷，不要手動假設只有 2 個 |
| 結束客服卡住 | 呼叫 `/agent/rooms` 找 `closed:false` 的 roomId，再呼叫 `/agent/close` 強制關閉 |
| LINE Push 429 | 免費方案 200 則/月；切換備用 channel 或改用 replyLine（不計配額） |

### HCH 兩個 LINE Channel

| Channel | ID | 用途 |
|---------|----|------|
| HCH | 2006734807 | 主要客服 bot |
| AI3.5 | 2006519220 | 備用 bot（月底 HCH 429 時切換） |

Webhook URL 兩者都指向 `https://gateway.james-huang.org/gateway`。

本機備份圖片：`C:\Lab2\chainsea\tool\richmenu-richmenu-*.png`

---

## Conformance Addendum

## When to Use
LINE Rich Menu 母子選單完整管理——(1) 從零設計/建立/重建主選單與子選單、Flex 九宮格功能選單、圖片產生上傳、 別名管理 (2) 切換 cbm-lite LINE channel（換 token/secret）並將既有 rich menu「完整複製搬遷」到新 channel （換帳號測試、配額用完換備用帳號、新 channel 上線）。合併自原本的 `cbm-richmenu-build` 與 `cbm-richmenu-channel-migrate` 兩個 skill。

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
