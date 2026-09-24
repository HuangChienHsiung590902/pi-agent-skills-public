---
name: sam3d-body-rigify-unreal-pipeline
description: >
  當使用者要把 SAM 3D Body 的單張人體重建結果轉成可動畫角色，經由 Blender Rigify 綁骨、Automatic Weights、FBX，
  再匯入 Unreal Engine 5.8 改姿勢、建立 Control Rig／Poseable Mesh、IK Rig／IK Retargeter 時使用。也用於 10.145.119.19 上的 SAM 3D Body Docker、
  T-pose 推理、正常姿勢與 T-pose 雙輸出、OBJ/NPZ 搬移、Gradio Web UI、NVIDIA Docker GPU runtime，以及 Rigify/UE 骨架方向、權重、關卡存檔與轉骨問題排查。
---

# SAM 3D Body → Rigify → Unreal Engine Pipeline

## When to Use

- 使用者要從單張人物圖產生 SAM 3D Body 人體 mesh。
- 使用者要把 SAM 3D Body 的 OBJ/NPZ 帶進 Blender 做 Rigify。
- 使用者要將 Rigify 角色匯出 FBX 並匯入 Unreal Engine 5.8。
- 使用者要在 UE 5.8 裡讓 `person_00` 改姿勢、舉手、或認為「骨骼對應好了」就能直接套 Manny 動畫。
- 使用者要建立 UE 5.8 的 Control Rig、Poseable Mesh、IK Rig、Retarget Root、Retarget Chains 或 IK Retargeter。
- 需要排查模型上下顛倒、T-pose 對不齊、Rigify Generate Rig 失敗、Automatic Weights、FBX 匯入、Untitled World Partition 存檔失敗。
- 本機 Blender Lab MCP 連不上、或要把已匯入的 `HumanMesh`（18439 頂點）做 Rigify／舞蹈時。
- 使用者要從人物圖片建立可綁骨、可做姿勢、可匯出 Unreal 的人形模型時。

不要用於：

- 一般 Blender 建模或材質編輯。
- 只想用 MCP 建立簡單幾何體人形原型時，改用 `blender-lab-mcp`；本 Skill 著重照片重建、T-pose、Rigify、動畫與 Unreal 管線。
- Hunyuan3D 物體生成／上色；該流程使用 `comfyui-blender-character-model`。
- SAM 3D Objects 的物體重建；本 Skill 只處理 SAM 3D Body 人體流程。

## Environment and Canonical Paths

