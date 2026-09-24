---
name: qwythos-27b-llamacpp-omniroute
description: 在遠端 GPU 主機 10.145.119.19 用 llama.cpp Docker 跑 Qwythos-27B-v1（empero-ai，Qwen3.5-27B 底座的推理/agent模型），並接進本機 OmniRoute 的 OpenAI-compatible provider。用在：要啟動、檢查、重裝 Qwythos-27B，處理 llama.cpp port 8081，使用 OmniRoute model id `qwythos/models/Qwythos-27B-Q4_K_M.gguf`，或評估這顆模型的推理/寫程式/動手操作能力時。
---

# Qwythos-27B-v1 on llama.cpp + OmniRoute

## 現況總覽（已完成部署，2026-08-20）

遠端主機：

```text
10.145.119.19  hostname: gigabyte  user: hch
GPU: 2 x RTX 4060 Ti 16GB
```

llama.cpp API：

```text
http://10.145.119.19:8081
```

本機 OmniRoute：

```text
Dashboard: http://localhost:20128
OpenAI-compatible endpoint: http://localhost:20128/v1
```

遠端模型檔（HF repo `empero-ai/Qwythos-27B-v1-GGUF`）：

```text
/home/hch/models/Qwythos-27B-Q4_K_M.gguf   # 16.13 GiB
```

遠端 compose 檔：

```text
/home/hch/llama-cpp-docker/docker-compose.qwythos-27b.yml
```

容器名：

```text
llama-cpp-qwythos-27b
```

docker image（**注意，不是共用 qwen38 那顆舊 image**，見下方「坑一」）：

```text
llama-cpp:fresh-20260820
```

OmniRoute provider：

```text
名稱: Qwythos 27B llama.cpp
字首: qwythos
baseUrl: http://10.145.119.19:8081/v1
connection: main
```

OmniRoute / pi 可用 model id：

```text
qwythos/models/Qwythos-27B-Q4_K_M.gguf
```

---

## 坑一：必須用「新編譯」的 llama.cpp，舊的 qwen38 image 載入會失敗

Qwythos-27B 底座是 **Qwen3.5**（混合 Gated-DeltaNet + 注意力層架構），跟 qwen38 用的
標準 Qwen3 架構不一樣。`qwen38-27b-llamacpp-omniroute` skill 裡那個 2026-08-12 build
的 `llama-cpp:local` image **不支援**這個新架構。必須重新 build 一份新的：

```bash
ssh hch@10.145.119.19 "cd /home/hch/llama-cpp-docker && docker build -t llama-cpp:fresh-20260820 . > /home/hch/llama-cpp-build-fresh.log 2>&1 &"
```

**這個 build 本身有一個獨立的坑**：`git clone --depth 1` 這步實測連續兩次都在
`RPC failed; curl 92 HTTP/2 stream 0 was not closed cleanly` 這個錯誤上失敗，是這台機器
對 GitHub 走 HTTP/2 不穩，不是隨機網路抖動。修法是在 Dockerfile 的 clone 那行強制用
HTTP/1.1：

```bash
ssh hch@10.145.119.19 "sed -i 's|RUN git clone --depth 1 https://github.com/ggerganov/llama.cpp.git .|RUN git -c http.version=HTTP/1.1 -c http.postBuffer=1048576000 clone --depth 1 https://github.com/ggerganov/llama.cpp.git .|' /home/hch/llama-cpp-docker/Dockerfile"
```

改完再重跑 build 就會成功（實測整個 build 約 8-10 分鐘）。build log 裡看到
`naming to docker.io/library/llama-cpp:fresh-20260820 done` 就是完成了。

**這份新 image 是通用的**，不是 Qwythos 專屬——之後如果要跑任何比 2026-08-12 更新的
模型架構（例如更新的 Qwen 版本），都可以先檢查這個 fresh image 能不能載入，不用每次
重新 build。

---

## 1. 檢查 / 啟動 / 停止

```bash
# 狀態 + 健康檢查
ssh hch@10.145.119.19 "docker ps -a --filter name=llama-cpp-qwythos-27b; curl -s http://127.0.0.1:8081/health || true"

# 啟動
ssh hch@10.145.119.19 "cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwythos-27b.yml up -d"

# 停止
ssh hch@10.145.119.19 "cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwythos-27b.yml down"

# 看 log
ssh hch@10.145.119.19 "docker logs --tail 120 llama-cpp-qwythos-27b 2>&1"
```

模型載入完成時 log 會看到 `llama_server: model loaded` + `listening on
http://0.0.0.0:8080`，沒有 `unknown architecture` 之類的錯誤就代表坑一那個架構支援
問題沒發生。

**這台機器同時間只能跑一個大模型**（見 `qwen38-27b-llamacpp-omniroute` 坑五/坑六，
`flm-audio-deploy` skill）——qwen38、flm-audio-server、Qwythos 三個都吃這兩張卡的
VRAM，換著跑之前記得先 `docker stop`/`down` 現在在用的那個。

