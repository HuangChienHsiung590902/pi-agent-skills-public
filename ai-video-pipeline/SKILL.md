---
name: ai-video-pipeline
description: 從劇本文字出發，用 Claude 分析場景缺口 → MiniMax-H3（ComfyUI）補生成缺少片段 → Remotion render 最終影片的完整 pipeline。用在：有一份故事劇本，想讓 AI 自動決定要生哪些片段、補齊缺口、再組裝成有字幕/轉場的完整影片時。伺服器固定為 10.145.119.19，ComfyUI port 8190，影片生成規則遵守 minimax-h3-comfyui skill。
---

# AI 影片 Pipeline：劇本 → 生成 → 剪輯

## 概念架構

```
你寫劇本（自然語言）
    ↓  Claude 分析
scenes.json（每個場景：描述、需要的動作、現有檔案或 null）
    ↓  對比 output-h3 現有 mp4
缺口清單（have: false 的場景）
    ↓  自動組 ComfyUI prompt，依序送伺服器生成（一次一段）
全部素材齊全
    ↓  生成 Remotion composition.tsx + scenes.json
Remotion render → 最終影片.mp4
```

---

## Step 1：劇本分析 → scenes.json

### 輸入格式（自然語言即可）
```
開場：角色走進舞台，對觀眾鞠躬
高潮：連續兩個後空翻，定格亮相
結尾：揮手道別，走出畫面
```

### 輸出格式 scenes.json
```json
[
  {
    "id": "scene_01",
    "order": 0,
    "desc_zh": "走進舞台，鞠躬",
    "comfyui_action": "walk into frame confidently, bow to camera, smile, full body visible",
    "duration_sec": 5,
    "have": false,
    "file": null
  },
  {
    "id": "scene_02",
    "order": 1,
    "desc_zh": "後空翻",
    "comfyui_action": "perform one clean stylized backward somersault in place, land safely facing camera",
    "duration_sec": 5,
    "have": true,
    "file": "/home/hch/ComfyUI/output-h3/video/MiniMax_H3/B_1000018219_07_07_backflip_00001_.mp4"
  }
]
```

### 分析規則
1. 每個明顯動作段落切成一個 scene（原則上每段 5 秒）。
2. 對照現有檔案清單（`ls /home/hch/ComfyUI/output-h3/video/MiniMax_H3/B_1000018219_*.mp4`），用檔名關鍵字比對 `comfyui_action`，匹配則 `have: true`。
3. 無法明確匹配的設 `have: false`，需補生成。

---

## Step 2：補生成缺口場景

### 規則（繼承自 minimax-h3-comfyui skill）
- **一次只送一段**，跑完確認輸出檔存在、RAM 恢復後再送下一段。
- 每段前 `docker restart comfyui-h3` 清記憶體。
- RAM available < 8GB 或 swap > 4GB 就停止，等恢復。
- 生成用的固定參數（與現有 B 段保持一致）：

```python
clip_alloc = 'cuda:0,2gb;cuda:1,5gb;cpu,*'
unet_alloc = 'cuda:0,3gb;cuda:1,9gb;cpu,*'
model      = 'minimax_h3_fl2va_int8_convrot.safetensors'
lora       = 'minimax_h3_turbo_v4_step600_comfyui_T8-convert.safetensors'
first_frame = '1000018219.png'   # 固定用原圖保持人物一致性
width, height, length = 384, 512, 124   # 5 秒 @ 24fps
steps = 4
```

### base_prompt 模板
```
Use the input image as the fixed identity and scene reference.
Preserve the same face, hairstyle, outfit, body shape, proportions,
background, camera angle, lighting, and visual style.
Single centered full-body character. No scene change, no background change,
no extra people, no extra limbs, no distorted face, no identity change.
Keep face front-facing and clearly visible most of the time.
Static camera. Audio: upbeat dance beat with rhythmic claps.
Segment action: {comfyui_action}. Keep the first-frame person and background
consistent for this entire 5-second clip.
```

