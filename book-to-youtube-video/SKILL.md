---
name: book-to-youtube-video
description: 把一本書（PDF/OCR文字）改編成寫實微電影風格的 YouTube 知識導讀影片：男聲旁白 + 繁中字幕 + AI 生成的角色一致場景素材。用在「幫我把這本書做成影片」「5分鐘/10分鐘說書影片」這類需求時。含角色一致性解法（Flux img2img 定裝照 + first_frame 鎖定）、長片背景生成 SOP（跑在遠端不怕本機關機）、ffmpeg 合成流程。這是 ai-video-pipeline 技能的書籍改編應用分支，影片生成本身遵守 `minimax-h3-comfyui` 的唯一遠端部署規則。
---

# 書籍 → YouTube 說書影片 Pipeline

## 概念架構

```
書籍 PDF/OCR文字（可用 anytxt-searcher-ocr skill 取得）
    ↓ 人工/LLM 改寫（絕不逐字照抄，避免版權問題）
旁白稿 narration.md（分「## 旁白稿」區塊，純段落文字）
    ↓ edge-tts 生成男聲/女聲旁白 mp3，量出實際秒數
男聲旁白 voiceover.mp3 + 依實際秒數重算的 subtitles_auto.srt
    ↓ 設計 scenes.json（每 8-10 秒一個鏡頭，含 visual_prompt）
    ↓ 角色一致性前處理（見下方「角色一致性」章節，必做，否則人物會換臉）
角色定裝照 + 每場景專屬起始幀（Flux txt2img/img2img）
    ↓ MiniMax-H3 img2video，每場景用專屬起始幀當 first_frame
一批獨立 mp4 片段（無旁白、無字幕）
    ↓ ffmpeg concat + 燒錄字幕 + 混音旁白
最終 YouTube mp4（16:9, 1920x1080）
```

**核心教訓（已踩過的坑，直接抄結論）**：
1. **絕對不能用純文字生成（T2V）多段影片再拼在一起** —— 同一個角色每段會長得不一樣（換臉、換衣服、換場景），觀眾一眼看出「這不是同一個人」。必須用角色定裝照 + `first_frame` 鎖定。
2. **本機的檔案操作要小心**——文件實測發生過整個專案資料夾被移進資源回收桶的事故（原因不明，非本 pipeline 指令所為）。做刪除操作前先確認目標路徑精確，且優先把中間產物也上傳/存一份到遠端，降低單點風險。
3. **10 分鐘以上的量級一定要背景執行**，不要期待在一個對話回合內等到結果。每段 8-10 秒的 MiniMax-H3 影片在雙 4060 Ti 上約需 13-20 分鐘（含起始幀生成+容器重啟+渲染），32 段 = 5-10 小時，必須用 `nohup ... & disown` 掛在遠端主機，不依賴本機或對話存活。

---

## Step 0：取得書籍文字

優先用專案已有的 OCR 流程（見 `anytxt-searcher-ocr` skill 或直接呼叫 Anytxt Searcher 的 `ATRpcServer.Searcher.V1.OCR` JSON-RPC）：
1. `pymupdf` 把 PDF 逐頁轉 PNG（`page.get_pixmap(matrix=fitz.Matrix(2,2))`，提高解析度利於 OCR）
2. 逐頁呼叫 OCR API，存成 jsonl
3. OpenCC `s2twp` 簡轉繁，套用高信心錯字修正表（人名、專有名詞、標點符號）
4. 輸出結構化 Markdown（含 `<!-- page: 0001 -->` 註解方便回查原頁）

**改編原則（版權合規，必須遵守）**：絕不逐字照抄或大量原文朗讀。旁白稿必須是「用自己的話重新敘述書中案例/觀點」的原創改寫，可引用具體數據/案例細節，但敘事語言、段落結構要重寫。

---

## Step 1：旁白稿與 TTS

`narration.md` 格式：
```markdown
# 標題

## 旁白稿

第一段文字...

第二段文字...
```

用 `edge-tts`（`pip install --user edge-tts`）生成男聲：
```python
import asyncio, edge_tts
voice = "zh-TW-YunJheNeural"  # 台灣繁中男聲；女聲可用 zh-TW-HsiaoChenNeural
communicate = edge_tts.Communicate(text, voice=voice, rate="-2%", pitch="-2Hz")
await communicate.save("voiceover.mp3")
```
用 `ffprobe -show_entries format=duration` 量出實際秒數，這是後續所有時間軸計算的基準——**不要用估計值，字數估的秒數跟 TTS 實際輸出常差 10-20%**。

字幕：依「段落字數權重」比例分配時間（不是均分），公式：
```python
weight = len(chunk_text) + 8   # +8 是給短句一個時間下限，避免太快閃過
duration = total_audio_dur * weight / sum(all_weights)
```

---

## Step 2：場景設計 scenes.json