---

## 2. docker-compose.qwythos-27b.yml 內容

```yaml
services:
  llama-cpp-qwythos-27b:
    image: llama-cpp:fresh-20260820
    container_name: llama-cpp-qwythos-27b
    command:
      - --host
      - 0.0.0.0
      - --port
      - "8080"
      - -m
      - /models/Qwythos-27B-Q4_K_M.gguf
      - -ngl
      - "99"
      - -c
      - "131072"
      - --flash-attn
      - "on"
      - --cache-type-k
      - q8_0
      - --cache-type-v
      - q8_0
      - --temp
      - "0.6"
      - --top-p
      - "0.95"
      - --top-k
      - "20"
      - --repeat-penalty
      - "1.05"
      - --jinja
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    ports:
      - "8081:8080"
    restart: "no"
    runtime: nvidia
    volumes:
      - /home/hch/models:/models:ro
```

實測 VRAM 用量約 11.5GB / 12GB（雙卡各一半），比 qwen38 略高但同量級。

---

## 3. 模型下載

HuggingFace repo：`empero-ai/Qwythos-27B-v1-GGUF`，只抓 `Qwythos-27B-Q4_K_M.gguf`
（15.78 GiB，官方標示；實際下載落地 16.13 GiB）。用跟 qwen38 skill 裡同一套
`hf_hub_download` 背景下載腳本模式即可，把 repo/filename 換掉。這個 repo**沒有**
mmproj 檔（Qwythos 雖然論文說繼承了 vision tower，但這個 GGUF release 目前只有純文字
權重，`/v1/models` 不會顯示 multimodal）。

---

## 4. 加進 OmniRoute

跟 `qwen38-27b-llamacpp-omniroute` skill 的「6. 加進 OmniRoute」完全同一套流程
（Providers → 新增 OpenAI 相容 → 填 base URL `http://10.145.119.19:8081/v1` →
新增連線 → API key 隨便填 dummy → Default Model / 驗證模型ID 都填
`/models/Qwythos-27B-Q4_K_M.gguf` → 檢查有效 → 儲存，儲存後會自動彈出「匯入模型」
視窗匯入 1 個模型）。這裡不重複貼截圖流程，照那份 skill的步驟做，只是換掉名稱/字首/
model 路徑。

---

## 5. 能力實測結論（2026-08-20）

用兩個 herdr pi pane（`--no-skills --mcp-config <空設定>`，避免大 prompt 拖慢，見
`herdr-pi-lightweight-panes` skill）分別測試：

- **純邏輯推理**：經典燈泡開關謎題，答案跟推理過程完全正確，繁體中文表達清楚。
- **純寫程式**：LIS（最長嚴格遞增子序列）O(n log n) 演算法跟程式碼完全正確；自己寫的
  10 組測試案例裡有 2 組「期望值」手算錯（不是演算法錯，是它出題時心算錯），程式邏輯
  本身沒問題（直接執行驗證過）。
- **動手操作工具（建檔、跑指令、讀檔）：不可靠**，詳見 memory
  [[project_qwythos27b_tool_calling_broken]]——實測過工具呼叫會卡住不回應，或生成
  近 9000 個 token 的失控回應（正常工具呼叫該是幾百字內），無法穩定吐出結構化
  tool_call。跟 Holo3.1 那次「pi 上工具呼叫實質壞掉」是同一種根因猜測（模型輸出格式
  沒被正確解析成 tool_calls）。**純文字問答/推理/寫程式可以放心用，但不要指派會
  實際動手操作檔案系統/執行指令的任務給它**，容易卡死整個 pi pane，卡住時直接關
  pane、必要時重啟容器（見上面「1. 檢查/啟動/停止」），不用一直等。

## 相關

- `qwen38-27b-llamacpp-omniroute`：同一台主機、同樣手法的姊妹部署，VRAM 共用限制、
  OmniRoute 註冊 UI 步驟細節都在那邊。
- `flm-audio-deploy`：同一台主機的第三個大模型服務，一樣要輪流開。
- `herdr-pi-lightweight-panes`：多個 pi pane 共用一顆自架模型時，`--no-skills` +
  `--mcp-config` 瘦身 prompt 的方法。
- memory [[project_qwythos27b_tool_calling_broken]]：動手能力測試的完整記錄。

---

## Conformance Addendum

## When to Use
在遠端 GPU 主機 10.145.119.19 用 llama.cpp Docker 跑 Qwythos-27B-v1（empero-ai，Qwen3.5-27B 底座的推理/agent模型），並接進本機 OmniRoute 的 OpenAI-compatible provider。用在：要啟動、檢查、重裝 Qwythos-27B，處理 llama.cpp port 8081，使用 OmniRoute model id `qwythos/models/Qwythos-27B-Q4_K_M.gguf`，或評估這顆模型的推理/寫程式/動手操作能力時。

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
