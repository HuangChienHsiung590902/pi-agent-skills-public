---
name: hunyuan-chibi-safe-animation
description: >
  把遠端 10.145.119.19 上的 Hunyuan3D 薄殼角色 mesh 做成會動、又不撕殼的 GLB。
  當使用者要讓混元角色動、揮手、轉身、下跪、套 kimodo 動作、SkinTokens 自動綁骨後畫面崩掉、
  或要重做 character_hunyuan_waving.glb 時使用。
  不要用於 SAM 3D Body / Rigify / Unreal（用 sam3d-body-rigify-unreal-pipeline），
  也不要用來產生 kimodo 骨架動畫本身（用 kimodo-cpp-text-to-motion）。
---

# Hunyuan 薄殼角色安全動畫

## When to Use

- 使用者要把 Hunyuan3D 產出的角色做成「會動的 3D」。
- 先前 SkinTokens 自動綁骨或 kimodo 下跪 retarget 把身體絞爛，要修或重做。
- 要重跑 `/home/hch/SkinTokens/io/character_hunyuan_waving.glb`。
- 要判斷「自動綁骨 + 動作 retarget」能不能用在這套 mesh。

不要用本 skill：

- 只要靜態 Hunyuan mesh / Paint：`comfyui-blender-character-model`
- 只要 kimodo 文字轉骨架動作（沒有套到混元角色）：`kimodo-cpp-text-to-motion`
- SAM 3D Body → Rigify → Unreal：`sam3d-body-rigify-unreal-pipeline`
- 遠端 Blender GUI / noVNC：`blender-unreal-linux-docker`

## 已驗證事實（2026-09-22）

遠端：`hch@10.145.119.19`（gigabyte，雙 RTX 4060 Ti 16GB）。

| 項目 | 路徑 / 值 |
|---|---|
| 原始角色 | `/home/hch/character_hunyuan_painted_no_bag_v3.glb`（約 9.6 MB，兩層重複殼） |
| 會動、外型正確 | `/home/hch/SkinTokens/io/character_hunyuan_waving.glb` |
| 外型正確、靜止 | `/home/hch/SkinTokens/io/character_hunyuan_fixed.glb` |
| **不要開** | `/home/hch/SkinTokens/io/character_hunyuan_kneel.glb`（下跪 retarget，身體被絞爛） |
| 腳本 | `/home/hch/SkinTokens/io/make_sway.py`（與本 skill `scripts/make_sway.py` 同步） |
| 預覽腳本 | 本 skill `scripts/render_preview.py` |
| 匯出 / 渲染 | Docker image `blender:5.2.1`，`docker run --rm --entrypoint blender` |
| SkinTokens 映像 | `skintokens:bpy`（自動綁骨用，**這套 chibi 不要拿來套動作**） |

原始 mesh 特徵：

- 兩塊幾乎相同的 mesh（各約 16 萬頂點）。匯出前只留一塊。
- 薄殼、頭髮與包包帶是獨立薄片。局部骨骼一轉，頭髮／袖口會扇成卡片。
- 靜止包圍盒大約 `1.00 × 1.02 × 1.97`，Z 約 `-0.98 … 0.99`。

## 不能做的事（已實測失敗）

1. **SkinTokens / TokenRig 自動綁骨再套 kimodo「有人下跪」**
   - SkinTokens 預測的 22 骨全擠在身體下半（頭骨約 `z=-0.115`，mesh 頭頂約 `z=0.99`）。
   - kimodo GLB 是 30 個 EMPTY（Hips / Spine / LeftArm…），不是 Armature。
   - 依骨頭位置做 swing retarget 後，靜止包圍盒看起來正常，一播就絞爛。使用者截圖就是這份 `character_hunyuan_kneel.glb`。
2. **把薄殼頂點分給手臂骨再轉肩／肘**
   - 連通分量顯示右側 `x>0.22` 碎成 50+ 片。權重常抓到頭髮或背帶，轉起來像紙片。
   - 偵測「手臂有沒有抬」要用該 cluster 的頂點位移，不要用全身包圍盒；包圍盒幾乎不變不代表沒變形。
