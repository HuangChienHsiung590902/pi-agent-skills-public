---
name: comfyui-blender-character-model
description: Generate a full-body character mesh from reference images through remote ComfyUI Hunyuan3D, then texture it with Hunyuan3D-Paint on host GPU and inspect in Blender. Use for image-to-3D character work and GPU paint texturing; not for generic Blender edits.
---

# ComfyUI Blender Character Model

Use the remote host `hch@10.145.119.19` (hostname often `gigabyte`) with two RTX 4060 Ti GPUs. ComfyUI runs in Docker `comfyui-h3` on host port `8190` (maps to container `8188`). Use local Blender 5.2 to inspect and deliver `.glb` / `.blend` files.

## When to Use

Use for reference-image generation feeding Hunyuan3D, GPU Shape/Paint, and Blender inspection/delivery of those characters. For an existing mesh, preserve it when the request is paint-only; do not regenerate its shape without authorization.

## Inputs and Outputs

- Inputs: approved design/reference images, optional existing mesh, GPU requirement, and requested output format.
- Outputs: textured GLB, a directly openable BLEND with packed textures when requested, and an actual model render. Label a usable first pass separately from a refined result.
- Before host installation or API troubleshooting, read [host-gpu-paint.md](references/host-gpu-paint.md).
- Before preparing views, repairing geometry, or claiming visual improvement, read [generation-and-quality.md](references/generation-and-quality.md).

## Workflow

1. Turn a single reference image into clean, consistent front, side, and back full-body reference images. Keep the subject centred, fully visible, and on a plain background.
2. Before Hunyuan3D conditioning, place each vertical view on a square canvas. Do not use a centre crop on a portrait image: it can discard the head or legs and produces a bust-only mesh.
3. Use the **Multi-View checkpoint/config pair**, not a single-view 2.1 config. The verified Multi-View host path is `/home/hch/ComfyUI/models/hy3d/mvroot/hunyuan3d-dit-v2-mv/` with `model.fp16.safetensors` linked to `/home/hch/ComfyUI/models/checkpoints/hunyuan3d-dit-v2-mv.safetensors`. Use `MVImageProcessorV2` / the Multi-View config and provide front, left, back, and optionally right images. Use the full model at about 30 steps and CFG 5 for structure-critical character work.
4. Save a `.glb`, import it into Blender, render a front inspection image, and check for all limbs, hands, feet, and bag/accessory silhouettes before presenting it.
5. If the mesh omits body parts, regenerate from corrected square full-body views. Do not add primitive limbs as a substitute for a failed image-to-3D reconstruction unless the user explicitly asks for a stylized manual repair.
6. When the user asks to texture / color / 上色 the mesh with Hunyuan, run **Hunyuan3D-Paint on the host GPU**. Shape DiT alone does not paint.

## Proven successful configuration (shape)

- The reliable recovery path was: generate three consistent full-body views, pad them to square without cropping, remove the plain background to transparent PNGs, then run the full multi-view checkpoint.
- Background removal used rembg core API with `bria-rmbg-2.0.onnx`; the ComfyUI background-removal node had no available model in this setup. Download the model once on the remote host and copy it into the container at `/root/.rembg/models/bria-rmbg/bria-rmbg.onnx`.
- Process `model_front_square.png`, `model_left_square.png`, and `model_back_square.png` into `model_front_cut.png`, `model_left_cut.png`, and `model_back_cut.png`. Use these cutout files as the three `LoadImage` inputs.
- The successful Hunyuan3D settings were `hunyuan3d-dit-v2-mv.safetensors`, 30 steps, CFG 5, Euler/flow-matching sampling, `octree_resolution=384`, `num_chunks=12000`, and the Multi-View processor/config with front/left/back/right conditioning. A verified four-view run produced `146678` vertices and `293412` faces in about 110 seconds after model load.
- **Do not load the 2.1 single-view checkpoint with `MVImageProcessorV2`.** The 2.1 single-view conditioner expects a 4-D tensor; Multi-View requires the Multi-View checkpoint whose config uses `DinoImageEncoderMV` and `MVImageProcessorV2`. Mixing them causes tensor shape errors or silently loses view conditioning.
- A tested four-view source sheet was split using inspected separator boundaries, not assumed equal quarters. After splitting, remove the separator/label pixels, run BRIA-RMBG, save transparent square cutouts, and inspect each cutout over a contrasting background before inference.
- Shape model install/source locations verified on the host: `/home/hch/Hunyuan3D-2.1`, `/home/hch/venvs/hy3d`, and 2.1 weights under `/home/hch/ComfyUI/models/hy3d/hunyuan3d-2.1/`.
- A grey or solid background, even when padded to square, can become a large rectangular mesh plane. Do not submit padded images until the alpha cutouts are verified.
- In this environment `onnxruntime-gpu` may warn that `libcublasLt.so.13` is missing and fall back to CPU. This is slow but functional for rembg; it is not by itself a job failure.
- Always perform a Blender render inspection. The verified successful result contained visible hands, feet, shoes, bow, hair, hoodie, and shoulder bag with no background plane. Only then deliver the GLB and imported BLEND.

