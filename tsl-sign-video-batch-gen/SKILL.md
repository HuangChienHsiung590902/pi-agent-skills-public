---
name: tsl-sign-video-batch-gen
description: 端到端流程：從 TSL@CCU 台灣手語辭典查詢中文詞彙、下載官方示範影片、去浮水印、套用任意角色形象（真人照片或卡通貼圖），用 MiniMax-H3 Ref2VA 批次生成「該角色比出這個手語動作」的短片，存到 C:\Users\HCH\Videos。用在：使用者要一次生成多個手語詞彙的示範影片、或换角色形象重新生成手語動作時。
---

# TSL 手語詞彙 → 角色動作影片 批次生成 SOP

2026-08-05 首次跑通，一次生成6個常用語（再見/對不起/高興/生氣/愛/朋友，含中途換角色形象）
累積的完整經驗。這是把 `twtsl-ccu-dict`（查詞典）和 `minimax-h3-comfyui`（跑模型）兩個 skill
串起來的實戰 SOP，本 skill 只講**怎麼串、批次跑的順序、這次踩到的新坑**；查詢 API 細節見
`twtsl-ccu-dict`，ComfyUI/DisTorch2/OOM 除錯細節見 `minimax-h3-comfyui`。

## 核心限制：GPU 是序列運算，不能並行催單

雙 4060Ti 這台機器同時間**只能真正跑一個 MiniMax-H3 任務**（見 `minimax-h3-comfyui`）。批次生成
20個詞彙不會比生成1個快很多，每支 512×512×3秒（73幀）配置實測穩定落在 **11-13分鐘**。
**下指令前要先讓使用者知道總耗時量級**（N支 × 12分鐘），不要無聲開始一個要跑數小時的任務。

**正確的批次節奏**：
1. 送出第 N 支的 API prompt
2. 用 `sleep` + 輪詢 `/history/<prompt_id>` 等待 `status_str == "success"`
3. 完成後立刻 `scp` 下載到 `C:\Users\HCH\Videos`
4. 立刻送出第 N+1 支（不要等使用者確認，除非使用者主動打斷換方向）
5. 每支完成後跟使用者回報一次進度（第幾支/總共幾支、檔名），不要悶頭跑完全部才一次回報

## 標準工作流程

### 步驟1：一次查完所有目標詞彙，篩選可用的

```bash
# 逐一 manualSearch 找 id，挑 _A（較常用打法）或無後綴的唯一詞條
curl "https://twtsl.ccu.edu.tw/api/manualSearch?name=<詞彙>&lang=zh&page=1&pageSize=5"
```

**常見陷阱**：很多常用語查不到或名稱跟你預期不同：
- 「你好」查不到（辭典裡沒有這個詞條，只有「給你好看」）
- 「謝謝」的辭典正式詞條名稱其實是「感謝_A」/「感謝_B」
- 「早安」「晚安」「沒問題」「了解」目前查無資料
- 查「愛」會混進「愛滋病」「戀愛」等關聯詞，需要挑精確符合 `name` 就是那兩個字的項目

**建議策略**：先準備比目標數量多50%的候選清單去查，再從查到的裡面選最終要跑的幾個，避免
現場才發現詞彙查無資料要重新想。

### 步驟2：批次查詳細資料（clip路徑+動作描述）

```bash
curl "https://twtsl.ccu.edu.tw/api/querySearch?id=<id>&lang=zh"
```
記下 `clip`（影片路徑）和 `description`（中文動作描述，直接當 prompt 素材）。

### 步驟3：批次下載影片、去浮水印

```bash
curl -o <slug>.mp4 "https://twtsl.ccu.edu.tw/<clip路徑>.mp4"
```

**浮水印座標每支影片不一定相同，不能只套一組座標**（這次6支裡有5支跟先前「吃飯」的座標不同）。
正確做法：

```bash
# 1. 抽一張中段畫格，人工核對浮水印範圍
ffmpeg -y -i <slug>.mp4 -vf "select='eq(n,10)'" -vframes 1 check.png
# 2. 用 PIL 掃描亮度找精確白色像素範圍（比人眼目測快且準）
python -c "
from PIL import Image
im = Image.open('check.png').convert('L')
px = im.load()
for y in range(400,480,5):
    row=''.join('W' if px[x,y]>200 else '.' for x in range(150,500,10))
    print(y,row)
"
```

