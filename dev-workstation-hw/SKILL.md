---
name: dev-workstation-hw
description: 本開發筆電(LG Gram 16 2025)的硬體規格、系統狀態、網路配置、電池健康度、已知限制與工作架構。使用者需要查閱本機能力/限制/IP/儲存空間/記憶體時使用。
triggers:
  - 筆電規格
  - 硬體
  - specs
  - 我的筆電
  - 本機
  - 工作站
  - laptop specs
  - workstation
  - 記憶體
  - RAM
  - 電池
  - SSD
  - 網路 IP
  - ZeroTier
  - 遠端主機
  - 本機限制
---

# 開發筆電硬體基本資料

**LG Gram 16 (型號 16T90TP-K.AD78C2)** — 2025 年款輕薄本

狀態：**良好**。系統乾淨、SSD/電池健康、配置均衡的現代 AI PC。

---

## 硬體規格

### CPU / GPU

| 項目 | 規格 | 說明 |
|---|---|---|
| **CPU** | Intel Core Ultra 7 **255H** | 16 核 16 執行緒 (6P + 8E + 2LPE),Arrow Lake-H |
| **GPU** | Intel **Arc 140T**(內顯,Xe2) | 共享 LPDDR5x,輕薄機標準。繁重 AI 運算已外包遠端 |
| **NPU** | 內建(Arrow Lake) | AI 推論加速(未充分利用) |

### 記憶體

| 項目 | 規格 |
|---|---|
| **容量** | **16GB**(LPDDR5x-8400) |
| **型態** | **封裝內焊死**,**不可擴充** |
| **速度** | DDR5-8400(LPDDR5x 高速晶粒) |
| **配置** | 8 個邏輯通道 × 2GB(SK Hynix) |

⚠️ **16GB 是硬上限** — 升級不可能。輕薄機代價。

### 儲存

| 磁碟 | 標籤 | 規格 | 狀態 |
|---|---|---|---|
| **主 SSD** | C: OS | SK Hynix 1TB NVMe (HFS001TEJ9X101N) | Healthy |
| **副區** | D: DATA | 虛擬分割,同 SSD | Healthy |

可用空間：C: **137 GB** (31% 已用) / D: **583 GB** (23% 已用) — **充足**。

### 螢幕

| 項目 | 規格 |
|---|---|
| 尺寸 | 16 吋(Gram 標配) |
| 規格 | OLED 2560×1600 @90Hz (推測,待確認) |

### 電池

| 項目 | 數據 |
|---|---|
| **設計容量** | 77,025 mWh |
| **現可充** | 72,230 mWh |
| **健康度** | **93.8%** (損耗 6.2%) |
| **循環次數** | 354 次(正常) |
| **現狀** | 98% charge,放電中 |

✅ 電池良好,預期還能再用 3-5 年。

---

## 系統資訊

| 項目 | 內容 |
|---|---|
| **OS** | Windows 11 專業版 10.0.26200 (25H2) |
| **安裝日期** | 2026-07-05 |
| **上次開機** | 2026-08-13 13:38 |
| **序號** | 502PGUH971990 |
| **BIOS** | American Megatrends v16T90TPF05 (2025-02-05) |

---

## 網路

### 有線 / 無線

| 介面 | 類型 | 規格 | IP/MAC |
|---|---|---|---|
| **Wi-Fi 2** | 無線 | Intel Wi-Fi 7 BE201 (320MHz 寬頻) | `192.168.100.147` / EC-4C-8C-30-71-4A |
| **ZeroTier** | 虛擬網 | ZeroTier One [08752e18b15e0bce] | `10.145.119.100` / CE-00-BC-44-45-5D |

### 重要連線

- **本機區網**:192.168.100.147
- **ZeroTier 虛擬網**:10.145.119.100(用來連線遠端工作主機)
- **遠端 GPU 主機**:10.145.119.19(ComfyUI/llama.cpp/xiaoai-voice-test 等)

---

## 即時狀態(採集於 2026-08-13)

| 項目 | 狀態 |
|---|---|
| **RAM 使用** | 6.8 / 15.5 GB (44%) 🟢 |
| **CPU 負載** | 5% 🟢 |
| **GPU 使用** | 5.6% 🟢 |
| **行程數** | 203 個(正常) |
| **待重開** | 無 ✅ |

### 記憶體大戶(Top 12)