## Rules and Limitations

- Host has **two NVIDIA GeForce RTX 4060 Ti** (~16 GB each). User requires Hunyuan3D-Paint to run on **GPU only** (no CPU substitute when they asked for Hunyuan paint).
- The native `ImageOnlyCheckpointLoader` keeps an individual Hunyuan3D shape pass on one GPU. Do not promise automatic two-GPU acceleration for that node. A second GPU can run a separate job, but parallel work does not speed one sampling pass.
- **Do not assume ComfyUI Docker has working CUDA.** `comfyui-h3` has been observed with `torch.cuda.is_available() == False` and NVML init failure even while the host `nvidia-smi` shows free GPUs. For Paint, prefer a **host venv** with CUDA torch (example: `/home/hch/venvs/hy3d`) rather than hoping the container sees the GPU.
- Docker Compose for `comfyui-h3` must request the nvidia runtime / device if you need GPU inside the container; fixing that is separate from host-side Paint.
- GPU Shape/Paint does not imply that xatlas UV unwrap, file export, and every preprocessing operation run on GPU. Report these stages accurately. Never substitute Blender CPU projection for a requested Hunyuan GPU Paint job.
- Prefer a verified, viewable first result followed by refinement. Do not call an unchanged file repaired or an uninspected render finished. Keep the original and export revisions to distinct paths.

## Hunyuan3D-Paint (GPU texturing) — required path when user wants 混元上色

Shape generation and texture painting are different stages.

### What does NOT count as painted

- `VAEDecodeHunyuan3D` → `VoxelToMesh` → `SaveGLB` produces a **grey** mesh (often `baseColorFactor` ~0.22, no `images` / textures in the GLB). Do not call this textured.
- Built-in `BakeTextureFromVoxel` / `PaintMesh` need voxel colors aligned to the mesh. Shape DiT voxels are not a substitute for Paint on an existing character mesh.
- Driving `Load3D` / `Load3DAdvanced` through the HTTP API is painful: they require a `LOAD_3D` viewport widget (`image` / `viewport_state`). Prefer host `hy3dgen` Paint over inventing Load3D API payloads.
- Cloud Comfy.org nodes (`TripoTextureNode`, `MeshyTexture*`, `Tencent3DTextureEditNode`) need Comfy.org login (`auth_token_comfy_org` / `api_key_comfy_org`). Without login you get `Unauthorized: Please login first`. Tencent texture edit also caps faces (~100k); a ~275k-face character needs decimation first.

### Weights and layout

- Download from Hugging Face `tencent/Hunyuan3D-2`:
  - `hunyuan3d-paint-v2-0-turbo` (or non-turbo paint)
  - `hunyuan3d-delight-v2-0`
- For Hunyuan3D 2.1 PBR Paint, the verified weights were downloaded from `tencent/Hunyuan3D-2.1` and stored under `/home/hch/ComfyUI/models/hy3d/hunyuan3d-2.1/hunyuan3d-paintpbr-v2-1/`. Its Shape/VAE weights are under the same tree.
- Put 2.0 Paint/Delight under a dedicated tree such as `/home/hch/ComfyUI/models/hy3d/` so `Hunyuan3DPaintPipeline.from_pretrained(that_root, subfolder="hunyuan3d-paint-v2-0-turbo")` finds delight beside paint.
- The verified 2.1 PBR Paint run used the host venv `/home/hch/venvs/hy3d`, the source `/home/hch/Hunyuan3D-2.1`, the original reference image as the texture reference, and a host-GPU process with `CUDA_VISIBLE_DEVICES=0`.
- **Do not leave empty placeholder directories** named like the HF folders at `/home/hch/ComfyUI/models/hunyuan3d-paint-v2-0-turbo`. Empty dirs confuse operators; delete them or fill them with real weights.
- Host system Python is PEP 668 managed. Use the existing venv (`/home/hch/venvs/hy3d`) for `huggingface_hub`, CUDA `torch`, and `hy3dgen`. Never `pip install` into system Python without a venv.

