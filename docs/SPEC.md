# HollowNest ComfyUI Workflow: specification

**Status:** v1.0, 2026-10-12. First written spec, reverse-engineered from the code as it stands.
**Owner:** Laila Harrison (KSU Office of Research). **Reviewer:** unassigned.
**Source of requirements:** the four workflow JSON files at the repo root and the README.md that shipped with them. No backend code is in this repo.

This document is the contract for the code. Every change starts here: update the spec, then the code, then the tests, in one pull request. Each requirement has an ID and says how it is verified. **Auto** means a test checks it on every pull request. **Manual** means a reviewer checks it by hand before approving. **None** means nothing checks it today.

## 1. Purpose

HollowNest is a 3D asset library for Kennesaw State University researchers who need models of their work for exhibits and augmented reality but have never used 3D software. These graphs are the generation step behind its Generate page. They split the job in two. First a cheap text-to-image sweep gives the user four images to choose from. Then the expensive image-to-3D step runs only on the one they picked. "Done" is a textured GLB in the ComfyUI output directory that the site can serve.

## 2. Scope

**In scope:** the ComfyUI graphs themselves. One text-to-image graph, two image-to-3D graphs, and one laid-out copy of the main image-to-3D graph for inspection in the ComfyUI canvas. The node settings, model references, and output paths they contain.

**Out of scope:** the HollowNest website and its Python backend, which build and submit these graphs. ComfyUI installation. Model weight downloads. Any quality check on the output model. Any record of which branch produced a given asset. User-editable style suffixes. The Hunyuan3D branch is not a second option offered to the user. It is only a substitute when TRELLIS.2 is missing.

## 3. Requirements

### T. Text to reference image, `01_text_to_image_zimage_seed_sweep.json`

| ID | Requirement | Verify |
|---|---|---|
| T1 | The graph loads `z_image_turbo_bf16.safetensors` through `UNETLoader`, `qwen_3_4b.safetensors` through `CLIPLoader` with type `lumina2`, and `ae.safetensors` through `VAELoader`. | Manual: open the JSON and read nodes 10, 10b, 10c. |
| T2 | The prompt sent to `TextEncodeZImageOmni` is the user text followed by the fixed style suffix: "Clean white background, product photography, single object centered, studio lighting, high detail, 3d render, museum exhibit, stylized, no text". | Manual: read node 11. |
| T3 | The latent is 1024 by 1024 with batch size 1. | Manual: read node 13. |
| T4 | `KSampler` runs 8 steps at CFG 1.0 with sampler `euler`, scheduler `simple`, denoise 1.0. | Manual: read node 14. |
| T5 | The same encoded prompt is wired to both the positive and negative inputs of `KSampler`. There is no separate negative prompt. | Manual: read node 14 links. |
| T6 | The seed is a fixed integer. The backend runs the graph once per seed, 42 through 45, so the same prompt always produces the same four images. | None. The repo ships one copy at seed 44. |
| T7 | The decoded image is saved with prefix `refs/<prompt-slug>_seed<N>` under the ComfyUI output directory. | Manual: read node 16. |

### R. Image to 3D, main branch, `02_image_to_3d_trellis2.json`

| ID | Requirement | Verify |
|---|---|---|
| R1 | The graph takes one image file from the ComfyUI input directory through `LoadImage`. The shipped name is `picked_seed.png`. | Manual: read node 15. |
| R2 | Background removal uses the ComfyUI_essentials `RemBGSession+` node with model `u2net: general purpose` on the CUDA provider, feeding `ImageRemoveBackground+`. The built-in TRELLIS.2 remover is not used. | Manual: read nodes 51a and 51b. |
| R3 | `LoadTrellis2Models` loads at resolution `1024_cascade` with precision `auto` and attention backend `auto`. | Manual: read node 50. |
| R4 | `Trellis2GetConditioning` receives the cutout image and its mask and uses background color `black`. | Manual: read node 52. |
| R5 | `Trellis2ImageToShape` runs sparse structure and shape stages at 12 steps each with guidance 7.5, seed 42, and `max_tokens` 49152. | Manual: read node 53. |
| R6 | `Trellis2ShapeToTexturedMesh` runs the texture stage at 12 steps with guidance 7.5 and seed 42. | Manual: read node 54. |
| R7 | `Trellis2ExportGLB` reads the voxel grid path, decimates to a 200,000 face target, bakes 2048 px textures, enables remesh, and writes with prefix `3D/hollownest_job`. | Manual: read node 55. |
| R8 | The output is a GLB with PBR textures. Face count is at or below 200,000. | Manual: open the GLB in Blender or a web viewer and check the face count. |

### H. Image to 3D, fallback branch, `03_image_to_3d_hunyuan3d_fallback.json`