**delogo 濾鏡有時去不乾淨、留下模糊殘影**（這次遇到，浮水印區域比原本抓的範圍更寬/更靠下，
`delogo` 邊界沒完全蓋住就會留斑駁殘影，比不去浮水印還顯眼）。**更可靠的替代方案：直接
`crop` 掉浮水印所在的整個底部窄條**，只要浮水印位置固定在畫面下緣、且不會切到手部動作範圍：

```bash
ffmpeg -y -i <slug>.mp4 -vf "crop=640:410:0:0" -c:v libx264 -crf 18 -c:a copy <slug>_clean.mp4
```
（原始畫面通常是 640×480，這裡裁成 640×410，砍掉下面 70px 含浮水印的區域。**裁切前務必先確認
手部動作沒有伸到畫面最下緣**，不然會把手切掉——這批全部是「身前/胸口高度」的手勢，裁下面
70px 是安全的，但換其他詞彙要重新核對。）

裁完務必再抽幀肉眼確認乾淨，不要假設一次做對：
```bash
ffmpeg -y -i <slug>_clean.mp4 -vf "select='eq(n,10)'" -vframes 1 verify.png
```

### 步驟4：上傳素材到 ComfyUI 伺服器

```bash
scp <slug>_clean.mp4 hch@10.145.119.19:/home/hch/ComfyUI/input/<slug>_sign.mp4
```
若要換角色形象，人物圖也要上傳；若圖片解析度太小（例如截圖裁出來只有 162×162），
**先用 PIL LANCZOS 放大到 512×512 再上傳**，太小的參考圖容易讓生成結果細節崩壞：
```python
from PIL import Image
Image.open('small.png').convert('RGB').resize((512,512), Image.LANCZOS).save('ref_512.png')
```

### 步驟5：組 R2V prompt（六段式格式）批次生成

沿用 `minimax-h3-comfyui` skill 裡驗證過的展平 API prompt 範本，**用 Python 模板函式產生每個
詞彙的 prompt**，不要每支手動重寫，容易漏欄位：

```python
import json

def build_prompt(name_zh, video_file, action_en, seed):
    return f"""<Subject 1> is the [角色描述，固定不變].
<Video 1> is the reference sign-language demonstration video, providing the exact hand motion
for the Taiwanese Sign Language sign meaning "{name_zh}" ({action_en}).

summary:
[reference generation] The target video shows <Subject 1> performing the complete Taiwanese
Sign Language gesture for "{name_zh}" exactly as demonstrated by the signer in <Video 1>, then
returning both hands to a relaxed neutral pose, in a plain, softly lit studio setting with a
static camera and no other subjects.

retention_analysis:
<Subject 1> (appears in [Shot 1]): fully_preserved - ...
<Video 1> (hand and arm motion for the sign "{name_zh}"): attribute_transfer - ...

detailed_description:
The target video is in a clean, realistic studio portrait style, static camera, plain softly
lit neutral background, medium shot framing the subject from the waist up, no camera movement.
[Shot 1] <Subject 1>, [外觀描述], stands centered in frame facing the camera with a calm,
neutral expression, both hands resting naturally at her sides. Following the exact hand motion
from <Video 1>, she performs the sign-language gesture: {action_en}. She repeats the motion
clearly at a calm, legible pace. After completing the gesture, she smoothly lowers both hands
back down, arms returning to a relaxed natural position resting at her sides, ending on a calm
neutral standing pose with both hands fully down and still, matching her starting pose.

overall_soundscape:
Quiet room tone with no distinct background noise.

non_diegetic_music:
N/A"""
```

`action_en` 直接把 `querySearch` 的中文 `description` 翻成英文動作描述即可，例如：
- 「雙手食指伸直相對，拉開成食指彎曲」→ `both hands with index fingers extended facing
  each other, then pulling apart while the index fingers curl/bend`
- 「一手掌心接觸另一手伸直的拇指指背上，並重複繞圈」→ `one hand's palm touches the back of
  the other hand's extended thumb, then makes a repeated circular motion`

### 步驟6：固定配置參數（省時版）