### Download reliability

- Paint + delight are multi-GB. If `.incomplete` sizes stop growing, inspect the process, network and logs before deciding it is stuck. Resume the specific failed download; do not repeatedly restart healthy jobs or kill unrelated processes.
- When Hugging Face is slow from this network, set `HF_ENDPOINT=https://hf-mirror.com` for the download process.
- Do not run several competing downloaders (extra agents, duplicate `snapshot_download`, huge unrelated `docker pull`) on the same link; they starve each other.
- Speak ETAs in plain language (minutes left, what is still downloading). Avoid jargon like「推環境」in user-facing updates.

### Run Paint

1. Confirm host CUDA in the venv: `python -c "import torch; assert torch.cuda.is_available()"`.
2. Install / import `hy3dgen` from a clone of `Tencent-Hunyuan/Hunyuan3D-2` (or pip install the package) in that venv.
3. Inputs that already work on this host:
   - Mesh: `/home/hch/ComfyUI/output-h3/3d/character_exact_turnaround_00001_.glb` (or a user-exported paint mesh)
   - Reference: `/home/hch/ComfyUI/input/exact_front_square.png` (and side/back squares when using multiview paint examples)
4. Inspect the actually imported source before selecting the API. The source used on 2026-09-08 accepted a list of PIL images; an older package's single-image error does not establish a limitation of every version. See the host reference for the tested GPU calls. If OOM, first investigate resolution/batch controls; do not silently introduce CPU model offload against the user's GPU-only constraint.
5. Export textured GLB under `/home/hch/ComfyUI/output-h3/3d/` or a job-specific directory and verify the GLB contains embedded images / textured materials before telling the user it is done. When the official Paint pipeline returns OBJ + MTL + JPG maps rather than a GLB, convert it to GLB with PBR material links and then inspect the resulting GLB; do not report the OBJ alone as the final deliverable.
6. For the verified 2.1 PBR Paint setup, install `pytorch-lightning` in the venv, pass `trust_remote_code=True` when loading the local `hunyuanpaintpbr` custom pipeline, and add the compatibility alias for old BasicSR imports if the installed torchvision lacks `torchvision.transforms.functional_tensor`:
   ```python
   import sys, types
   import torchvision.transforms.functional as functional
   compat = types.ModuleType("torchvision.transforms.functional_tensor")
   compat.rgb_to_grayscale = functional.rgb_to_grayscale
   sys.modules["torchvision.transforms.functional_tensor"] = compat
   ```
   These are compatibility fixes for this verified environment, not reasons to change the system Python.
7. Use a distinct revision directory for every Shape/Paint attempt, preserve the original GLB, and record the actual GPU stage, output path, file size, and embedded-texture check.
6. Copy results with `scp` between the host and local workspace and verify the destination. Repeated large Desktop copies through a file bridge have failed; do not rely on that as the only delivery path.

## Identity and quality limits

- A successful Shape + Paint pipeline does **not** guarantee that the face is identical to the reference photo. Shape reconstructs geometry and Paint synthesizes a texture from views; it is not a face-identity reconstruction or photographic face transfer.
- If the user requires the original face to match closely, say so explicitly before running Paint. Use an approved face-reconstruction / local face projection / Blender face-retouch stage after the Shape result, and compare a close-up render against the source. Do not claim that PBR Paint alone preserved identity.
- Do not send a generated A-pose that changes the face or clothing without approval. Preserve the user's original image and all generated revisions separately. A grey Shape preview and a PBR Paint preview must both be inspected; neither proves identity fidelity by itself.

## Three-view texture painting in Blender (fallback only)

Use this only when the user explicitly wants a fast Blender material/UV color from a turnaround **and** has not required Hunyuan GPU Paint. It is a material and UV operation: do not regenerate, reimport, alter vertices, or cut geometry while applying the texture.