### 輸出檔命名規則
```
B_1000018219_{order:02d}_{scene_id}_{suffix}.mp4
```
例：`B_1000018219_00_scene_01_walk_bow_00001_.mp4`

---

## Step 3：Remotion 專案結構

### 目錄
```
/home/hch/remotion-project/       # 伺服器
  package.json
  remotion.config.ts
  src/
    Root.tsx
    Composition.tsx
    scenes.json                   # Step 1 產出，有 file 路徑
    components/
      Subtitle.tsx
      FadeTransition.tsx
```

本機也可以跑（需 Node.js 18+）：
```
C:\Users\HCH\remotion-project\
```

### 安裝
```bash
mkdir -p /home/hch/remotion-project && cd /home/hch/remotion-project
npm init -y
npm install remotion @remotion/cli @remotion/media-utils
```

### Composition.tsx 模板
```tsx
import { Composition } from 'remotion';
import { MainVideo } from './MainVideo';
import scenes from './scenes.json';

export const RemotionRoot = () => (
  <Composition
    id="DanceVideo"
    component={MainVideo}
    durationInFrames={scenes.reduce((s, sc) => s + sc.duration_sec * 24, 0)}
    fps={24}
    width={384}
    height={512}
    defaultProps={{ scenes }}
  />
);
```

### MainVideo.tsx 模板
```tsx
import { Sequence, Video, useCurrentFrame, interpolate } from 'remotion';

export const MainVideo = ({ scenes }) => {
  let offset = 0;
  return (
    <>
      {scenes.map((sc, i) => {
        const from = offset;
        const dur  = sc.duration_sec * 24;
        offset += dur;
        return (
          <Sequence key={sc.id} from={from} durationInFrames={dur}>
            {/* 影片 */}
            <Video src={sc.file} />

            {/* 淡入轉場（前 12 幀）*/}
            <FadeIn durationFrames={12} />

            {/* 字幕 */}
            <Subtitle text={sc.desc_zh} />
          </Sequence>
        );
      })}
    </>
  );
};
```

### FadeIn 元件
```tsx
import { useCurrentFrame, interpolate, AbsoluteFill } from 'remotion';

export const FadeIn = ({ durationFrames = 12 }) => {
  const frame = useCurrentFrame();
  const opacity = interpolate(frame, [0, durationFrames], [1, 0], {
    extrapolateLeft: 'clamp', extrapolateRight: 'clamp'
  });
  return (
    <AbsoluteFill style={{ backgroundColor: 'black', opacity }} />
  );
};
```

### Subtitle 元件
```tsx
import { useCurrentFrame } from 'remotion';

export const Subtitle = ({ text }) => (
  <div style={{
    position: 'absolute', bottom: 40, width: '100%',
    textAlign: 'center', color: 'white',
    fontSize: 28, fontWeight: 'bold',
    textShadow: '0 2px 8px rgba(0,0,0,0.8)',
    fontFamily: 'Noto Sans TC, sans-serif'
  }}>
    {text}
  </div>
);
```

### render 指令
```bash
cd /home/hch/remotion-project
npx remotion render DanceVideo out/final.mp4 \
  --codec=h264 --crf=18 --fps-override=24

# 或只 render 某個時間段（除錯用）
npx remotion render DanceVideo out/preview.mp4 \
  --frames=0-120
```

---

## Step 4：完整執行腳本（pipeline.py）

放在 `/home/hch/ai_video_pipeline/pipeline.py`，接受 `scenes.json` 輸入：