3. **宣稱 kimodo 動作能直接套到混元角色**
   - kimodo 是 SOMA 30 關節；SkinTokens 骨頭叫 `bone_0`…。兩邊骨架不同名、也對不齊身材。

## Procedure

### 1. 確認來源 mesh

```bash
ssh hch@10.145.119.19 'ls -lh /home/hch/character_hunyuan_painted_no_bag_v3.glb'
```

不要用 `character_hunyuan_kneel.glb` 當輸入。那份已經壞了。

### 2. 重做會動的 GLB

在遠端用官方 Blender 5.2.1 映像跑腳本（`skintokens:bpy` 缺 EGL，渲染會 segfault；匯出用 blender 映像較穩）：

```bash
scp scripts/make_sway.py hch@10.145.119.19:/tmp/make_sway.py
ssh hch@10.145.119.19 'docker run --rm --entrypoint blender -v /home/hch:/home/hch -v /tmp/make_sway.py:/tmp/make_sway.py blender:5.2.1 -b -P /tmp/make_sway.py'
```

腳本做的事：

1. 只留一塊 mesh。
2. 全頂點掛在一根 `root` 骨（權重 1.0）。
3. 整隻角色做左右轉身 + 上下輕晃（第 1–48 幀，24 fps，約兩秒一輪）。
4. 可選 shape key `Wave`：只移動鏡頭看得到的那隻袖／手（近側、負 Y），不要動頭髮。
5. 匯出 `/home/hch/SkinTokens/io/character_hunyuan_waving.glb`。

目前已驗證的關鍵幀（`make_sway.py`）：

| 幀 | 轉身 | 上移 | Wave |
|---|---|---|---|
| 1 | 0° | 0 | 0 |
| 12 | +24° | 0.08 | 1 |
| 24 | 0° | 0 | 0 |
| 36 | -18° | 0.08 | 1 |
| 48 | 0° | 0 | 0 |

### 3. 驗證動作真的在檔裡，而且外型沒爛

只看靜止圖不夠。必須：

```bash
ssh hch@10.145.119.19 "docker run --rm --entrypoint blender -v /home/hch:/home/hch blender:5.2.1 -b --python-expr \"
import bpy
bpy.ops.import_scene.gltf(filepath='/home/hch/SkinTokens/io/character_hunyuan_waving.glb')
print('actions', [a.name for a in bpy.data.actions])
arm=next(o for o in bpy.data.objects if o.type=='ARMATURE')
bpy.context.scene.frame_set(1)
print('f1', tuple(round(x,3) for x in arm.matrix_world.translation), tuple(round(x,3) for x in arm.matrix_world.to_euler()))
bpy.context.scene.frame_set(12)
print('f12', tuple(round(x,3) for x in arm.matrix_world.translation), tuple(round(x,3) for x in arm.matrix_world.to_euler()))
\""
```

預期：有 `CharRigAction`；第 12 幀 `location.z` 約 `0.08`、yaw 約 `0.42` rad（24°）。

再渲染第 1 與第 12 幀，**目視**臉、蝴蝶結、外套、腿完整，沒有扇狀紙片：

```bash
scp scripts/render_preview.py hch@10.145.119.19:/tmp/render_preview.py
ssh hch@10.145.119.19 'docker run --rm --entrypoint blender -v /home/hch:/home/hch -v /tmp/render_preview.py:/tmp/render_preview.py blender:5.2.1 -b -P /tmp/render_preview.py -- /home/hch/SkinTokens/io/character_hunyuan_waving.glb 1'
ssh hch@10.145.119.19 'docker run --rm --entrypoint blender -v /home/hch:/home/hch -v /tmp/render_preview.py:/tmp/render_preview.py blender:5.2.1 -b -P /tmp/render_preview.py -- /home/hch/SkinTokens/io/character_hunyuan_waving.glb 12'
```

預覽寫到 `/home/hch/SkinTokens/io/preview-f{幀}.png`。回覆前要讀這兩張圖，不要只報「有 action」。

在 Blender 播放：時間軸 1–48，空白鍵。

### 4. 使用者只要「會動、不要爛」時的預設

優先交 `character_hunyuan_waving.glb`。

若使用者堅持大幅度關節動作（下跪、揮手過頭）：

