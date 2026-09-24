---
name: qwen38-27b-llamacpp-omniroute
description: 在遠端 GPU 主機 10.145.119.19 用 llama.cpp Docker 跑 Qwen3.8-27B Abliterated / 越獄版 GGUF，並把它接進本機 OmniRoute／清箋的 OpenAI-compatible provider。用在：要啟動、檢查、重裝 Qwen3.8 27B abliterated，或使用 model id `/models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf` 時。現行宿主 port 是 8080（與 30B-A3B、Spark 輪流，見 qwen-spark-docker-desktop-switch）；歷史文件裡的 8081 不要當現行 endpoint。
---

# Qwen3.8-27B Abliterated on llama.cpp + OmniRoute

## 現況總覽（已完成部署）

遠端主機：

```text
10.145.119.19  hostname: gigabyte  user: hch
GPU: 2 x RTX 4060 Ti 16GB
```

llama.cpp API（現行；與 30B-A3B、Spark 輪流佔用，不能同時開）：

```text
http://10.145.119.19:8080
```

歷史文件與部分 OmniRoute provider 曾寫 8081。2026-09 之後 27B compose 的 `ports` 是 `8080:8080`。先 `curl /v1/models` 確認現在 8080 是不是這顆 27B。

**保留容器**：使用者會在 27B／30B／Spark 之間換來換去。切走時用 `docker stop` 或 Docker Desktop Stop，**不要** `docker compose down`（會 Removed 容器）。誤刪後：`docker compose -f docker-compose.qwen38-27b-abliterated.yml up --no-start`。

本機 OmniRoute：

```text
Dashboard: http://localhost:20128
OpenAI-compatible endpoint: http://localhost:20128/v1
```

遠端模型檔：

```text
/home/hch/models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf        # 約 16G
/home/hch/models/mmproj-Qwen3.8-27B-ABLITERATED-F16.gguf    # 約 885M
```

遠端 compose 檔：

```text
/home/hch/llama-cpp-docker/docker-compose.qwen38-27b-abliterated.yml
```

容器名：

```text
llama-cpp-qwen38-27b-abliterated
```

OmniRoute provider node：

```text
id: openai-compatible-chat-7bc1f0dc-4afe-487a-8c0e-96555ad5954a
name: Qwen3.8 27B Abliterated llama.cpp
prefix: qwen38
baseUrl: http://10.145.119.19:8081/v1
connection: main
```

OmniRoute 可用 model id：

```text
qwen38//models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
openai-compatible-chat-7bc1f0dc-4afe-487a-8c0e-96555ad5954a//models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
```

優先使用短一點的：

```text
qwen38//models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
```

> 注意：裸 alias `qwen38` 目前不能直接當 model id 使用，OmniRoute 會回 `Unable to determine provider for model 'qwen38'`。必須使用 provider-prefixed model id。

> pi 啟動時直接用 `pi --provider omni --model "qwen38/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf"`（單斜線、不含 `/models/` 路徑）實測可行，OmniRoute 會自己解析成功，不用堆完整 `/models/` 路徑或那串 UUID provider id。這是給 pi CLI 或 herdr pane 啟動 pi 時最簡潔的寫法。

---

## 1. 檢查遠端 llama.cpp 狀態

```bash
ssh hch@10.145.119.19 "hostname; docker ps -a --filter name=llama-cpp-qwen38-27b-abliterated; curl -s http://127.0.0.1:8081/health || true"
```

健康檢查成功應回：

```json
{"status":"ok"}
```

查模型：

```bash
ssh hch@10.145.119.19 "curl -s http://127.0.0.1:8081/v1/models"
```

查看 logs：

```bash
ssh hch@10.145.119.19 "docker logs --tail 120 llama-cpp-qwen38-27b-abliterated 2>&1"
```

模型載入完成時 log 會看到：

```text
llama_server: model loaded
llama_server: listening on http://0.0.0.0:8080
```

---

## 2. 啟動 / 停止

啟動：

```bash
ssh hch@10.145.119.19 "cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwen38-27b-abliterated.yml up -d"
```

停止：

```bash
ssh hch@10.145.119.19 "cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwen38-27b-abliterated.yml down"
```

重啟：

```bash
ssh hch@10.145.119.19 "cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwen38-27b-abliterated.yml down && docker compose -f docker-compose.qwen38-27b-abliterated.yml up -d"
```