1. Start from a preserved `.blend` copy of the approved mesh. Keep a separate output file for the textured version.
2. Split the turnaround into the three views and remove their backgrounds before sampling colors. Sampling a white or grey backdrop causes visible white edge bleeding on hair, sleeves, and shoes.
3. Create UV coordinates per mesh loop from the relevant view: front-facing surfaces use the front panel, back-facing surfaces use the back panel, and side-facing surfaces use the side panel. Base the choice on the face normal, then blend colors across the transition rather than hard-switching at one normal threshold.
4. Extend cutout foreground colors a short distance into transparent pixels before sampling. This prevents border pixels from pulling in the original backdrop.
5. Bake or store the blended result as vertex color / a packed image material. Pack source images into the `.blend` so the user does not lose colors after moving the file.
6. Render front, side, and back in Eevee or material preview before delivery. Object Solid mode is grey even when materials are correctly assigned; tell the user to use Material Preview (`Z`, then `M`) if they are inspecting in the viewport.

### Texture quality gates

- Do not deliver a first-pass projection without a material render. A planar front projection stretches severely around the head and arms.
- Do not use a hard front/side/back material boundary: it creates seams at the hairline, cheeks, sleeves, and bow.
- If the view panels differ in lighting or generated detail, expect some softening at blends; preserve clear reference features such as eyes, bow, hoodie drawstrings, bag, socks, and shoes from the most front-facing source.
- When an unwanted back accessory is fused into the body mesh, do not delete a broad bounding-box selection. It can hollow the hoodie. Restore the untouched mesh first and use a manually verified local edit only when a clean boundary exists.

## Pitfalls

- Converting RGB to RGBA does not remove its background. A generated image requested as transparent may still be opaque. Inspect alpha and cutout previews, not the prompt alone.
- Do not crop a three-view sheet into equal thirds without checking each silhouette. Hands and ears may cross those boundaries.
- A SAM 3D Body mesh (`HumanMesh`, 18439 verts) is a nude parametric body. Hunyuan3D-Paint on that mesh still will not grow hair or a dress; it only paints existing surfaces. Tell the user this before offering Paint. Blender vertex-color / planar projection of a turnaround onto that mesh is not Paint and was rejected as a smear; restore grey if asked.
- Paint runs on host `10.145.119.19` and requires sending the mesh plus reference images. Do not upload until the user authorizes that host.
- GPU utilization at zero means neither success nor failure; GPU memory allocated means neither active inference nor completion. Use process, stage log, exit status and output evidence together.
- Deleting by a broad bounding box previously removed legs while leaving the bag. Inspect local geometry and coordinate transforms before any targeted edit; see the quality reference.
- GLB import success, embedded texture presence, and saving as BLEND are structural checks, not proof that the face or texture improved.

## Verified 2.1 run record

- Host: `hch@10.145.119.19`; GPUs: two RTX 4060 Ti 16GB; host venv: `/home/hch/venvs/hy3d`; local source: `/home/hch/Hunyuan3D-2.1`.
- Shape weights: Hunyuan3D 2.1 files under `/home/hch/ComfyUI/models/hy3d/hunyuan3d-2.1/`; Multi-View Shape weights/config under `/home/hch/ComfyUI/models/hy3d/mvroot/` and `/home/hch/ComfyUI/models/checkpoints/`.
- Verified Multi-View Shape output: `/home/hch/hy3d-jobs/hunyuan21-turnaround/hunyuan21_turnaround_multiview20_shape.glb`, 146,678 vertices, 293,412 faces.
- Verified Paint output was converted to PBR GLB at `/home/hch/hy3d-jobs/hunyuan21-turnaround/paint_v2/hunyuan21_turnaround_multiview20_painted.glb`; the local copied revision had SHA-256 `25da3570d3596d294b33a46bdd5f6852f5db61f6f059692d216a47d0c3609768`.
- The final visual check showed a valid textured GLB, but the face was still not identical to the original photo and the result remained a first usable PBR pass, not a finished identity-preserving reconstruction.

## Verification

Keep the original `.glb`, save an imported `.blend`, and provide a preview image. Open the `.blend` in Blender only after the inspection passes. For Paint results, confirm embedded textures in the GLB (not flat grey) before delivery.

Reopen the saved BLEND in a separate Blender process and render it with packed textures. Check front/side/back and a face close-up for refinement tasks. Do not mistake Solid viewport shading for missing materials, or tell the user to import a GLB when they requested a directly openable BLEND. Report residual defects explicitly.