1. svchost — 1.4 GB(Windows 系統服務)
2. msedgewebview2 — 1.0 GB(Edge 內核)
3. chrome — 1.0 GB(瀏覽器)
4. claude — 529 MB(Claude Code TUI)
5. shandianshuo — 365 MB(**本機閃電說 ASR 常駐**)
6. explorer — 346 MB(檔案管理)
7. dwm — 192 MB(桌面視窗管理)
8. WindowsTerminal — 177 MB(終端機)
9. SearchHost — 167 MB(Windows 搜尋)
10. pwsh — 163 MB(PowerShell)

---

## 已知限制 & 設計決策

### 限制

1. **16GB RAM 不可擴充**
   - 焊死在主板上(LPDDR5x),無升級路徑
   - 同時跑 Chrome + Edge WebView + Claude + ASR 時偶可壓力大
   - **但架構迴避了此限制** — 見下方

2. **內顯 Arc 140T 不適合本機重運算**
   - 輕薄機內顯標準
   - 大型 AI 模型推論/訓練不夠力

3. **驅動偏舊**(Intel Arc 驅動 2024-11,已 ~9 個月)
   - 建議更新以獲得最新效能/穩定性修正

### 工作架構(對應限制的策略)

**本機 = 控制端,重運算 = 外包遠端**

```
本機(192.168.100.147 / 10.145.119.100)
  ├─ Claude Code(TUI)     — 提示與決策
  ├─ Chrome/Edge          — 瀏覽與測試
  ├─ 終端機(pwsh/bash)    — 輕量工具
  └─ shandianshuo(ASR)    — 本機語音轉文
        ↓ ZeroTier tunnel
遠端 GPU 主機(10.145.119.19)
  ├─ ComfyUI:8190         — 圖像生成 / MiniMax-H3
  ├─ llama.cpp:8181       — 本地 LLM 推論(Qwythos)
  ├─ ollama:11434         — 多模態模型
  ├─ Cloudflare Tunnel    — 對外服務路由
  └─ xiaoai-voice-test    — 語音助理測試環境
```

**優點**：本機乾淨,16GB 約束迴避,遠端可充分利用多 GPU。

---

## 待辦項

> 採集日期 2026-08-13 時的建議:

### 🔴 優先

- **Windows Defender 即時保護目前關閉**
  - 檢查是否刻意關閉(via `win11debloat`)或系統異常
  - 若無特殊原因,建議開啟:
    ```powershell
    Set-MpPreference -DisableRealtimeMonitoring $false
    ```

### 🟡 次要

- **Intel Arc 驅動更新**(從 2024-11 → latest)
  - 執行 Intel Arc Control Center 檢查更新

- **Node.js/npm 版本檢查**
  - Pipecat、ComfyUI 等工具可能要求特定版本

---

## 快速查詢指令

```powershell
# 檢查現況
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer,Model,TotalPhysicalMemory
Get-CimInstance Win32_OperatingSystem | Select-Object Caption,BuildNumber,LastBootUpTime
Get-NetAdapter -Name "Wi-Fi*","ZeroTier*" | Format-Table Name,Status,LinkSpeed

# 檢查 RAM/CPU 負載
Get-Process | Group-Object ProcessName | Sort-Object {($_.Group | Measure-Object WorkingSet64 -Sum).Sum} -Descending | Select-Object -First 10 Name, @{N='RAM_GB';E={[math]::Round(($_.Group | Measure-Object WorkingSet64 -Sum).Sum/1GB,2)}}

# 檢查電池健康度
powercfg /batteryreport /XML /OUTPUT "$env:TEMP\battery.xml"
# 然後用記事本開 $env:TEMP\battery.xml,搜尋 DesignCapacity / FullChargeCapacity

# 測試遠端連線
Test-NetConnection 10.145.119.19 -Port 8190  # ComfyUI
Test-NetConnection 10.145.119.19 -Port 8181  # llama.cpp
```

---

## 相關 Skills

- `dotfiles-to-d-system` — 配置集中化(若 C 槽要凍結時用)
- `aipower-docker-local` — 本機 ECP/aipower 實例
- `minimax-h3-comfyui` — 遠端唯一 ComfyUI / MiniMax-H3 雙 GPU 部署
- `llama-vision-test` — 遠端 LLM 視覺模型
- `xiaoai` — 遠端語音助理棧
- `windows-asr-shandianshuo` — 本機 ASR 常駐服務
- `win11debloat` — Windows 精簡(若之前用過)

---

## Conformance Addendum

## When to Use
本開發筆電(LG Gram 16 2025)的硬體規格、系統狀態、網路配置、電池健康度、已知限制與工作架構。使用者需要查閱本機能力/限制/IP/儲存空間/記憶體時使用。

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