---

## 3. 直接測 llama.cpp

健康檢查：

```bash
curl http://10.145.119.19:8081/health
```

模型列表：

```bash
curl http://10.145.119.19:8081/v1/models
```

Chat completions：

```bash
curl http://10.145.119.19:8081/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen",
    "messages": [
      {"role": "user", "content": "/no_think 用繁體中文簡短介紹你自己"}
    ],
    "max_tokens": 512,
    "temperature": 0.6,
    "stream": false
  }'
```

### Qwen thinking 注意

這顆 Qwen 預設可能會輸出 `reasoning_content`。若要直接回答，prompt 前面加：

```text
/no_think
```

實測速度約：

```text
15 tokens/s
```

---

## 4. 模型下載 / 重裝

HuggingFace repo：

```text
Blackfrost-AI/Qwen3.8-27B-ABLITERATED-GGUF
```

下載檔案：

```text
Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
mmproj-Qwen3.8-27B-ABLITERATED-F16.gguf
```

背景下載腳本：

```bash
ssh hch@10.145.119.19 "cat > /home/hch/download_qwen38_27b_abliterated.py <<'PY'
from huggingface_hub import hf_hub_download
import os, shutil

repo = 'Blackfrost-AI/Qwen3.8-27B-ABLITERATED-GGUF'
items = [
    ('Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf', '/home/hch/models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf'),
    ('mmproj-Qwen3.8-27B-ABLITERATED-F16.gguf', '/home/hch/models/mmproj-Qwen3.8-27B-ABLITERATED-F16.gguf'),
]

tmp = '/home/hch/models/_qwen38_27b_abliterated_tmp'
os.makedirs(tmp, exist_ok=True)

for fname, dest in items:
    if os.path.exists(dest) and os.path.getsize(dest) > 1024 * 1024:
        print(f'[skip] {dest} exists ({os.path.getsize(dest)/1024/1024/1024:.2f} GiB)', flush=True)
        continue
    print(f'[download] {repo}/{fname} -> {dest}', flush=True)
    p = hf_hub_download(repo_id=repo, filename=fname, local_dir=tmp)
    shutil.move(p, dest)
    print(f'[done] {dest} ({os.path.getsize(dest)/1024/1024/1024:.2f} GiB)', flush=True)

print('[all done]', flush=True)
PY
nohup python3 /home/hch/download_qwen38_27b_abliterated.py > /home/hch/download_qwen38_27b_abliterated.log 2>&1 &"
```

看下載進度：

```bash
ssh hch@10.145.119.19 "ps -ef | grep -E 'download_qwen38|huggingface' | grep -v grep || true; tail -80 /home/hch/download_qwen38_27b_abliterated.log; ls -lh /home/hch/models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf /home/hch/models/mmproj-Qwen3.8-27B-ABLITERATED-F16.gguf 2>/dev/null || true"
```

---

## 5. llama.cpp compose 檔內容

檔案：

```text
/home/hch/llama-cpp-docker/docker-compose.qwen38-27b-abliterated.yml
```

內容：

```yaml
services:
  llama-cpp-qwen38-27b-abliterated:
    build: .
    image: llama-cpp:local
    container_name: llama-cpp-qwen38-27b-abliterated
    command:
      - --host
      - 0.0.0.0
      - --port
      - "8080"
      - -m
      - /models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
      - --mmproj
      - /models/mmproj-Qwen3.8-27B-ABLITERATED-F16.gguf
      - -ngl
      - "99"
      - -c
      - "262144"
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

---

## 6. 加進 OmniRoute（已完成；重建時照做）

### 6.1 使用 dashboard 新增 OpenAI-compatible provider

頁面：

```text
http://localhost:20128/dashboard/providers
```

如果要用 UI 操作 dashboard，先使用 `connect-chrome` skill 接管已開啟的 Chrome debug port 9222；不要自動啟動新 Chrome。

在 dashboard：

```text
Providers → API 金鑰相容提供者 → 新增 OpenAI 相容
```

填：

```text
名稱: Qwen3.8 27B Abliterated llama.cpp
字首: qwen38
API 類型: 聊天完成
基礎 URL: http://10.145.119.19:8081/v1
API 金鑰（用於檢查）: dummy
模型 ID（選用）: /models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
```

按「檢查」，顯示「有效」後按「新增」。

### 6.2 新增 connection

進 provider detail 頁後，新增 API key connection：

```text
名稱: main
API key: dummy
預設模型: /models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
驗證模型 ID: /models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
優先順序: 1
```

按「檢查」，顯示「有效」後「儲存」。

### 6.3 從 /models 匯入

在 provider detail 頁的「可用模型」按：

```text
從 /models 匯入
```

匯入後應看到：

```text
qwen38//models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
```

---

## 7. OmniRoute 驗證

列 provider nodes：

```bash
omniroute api provider-nodes get-api-provider-nodes
```

列 OmniRoute models：

```bash
python - <<'PY'
import urllib.request, json
j = json.load(urllib.request.urlopen('http://localhost:20128/v1/models'))
for m in j.get('data', []):
    mid = m.get('id', '')
    if any(s.lower() in mid.lower() for s in ['qwen38', 'qwen3.8', 'abliterated']):
        print(mid)