1. 先講清楚這套 mesh 是薄殼，局部蒙皮會撕開。
2. 需要重新拓撲或實體化之後才能做可靠的手臂／腿骨骼。
3. 不要再把 kimodo SOMA 直接 retarget 上去交差。

## A-pose 真人角色：完整已驗證流程（2026-09-22）

A-pose 是避免前一個 chibi 角色失敗的關鍵：手臂和身體有明確空隙，SkinTokens 能辨識上臂、前臂、手指與完整雙腿。

```text
A-pose 參考圖
→ 去背並補透明方形畫布
→ Hunyuan3D-2.1 Shape
→ Hunyuan3D-Paint PBR（必須在綁骨前）
→ SkinTokens 自動綁骨
→ 對 SkinTokens 骨架寫正確走路 cycle
→ 有材質、含動畫的 GLB
```

實際 A-pose job：`/home/hch/hy3d-jobs/apose/`

| 階段 | 實際輸出 |
|---|---|
| 參考圖 | `source.png` |
| Shape | `apose_shape.glb`（約 172k 頂點） |
| Paint | `paint/apose_painted.obj`、`.jpg`、roughness / metallic maps |
| SkinTokens | `paint/apose_painted_rigged.glb`（52 骨） |
| **正確走路** | `apose-forward-walk.glb` |

### 為什麼第一版走路怪

`apose-walk.glb` 錯把大腿本地 X 當成前進軸，結果雙腿主要往左右／交叉擺。不要再把它交付為正確走路。

先用 Blender probe 驗證本地軸：

```text
右髖 bone_44，左髖 bone_48
- right hip + local Y：右腳向世界 Y 正向跨出
- left hip - local Y：左腳向世界 Y 正向跨出
膝蓋 bone_45 / bone_49
- local X 正向：收腿時抬高腳踝並向前帶出
```

不要只憑骨頭軸名稱猜；要實際旋轉約 25–35°，讀取 knee / ankle world coordinates，確認跨步方向後才寫 cycle。

### 正確走路的最小 cycle

使用本 skill 的 `scripts/animate-forward-walk.py`，它輸出 `apose-forward-walk.glb`：

```bash
scp scripts/animate-forward-walk.py hch@10.145.119.19:/tmp/animate-forward-walk.py
ssh hch@10.145.119.19 'docker run --rm --entrypoint blender \
  -v /home/hch:/home/hch -v /tmp/animate-forward-walk.py:/tmp/animate-forward-walk.py \
  blender:5.2.1 -b -P /tmp/animate-forward-walk.py'
```

週期每 32 幀（24 FPS）：

1. 右腳前方 heel contact，右膝幾乎伸直。
2. 左膝 local X 約 `+54°`，左腳明顯抬高並向前跨。
3. 左腳前方 heel contact。
4. 右膝 local X 約 `+54°`，右腳明顯抬高並向前跨。
5. 手臂用相反相位小幅擺動，身體只有小幅上下 bob。

這個角色有高跟鞋，不要強迫雙腳完全貼地；在擺動幀離地是正常的。輸出是第 1–176 幀、24 FPS，約 7.3 秒，含緩慢世界 Y 位移。

### 真人 A-pose 路徑的驗證

至少渲染：右腳接觸、左腳高膝、左腳接觸、右腳高膝四個影格（例如 8、16、24、32）。判斷條件：

- 擺動腳的**膝蓋真的彎曲**，腳踝向上向前，不是整條腿左右平移。
- 兩腳不長時間重疊、交叉或同時離地。
- 手臂與腿反向，但不超過走路需要的幅度。
- 用 Material Preview 或 Eevee 預覽檢查 Paint；Workbench 一律顯示灰色，不能當成沒上色。

## Kimodo SOMA → SkinTokens 52 骨 Retarget（已完成）

當使用者要求動作必須由 Kimodo 生成時，不能再用手寫 walk keyframes 冒充 Kimodo。已驗證流程：

```text
Kimodo prompt
→ SOMA RP v1.1（soma30）animation.glb
→ Blender 讀取 30 個 animated EMPTY
→ global rotation-delta retarget
→ SkinTokens 52 骨中對應的 22 個人體主骨
→ 有 Paint 材質的動畫 GLB
```

