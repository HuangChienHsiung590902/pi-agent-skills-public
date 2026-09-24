---
name: ue-editor-zh-hant-localization
description: >-
  當使用者問 Unreal Engine 編輯器有沒有繁體中文、要把 UE 編輯器介面做成繁中／zh-Hant、
  發現官方 Localization 只有 zh-Hans、或繁中開了但偏好設定側欄仍有英文
  （Asset Tools、Control Rig Edit Mode、Sequencer Simple View Settings、
  Model Context Protocol、Scribble、Gizmos）時使用。分兩階段：
  (1) UnrealLocres 匯出官方簡中 locres、OpenCC s2twp 轉繁中、寫回 zh-Hant 並補 locmeta；
  (2) 對官方簡中本來就沒翻的標題式顯示名稱做詞庫英翻中。
  不用於遊戲專案、UMG、String Table、屬性欄位精翻或插件 PO 本地化。
---

# UE Editor 繁中本地化（zh-Hans → zh-Hant，再補剩餘英文標題）

Epic 發行的 Unreal Editor **沒有官方繁體中文**。`Engine/Content/Localization` 的中文只有 `zh-Hans`。本 Skill 先用官方簡中 locres 做 OpenCC 機械轉換產生 `zh-Hant`，再把官方從未翻譯的**標題式顯示名稱**英翻中。這不是 Epic 官方繁中。

## When to Use

- 使用者問「UE 5.8／Unreal Editor 有沒有繁體中文」。
- 使用者說「幫我做繁中本地化」，且上下文是**編輯器介面**，不是遊戲字串。
- 官方目錄只有 `zh-Hans`，沒有 `zh-Hant`／`zh-TW`。
- Epic 更新後繁中消失，要重做階段 1。
- 編輯器已是繁中，但「編輯器偏好設定」側欄下半仍是英文（Asset Tools、Control Rig Edit Mode 等）。那是階段 2，不是 OpenCC 漏轉。
- 要把 Editor Language 切到 `zh-Hant`，或還原這項修改。

不要用於：

- 遊戲專案／UMG／String Table／Gather Text。
- 細節面板裡約 3 萬條屬性欄位名稱、ToolTips 裡的 C++、Keywords。
- 插件 UI 的 PO／commandlet 翻譯。
- SAM 3D Body、Rigify、IK Rig（用 `sam3d-body-rigify-unreal-pipeline`）。

## Inputs and Outputs

### Inputs

- Unreal Engine 安裝路徑。本機預設：`C:\Program Files\Epic Games\UE_5.8`。
- 階段 1：官方 `zh-Hans` locres（`Editor`、`Engine`、`Category`、`Keywords`、`PropertyNames`、`ToolTips`）。
- 階段 2：已存在的 `zh-Hant` locres（沒有就先跑階段 1）。
- `UnrealLocres`（執行時下載到暫存）、階段 1 另需 `opencc-python-reimplemented`。

### Outputs