```python
#!/usr/bin/env python3
"""
usage: python3 pipeline.py scenes.json
  - 找出 have:false 的 scene
  - 依序生成（一段一段，每段前重啟容器）
  - 生成完更新 scenes.json 的 have/file
  - 最後觸發 Remotion render
"""
import json, subprocess, time, pathlib, sys, urllib.request

COMFY_BASE   = 'http://localhost:8190'
OUT_DIR      = pathlib.Path('/home/hch/ComfyUI/output-h3/video/MiniMax_H3')
REMOTION_DIR = pathlib.Path('/home/hch/remotion-project')
SCENES_FILE  = pathlib.Path(sys.argv[1])
scenes       = json.loads(SCENES_FILE.read_text())

# ── 生成函式 ──────────────────────────────────────────────────────────────

clip = 'cuda:0,2gb;cuda:1,5gb;cpu,*'
unet = 'cuda:0,3gb;cuda:1,9gb;cpu,*'
BASE_PROMPT = (
  'Use the input image as the fixed identity and scene reference. '
  'Preserve the same face, hairstyle, outfit, body shape, proportions, '
  'background, camera angle, lighting, and visual style. '
  'Single centered full-body character. No scene change, no background change, '
  'no extra people, no extra limbs, no distorted face, no identity change. '
  'Keep face front-facing and clearly visible most of the time. '
  'Static camera. Audio: upbeat dance beat with rhythmic claps. '
  'Segment action: {action}. Keep the first-frame person and background '
  'consistent for this entire 5-second clip.'
)

def http(path, data=None, timeout=30):
    req = urllib.request.Request(
        COMFY_BASE + path, data=data,
        headers={'Content-Type': 'application/json'} if data else {}
    )
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return r.read().decode()

def wait_ready(retries=90):
    for _ in range(retries):
        try:
            http('/system_stats', timeout=3)
            return True
        except Exception:
            time.sleep(2)
    return False

def check_ram_ok():
    with open('/proc/meminfo') as f:
        lines = {k: int(v.split()[0]) for k, v in
                 (l.strip().split(':', 1) for l in f if ':' in l)}
    avail_gb = lines.get('MemAvailable', 0) / 1024 / 1024
    swap_used_gb = (lines.get('SwapTotal', 0) - lines.get('SwapFree', 0)) / 1024 / 1024
    return avail_gb >= 8 and swap_used_gb <= 4

def make_prompt(scene):
    prefix = f'video/MiniMax_H3/B_1000018219_{scene["order"]:02d}_{scene["id"]}'
    prompt = BASE_PROMPT.format(action=scene['comfyui_action'])
    idx = scene['order']
    api = {
        '1':  {'class_type': 'LoadImage', 'inputs': {'image': '1000018219.png'}},
        '2':  {'class_type': 'VAELoaderDisTorch2MultiGPU', 'inputs': {'vae_name': 'minimax_h3_video_vae_fp16.safetensors', 'compute_device': 'cuda:0', 'virtual_vram_gb': 0, 'donor_device': 'cpu', 'expert_mode_allocations': '', 'eject_models': True}},
        '3':  {'class_type': 'VAELoaderDisTorch2MultiGPU', 'inputs': {'vae_name': 'minimax_h3_audio_vae_fp32.safetensors', 'compute_device': 'cuda:0', 'virtual_vram_gb': 0, 'donor_device': 'cpu', 'expert_mode_allocations': '', 'eject_models': True}},
        '4':  {'class_type': 'UNETLoaderDisTorch2MultiGPU', 'inputs': {'unet_name': 'minimax_h3_fl2va_int8_convrot.safetensors', 'weight_dtype': 'default', 'compute_device': 'cuda:0', 'virtual_vram_gb': 4, 'donor_device': 'cuda:1', 'expert_mode_allocations': unet, 'eject_models': True}},
        '5':  {'class_type': 'LoraLoaderBypassModelOnly', 'inputs': {'model': ['4', 0], 'lora_name': 'minimax_h3_turbo_v4_step600_comfyui_T8-convert.safetensors', 'strength_model': 1}},
        '6':  {'class_type': 'CLIPLoaderDisTorch2MultiGPU', 'inputs': {'clip_name': 'qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors', 'type': 'minimax', 'device': 'cuda:1', 'virtual_vram_gb': 4, 'donor_device': 'cuda:0', 'expert_mode_allocations': clip, 'eject_models': True}},
        '7':  {'class_type': 'MiniMaxH3AudioConditioningT8', 'inputs': {'clip': ['6', 0], 'video_vae': ['2', 0], 'audio_vae': ['3', 0], 'prompt': prompt, 'width': 384, 'height': 512, 'length': 124, 'task_type': 'I2VA', 'audio_mode': 'native', 'audio_denoise_strength': 0.35, 'add_source_as_reference': True, 'prompt_primary_audio_ordinal': 0, 'strict_prompt_tags': True, 'ref_image_size': 'match', 'reference_video_policy': 'official_2_to_15s', 'first_frame': ['1', 0]}},
        '8':  {'class_type': 'MiniMaxH3DualClockSamplerT8', 'inputs': {'model': ['5', 0], 'av_latent': ['7', 1], 'steps': 4, 'shift_video': 12, 'shift_audio': 3, 'sampler_name': 'dual_clock_euler', 'scheduler': 'native_flow'}},
        '9':  {'class_type': 'RandomNoise', 'inputs': {'noise_seed': 82190100 + idx}},
        '10': {'class_type': 'BasicGuider', 'inputs': {'model': ['8', 0], 'conditioning': ['7', 0]}},
        '11': {'class_type': 'SamplerCustomAdvanced', 'inputs': {'noise': ['9', 0], 'guider': ['10', 0], 'sampler': ['8', 1], 'sigmas': ['8', 2], 'latent_image': ['7', 1]}},
        '12': {'class_type': 'MiniMaxH3AVDecodeT8', 'inputs': {'av_latent': ['11', 0], 'video_vae': ['2', 0], 'audio_vae': ['3', 0]}},
        '13': {'class_type': 'CreateVideo', 'inputs': {'images': ['12', 0], 'fps': 24, 'audio': ['12', 1], 'bit_depth': 8}},
        '14': {'class_type': 'SaveVideo', 'inputs': {'video': ['13', 0], 'filename_prefix': prefix, 'format': 'mp4', 'codec': 'auto'}},
    }
    return {'prompt': api, 'client_id': f'pipeline-{scene["id"]}'}, pathlib.Path(prefix).name

def generate_scene(scene):
    print(f'\n=== generate {scene["id"]} ===')
    if not check_ram_ok():
        print('RAM 壓力過高，停止')
        sys.exit(1)
    subprocess.run(['docker', 'restart', 'comfyui-h3'], check=True)
    time.sleep(5)
    if not wait_ready():
        print('ComfyUI 啟動超時')
        sys.exit(1)
    payload, prefix = make_prompt(scene)
    http('/prompt', json.dumps(payload).encode())
    print(f'submitted {scene["id"]}')
    t0 = time.time()
    while time.time() - t0 < 3600:
        q = json.loads(http('/queue', timeout=10))
        matches = sorted(OUT_DIR.glob(prefix + '*.mp4'))
        if matches and not q.get('queue_running') and not q.get('queue_pending'):
            f = matches[-1]
            print(f'DONE {f} ({f.stat().st_size} bytes)')
            return str(f)
        time.sleep(30)
    raise TimeoutError(f'scene {scene["id"]} timeout')

# ── 主流程 ────────────────────────────────────────────────────────────────

for sc in scenes:
    if sc.get('have'):
        print(f'skip {sc["id"]} (already have)')
        continue
    f = generate_scene(sc)
    sc['have'] = True
    sc['file'] = f
    SCENES_FILE.write_text(json.dumps(scenes, ensure_ascii=False, indent=2))
    print(f'scenes.json updated')

print('\n=== 所有場景齊全，觸發 Remotion render ===')
# 先把 scenes.json 複製進 remotion 專案
import shutil
shutil.copy(SCENES_FILE, REMOTION_DIR / 'src' / 'scenes.json')
result = subprocess.run(
    ['npx', 'remotion', 'render', 'DanceVideo', 'out/final.mp4',
     '--codec=h264', '--crf=18'],
    cwd=str(REMOTION_DIR), text=True,
    stdout=subprocess.PIPE, stderr=subprocess.STDOUT
)
print(result.stdout)
print('render return code:', result.returncode)
```