已驗證 Kimodo walk：

| 項目 | 值 |
|---|---|
| animation id | `5c081102cd987903` |
| model | `soma-rp-v1.1` |
| text quantization | `q4_k_m` |
| frames / steps / seed | `150 / 100 / 42` |
| metadata status | `ready` |
| source | `/home/hch/kimodo.cpp/demo-output/5c081102cd987903/animation.glb` |
| retarget output | `/home/hch/hy3d-jobs/apose/apose-kimodo-walk-inplace.glb` |
| desktop copy | `apose-kimodo-walk.glb` |

Kimodo 生成 API 必須明確帶 model 與 quantization；否則 UI 上次選到不可用的 SMPL-X 時會回 409：

```bash
curl -X POST http://127.0.0.1:8094/api/generate \
  -H 'Content-Type: application/json' \
  -d '{
    "prompt":"A woman walks forward naturally with clear alternating steps. Each swing leg bends at the knee, lifts the foot, reaches forward, and lands heel first. Arms swing gently opposite to the legs. Upright steady posture. No waving, no jumping, no dancing.",
    "frames":150,"steps":100,"seed":42,
    "model":"soma-rp-v1.1","text_quantization":"q4_k_m"
  }'
```

等待 metadata `status=ready`，並確認目錄同時有 `animation.glb`、`local_rotations_xyzw.f32`、`root_positions.f32`。CPU worker 跑 150 幀可能要數分鐘；不要重複送單。

Retarget 使用本 skill：

```bash
scp scripts/retarget-kimodo-soma-to-skintokens.py hch@10.145.119.19:/tmp/retarget.py
ssh hch@10.145.119.19 'docker run --rm --entrypoint blender \
  -v /home/hch:/home/hch -v /tmp/retarget.py:/tmp/retarget.py blender:5.2.1 \
  -b -P /tmp/retarget.py -- \
  /home/hch/hy3d-jobs/apose/paint/apose_painted_rigged.glb \
  /home/hch/kimodo.cpp/demo-output/5c081102cd987903/animation.glb \
  /home/hch/hy3d-jobs/apose/apose-kimodo-walk-inplace.glb inplace'
```

### 骨架對應

- 軀幹：`bone_0..5` → Hips / Spine1 / Spine2 / Chest / Neck1 / Head
- 右臂：`bone_6..9` → RightShoulder / Arm / ForeArm / Hand
- 左臂：`bone_25..28` → LeftShoulder / Arm / ForeArm / Hand
- 右腿：`bone_44..47` → RightLeg / Shin / Foot / ToeBase
- 左腿：`bone_48..51` → LeftLeg / Shin / Foot / ToeBase
- SkinTokens 額外的手指骨保留 bind pose；SOMA compact 沒有完整手指動作。

⚠️ **2026-09-22 修正：不要把 Kimodo 的 world quaternion delta 直接套到 SkinTokens 的 local pose bone。** 兩套骨架的 rest axes／parent space 不同，這樣會把手部旋轉誤差放大，造成手掌與腳踝亂抖。

正確 retarget 順序：

1. Kimodo SOMA GLB 的 150 個 sample 在匯入 Blender 後 keyframe 間隔是 `0.8` frame（約 30 FPS），不能用 `scene.frame_set(0..149)` 當作等距 24 FPS 取樣；要先把 source time 軸轉成每 sample 一幀，或以 `sample_index * 0.8` 精確取樣。
2. 先讀取 source 每根骨頭相對 parent 的 **local rest-relative quaternion delta**，不要直接取 world quaternion。
3. 按 parent → child 順序，把 local delta 寫入 SkinTokens 對應的 pose bone；不能只用同名骨頭直接複製 local quaternion，因為兩邊 rest axes 不同。
4. 對 `ForeArm/Hand` 與 `Shin/Foot` 做輕量 3-point smoothing 即可；不要用大半徑平滑抹掉跑跳動作。
5. 重新輸出後量測手、前臂、小腿、腳的相鄰影格 quaternion 差，確認沒有異常尖峰。