| ID | Requirement | Verify |
|---|---|---|
| H1 | The graph takes the same `LoadImage` input as R1, shipped as `picked_seed.png`. | Manual: read node 15. |
| H2 | The image is padded to 518 by 518 with lanczos interpolation, method `pad`, condition `always`, multiple of 2. | Manual: read node 52. |
| H3 | Background removal uses `TransparentBGSession+` in `base` mode with JIT on, feeding `ImageRemoveBackground+`. | Manual: read nodes 55 and 17. |
| H4 | The cutout is composited onto a flat 512 by 512 canvas built from a `SolidMask` of value 0.8, at offset 0, 0, without resizing the source. | Manual: read nodes 60, 61, 64. |
| H5 | `Hy3DModelLoader` loads `hy3dgen/hunyuan3d-dit-v2-0-mv-fast-fp16.safetensors` with attention mode `sdpa` and cuBLAS ops off. | Manual: read node 19. |
| H6 | `Hy3DGenerateMesh` runs 50 steps at guidance 5.5, seed 42. `Hy3DVAEDecode` uses box 1.01, octree resolution 512, 8000 chunks, marching cubes level 0.0, algorithm `mc`. | Manual: read nodes 20 and 20b. |
| H7 | `Hy3DPostprocessMesh` removes floaters and degenerate faces, reduces to at most 50,000 faces, and does not smooth normals. The mesh is then UV wrapped. | Manual: read nodes 21 and 22. |
| H8 | `Hy3DDelightImage` uses model `hunyuan3d-delight-v2-0` at 50 steps, 512 by 512, image CFG 1.0, seed 42, on the composited image. | Manual: read nodes 30 and 31. |
| H9 | `Hy3DCameraConfig` defines six cameras at azimuths 0, 90, 180, 270, 0, 180 and elevations 0, 0, 0, 0, 90, -90 with view weights 1, 0.1, 0.5, 0.1, 0.05, 0.05, distance 1.45, ortho scale 1.2. | Manual: read node 32. |
| H10 | `Hy3DRenderMultiView` renders at 1024 px with a 2048 px texture in world normal space. `Hy3DSampleMultiView` uses `hunyuan3d-paint-v2-0` at view size 512, 25 steps, seed 42. | Manual: read nodes 33, 34, 35. |
| H11 | The texture is baked from the multiview images, vertex inpainted, applied to the mesh, converted with `TrimeshToMESH`, and written by `SaveGLB` with prefix `3D/hollownest_job`. | Manual: read nodes 36 through 40. |
| H12 | The output is a GLB with a 2048 px texture and at most 50,000 faces. | Manual: open the GLB and check the face count. |

### B. Branch selection

| ID | Requirement | Verify |
|---|---|---|
| B1 | TRELLIS.2 is the default image-to-3D branch. | None. The rule lives in the backend, not this repo. |
| B2 | The Hunyuan3D graph runs only when the TRELLIS.2 nodes are not installed on the ComfyUI server. | None. The backend checks installed nodes. |
| B3 | Both branches read the same input filename and write to the same output prefix, so the site does not need to know which one ran. | Manual: compare `LoadImage` and the save node in graphs 02 and 03. |

### L. Laid-out graph, `hollownest-trellis2-workflow.json`

| ID | Requirement | Verify |
|---|---|---|
| L1 | The file is ComfyUI native workflow format version 0.4 and loads onto the canvas with three groups: Inputs, Generate, and Export and preview. | Manual: drag the file into ComfyUI. |
| L2 | The TRELLIS.2 settings match graph 02: `1024_cascade`, 12 steps per stage, guidance 7.5, seed 42 fixed, 200,000 faces, 2048 px textures, remesh on. | Manual: compare widget values on nodes 68, 73, 74, 75 against graph 02. |
| L3 | The mask comes from the loaded image's alpha channel through `InvertMask`, not from a rembg node. The input image must already be cut out. | Manual: read nodes 1 and 11. |
| L4 | The graph ends in a `Preview3D` node fed by the exported GLB path, and writes with prefix `3D/Hollownest`. | Manual: read nodes 64 and 74. |

### D. Data and files

| ID | Requirement | Verify |
|---|---|---|
| D1 | No graph contains credentials, API keys, or absolute paths outside the ComfyUI input and output directories. | Manual: grep the JSON files. |
| D2 | Reference images land under `refs/` and 3D models under `3D/` in the ComfyUI output directory. | Manual: read the save nodes. |
| D3 | Every sampler seed is fixed in the file so a run can be reproduced from the JSON alone. | Manual: confirm no seed is wired to a random or increment control. |

## 4. Interfaces

### Files

| File | Format | Role |
|---|---|---|
| `01_text_to_image_zimage_seed_sweep.json` | ComfyUI API prompt, 8 nodes | Text to image |
| `02_image_to_3d_trellis2.json` | ComfyUI API prompt, 8 nodes | Image to 3D, main |
| `03_image_to_3d_hunyuan3d_fallback.json` | ComfyUI API prompt, 23 nodes | Image to 3D, fallback |
| `hollownest-trellis2-workflow.json` | ComfyUI workflow v0.4, 9 nodes, 3 groups | Canvas copy of the main branch |