---

## 使用方式（整個 pipeline 一鍵）

```bash
# 1. 在本機寫好劇本，讓 Claude 幫你分析出 scenes.json
# 2. 把 scenes.json 傳上伺服器
scp scenes.json hch@10.145.119.19:/home/hch/ai_video_pipeline/

# 3. 背景執行整個 pipeline
ssh hch@10.145.119.19 \
  'nohup python3 /home/hch/ai_video_pipeline/pipeline.py \
   /home/hch/ai_video_pipeline/scenes.json \
   > /home/hch/ai_video_pipeline/pipeline.log 2>&1 &
   echo PID=$!'

# 4. 查看進度
ssh hch@10.145.119.19 'tail -f /home/hch/ai_video_pipeline/pipeline.log'
```

---

## 劇本分析 prompt（讓 Claude 幫你生成 scenes.json）

當使用者給劇本時，用以下規則分析並直接輸出 scenes.json（不要問問題，直接給 JSON）：

1. 每個明顯動作/情境切一個 scene，id 用 `scene_{order:02d}`。
2. `comfyui_action` 要英文，具體描述動作（人物、動作、方向、表情），20-40 字。
3. `duration_sec` 預設 5，特別短的動作可設 3。
4. 對照 `現有檔案清單` 比對（呼叫 `ssh hch@10.145.119.19 "ls /home/hch/ComfyUI/output-h3/video/MiniMax_H3/B_1000018219_*.mp4"` 取得），能匹配的設 `have: true` 並填 `file`。
5. 輸出純 JSON，不要加 markdown code block。

