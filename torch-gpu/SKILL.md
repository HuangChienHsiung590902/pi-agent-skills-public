---
name: torch-gpu
description: 使用本機已建好的 PyTorch GPU 虛擬環境 (C:\Users\HCH\venvs\torch-gpu) 來執行 Python，讓程式自動吃到 NVIDIA GPU (CUDA)。當使用者要求「用 GPU 跑」、「進 torch 環境」、「啟用 PyTorch 環境」、「用 cuda 執行」、或要跑任何需要 PyTorch/深度學習且想用顯卡加速的 Python 程式時使用。
---

在本機已建立好的 PyTorch GPU 虛擬環境中執行 Python。此環境已安裝 GPU 版 PyTorch (cu121)，可直接使用 NVIDIA GPU。

## 環境資訊

- **venv 路徑**：`C:\Users\HCH\venvs\torch-gpu`
- **Python 直接執行檔**：`C:\Users\HCH\venvs\torch-gpu\Scripts\python.exe`（Python 3.12）
- **pip 執行**：`C:\Users\HCH\venvs\torch-gpu\Scripts\python.exe -m pip ...`
- **已安裝**：torch 2.5.1+cu121、torchvision、torchaudio、numpy
- **GPU**：NVIDIA GeForce RTX 4060 Laptop GPU（驅動支援 CUDA 12.2）

## 核心原則

**永遠用完整路徑的 venv python，不要用系統的 `python`。**
系統預設 `python` 是 Microsoft Store 的 3.13，沒有 PyTorch；要用 GPU 一定要走這個 venv 的執行檔。

## 步驟

### 1. 執行使用者的 Python 程式（用 GPU）

執行腳本檔：
```powershell
& C:\Users\HCH\venvs\torch-gpu\Scripts\python.exe 你的腳本.py
```

執行一段 inline 程式：
```powershell
& C:\Users\HCH\venvs\torch-gpu\Scripts\python.exe -c "import torch; print(torch.cuda.is_available())"
```

在程式碼中選擇裝置的標準寫法（寫新程式時請套用）：
```python
import torch
device = "cuda" if torch.cuda.is_available() else "cpu"
# model = model.to(device)
# data = data.to(device)
```

### 2. 安裝額外套件到此環境

```powershell
& C:\Users\HCH\venvs\torch-gpu\Scripts\python.exe -m pip install 套件名稱
```

> 注意：若要裝會用到 GPU 的 PyTorch 生態套件，確保與 cu121 / torch 2.5.1 相容。

### 3. 驗證 GPU 是否可用（需要時）

```powershell
& C:\Users\HCH\venvs\torch-gpu\Scripts\python.exe -c "import torch; print('cuda available:', torch.cuda.is_available()); print('device:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU only')"
```

預期看到 `cuda available: True` 與 `NVIDIA GeForce RTX 4060 Laptop GPU`。

## 疑難排解

- **`torch.cuda.is_available()` 回傳 False**：先跑 `nvidia-smi` 確認驅動正常；確認確實是用 venv 的 python 而非系統 python。
- **`ModuleNotFoundError: No module named 'torch'`**：表示誤用了系統 python，請改用上面的完整路徑執行檔。
- **`Activate.ps1` 被執行原則擋住**：不需要啟用，直接用完整路徑的 `python.exe` 即可；或執行一次 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`。

---

## Conformance Addendum

## When to Use
使用本機已建好的 PyTorch GPU 虛擬環境 (C:\Users\HCH\venvs\torch-gpu) 來執行 Python，讓程式自動吃到 NVIDIA GPU (CUDA)。當使用者要求「用 GPU 跑」、「進 torch 環境」、「啟用 PyTorch 環境」、「用 cuda 執行」、或要跑任何需要 PyTorch/深度學習且想用顯卡加速的 Python 程式時使用。

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