這次錯誤方法的實測：原始 Kimodo 手部最大相鄰影格旋轉約 18–19°，直接 world-to-local 錯誤重定向後被放大到約 89–91°。改成正確 local-delta retarget 後，手部最大值降到約 16–17°，平均值約 4°。

本次修正版案例：

```text
/home/hch/hy3d-jobs/runjump/final-localdelta/runjump-travel-20s-localdelta.glb
D:\3D動畫\runjump-20260922\runjump-travel-20s-corrected.glb
```

預設交付 `inplace`：保留 Kimodo vertical bounce、移除 X/Y root travel，避免固定鏡頭的 GLB viewer 裡角色走出畫面。需要真正位移時把最後參數改成 `travel`。

### Kimodo Retarget 驗證

1. metadata 是 `ready`，不可只看 HTTP 200。
2. 輸出有 150 幀，且 Material Preview 可見 Hunyuan3D-Paint 貼圖。
3. 至少渲染 1、38、75、112、150 幀；角色要保持在畫面，腿與手臂姿勢有變。
4. 若 `travel` 版固定鏡頭後段全黑，通常只是角色已走出鏡頭，不代表 mesh 消失；改 `inplace` 或使用跟拍攝影機。
5. 回覆明確揭露 animation id、Kimodo model、frames、status、retarget output；不能把手寫 keyframes 說成 Kimodo。

## SkinTokens 備註（相關、但不是這條動畫的正確路徑）

自動綁骨服務裝在 `/home/hch/SkinTokens`，image `skintokens:bpy`，腳本 `/home/hch/auto-rig/rig.sh`。推理大約要 14 GB 空閒顯存；llama 佔卡時先 `docker stop llama-cpp-qwen38-27b-abliterated`。長頸鹿測試可以，這套 chibi 靜止綁骨外型還在，但**不能**再套 kimodo 動作。

## Pitfalls

- 兩層重複殼會讓權重與渲染打架；動畫腳本必須刪掉第二塊。
- `skintokens:bpy` 可 import `bpy`，但渲染缺 `libEGL`，常 exit 139。預覽用 `blender:5.2.1`。
- 10° 轉身在靜態截圖幾乎看不出來。驗證用 ≥20° 或直接比對 armature 矩陣，不要用「兩張圖看起來一樣」當失敗證據。
- Hunyuan 匯出的 GLB 再進 Blender 再出，頂點數可能膨脹；以目視與包圍盒為準。
- 不要為了跑 SkinTokens 就 `docker system prune`，同機還有 n8n、new-api、kimodo。
- 第一版 `apose-walk.glb` 的腿部前進軸寫錯，不要複用；改用 `scripts/animate-forward-walk.py` 與 `apose-forward-walk.glb`。
- llama 停掉後記得問使用者要不要開回來；本 skill 預設不自動重啟 8080。
- Kimodo UI 可能保留不可用的 `smplx-rp-v1` 選擇；呼叫 API 時務必指定 `model=soma-rp-v1.1`，否則回 409。
- **不要再使用舊版直接 world quaternion retarget 的 `retarget-kimodo-soma-to-skintokens.py` 邏輯**：它會因 source 30 FPS／Blender 24 FPS 時間軸錯位，以及 world/local rest-axis 不匹配，讓手部旋轉尖峰放大到約 90°。改用 local-delta 版本，並先做 0.8 frame 時間軸校正。
- Kimodo `travel` 動作在固定攝影機預覽中後段可能全黑；這是走出鏡頭。預設交付 `inplace`。

## Verification

1. `character_hunyuan_waving.glb` 存在且可被 `blender:5.2.1` 匯入。
2. 有 armature action；第 12 幀相對第 1 幀有非零旋轉或位移。
3. 第 1 與第 12 幀渲染：全身完整，沒有下半身皺成一團、沒有頭髮扇狀卡片。
4. 回覆寫明檔案路徑，並註明不要開 `character_hunyuan_kneel.glb`。
5. A-pose 真人走路時，四個 gait 關鍵幀都要確認至少有一隻腳是「屈膝抬起向前跨」而不是左右擺。
6. 使用者指定 Kimodo 時，必須能指出 `animation id`、metadata `ready`、model `soma-rp-v1.1` 與 retarget 腳本；否則不算完成。
