---
name: openjev
description: 當任務需要 NLI 判定（entailment/contradiction/neutral）、多選題 rerank、拿參考答案評分候選、或偏好/決策類問題（如「明天吃什麼」）時，呼叫本機 openjev Docker 服務。不生成文字，只在給定的選項/陳述裡「判」；偏好決策一律走「約束排除法」（contradiction 分數），不用 entailment 當推薦。
---

# openjev — NLI cross-encoder 服務

## 端點

| 項目 | 值 |
|---|---|
| Host | `10.145.119.19`（ssh: `hch@10.145.119.19`） |
| Port | **8090** |
| Base URL | `http://10.145.119.19:8090` |
| 容器名 | `openjev-svc` |
| Image | `openjev:cuda`（CPU bf16 模式） |
| 權重 | 主機掛載（ro），不隨容器消失 |

## 端點說明

### `POST /predict`
NLI 三類判定。輸入一組 (premise, hypothesis) pair，回 contradiction/entailment/neutral 概率。

```json
{"pairs": [{"premise": "...", "hypothesis": "..."}]}
```

回應：
```json
{"probabilities": [{"contradiction": 0.01, "entailment": 0.94, "neutral": 0.05}]}
```

### `POST /rerank`
多選題，回 entailment 最高的選項 index + choice。

```json
{"question": "...", "options": ["A", "B", "C"], "hyp_fmt": "The correct answer is: {}"}
```

回應：
```json
{"index": 1, "choice": "B", "entailment": [0.02, 0.91, 0.03]}
```

### `POST /grade`
拿參考答案評分候選答案。

```json
{"question": "...", "reference": "完整參考答案句", "candidate": "候選答案"}
```

回應：
```json
{"label": "entailment|contradiction|neutral", "probabilities": {"...": 0.94}}
```

### `GET /health`
回 device、weights path、max_len。

## 呼叫範例（bash）

```bash
B=http://10.145.119.19:8090

# predict
curl -fsS $B/predict -H 'Content-Type: application/json' \
  -d '{"pairs":[{"premise":"It is raining.","hypothesis":"The ground is wet."}]}'

# rerank
curl -fsS $B/rerank -H 'Content-Type: application/json' \
  -d '{"question":"2+2=?","options":["3","4","5"]}'

# grade（reference 必須是完整句，不能只給數字/單詞）
curl -fsS $B/grade -H 'Content-Type: application/json' \
  -d '{"question":"When did WW2 end?","reference":"World War 2 ended in 1945.","candidate":"It ended after the atomic bombs, in August 1945."}'
```

## Python（requests）

```python
import requests
B = "http://10.145.119.19:8090"
r = requests.post(f"{B}/rerank", json={"question":"...", "options":[...]}).json()
g = requests.post(f"{B}/grade", json={"question":"...", "reference":"...", "candidate":"..."}).json()
p = requests.post(f"{B}/predict", json={"pairs":[{"premise":"...","hypothesis":"..."}]}).json()
```

## 使用準則（重要）

1. **不生成文字**——只能在給定的選項/陳述裡「選」或「判」。
2. **無參考 rerank 在知識密集題上≈隨機**（GPQA ~chance、MMLU 0.47）。有參考的 grade 才可靠（0.94+）。
3. **Grade 的 reference 必須是完整句**。短 reference（「1989」）會失靈（回 neutral）。
4. **Contradiction bias**：無關係的 pair 可能被誤判為 contradiction（~0.99）。對「不確定」的 pair 要保守。
5. **CPU 延遲 ~7s/次**。非即時；GPU + `causal-conv1d`/`flash-linear-attention` 可壓到 ~50ms。
6. **max_len = 4096 tokens**，超長輸入被截斷。

## 管理（ssh hch@10.145.119.19）

```bash
# 狀態
docker ps | grep openjev-svc

# 日誌
docker logs --tail 50 openjev-svc

# 重啟（權重在主機，不隨容器消失）
docker restart openjev-svc

# 切 GPU（需先釋放佔卡程序）
docker stop openjev-svc && docker rm openjev-svc
HUB=/home/hch/.cache/huggingface/hub
SHA=$(ls "$HUB"/models--AlexWortega--openjev/snapshots/ | head -n1)
docker run -d --name openjev-svc --gpus device=0 -p 8090:8090 \
  -v "$HUB":/hfhub:ro \
  -e OPENJEV_WEIGHTS="/hfhub/models--AlexWortega--openjev/snapshots/$SHA/qwen3.5-4b-nli" \
  -e OPENJEV_DEVICE=cuda \
  openjev:cuda

# GPU 加速（需先裝）
docker exec openjev-svc pip install causal-conv1d flash-linear-attention
```

## 建置（重裝時）

```bash
# 在 hch@10.145.119.19
cd /home/hch/openjev-server   # Dockerfile + requirements.txt + server.py + modeling_openjev.py
docker build -t openjev:cuda .
```

## 偏好/決策類問題（「明天吃什麼」型）

這類問題**沒有事實答案**，entailment 訊號≈0（實測：5 選全在 0.01–0.02、比隨機還平）。唯一有意義的用法是**約束排除法**：

1. **先向使用者收集約束**（口味、天氣、預算、不想重複、忌口）——沒有約束就不要硬跑。
2. 把每個選項寫成一句**具體陳述**塞進 hypothesis，premise 放約束：
   ```bash
   curl -fsS $B/predict -H 'Content-Type: application/json' -d '{
     "pairs": [
       {"premise": "It is hot and humid in Taipei. I want something light and cheap.",
        "hypothesis": "I will eat heavy hot pot for dinner."},
       {"premise": "It is hot and humid in Taipei. I want something light and cheap.",
        "hypothesis": "I will eat a cold noodle salad for dinner."}
     ]}'
   ```
3. **用 contradiction 分數當排除訊號**（分數高＝違反約束，砍掉），剩下的由使用者/agent 依偏好決定。**禁止把 entailment 分數當推薦分數。**
4. 若所有選項的 contradiction 都 <0.3（無訊號），直接回報「openjev 對此無訊號」並改由偏好判斷，不得偽裝成模型建議。

## 適用 / 不適用

| ✓ 適用 | ✗ 不適用 |
|---|---|
| MCQ 自動評分（有參考） | 開放域 QA / 生成式推理 |
| Claim verification（premise↔hypothesis） | 知識密集無參考 rerank |
| 遊戲狀態判定（zero-shot） | 長文生成 / summary |
| 內容一致性守護 | **偏好/決策（「吃什麼」型）——僅可走約束排除法，且常無訊號** |
| 約束排除（contradiction 分數） | 多輪對話 |
| 已縮小到 3–5 個重疊候選時的可選第二段打分（見 `skills-openjev-routing`，預設關閉） | **選 `D:\OB\skills` 的主路由**（永遠走 `obsidian-mcp-skill-router`，不要用 `/rerank` 掃 catalog） |
