---
name: win11debloat
description: 當使用者想要清除 Windows 11 臃腫軟體、停用遙測、移除預裝應用程式，或自訂 Windows 11 UI 和隱私設定時使用
---

# Win11Debloat MCP

Win11Debloat 是一個 Windows 11 精簡化腳本，可移除臃腫軟體、停用遙測，並自訂 Windows 11 UI。

## MCP 工具

此 skill 提供三個 MCP 工具：

### 1. run_debloat

以指定選項執行 Win11Debloat。**需要管理員權限。**

```python
# 隱私強化 - 停用所有遙測
win11debloat_run_debloat(
    disable_telemetry=True,
    disable_search_history=True,
    disable_bing=True,
    disable_delivery_optimization=True,
    disable_update_asap=True,
    prevent_update_reboot=True,
    disable_fast_startup=True,
    disable_bitlocker=True,
    disable_modern_standby=True,
    disable_storage_sense=True,
    create_restore_point=True,
    silent=True
)

# UI 清理
win11debloat_run_debloat(
    enable_dark_mode=True,
    taskbar_align_left=True,
    hide_taskview=True,
    hide_chat=True,
    disable_copilot=True,
    disable_widgets=True,
    revert_context_menu=True
)

# 移除臃腫軟體
win11debloat_run_debloat(
    remove_gaming_apps=True,
    remove_comm_apps=True,
    remove_hp_apps=True,
    create_restore_point=True
)

# 安全預設精簡版（建議首次執行使用）
win11debloat_run_debloat(
    run_defaults_lite=True,
    create_restore_point=True
)
```

### 2. get_presets

回傳預定義的預設配置：

- `privacy_boost` - 停用遙測、Bing、搜尋記錄
- `debloate_basics` - 移除遊戲/通訊/HP 臃腫軟體
- `ui_cleanup` - 清理工作列（隱藏工作檢視、Copilot、小工具、聊天）
- `game_performance` - 停用 Game Bar、DVR
- `start_menu_fixes` - 清理開始功能表
- `quick_defaults_lite` - 執行安全最小精簡
- `full_debloat` - 積極完整精簡

### 3. list_options

列出所有可用的 Win11Debloat 選項，依類別整理。

## 所有可用選項

| 類別 | 選項 |
|------|------|
| **應用程式移除** | `remove_apps`, `remove_gaming_apps`, `remove_comm_apps`, `remove_hp_apps`, `remove_w11_outlook`, `force_remove_edge` |
| **Windows 功能** | `enable_sandbox`, `enable_wsl`, `disable_dvr`, `disable_game_bar` |
| **隱私** | `disable_telemetry`, `disable_search_history`, `disable_bing`, `disable_delivery_optimization`, `disable_update_asap`, `prevent_update_reboot`, `disable_fast_startup`, `disable_bitlocker`, `disable_modern_standby`, `disable_storage_sense` |
| **UI** | `enable_dark_mode`, `disable_transparency`, `disable_animations`, `taskbar_align_left`, `hide_taskview`, `hide_search`, `hide_chat`, `disable_copilot`, `disable_recall`, `disable_widgets` |
| **開始功能表** | `disable_start_recommended`, `disable_start_all_apps`, `clear_start`, `replace_start` |
| **檔案總管** | `explorer_to_home`, `explorer_to_this_pc`, `explorer_to_downloads`, `hide_home`, `hide_gallery`, `hide_onedrive`, `hide_3d_objects` |
| **右鍵選單** | `revert_context_menu` |
| **協助工具** | `disable_drag_tray`, `disable_mouse_acceleration`, `disable_sticky_keys`, `disable_window_snapping`, `disable_snap_assist`, `disable_snap_layouts` |
| **執行模式** | `run_defaults`, `run_defaults_lite`, `silent`, `verbose`, `no_restart_explorer`, `create_restore_point` |

## 注意事項

- 需要**管理員權限**（會出現 UAC 提示）
- 腳本首次執行時會從 GitHub 下載最新版本
- 使用 `create_restore_point=True` 在變更前建立系統還原點
- 部分變更需要重新啟動才能生效

---

## Conformance Addendum

## When to Use
當使用者想要清除 Windows 11 臃腫軟體、停用遙測、移除預裝應用程式，或自訂 Windows 11 UI 和隱私設定時使用

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