---

## 注意事項

- **人物一致性**：所有場景都用同一張 `1000018219.png` 作 first_frame，動作越複雜臉的漂移越明顯，這是模型限制。
- **動作銜接**：兩段之間的動作靠 Remotion 的 FadeIn（12 幀黑色淡入）稍作掩蓋，但不能保證物理上連貫。
- **Remotion 需要 Node.js 18+**：伺服器若沒裝，先 `nvm install 18 && nvm use 18`。
- **render 在伺服器執行**：影片檔路徑要是伺服器本機絕對路徑，不能用 Windows 路徑。
- **缺口補生成時間**：每段約 8-10 分鐘，背景執行用 `nohup`，不要在互動時段等待。
- **ffmpeg concat DTS 警告**：用 `-c copy` 串接 MiniMax-H3 輸出時，段落交接處音訊會出現 `Non-monotonic DTS` 警告，播放不受影響；要修乾淨需用 Remotion 重新 render（重新編碼）。
- **concat 不能用 glob `*`**：要明確列出每段的 `_00001_` 檔案路徑，避免同一段的重試檔案（`_00002_`、`_00003_`）混入。

---

## 2026-08-10 實測紀錄（1000018219 舞蹈影片）

- 角色：`1000018219.png`
- 12 段 × 5 秒，模型：FL2VA INT8 + T8 Turbo，384×512，4 steps
- 每段約 8–10 分鐘，12 段合計約 2 小時（含每段重啟容器）
- 全部用 `nohup` 背景跑，使用者關機後伺服器自行完成
- concat 結果：`B_1000018219_dance_12x5s_concat.mp4`，9.4MB，62 秒
- 已拉到本機：`C:\Users\HCH\Videos\B_1000018219_dance_12x5s_concat.mp4`
- 執行腳本：`/home/hch/h3_b_segments/run_one_b_segment.py`（單段）、`/home/hch/h3_b_segments/remaining.sh`（背景多段）

---

## Conformance Addendum

## When to Use
從劇本文字出發，用 Claude 分析場景缺口 → MiniMax-H3（ComfyUI）補生成缺少片段 → Remotion render 最終影片的完整 pipeline。用在：有一份故事劇本，想讓 AI 自動決定要生哪些片段、補齊缺口、再組裝成有字幕/轉場的完整影片時。伺服器固定為 10.145.119.19，ComfyUI port 8190，影片生成規則遵守 minimax-h3-comfyui skill。

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