- `Engine\Content\Localization\<Namespace>\zh-Hant\<Namespace>.locres`
- 階段 1：各 `*.locmeta` cultures 含 `zh-Hant`；可選 `EditorSettings.ini` 的 `Language=zh-Hant`
- 階段 2：`PropertyNames` class-level `UObjectDisplayNames`、以及 `Editor`／`Category` 的標題式英文
- 暫存：`%TEMP%\pi-work\ue-editor-zh-hant-*\`（不寫入目前工作目錄）

## Procedure

相對路徑以本 Skill 資料夾為基準。寫 locres 前必須關閉 Unreal Editor。

### 階段 1：官方簡中 → 繁中

1. 讀 `scripts/convert_ue_editor_zh_hant.py` 與 `references/locres-and-locmeta.md`。
2. 唯讀確認：`Editor\` 有 `zh-Hans`；`EditorTutorials` 通常連簡中都沒有，跳過。`Engine\Binaries\ThirdParty\DotNet\**\zh-Hant` 是 .NET 分析器，不是編輯器 UI。
3. 執行：

   ```powershell
   python D:\OB\skills\ue-editor-zh-hant-localization\scripts\convert_ue_editor_zh_hant.py convert --engine "C:\Program Files\Epic Games\UE_5.8"
   ```

4. 驗證：

   ```powershell
   python D:\OB\skills\ue-editor-zh-hant-localization\scripts\convert_ue_editor_zh_hant.py verify --engine "C:\Program Files\Epic Games\UE_5.8"
   ```

### 階段 2：剩餘英文標題 → 繁中

官方 `zh-Hans` 的 `UObjectDisplayNames` 很多本來就是英文（外掛／新模組沒進中文包）。OpenCC 轉不了英文。詳見 `references/leftover-english-titles.md`。

1. 讀 `scripts/translate_en_display_titles.py`。先產 CSV 抽樣，確認截圖那批標題無誤，再 import。
2. 預覽（不寫 Program Files）：

   ```powershell
   python D:\OB\skills\ue-editor-zh-hant-localization\scripts\translate_en_display_titles.py csv --engine "C:\Program Files\Epic Games\UE_5.8"
   ```

3. 關閉編輯器後寫入：

   ```powershell
   python D:\OB\skills\ue-editor-zh-hant-localization\scripts\translate_en_display_titles.py import --engine "C:\Program Files\Epic Games\UE_5.8"
   ```

   編輯器還開著時，加 `--stop-editor` 才允許腳本結束 `UnrealEditor*`。預設是偵測到行程就停止。
4. 請使用者**重開編輯器**，檢查「編輯器偏好設定」側欄。

還原階段 1 用 convert 的 `restore`。階段 2 的 locres 備份在該次 `%TEMP%\pi-work\ue-editor-zh-hant-en2zh-*\backup_locres\`。

## Rules and Limitations

- 非官方機械轉換／詞庫翻譯，不是 Epic 釋出的繁中。
- 階段 1 只動六個核心 namespace；不要順便掃 Plugins。
- 階段 2 只翻**標題式顯示名稱**：`PropertyNames` 的 class-level `UObjectDisplayNames`（key 不含 `:`）、`Editor`／`Category` 的短標題。不翻屬性欄位、句子、CamelCase 識別字、純縮寫、ToolTips、Keywords、Engine。
- 片語必須用詞界，禁止子字串替換（`Compare` 不可變成 `CompARe`）。
- 專有名詞保留：Control Rig、Sequencer、Niagara、MetaHuman、IK Rig 等。
- 寫入 `Program Files`。Epic 驗證／更新可能覆蓋，屆時重跑對應階段。
- OpenCC `s2twp`：Documentation 語境的「文件」不要改成「檔案」。
- `*.locmeta` 只追加 `zh-Hant`，不要改 GUID／native。
- 暫存與 `UnrealLocres.exe` 放 `%TEMP%\pi-work\...`，不要提交進 Skill 或專案。
- 改語系或 locres 後必須重開編輯器。

## Pitfalls

- 把偏好設定裡剩下的英文當成階段 1 失敗。先查 `PropertyNames` CSV：source 若已是英文，要跑階段 2。
- 子字串替換會破壞識別字（`ArcCos`、`AnimSharing`、`Compare`）。
- 把完整英文句子逐詞翻成中英混雜；階段 2 應跳過句子。
- 編輯器還開著就寫 locres。
- 把 .NET 的 `zh-Hant` 當成編輯器已有繁中。
- 以為 `EditorTutorials` 有簡中可轉。
- YAML `description` 含冒號時必須用 `>-`。

## Verification

- 階段 1：`verify` 顯示六個 `zh-Hant/*.locres` 存在，`locmeta` 含 `zh-Hant`；抽樣 `內容瀏覽器`、`設定`、`儲存`、`資料夾`、`預設`。
- 階段 2：`csv` 結束時截圖標題應對上：資產工具、音訊編輯器專案設定、內容瀏覽器我的最愛專案設定、Control Rig 編輯模式、資料表編輯器設定、操控器、媒體合成編輯器設定、模型上下文協定、塗鴉、選取集設定、Sequencer 簡易檢視設定、可工具化時間軸（預設）設定。
- 重開編輯器後，偏好設定側欄那串英文應變繁中；細節面板屬性名仍可能是英文。
- Skill：資料夾名 = frontmatter `name` = `ue-editor-zh-hant-localization`；索引搜得到；本 Skill 稽核無 error。