PY
```

測 OmniRoute chat：

```bash
python - <<'PY'
import urllib.request, json
model = 'openai-compatible-chat-7bc1f0dc-4afe-487a-8c0e-96555ad5954a//models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf'
payload = {
    'model': model,
    'messages': [{'role': 'user', 'content': '/no_think 請只回答兩個字：成功'}],
    'max_tokens': 64,
    'temperature': 0.1,
    'stream': False,
}
req = urllib.request.Request(
    'http://localhost:20128/v1/chat/completions',
    data=json.dumps(payload).encode('utf-8'),
    headers={'Content-Type': 'application/json', 'Authorization': 'Bearer dummy'},
)
with urllib.request.urlopen(req, timeout=120) as r:
    print(r.read().decode('utf-8', 'ignore')[:2000])
PY
```

---

## 8. 已知坑

### 坑一：裸 alias `qwen38` 不可用

曾嘗試：

```bash
omniroute api models post-api-models-alias --body '{"alias":"qwen38","model":"qwen38//models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf"}'
```

但實測 `/v1/chat/completions` 使用裸 `qwen38` 會失敗：

```text
Unable to determine provider for model 'qwen38'. Use a provider/model prefix ...
```

所以使用者或 pi model picker 應使用 provider-prefixed id：

```text
qwen38//models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf
```

### 坑二：OmniRoute 可能回 SSE chunk

有時即使請求沒明確指定 streaming，OmniRoute/上游回來可能是 `data: ...` chunk 形式。若用簡單 `json.load()` 解析會炸。測試時建議加：

```json
"stream": false
```

並準備處理 SSE 格式。

### 坑三：終端機中文亂碼不是模型壞

Windows bash/cmd 顯示中文可能變成 `���`，這通常是終端機編碼問題。若要驗證內容，保存 UTF-8 檔案或用瀏覽器/支援 UTF-8 的終端查看。

### 坑五：跟 flm-audio-server 搶 VRAM，啟動前一定要先確認

這台機器（`10.145.119.19`）兩張 RTX 4060 Ti 各只有 16GB。`flm-audio-server`（見
`flm-audio-deploy` skill）正常運作時兩張卡各吃約 8.2GB/8.4GB，剩下可用空間只有
~7-8GB/卡。Qwen3.8-27B Q4_K_M 光權重就要 16GB+（尚未含 KV cache），跟 flm-audio 同時開
根本塞不下。

實測症狀：容器 `docker ps` 顯示 `Up`，log 也印出一堆正常的 tensor 載入訊息，但接著整個
卡住不動——`docker top` 看容器內 `llama-server` process 的 CPU time 幾十秒都不動
（例如卡在 `00:00:01`），GPU process list（`nvidia-smi --query-compute-apps`）裡這個
process 只吃了 ~120MiB，代表它根本沒真的把模型層搬上 GPU，卡死在記憶體配置那一步。
啟動前先跑：

```bash
ssh hch@10.145.119.19 "nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv"
```

如果兩張卡已經各用了 8GB+，先停掉 flm-audio-server 騰出空間再啟動 qwen38：

```bash
ssh hch@10.145.119.19 "docker stop flm-audio-server"
# 啟動 qwen38 之後，nvidia-smi 應該顯示兩張卡各用約 12GB（context 32768 時）
# 或約 15GB（context 調到 262144 時，見下方 context window 章節）
```

反過來，如果之後要換回用 flm-audio-server，記得先 `docker compose ... down` 停掉
qwen38，再 `docker start flm-audio-server`。這兩個服務目前设計上不能同時開。

### 坑六：n_slots=4 + kv_unified=true，多個並發請求會共用同一個 context pool

啟動 log 會印 `n_slots = 4, kv_unified = 'true'`，代表 `-c` 設定的 context window
是所有並發請求共用的一個池子，不是每個 slot各自獨立擁有那麼多。如果同時有 3 個 pi
pane（或其他 client）都在打這個 endpoint，且每個請求的 prompt 都很大（例如 pi 帶了完整
skills 索引，可能單一請求就吃到 8-9 萬 token），3 個一起送很容易加總超過預設的
`32768`，導致 `[500]: Context size has been exceeded`。

實測解法：把 `-c` 從 `32768` 調到 `262144`（本檔案目前的值），VRAM 用量會從約
12GB/12GB 漲到約 15GB/15GB（在沒有 flm-audio-server 佔用時還撐得住，非常接近上限，
沒什麼餘裕）。如果之後要讓更多 pane 同時打，或 pi 的 prompt 又變大，可能還要再往上調，
但要注意這台機器 VRAM 已經很緊繃，調太大會直接 OOM 開不起來（症狀同坑五：卡住不動、
process 幾乎不吃 GPU）。

真正治本的做法是讓每個 pi pane 自己把 prompt 變小，見
`herdr-pi-lightweight-panes` skill 的 `--no-skills` + `--mcp-config` 技巧——把
pi 的 prompt 從 ~88k token 壓到 ~3k token 之後，3 個 pane 同時打 32768 context 都
綽綽有餘，不需要一直往上加 context window。

### 坑七：`compose down` 會刪掉容器，使用者只要輪流切換時不要用

2026-09-21 為了上 30B 對 27B 做了 `docker compose down`，容器變成 Removed。使用者要求 27B Docker 必須留著以便換來換去。正確切走：`docker stop llama-cpp-qwen38-27b-abliterated`。誤刪後，在 30B 已佔 8080 時用：

```bash
cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwen38-27b-abliterated.yml up --no-start
```

容器應為 Created／Stopped，不要立刻 `up -d`。輪流切換細節見 `qwen-spark-docker-desktop-switch`；30B 部署見 `qwen3-30b-a3b-llamacpp`。

### 坑四：這顆有 vision/mmproj，但文字用途優先

此模型載入了：

```text
mmproj-Qwen3.8-27B-ABLITERATED-F16.gguf
```

`/v1/models` 會顯示 `multimodal`。但目前主要用途是文字 chat；若要測圖像輸入，需用 OpenAI-compatible image content 格式另外測。

---

## 9. 快速回復流程

若使用者說「Qwen3.8 掛了 / OmniRoute 找不到 / llama.cpp 8081 不通」，依序做：

```bash
# 1. 遠端容器與健康檢查
ssh hch@10.145.119.19 "docker ps -a --filter name=llama-cpp-qwen38-27b-abliterated; curl -s http://127.0.0.1:8081/health || true"

# 2. 如果沒跑，啟動
ssh hch@10.145.119.19 "cd /home/hch/llama-cpp-docker && docker compose -f docker-compose.qwen38-27b-abliterated.yml up -d"

# 3. 等 model loaded
ssh hch@10.145.119.19 "docker logs --tail 120 llama-cpp-qwen38-27b-abliterated 2>&1"

# 4. 本機 OmniRoute 確認 model id
python - <<'PY'
import urllib.request, json
j = json.load(urllib.request.urlopen('http://localhost:20128/v1/models'))
for m in j.get('data', []):
    mid = m.get('id', '')
    if 'qwen38' in mid.lower() or 'abliterated' in mid.lower():
        print(mid)
PY
```

---

## Conformance Addendum

## When to Use
在遠端 GPU 主機 10.145.119.19 用 llama.cpp Docker 跑 Qwen3.8-27B Abliterated / 越獄版 GGUF，並把它接進本機 OmniRoute 的 OpenAI-compatible provider。用在：要啟動、檢查、重裝 Qwen3.8 27B abliterated，處理 llama.cpp port 8081，或使用 OmniRoute model id `qwen38//models/Qwen3.8-27B-ABLITERATED-Q4_K_M.gguf` 時。

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