### Inputs the backend rewrites per job

| Graph | Node id | Input | Shipped value |
|---|---|---|---|
| 01 | 11 | `prompt` | "an orange shoe." plus the style suffix |
| 01 | 14 | `seed` | 44 |
| 01 | 16 | `filename_prefix` | `refs/an-orange-shoe_seed44` |
| 02 | 15 | `image` | `picked_seed.png` |
| 02 | 55 | `filename_prefix` | `3D/hollownest_job` |
| 03 | 15 | `image` | `picked_seed.png` |
| 03 | 40 | `filename_prefix` | `3D/hollownest_job` |

### Outputs

| Graph | Node | Output | Location |
|---|---|---|---|
| 01 | `SaveImage` | PNG, 1024 by 1024 | `output/refs/<prompt-slug>_seed<N>` |
| 02 | `Trellis2ExportGLB` | GLB, up to 200,000 faces, 2048 px PBR textures | `output/3D/hollownest_job` |
| 03 | `SaveGLB` | GLB, up to 50,000 faces, 2048 px texture | `output/3D/hollownest_job` |
| 04 | `Trellis2ExportGLB` plus `Preview3D` | GLB, same limits as graph 02 | `output/3D/Hollownest` |

### Models referenced

| Model | Graph | Loader |
|---|---|---|
| `z_image_turbo_bf16.safetensors` | 01 | `UNETLoader` |
| `qwen_3_4b.safetensors` | 01 | `CLIPLoader`, type `lumina2` |
| `ae.safetensors` | 01 | `VAELoader` |
| TRELLIS.2, `1024_cascade` | 02, 04 | `LoadTrellis2Models` |
| rembg `u2net: general purpose` | 02 | `RemBGSession+` |
| `hy3dgen/hunyuan3d-dit-v2-0-mv-fast-fp16.safetensors` | 03 | `Hy3DModelLoader` |
| `hunyuan3d-delight-v2-0` | 03 | `DownloadAndLoadHy3DDelightModel` |
| `hunyuan3d-paint-v2-0` | 03 | `DownloadAndLoadHy3DPaintModel` |
| TransparentBG, `base` | 03 | `TransparentBGSession+` |

### Node packs

| Pack | Used by | Pinned version |
|---|---|---|
| ComfyUI-TRELLIS2 | 02, 04 | Graph 04 records three different commit hashes across its TRELLIS.2 nodes: `aa154e8`, `64864b1`, `96c6da1`. |
| ComfyUI_essentials | 02, 03 | Not pinned. |
| ComfyUI-Hunyuan3DWrapper | 03 | Not pinned. |
| comfy-core | 04 | 0.5.0 and 0.5.1, frontend 1.38.13. |

There are no environment variables, CLI flags, or HTTP routes in this repo. The graphs are posted to the standard ComfyUI `/prompt` endpoint by the backend.

## 5. Open decisions

1. **No quality check before saving.** One test object, a stone, came out with a strange shiny texture and still got saved. Nothing in the graphs or backend rejects a bad model.
2. **No record of which branch made each model.** The asset record should store it. Today the only tell is the face count, 200,000 cap for TRELLIS.2 against 50,000 for Hunyuan3D.
3. **Two copies of the graph have drifted.** The site and the pipeline repo each hold their own copy. This repo is a third. Which one is canonical is undecided.
4. **The style suffix is fixed.** "Museum exhibit, stylized" works for research props but not for every object. Letting users change it is not built.
5. **Output prefix mismatch.** The laid-out graph writes to `3D/Hollownest`. The API graphs write to `3D/hollownest_job`. Unknown whether this is intended.
6. **Fallback canvas size mismatch.** Graph 03 pads the image to 518 px, then composites it onto a 512 px canvas with `resize_source` off. Three pixels on each edge are cropped. Unknown whether this is intended.
7. **No negative prompt.** Graph 01 wires the same conditioning to both positive and negative. At CFG 1.0 the negative has no effect, so this is harmless today, but it would matter if CFG is raised.
8. **Mixed TRELLIS.2 pins.** The laid-out graph records three different commit hashes for the TRELLIS.2 node pack. The DGX install should be pinned to one.
9. **No test or CI.** Every requirement is verified by hand. A JSON schema check and a node-presence check against a ComfyUI `/object_info` dump would turn most of section 3 into Auto.
10. **Backend not in repo.** Requirements B1, B2, and T6 describe behavior that lives in the HollowNest site backend, which is not here. They cannot be verified from this repo alone.

## 6. How changes ship

1. Open a branch. If behavior changes, update this spec first, in the same branch.
2. Open a pull request into `main`.
3. No CI today. Load each changed graph into ComfyUI on the DGX Spark and run it end to end before asking for review. For the API graphs, confirm the output file lands at the expected prefix.
4. One approving review before merge.

## 7. Review log

None yet.
