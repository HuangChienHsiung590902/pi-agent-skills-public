---
name: ecp-flutter
description: ECP Mobile (Flutter, C:\com\APP) 開發規範——固定專案位置與 lib/ feature-first 原始碼結構（非後端 tool\src），加上 Windows Android build 三大致命陷阱（Kotlin plugin 未套用、Developer Mode 未開、AGP 版本衝突）
triggers:
  - ClassNotFoundException MainActivity
  - flutter build apk
  - compileDebugKotlin
  - flutter_inappwebview AGP
  - MainActivity not found
  - Kotlin plugin
  - build.gradle.kts
  - Developer Mode symlink
  - ecp mobile
  - C:\com\APP
  - flutter lib 結構
  - flutter 原始碼放哪
---

# ECP Mobile（Flutter）開發規範

## 〇、專案位置與原始碼結構（固定，與後端 Java 完全不同）

ECP 行動版 Flutter 專案：**專案根固定在 `C:\com\APP`**（package 名 `ecp`，即 e-Contact+ 行動版）。

**Dart 原始碼一律放 `lib/`，不是 `src/`。** 這是 Dart 套件系統的硬性規定——import 以 `package:ecp/...` 解析到 `lib/`，放到 `src/` 會 import 不到。**不要套用後端 `tool\src` 那套；那只適用 aipower Java。**

固定結構（feature-first，已在用，務必沿用，不要把檔案散落到 `lib/` 根）：

| 路徑 | 用途 |
|------|------|
| `lib/main.dart` | App 進入點 |
| `lib/core/` | 跨功能基礎建設：`api/`（`api_client.dart`、`entity_service.dart`）、`auth/`（`auth_service.dart`）、`models/`（`base_model.dart`） |
| `lib/features/<feature>/` | **功能模組各自一包**（現有：`contacts`、`dashboard`、`helpdesk`、`login`、`tasks`、`zeppelin`） |
| `lib/shared/` | 跨功能共用 UI：`theme/`、`widgets/` |
| `test/` | 測試，鏡像 `lib/` 結構 |
| `android/`、`windows/` | 平台原生工程（build 設定見下方三大陷阱） |
| `build/`、`.dart_tool/` | 產生物——**勿手改、勿提交** |

### 規範
1. **新功能 → 開 `lib/features/<feature>/` 自己一包**，所有該功能的 page/model/service/provider 放裡面，不要丟到 `lib/` 根目錄或別的 feature。
2. **跨功能共用**：UI 元件 → `lib/shared/widgets`；主題 → `lib/shared/theme`；服務 / 模型 / API → `lib/core`。命名刻意對應後端六大核心（`entity_service.dart`↔`EntityService`、`base_model.dart`↔`BaseModel`）。
3. **檔名一律 `snake_case`**（`login_page.dart`、`task_model.dart`）；類別 `PascalCase`。慣用後綴：`_page.dart`(畫面)、`_model.dart`(資料)、`_service.dart`(服務)、`_provider.dart`(provider 狀態)。
4. **import 用 `package:ecp/...` 絕對路徑**，避免 `../../..` 相對地獄。
5. 狀態管理用 `provider`、路由用 `go_router`、HTTP 用 `dio`（見 `pubspec.yaml`，沿用既有套件，勿任意引入同類替代品）。
6. `lib/` 以 `.dart` 原始碼為主；產生物（`build/`、`.dart_tool/`）勿提交。`.gitignore` 已含 `.omc/`、`**/.omc/` 作為保險，誤生成的 `.omc` 可直接 `rm -rf`。

> **指令在哪執行不設限**：可從專案根或視需要進入子目錄執行（`flutter analyze`、`dart`、測試、診斷等），以利檢查與排錯。
> 後端 aipower Java 的路徑規範（`tool\src`→`tool\classes`→`WEB-INF\classes`）見 `ecp-java-core` skill；兩者互不適用。

---

# Flutter Android Build on Windows — 三大致命陷阱

## The Insight

Flutter Android 專案在 Windows 上 build，有三個獨立的陷阱，每一個都會造成 app 完全無法啟動但錯誤訊息誤導性很高。這三個問題必須全部正確才能 work。

---

## 陷阱一：`org.jetbrains.kotlin.android` 未在 `build.gradle.kts` 顯式套用

### 症狀
```
java.lang.ClassNotFoundException: Didn't find class "com.xxx.xxx.MainActivity"
```
`flutter build apk` 成功、APK 存在、但 app 一開就閃退。