每個場景一個物件：
```json
{
  "id": "scene_03_daycare_waiting",
  "order": 3,
  "duration_sec": 8,
  "desc_zh": "中文場景描述",
  "voiceover": "這段對應的旁白文字",
  "visual_prompt": "英文寫實影片生成 prompt",
  "have": false,
  "file": null
}
```
- 每 8-10 秒一個鏡頭，5 分鐘影片約需 16 個場景，10 分鐘約需 30-35 個。
- `visual_prompt` 一定要包含：`photorealistic`, `documentary style`, `35mm film look`, `no text, no subtitles, no watermark`（避免模型自己生出讀不出來的假字）。
- 有真人角色出現的場景，`visual_prompt` 開頭要寫 `Same [角色描述], same face,`（配合下方角色一致性流程）。

---

## Step 3：角色一致性（必做，這是本技能最重要的部分）

MiniMax-H3 目前**沒有 IPAdapter/PuLID/FaceID 這類人臉鎖定節點**（先檢查 `curl http://<host>:<port>/object_info | grep -i 'IPAdapter\|PuLID\|FaceID'` 確認，未來可能裝上就不必这么麻烦）。在沒有這些節點的情況下，正確做法分兩層：

### 3a. 生成角色定裝照（一次性，每個角色一張）
用伺服器上的 Flux（`flux1-schnell-fp8.safetensors`，4 步快速取樣即可，`cfg=1.0`）生成正面照：
```python
prompt_flux_t2i = {
    "1": {"class_type": "CheckpointLoaderSimple", "inputs": {"ckpt_name": "flux1-schnell-fp8.safetensors"}},
    "2": {"class_type": "CLIPTextEncode", "inputs": {"text": character_desc, "clip": ["1", 1]}},
    "3": {"class_type": "CLIPTextEncode", "inputs": {"text": "", "clip": ["1", 1]}},
    "4": {"class_type": "EmptyLatentImage", "inputs": {"width": 1024, "height": 1024, "batch_size": 1}},
    "5": {"class_type": "KSampler", "inputs": {"model": ["1",0], "seed": seed, "steps": 4, "cfg": 1.0,
          "sampler_name": "euler", "scheduler": "simple", "positive": ["2",0], "negative": ["3",0],
          "latent_image": ["4",0], "denoise": 1.0}},
    "6": {"class_type": "VAEDecode", "inputs": {"samples": ["5",0], "vae": ["1",2]}},
    "7": {"class_type": "SaveImage", "inputs": {"images": ["6",0], "filename_prefix": f"charref_{name}"}}
}
```
角色描述要具體到：種族外貌、年齡、髮型、服裝、場景基調（例如「Asian female daycare teacher in her late 20s, warm cardigan, daycare classroom warm lighting」），這是後續每個場景 img2img 的「錨點」。

### 3b. 每個場景生成專屬起始幀（img2img，保留臉、換場景）
拿角色定裝照當底圖，用 `denoise≈0.5-0.6`（保留夠多原圖特徵但允許換背景/姿勢）：
```python
prompt_flux_img2img = {
    ... 同上，除了：
    "L": {"class_type": "LoadImage", "inputs": {"image": ref_image_filename}},  # 先用 /upload/image 上傳角色定裝照
    "V": {"class_type": "VAEEncode", "inputs": {"pixels": ["L",0], "vae": ["1",2]}},
    "5": {..., "latent_image": ["V",0], "denoise": 0.55}
}
```
prompt 文字要用 `"Same [角色], same face, [新場景/新動作/新表情描述]"` 開頭，明確要求模型保留臉。

**這一步不是完美方案**——Flux img2img 沒有真正的身份鎖定，只是「用同一張臉當強力參考」，跨場景仍可能有些微差異，但比純文字生成好非常多。若伺服器之後裝了 IPAdapter FaceID / PuLID，優先改用那個，一致性會更好。

### 3c. 每段影片用專屬起始幀當 first_frame
```python
prompt["104"]["inputs"]["first_frame"] = ["114", 0]  # MiniMaxH3ImageToVideo 節點
prompt["114"] = {"class_type": "LoadImage", "inputs": {"image": frame_filename}}  # 先 /upload/image 上傳
```
沒有角色的空鏡（城市街景、書桌 B-roll 等）可以不用 first_frame，直接純文字 T2V 生成即可，不影響一致性問題。

---

## Step 4：MiniMax-H3 影片生成（長片背景執行 SOP）

參數與踩坑細節完全遵守 `minimax-h3-comfyui` skill，這裡只補充「長片批次背景執行」的做法：

### 4a. 一定要在遠端主機上背景跑，不要在本機/對話裡等
```bash
# 上傳 pipeline 腳本 + 角色參考圖 + template_prompt.json 到遠端
scp master_pipeline.py character_refs/*.png template_prompt.json hch@<host>:/home/hch/<project>/

# 在遠端用 nohup + disown 背景啟動，SSH 斷線也不會被殺
ssh hch@<host> "cd /home/hch/<project> && nohup python3 master_pipeline.py > pipeline_stdout.log 2>&1 < /dev/null & disown"
```