- 遠端 GPU 主機：`hch@10.145.119.19`，hostname 通常是 `gigabyte`。
- 遠端 SAM 專案：`/home/hch/sam-3d/`。
- 遠端 Body 容器：`sam3d-body`，`restart: no`，以 `pytorch/pytorch:2.5.1-cuda12.1-cudnn9-devel` 為基礎。
- 遠端 Body checkpoint：`/home/hch/sam-3d/hf-cache/sam-3d-body-dinov3/`。
- 遠端輸出：`/home/hch/sam-3d/outputs/`。
- 本機結果目錄：`C:\Users\HCH\Downloads\sam-3d-body-demo\`；實際也用過 `D:\sam-3d-body-demo\`（參考圖／四視圖）。
- T-pose 結果目錄：`C:\Users\HCH\Downloads\sam-3d-body-demo\tpose\`。
- Blender：本機 Blender 5.2。官方 Blender Lab MCP（https://www.blender.org/lab/mcp-server/）：
  - addon：`MCP` 1.0.0，TCP `127.0.0.1:9876`（須顯示 Server is running）。
  - server：`D:/MCP/blender-mcp/.venv/Scripts/blender-mcp.exe`（repo `lab/blender_mcp`）。
  - Pi：`D:/.system/.pi/agent/mcp.json` 的 `blender`，`lifecycle: lazy`。本 session 先 `mcp({ connect: "blender" })` 再呼叫工具。
- 本機動畫成果例：`D:\BlenderProjects\happy-dance\happy-dance-rigged.blend`。
- Unreal 專案：`C:\Users\HCH\Documents\Unreal Projects\MyProject\MyProject.uproject`。
- Unreal Engine：`C:\Program Files\Epic Games\UE_5.8\Engine\Binaries\Win64\UnrealEditor.exe`。

## Procedure

### 1. 確認 SAM 3D Body Docker 與 checkpoint

1. SSH 到 `hch@10.145.119.19`，唯讀檢查：
   ```bash
   docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
   docker exec sam3d-body python -c "import torch; print(torch.__version__, torch.cuda.is_available(), torch.cuda.device_count())"
   ```
2. GPU 推理前，確認不與需要 GPU 的服務同時搶 VRAM；使用者要求時停用 `comfyui-h3`，不要自動刪除或 prune 其他容器。
3. Hugging Face gated checkpoint 必須先確認帳號對下列 repo 顯示 `ACCEPTED`：
   - `facebook/sam-3d-body-dinov3`
   - `facebook/sam-3d-objects`（只有需要 Objects 時）
4. Token 只用於當次下載或遠端受限檔案，不寫進 Skill、Git、公開 compose 或回報內容。
5. Body 必須存在：
   - `model.ckpt`
   - `assets/mhr_model.pt`
   - `model_config.yaml`
6. 若使用遠端 Web UI，確認 Docker Compose 的 `sam3d-body` 服務包含 `ports: ["7860:7860"]`、`command: ["python", "/workspace/body/webui.py"]`，且 Web UI 腳本透過 bind mount 位於 `/home/hch/sam-3d/body/webui.py`。
7. `sam3d-body` 必須明確使用 NVIDIA runtime；目前已驗證的 compose 設定為：
   ```yaml
   gpus: all
   runtime: nvidia
   ```
   只寫 `gpus: all` 在本機環境可能讓 `/dev/nvidia*` 存在，但 PyTorch 仍回報 `No CUDA GPUs are available`。

### 2. SAM 3D Body Web UI

1. 開啟瀏覽器：
   ```text
   http://10.145.119.19:7860
   ```
2. 上傳單人全身圖片，按「產生正常姿勢 + T-pose」。第一次推論會載入模型，可能需要數分鐘；後續使用共用的 Hugging Face／Torch cache。
3. Web UI 每次推論同時提供兩組結果：
   - `person_00_pose.obj`、`person_00_pose.npz`：保留輸入圖片推論出的原始姿勢。
   - `person_00_tpose.obj`、`person_00_tpose.npz`：保留身形比例、改成 MHR canonical T-pose。
   - 正常姿勢版 NPZ 的 `canonical_tpose=False`；T-pose 版 NPZ 的 `canonical_tpose=True`。
4. 遠端結果會寫入 `/home/hch/sam-3d/outputs/webui/<job-id>/`；container 內是 `/workspace/outputs/webui/<job-id>/`。
5. 目前 Web UI 沒有登入驗證，只適合內網。不要啟用 Gradio `share=True` 或直接暴露到公網。
6. 預覽圖仍是原始輸入圖片加人物框與「正常姿勢 + T-pose」標籤，不是輸出 mesh 的 3D 渲染；判斷結果要看下載的 OBJ 與 NPZ。
7. 使用選擇：需要保留圖片原始動作時使用 `person_00_pose.obj`；要進行 Blender Rigify 綁骨、FBX 或 Unreal retarget 時使用 `person_00_tpose.obj`。
8. 啟停與 HTTP 檢查：
   ```bash
   ssh hch@10.145.119.19 "docker start sam3d-body 2>/dev/null || true"
   ssh hch@10.145.119.19 "docker ps --filter name=^/sam3d-body$; curl -fsSI http://127.0.0.1:7860/"
   ```
9. 若需要從現有圖片直接跑 CLI，仍可使用既有腳本；此腳本輸出原始推論姿勢，不會自動產生 T-pose：
   ```bash
   ssh hch@10.145.119.19 'docker exec sam3d-body python /workspace/outputs/run_body_export.py \\
     --image /workspace/outputs/input.jpg \\
     --output_dir /workspace/outputs/result \\
     --checkpoint_path /workspace/hf-cache/sam-3d-body-dinov3/model.ckpt \\
     --mhr_path /workspace/hf-cache/sam-3d-body-dinov3/assets/mhr_model.pt'
   ```
10. Web UI 的 T-pose 輸出不是把原始 OBJ 做整體旋轉，而是保留推論出的 `shape_params` 與 `scale_params`，將 `global_rot`、`body_pose_params`、`hand_pose_params`、`expr_params` 歸零後，呼叫 MHR `head_pose.mhr_forward()` 重新生成 mesh。這會同時處理手臂、腿、軀幹與手部姿勢；正常姿勢版則直接使用 SAM 3D Body 原始 `pred_vertices + pred_cam_t`。

### 3. GPU 與 Web UI 故障排查

1. 先分開確認主機與 container：
   ```bash
   ssh hch@10.145.119.19 "nvidia-smi"
   ssh hch@10.145.119.19 "docker exec sam3d-body nvidia-smi"
   ssh hch@10.145.119.19 "docker exec sam3d-body python -c \\\"import torch; print(torch.cuda.is_available(), torch.cuda.device_count())\\\""
   ```
2. 若主機 `nvidia-smi` 正常，但 container 的 `torch.cuda.is_available()` 是 `False` 或推論出現 `No CUDA GPUs are available`：
   - 檢查 `docker inspect sam3d-body` 的 `Runtime` 是否為 `nvidia`。
   - 檢查 `/home/hch/sam-3d/docker-compose.yml` 的 `sam3d-body` 是否有 `runtime: nvidia`。
   - 執行 `docker compose -f /home/hch/sam-3d/docker-compose.yml up -d --force-recreate sam3d-body`，不要先刪除模型 cache 或輸出。
   - 再驗證 `docker exec sam3d-body nvidia-smi`、PyTorch CUDA 與 HTTP `200 OK`。
3. 已驗證的正常結果：
   ```text
   Runtime=nvidia
   cuda_available=True
   device_count=2
   HTTP/1.1 200 OK
   ```
4. Docker GPU 測試可用一次性 container 驗證 runtime，不要把測試 container 留在主機：
   ```bash
   docker run --rm --gpus all --entrypoint nvidia-smi sam3d-body:latest
   ```
5. 不要把只有 HTTP `200 OK` 誤報成推論成功；必須實際上傳圖片並確認輸出 OBJ／NPZ 已產生。
6. T-pose 推論成功的最低驗證條件：
   - `person_00_tpose.obj` 存在且大小大於零。
   - NPZ 的 `canonical_tpose` 為 `True`。
   - `body_pose_params` 與 `hand_pose_params` 全部為零或接近零。
   - `global_rot` 為 `[0, 0, 0]` 或接近零。
   - `vertices` 形狀為 `(18439, 3)`、`joints` 形狀為 `(127, 3)`。
7. 已驗證的 Web UI T-pose 輸出條件：
   ```text
   /workspace/outputs/webui/<job-id>/person_00_tpose.obj
   /workspace/outputs/webui/<job-id>/person_00_tpose.npz
   canonical_tpose=True
   body_pose_params 最大絕對值=0
   hand_pose_params 最大絕對值=0
   global_rot=[0, 0, 0]
   ```
8. 已驗證的雙輸出範例：
   ```text
   /workspace/outputs/webui/85bfc159a783/person_00_pose.obj
   /workspace/outputs/webui/85bfc159a783/person_00_pose.npz
   /workspace/outputs/webui/85bfc159a783/person_00_tpose.obj
   /workspace/outputs/webui/85bfc159a783/person_00_tpose.npz
   ```
   正常姿勢與 T-pose 是同一次模型推論產生，不需要對圖片重跑第二次。

### 4. 使用 T-pose 輸入

1. 優先使用單一正面 T-pose 圖，不要把前、後、左、右、頭手腳特寫拼圖整張丟給 Body。
2. 從角色 turnaround sheet 只裁出 Front View；裁切結果先目視確認頭、手、腳完整且沒有混入下一格。
3. T-pose 比跳舞或大幅遮擋姿勢適合 Rigify、Mixamo、AccuRIG 與 UE retarget。
4. 用遠端 `sam3d-body` 跑推理，輸出 OBJ/NPZ；推理腳本應保留：
   - mesh OBJ
   - NPZ 的 vertices、keypoints_3d、joints、faces
   - bbox 預覽圖

### 5. 搬移與檢查輸出

1. 從遠端複製結果到本機，並比對檔案大小；不要複製 checkpoint 或 token。
2. 本機至少保留：
   - `person_00.obj`
   - `person_00.npz`
   - `person_00_bbox.jpg`
3. NPZ 的 `keypoints_3d` 通常是 MHR70，另有 `joints`。關節座標是輔助對骨資料，不等於現成 Armature 或 skin weights。

### 6. 從人物圖片建立可動畫人形的標準選擇

依目標選擇來源：

| 需求 | 建議來源 |
|---|---|
| 人體身形、後續 Rigify/Unreal | SAM 3D Body，優先使用 `person_00_tpose.obj` |
| 服裝、髮型、配件與角色外觀 | Hunyuan3D 或其他角色生成器，再在 Blender 整理拓撲 |
| 只是測試 MCP 或流程 | `blender-lab-mcp` 透過 Python 建立簡單幾何體人形 |
| 手動高品質角色 | Mirror 建模 → Sculpt → Retopology → UV/材質 → Rigify |

推薦完整流程：

```text
正面全身 T-pose 圖片
→ SAM 3D Body
→ person_00_tpose.obj / .npz
→ Blender 檢查頭腳方向與尺寸
→ Rigify Human Meta-Rig
→ 對齊骨架
→ Generate Rig
→ Automatic Weights
→ AnimChar Pose Editor / Motion Library
→ FBX
→ Unreal Engine
```

注意：一般站姿或雙手貼身的模型不適合直接 Automatic Weights；若要做手語、舞蹈或舉手動作，輸入最好是雙臂張開的 T-pose 或 A-pose。MCP 可以協助匯入、檢查、命名、建立骨架與測試，但不應假設它會自動產生高品質人體拓撲、臉部、髮型、服裝或正確權重；每一步都要讀取目前場景並驗證結果。

### 7. Blender MCP 與座標方向

1. Blender 附加元件必須啟用名稱為 `MCP` 的 Blender Lab addon，並顯示 `Server is running`；Rigify 是另一個 addon。Pi 端若顯示 lazy／not connected，先 `mcp({ connect: "blender" })`。官方 zip 是 `mcp-1.0.0`；本機已裝同版時不要重裝。
2. 連線後先讀場景與物件摘要，不要猜物件名稱、旋轉或尺寸。GLB 匯入常見名稱是 `HumanMesh`（parent 常為 `Node_176`），頂點數 **18439** 可當成 SAM 3D Body 指紋。
3. SAM Body OBJ 匯入後需檢查世界 bounds、頭腳方向與 mesh rotation。
4. 本流程的關鍵座標坑：
   - 之前錯誤使用 `rotation_euler.x = +90°` 會使人物上下倒置。
   - 正確使用 `rotation_euler.x = -90°`，即 `(x, y, z) → (x, z, -y)`。
5. 確認頭在上、腳在下後，才建立 Meta-Rig；不要在方向未確認前 Generate Rig 或綁權重。
6. **綁骨姿勢：** 必須用手臂離開軀幹的 T-pose 或明顯 A-pose。雙手貼著胯的站姿做 Automatic Weights 後，一舉手會把髖／腰頂點拉成「披風」。先量寬高比：站姿貼身約 0.3，張開手臂約 ≥0.6。
7. NPZ **可選**。Blender 不會匯入 `.npz`；使用者只拖 OBJ／GLB 時場景裡不會有 NPZ。沒有 MHR70 也能用幾何對齊 Rigify，手指與臉較粗。有 NPZ 再補對手指。

### 8. Rigify 對骨與 Generate Rig

1. 啟用 Blender 內建 `Rigify`；不需要額外安裝 Rigify Feature Set。
2. 建立 Human Meta-Rig，依 MHR70 對齊：
   - 髖：left/right hip
   - 軀幹：hip midpoint → neck → nose/head
   - 手臂：shoulder → elbow → wrist
   - 腿：hip → knee → ankle → toe/heel
   - 手指：MHR tip/first/second/third 反向建立 Rigify finger chain
3. MHR 關節與 Rigify 骨架不是一對一相同定義；臉部只能合理縮放／平移，不能宣稱表情骨完全精準。
4. Rigify metarig 的拓撲不能任意改壞。尤其：
   - `spine.004` 是 `spines.super_head` 的起點。
   - 要保留 stock Rigify 的 `spine.004.use_connect = False`。
   - `spine.005`、`spine.006` 與基本 spine chain 需維持 stock parent/connect 關係。
   - 否則會出現：`RIGIFY ERROR: Bone 'spine.004': Cannot connect chain - bone position is disjoint.`
5. 最安全的修復方式是建立一副 stock Human Meta-Rig，複製每根同名骨頭的 parent/connect topology，再把已對齊骨頭的 connected child head 重設為 parent tail。
6. Generate Rig 成功後，確認生成 `rig`，再進行 Automatic Weights。
7. **Blender 5 / Rigify 實測：** `bpy.ops.pose.rigify_generate()` 產生的 `rig` 常在世界原點 `(0,0,0)`，即使 `metarig` 已放在人物腳底。權重前必須：mesh `parent_clear(KEEP_TRANSFORM)` → `rig.location = metarig.location` → 再 Automatic Weights。否則 vertex groups 是空的，mesh 不會跟骨頭動。
8. 旋轉手臂 chain 時，connected bone 不能任意寫 `head`；依 parent 深度排序，只改未 connect 的 head 與所有 tail。

### 9. Automatic Weights 與變形測試

1. Mesh 與生成的 `rig` 都選取，`rig` 設為 active。
2. 執行 `Parent → With Automatic Weights`。
3. 驗證：
   - mesh parent 是 `rig`
   - mesh 有一個 Armature modifier 且 object 是 `rig`
   - 所有 mesh vertices 都至少有一個 vertex group assignment
4. 用 Rigify FK/IK controls 測試至少：
   - 左右手臂／肘
   - 左右大腿／膝
   - 脊椎
5. 記錄最大與平均頂點位移，確認 mesh 真的跟骨架變形；測試後清除 pose transforms，恢復 T-pose，再保存 `.blend`。用 `evaluated_get` + `to_mesh()` 比 bbox，不要只讀未評估的 `mesh.vertices`。
6. Blender 5 layered Action：寫關鍵幀必須 **先 `scene.frame_set(f)`，再設 pose，再 `keyframe_insert`**。先設 pose 再 `frame_set` 會把值清成 0。循環用 `action.use_cyclic = True` 與 `action.frame_start/end`；`Action.fcurves` 已不存在，改走 `action.layers[0].strips[0].channelbags[0].fcurves`。

### 10. FBX 匯出給 UE

1. 只選 mesh `person_00` 與 generated `rig`；不要選 Camera、Light、metarig。
2. 匯出路徑：
   `C:\Users\HCH\Downloads\sam-3d-body-demo\tpose\person_00_ue.fbx`
3. 建議 Blender FBX 設定：
   - `use_selection=True`
   - `object_types={'ARMATURE','MESH'}`
   - `use_mesh_modifiers=True`
   - `use_armature_deform_only=True`
   - `add_leaf_bones=False`
   - `bake_anim=False`
   - `apply_unit_scale=True`
   - `axis_forward='-Z'`
   - `axis_up='Y'`
4. FBX 匯出後先確認檔案存在與大小，再在 UE 匯入；不要把成功匯出誤報成 UE 已驗證。

### 11. Unreal Engine 5.8：先放進已存關卡，不要匯入場景

1. UE 專案必須啟用：
   - `IKRig`
   - `PythonScriptPlugin`
   - `EditorScriptingUtilities`
   - `ModelContextProtocol`
   - 操編輯器用：`EditorToolset`、`PluginToolset`（CmdLink／MCP 細節見 `ue-cmdlink-mcp`）
2. MyProject 已啟用上述 MCP／EditorToolset。不要開 `AllToolsets`。
3. **不要**對未存檔的 World Partition 新圖（`/Temp/Untitled_1`）做 FBX「匯入場景」。會留下 `/Temp/__ExternalActors__/Untitled_1/...`，另存為時報「儲存包失敗」。正確：開已存關卡（例如 `/Game/FirstPerson/Lvl_FirstPerson`），再把 **Skeletal Mesh** `/Game/person_00` 放進去。
4. 儲存 Content Browser 後確認磁碟：
   `C:\Users\HCH\Documents\Unreal Projects\MyProject\Content\`
5. 匯入後通常會有：
   - `/Game/person_00.person_00`
   - `/Game/person_00_Skeleton.person_00_Skeleton`
   - `/Game/person_00_PhysicsAsset.person_00_PhysicsAsset`
6. 2026-09-10 已驗證：用 MCP `SceneTools.add_to_scene_from_asset` 把 `/Game/person_00` 放到 `Lvl_FirstPerson`，actor 標籤 `person_00_step1`。使用者若說自己看著編輯器，不要截圖塞對話。
7. Blender 對過 MHR、FBX 有 Armature，**不等於** UE 能播 Manny 的 `MM_*`。Manny 骨名是 `pelvis`／`hand_l`；`person_00` 是 Rigify `DEF-spine`／`DEF-upper_arm_L`。沒做 IK Retargeter 之前，拖 `MM_Rifle_Jog_Fwd` 不會正確驅動這隻。

### 11b. 在 UE 裡立刻改姿勢（Poseable Mesh；已驗證）

目標：證明權重能動，例如舉起 `DEF-upper_arm_L`。這不是最終動畫管線。

1. 元件用 MCP `editor_toolset.toolsets.actor.ActorTools.add_component`。`SkeletalMeshActor` **沒有** `add_component_by_class`。
2. 加 `PoseableMeshComponent`（`/Script/Engine.PoseableMeshComponent`）。
3. CmdLink 的 `PY` 多行會被拆壞；腳本寫 `%TEMP%\pi-work\ue-ikrig\`，再：
   `cmdlink PY "exec(open(r'.../script.py', encoding='utf-8').read())"`
4. Python 有效順序（2026-09-10 實測）：
   - `sk.set_visibility(False)` 藏原本 SkeletalMeshComponent
   - `pmc.set_skinned_asset_and_update(mesh, True)`（`set_skeletal_mesh` 已改名）
   - `pmc.set_visibility(True)`；確認 `pmc.is_visible()` 為 True，否則畫面仍是 T-pose
   - `pmc.set_bone_rotation_by_name('DEF-upper_arm_L', rotator, unreal.BoneSpaces.WORLD_SPACE)`
5. 用左右手世界座標差驗證，不要只信 API 回傳。舉左手時 `DEF-hand_L.z` 應明顯高於 `DEF-upper_arm_L.z`（實測約 +45 cm 才看得見舉手）。
6. 旋轉方向因 Rigify 軸而異；可掃一組 pitch/yaw/roll 取 `hand.z - arm.z` 最大者。實測舉手較有效：`Rotator(-90, 180, -90)`、WORLD_SPACE。

### 11c. Control Rig 資產（半完成）

1. 建立：
   `unreal.ControlRigBlueprintFactory.create_control_rig_from_skeletal_mesh_or_skeleton(mesh, False)`
   再 rename 到 `/Game/CR_person_00`（工廠預設可能是 `/Game/person_00_CtrlRig`）。2026-09-10 此資產已存在。
2. `unreal.ControlRigBlueprintLibrary.set_preview_mesh(bp, mesh, True)`。
3. 把 `ControlRigComponent` 加到 actor 後，`set_control_rig_class` 已 deprecated；`set_control_rig_asset_reference(bp)` 對 Blueprint 物件會 TypeError。runtime `get_control_rig()` 可能有實例，但 **`set_bone_transform` 改數字、viewport 皮膚常常不動**。
4. 要給使用者「看得見的舉手」時，走 8b Poseable Mesh，不要停在 Control Rig 數字。Control Rig 圖裡加 FK 控制器、給人在編輯器手轉，是另一步，尚未驗證自動化。

### 11d. Unreal Engine 5.8 IK Rig（尚未在 MyProject 建立）

截至 2026-09-10，`/Game/IK/IKR_person_00` **不存在**。Content 有 `CR_person_00`，沒有 IK Rig。下列是下一步，不要寫成已完成。

1. UE Python 可使用 `unreal.IKRigDefinitionFactory` 建立 IK Rig，使用 `unreal.IKRigController`：
   - `set_skeletal_mesh(mesh)`
   - `set_retarget_root('DEF-spine')` 或實際存在的骨名
   - `set_root_motion_bone('root')`
   - `add_retarget_chain(name, start_bone, end_bone, '')`
6. 由 Rigify deform bones 建立最小 retarget chains：
   - Spine：`DEF-spine` → `DEF-spine_006`
   - LeftArm：`DEF-upper_arm_L` → `DEF-hand_L`
   - RightArm：`DEF-upper_arm_R` → `DEF-hand_R`
   - LeftLeg：`DEF-thigh_L` → `DEF-toe_L`
   - RightLeg：`DEF-thigh_R` → `DEF-toe_R`
   - LeftHand：`DEF-hand_L` → `DEF-hand_L`
   - RightHand：`DEF-hand_R` → `DEF-hand_R`
   - Head：`DEF-spine_004` → `DEF-spine_006`
7. 建議資產路徑：`/Game/IK/IKR_person_00`。
8. 建立後用 `get_retarget_chains()`、`get_retarget_root()`、`get_skeletal_mesh()` 重新讀回驗證，並確認 `/Game/IK/IKR_person_00.uasset` 存在。

### 12. IK Retargeter 與 Manny/Quinn

1. 先搜尋專案與 Engine content 是否有 Manny／Quinn 的 IK Rig；空白 UE 專案可能沒有，不能自行假設存在。
2. 若有來源 IK Rig，建立 IK Retargeter，設定 source IK Rig 與 target `IKR_person_00`。
3. 使用 `unreal.IKRetargeterController` 可操作：
   - `set_ik_rig`
   - `set_preview_mesh`
   - `auto_map_chains`
   - `add_default_ops`
   - `get_all_chain_settings`
4. 若沒有 Manny／Quinn 或來源動畫，先交付角色 IK Rig，明確標註 Retargeter 尚未建立，不要虛構動畫測試結果。

## Pitfalls

- 不要將 Hugging Face token 寫進 `SKILL.md`、compose、Git 或輸出回報。
- SAM 3D Body 輸出是裸體參數人體，不包含參考圖的洋裝、頭髮、鞋子或材質。四視圖／參考圖投影到這顆 mesh 只會變成皮膚上的汙漬，不是穿上衣服；要上色走 `comfyui-blender-character-model`（Hunyuan3D-Paint），且 Paint 也不會長出頭髮或裙襬幾何。
- 不要把 `person_00.npz` 的關節點誤稱為已完成骨架或 skin weights。沒有 NPZ 也可以 Rigify，不要因此停工。
- 不要用雙手貼胯的站姿做 Automatic Weights。
- Generate 後若 `rig.location` 是原點而人物不在原點，先搬 rig 再綁權重。
- 不要在 Blender 5 先改 pose 再 `frame_set` 後 keyframe；動畫曲線會全是 0。
- 不要在未確認 OBJ 方向前手動加 `+90°`；這次實際造成上下顛倒的根因是旋轉方向用反。
- 不要把 Rigify 全部 706 根控制骨頭匯入 UE；FBX 使用 deform bones only。
- Rigify Generate Rig 失敗時先檢查 metarig 的 parent/connect topology，不要直接重建或繼續 Automatic Weights。
- UE Content Browser 顯示匯入資產不代表已寫入磁碟；必須 Save All 後檢查 `.uasset`。
- UE MCP 只有 AgentSkillToolset 時，不能宣稱能建立 IK Rig；先啟用並重啟 `PluginToolset`、`SlateInspectorToolset`，或透過 UE Python 執行確定性腳本。
- 不要把空白專案當成一定有 Manny／Quinn；先查資產。
- 不要把建立 `IKR_person_00` 說成已完成動畫重定向；必須另行驗證 source IK Rig、retargeter 與實際動畫。
- 所有測試／暫存腳本放在系統暫存目錄，例如 `C:\Users\HCH\AppData\Local\Temp\pi-work\ue-ikrig\`；正式 Skill 只保留可重用文件與必要腳本。

## Verification

- Skill folder 是 `D:\OB\skills\sam3d-body-rigify-unreal-pipeline\`。
- `SKILL.md` frontmatter 的 `name` 與資料夾名稱完全一致。
- YAML frontmatter 可解析，且 description 具有 trigger 語意。
- SAM Body checkpoint、OBJ/NPZ、Rigify `.blend`、FBX 的實際路徑與檔案存在性已核對。
- Blender 驗證項目：
  - `rig` 存在，且 `rig.location` 對齊人物（不是誤留在原點）。
  - mesh 有 Armature modifier 指向 `rig`。
  - 所有 vertices 都有權重群組，且至少一個 deform group 權重 > 0。
  - 舉手／抬腿時 evaluated bbox 有變化。
  - Pose 測試後已恢復 T-pose。
- UE 驗證項目：
  - `person_00` Skeletal Mesh、Skeleton、Physics Asset 已 Save All。
  - `/Game/IK/IKR_person_00.uasset` 存在。
  - IK Rig 的 skeletal mesh、Retarget Root 與 chains 可由 UE Python 重新讀取。
  - 若沒有來源 Manny／Quinn IK Rig，回報「角色 IK Rig 已建立，但動畫 Retargeter 尚未完成」。
- 新增 Skill 後執行：
  ```powershell
  python D:\OB\skills\obsidian-mcp-skill-router\scripts\rebuild_skills_index.py
  ```
- 最後確認 `SKILLS_INDEX.md`、`_consolidation/SKILLS_CATALOG.json`、`_consolidation/SKILLS_BY_CATEGORY.md`、`_consolidation/SKILLS_TAG_INDEX.md` 均包含新 Skill，且索引數量與實際 Skill 目錄一致。

## Inputs and Outputs

### Inputs

- 單張人物圖片、SAM 3D Body checkpoint，以及遠端 `sam3d-body` Docker/GPU 環境。
- OBJ/NPZ、Blender 5.2/Rigify、FBX 與 Unreal Engine 5.8 專案。

### Outputs

- 正常姿勢與 T-pose 的 OBJ/NPZ、Rigify 綁骨與權重驗證、UE Skeletal Mesh/IK 資產。
- 每個階段都要標註已驗證、半完成或尚未建立的結果，不把推測當成完成。

## Rules and Limitations

- Hugging Face token、遠端 credential 與未授權檔案不可寫入 Skill、Git 或回報。
- GPU 推理前先確認不會與其他服務搶 VRAM；不得為了測試任意刪除 cache、輸出或容器。
- 方向、權重、FBX、UE Retargeter 必須逐層驗證；HTTP 200 或資產顯示不等於推理、變形或動畫成功。
- 未找到來源 IK Rig/動畫時，必須明確回報尚未完成，不得虛構 Manny/Quinn 重定向結果。