### 根本原因
Flutter 新專案的 `android/app/build.gradle.kts` 預設只有：
```kotlin
plugins {
    id("com.android.application")
    id("dev.flutter.flutter-gradle-plugin")
}
```
**沒有** `id("org.jetbrains.kotlin.android")`。沒有這行，Gradle task 清單中不會出現 `:app:compileDebugKotlin`，`MainActivity.kt` 從來不會被編譯，APK 裡只有資源 class (`R`) 而沒有 `MainActivity`。

### 修法
```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")   // ← 必須加這行
    id("dev.flutter.flutter-gradle-plugin")
}
```

### 驗證方式
Build 後確認 APK 裡有 MainActivity：
```python
import zipfile
with zipfile.ZipFile('build/app/outputs/flutter-apk/app-debug.apk') as z:
    for name in z.namelist():
        if name.endswith('.dex'):
            if b'Lcom/xxx/xxx/MainActivity;' in z.read(name):
                print('FOUND in', name)
```

---

## 陷阱二：Windows Developer Mode 未開啟

### 症狀
`flutter pub get` 時出現警告：
```
Building with plugins requires symlink support.
Please enable Developer Mode in your system settings.
```
但 `flutter build apk` 仍然成功、APK 存在。

### 根本原因
Flutter 在 Windows 上需要 symlink 把 Kotlin 原始碼連結進 Gradle 編譯目錄。沒開 Developer Mode 就沒有 symlink 權限，導致 Kotlin 原始碼被靜默跳過（不報 error）。

### 修法
開啟 Windows Developer Mode：
```
start ms-settings:developers
```
開完之後必須 `flutter clean && flutter build apk --debug` 重建。

---

## 陷阱三：AGP 9.x 與 `flutter_inappwebview` 不相容

### 症狀
```
`getDefaultProguardFile('proguard-android.txt')` is no longer supported since AGP 9.0
BUILD FAILED
```

### 根本原因
`flutter_inappwebview_android 1.1.3` 及以下版本在 `build.gradle` 裡使用了 AGP 9.0 移除的 `proguard-android.txt`。

### 修法
`android/settings.gradle.kts` 把 AGP 降至 8.x：
```kotlin
plugins {
    id("dev.flutter.flutter-plugin-loader") version "1.0.0"
    id("com.android.application") version "8.7.3" apply false   // ← 9.x → 8.x
    id("org.jetbrains.kotlin.android") version "2.1.0" apply false
}
```
同時移除 `android/app/build.gradle.kts` 裡的 AGP 9 only 語法：
```kotlin
// 刪除這整個 block（AGP 9 才支援的 top-level kotlin {}）
kotlin {
    compilerOptions {
        jvmTarget = org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17
    }
}
```

---

## 附加：Dart SDK 版本對齊

`pubspec.yaml` 的 `sdk` 版本必須 ≤ 本機安裝的 Dart 版本：
```yaml
environment:
  sdk: ^3.11.0   # 必須與 flutter --version 顯示的 Dart 版本相符
```
不符合時 `flutter pub get` 直接失敗，所有後續都無法進行。

---

## 完整 Checklist（每次 Flutter Android build 前確認）

- [ ] `pubspec.yaml` sdk 版本 ≤ 本機 Dart 版本
- [ ] Windows Developer Mode 已開啟
- [ ] `android/app/build.gradle.kts` 有 `id("org.jetbrains.kotlin.android")`
- [ ] `android/settings.gradle.kts` AGP 版本與 plugins 相容
- [ ] 有用 `flutter_inappwebview` 時 AGP ≤ 8.x

## Recognition Pattern

遇到 `ClassNotFoundException: MainActivity` → 先查 Gradle task list 有無 `compileDebugKotlin`，沒有就是 Kotlin plugin 未套用或 Developer Mode 問題。

---

## Conformance Addendum

## When to Use
ECP Mobile (Flutter, C:\com\APP) 開發規範——固定專案位置與 lib/ feature-first 原始碼結構（非後端 tool\src），加上 Windows Android build 三大致命陷阱（Kotlin plugin 未套用、Developer Mode 未開、AGP 版本衝突）

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

## Verification
1. Confirm the intended files, services, or outputs exist in their expected state.
2. Run the most relevant syntax, configuration, build, or runtime check available for this Skill.
3. Report what was changed, what was tested, and any remaining limitation.