這次全部用：`width=512, height=512, length=73`（~3秒），UNet/CLIP `expert_mode_allocations`
用 `cuda:0,3gb;cuda:1,3gb;cpu,*`。**這個配置目前是「動作完整（有收尾）+ 尚可接受耗時」的
甜蜜點**，比 `minimax-h3-comfyui` 記載的 1.6秒(39幀)版本更好——39幀太短，動作會像是硬生生
被截斷、看不到收尾（使用者原話：「感覺很像被截掉了的樣子，至少要讓手放下來吧」）。

**73幀（~3秒）的動作結構要點**：prompt 裡明確分三段寫：① 起手勢（手放鬆下垂）② 核心手語
動作（重複2-3次讓動作清楚）③ 收尾（手放下回到跟起手勢一致的姿勢）。不寫收尾段，模型有機率
在動作做到一半時就把幀數用完，畫面會卡在不自然的中間姿勢。

### 步驟7：不同角色形象的 prompt 差異

**真人寫實角色**（例如「黑色露肩洋裝長髮女孩」）：`detailed_description` 開頭寫
`clean, realistic studio portrait style`，角色描述用具體的服裝/髮型/飾品細節。

**卡通/貼圖角色**：**必須**在 prompt 最前面加畫風約束，否則容易被生成得偏寫實、跑掉原本的
扁平卡通感（`minimax-h3-comfyui` skill 已有記載，這次批次生成再次驗證有效）：
```
Cartoon illustration animation, flat cel-shaded 2D sticker style, thick bold black outlines,
simple plain white background, no camera movement, character stays centered in frame, no
camera movement, no added shading or gradients beyond the original flat colors.
```
角色外觀描述也要盡量具體到「線條粗細、色塊範圍、五官形狀」這種向量圖特徵，而不是只說
「一個笑臉」，例如：`round white flat cartoon face with thick black bold outline, two black
slanted crescent-shaped closed happy eyes, ...`。這次卡通角色（笑臉貼圖）成功生成後畫風
完全沒跑掉，開場/收尾姿勢也對齊了角色原本的手部姿勢（角色本來就是雙手交握狀，正好貼合
「朋友」手語動作的雙手相握輕搖）。

## 這次批次生成實測記錄（2026-08-05，6支）

| 詞彙 | id | clip路徑 | 角色 | 配置 | 耗時 |
|---|---|---|---|---|---|
| 再見 | 902 | video/g/good_bye | 黑洋裝女孩 | 512×512×73幀 | 11分37秒 |
| 對不起 | 2001 | video/s/sorry | 黑洋裝女孩 | 512×512×73幀 | ~12分 |
| 高興 | 958 | video/h/happy | 黑洋裝女孩 | 512×512×73幀 | ~12分 |
| 生氣 | 76 | video/a/angry_a | 黑洋裝女孩 | 512×512×73幀 | ~12分 |
| 愛 | 1240 | video/l/love | 黑洋裝女孩 | 512×512×73幀 | ~12分 |
| 朋友 | 849 | video/f/friend | 笑臉貼圖角色（換形象）| 512×512×73幀 | ~13分 |

另外先前已做過：吃飯（id 649，2個版本：1.6秒版動作被截斷、3秒版動作完整）、感謝/謝謝
（id 2196）。

全部產出檔案在 `C:\Users\HCH\Videos\sign_<slug>.mp4`（依 memory
`feedback_video_output_location.md` 的硬性規則）。

## 相關 skill

- `twtsl-ccu-dict` — TSL@CCU 辭典 API 細節（`manualSearch`/`querySearch`/`handSearch` 等）
- `minimax-h3-comfyui` — ComfyUI 伺服器環境、DisTorch2 分卡設定、OOM 除錯、R2V API 展平範本

---

## Conformance Addendum

## When to Use
端到端流程：從 TSL@CCU 台灣手語辭典查詢中文詞彙、下載官方示範影片、去浮水印、套用任意角色形象（真人照片或卡通貼圖），用 MiniMax-H3 Ref2VA 批次生成「該角色比出這個手語動作」的短片，存到 C:\Users\HCH\Videos。用在：使用者要一次生成多個手語詞彙的示範影片、或换角色形象重新生成手語動作時。

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