### 4b. pipeline 腳本內建規則（見本 skill 目錄下 `scripts/master_pipeline_template.py`）
- **一次只跑一段**，每段跑完 `docker restart comfyui-h3` 清記憶體再送下一段（不要圖快省略這步，會累積 OOM 風險）。
- 進度寫入 `pipeline_log.txt`（人類可讀）+ `pipeline_state.json`（結構化，`{scene_id: "done"|"frame_failed"|"video_failed"}`），**腳本重跑時自動跳過已完成的場景**（讀 state.json + 檢查 clip 檔案是否存在），支援中斷後繼續。
- SERVER 位址在遠端主機上要用 `http://127.0.0.1:<容器映射的外部port>`（例如 comfyui-h3 是 8190），不是 8188（那是容器內部 port，host 上打不到）。

### 4c. 查進度（隨時可從任何連線查詢，不需要保留原本的 session）
```bash
ssh hch@<host> "ps aux | grep master_pipeline | grep -v grep"          # 確認還活著
ssh hch@<host> "tail -50 /home/hch/<project>/pipeline_log.txt"          # 看最新動態
ssh hch@<host> "cat /home/hch/<project>/pipeline_state.json"            # 看完成清單
```

### 4d. 時間估算
| 每段長度 | 單段耗時（含幀生成+重啟+渲染） | N 段總時間 |
|---|---|---|
| 8秒 (192 frame) | 約 13-16 分鐘 | N × 14 分鐘 |
| 10秒 (240 frame) | 約 15-20 分鐘 | N × 17 分鐘 |

5 分鐘影片（約16段）≈ 3.5-4.5 小時；10 分鐘影片（約32段）≈ 7.5-9.5 小時。**照實回報這個量級給使用者，不要淡化**，讓使用者決定是否真的要等這麼久或縮減場景數。

---

## Step 5：合成最終影片

```python
# 1. 每段素材 loop 填滿分配到的秒數，統一 scale/pad 成 1920x1080
ffmpeg -y -stream_loop -1 -i clip.mp4 -t {duration} \
  -vf "scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2,setsar=1" \
  -an -c:v libx264 -preset medium -crf 20 segment.mp4

# 2. concat 所有片段（先寫 concat_list.txt: file 'xxx.mp4' 一行一個）
ffmpeg -y -f concat -safe 0 -i concat_list.txt -c copy video_concat.mp4

# 3. 混入旁白 + 燒錄字幕
ffmpeg -y -i video_concat.mp4 -i voiceover.mp3 \
  -vf "subtitles='subtitles_auto.srt':fontsdir='C\:/Windows/Fonts':force_style='FontName=Microsoft JhengHei,FontSize=28,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,BorderStyle=1,Outline=2,Shadow=1,MarginV=70,Alignment=2'" \
  -c:v libx264 -preset medium -crf 20 -c:a aac -b:a 192k -shortest final.mp4
```

**Windows 路徑注意**：ffmpeg 的 `subtitles=` filter 對路徑裡的 `:` 要跳脫（`D:` → `D\:`），且 `fontsdir` 同理。

---

## 相關 Skill

- 通用「劇本→場景缺口分析→生成→Remotion 組裝」流程 → [`ai-video-pipeline`](../ai-video-pipeline)
- MiniMax-H3 本身的模型知識、OOM 除錯、DisTorch2 分層卸載與唯一 Docker Compose 雙 GPU 部署 → [`minimax-h3-comfyui`](../minimax-h3-comfyui)
- PDF/圖片 OCR 轉繁體 Markdown → 若有獨立 skill 則參照，否則見本文 Step 0 的 pymupdf + Anytxt Searcher 流程

## 已知限制 / TODO

- 角色一致性目前是 img2img 近似解法，非完美；伺服器若安裝 IPAdapter FaceID / PuLID / InstantID，應優先改用（一致性會顯著提升）。
- MiniMax-H3 單段影片上限約 15 秒，無法一次生成長鏡頭，只能多段剪接。
- 沒有裝 ComfyUI-Manager 自動裝 IPAdapter 的話，需要 `install_custom_node` 手動裝，重啟 ComfyUI 後才會生效。

---

## Conformance Addendum

## When to Use
把一本書（PDF/OCR文字）改編成寫實微電影風格的 YouTube 知識導讀影片：男聲旁白 + 繁中字幕 + AI 生成的角色一致場景素材。用在「幫我把這本書做成影片」「5分鐘/10分鐘說書影片」這類需求時。含角色一致性解法（Flux img2img 定裝照 + first_frame 鎖定）、長片背景生成 SOP（跑在遠端不怕本機關機）、ffmpeg 合成流程。這是 ai-video-pipeline 技能的書籍改編應用分支，影片生成本身遵守 `minimax-h3-comfyui` 的唯一遠端部署規則。

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
