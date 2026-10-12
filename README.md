# HollowNest ComfyUI Workflow

HollowNest is a 3D asset library for researchers at Kennesaw State University. Its Generate page turns a text prompt or a photo into a textured 3D model. This repo holds the ComfyUI graphs that run behind that page on the Office of Research NVIDIA DGX Spark.

**Status:** Active, as of 2026-10-12. The graphs are in use by the live site and still being tuned.
**Owner:** Laila Harrison, KSU Office of Research.

**Live site:** https://hollownest-one.vercel.app/generate

![One prompt, four seeds, one pick, one 3D model](seed_to_3d.png)

## What it does

- **Text to reference image.** Z-Image Turbo turns a prompt into a product-style reference image. The site runs it at seeds 42 to 45 so the user gets four options.
- **Image to 3D, main branch.** TRELLIS.2 turns the picked image into a mesh with physically based rendering textures. PBR is the texture standard most 3D viewers and AR apps expect.
- **Image to 3D, fallback branch.** Hunyuan3D runs only when TRELLIS.2 is not installed. It cleans the image, builds the mesh, removes baked lighting, and bakes a texture from six cameras.
- **Hardware-aware routing.** The backend checks which nodes are installed on the DGX and picks the 3D branch to match.
- **Repeatable output.** Every graph uses fixed seeds, so the same prompt gives the same four images every time.
- **A fixed house style.** Every prompt gets the same style suffix: white background, studio lighting, single centered object. The 3D step always gets clean input.

## Quick start

These are graph files, not a program. You need a running ComfyUI with the three node packs and the model files below.

```bash
git clone https://github.com/lailajtharrison/hollownest-comfyui-workflow.git

# Install the node packs into your ComfyUI custom_nodes directory.
cd /path/to/ComfyUI/custom_nodes
git clone https://github.com/PozzettiAndrea/ComfyUI-TRELLIS2
git clone https://github.com/cubiq/ComfyUI_essentials
git clone https://github.com/kijai/ComfyUI-Hunyuan3DWrapper   # fallback branch only

# Inspect a graph: drag hollownest-trellis2-workflow.json onto the ComfyUI canvas.

# Run an API graph against a local ComfyUI server.
cd /path/to/hollownest-comfyui-workflow
python3 -c 'import json,urllib.request as u; g=json.load(open("01_text_to_image_zimage_seed_sweep.json")); \
  u.urlopen(u.Request("http://127.0.0.1:8188/prompt", json.dumps({"prompt": g}).encode(), {"Content-Type": "application/json"}))'
```

**Model files the graphs expect** in the ComfyUI models directory:

| Graph | File or model id | Loader node |
|---|---|---|
| Z-Image | `z_image_turbo_bf16.safetensors` | `UNETLoader` |
| Z-Image | `qwen_3_4b.safetensors`, type `lumina2` | `CLIPLoader` |
| Z-Image | `ae.safetensors` | `VAELoader` |
| TRELLIS.2 | TRELLIS.2 weights at `1024_cascade` resolution | `LoadTrellis2Models` |
| TRELLIS.2 | rembg `u2net: general purpose` on CUDA | `RemBGSession+` |
| Hunyuan3D | `hy3dgen/hunyuan3d-dit-v2-0-mv-fast-fp16.safetensors` | `Hy3DModelLoader` |
| Hunyuan3D | `hunyuan3d-delight-v2-0` | `DownloadAndLoadHy3DDelightModel` |
| Hunyuan3D | `hunyuan3d-paint-v2-0` | `DownloadAndLoadHy3DPaintModel` |

The exact ComfyUI version is not pinned. The laid-out graph was saved from ComfyUI frontend 1.38.13 with comfy-core 0.5.1 nodes.

## Repository layout

| File | Nodes | What it does |
|---|---|---|
| `hollownest-trellis2-workflow.json` | 9 | **Start here.** The TRELLIS.2 graph in ComfyUI's native format, grouped into Inputs, Generate, and Export and preview. |
| `01_text_to_image_zimage_seed_sweep.json` | 8 | API graph. Z-Image Turbo, prompt to one 1024 by 1024 reference image at one seed. |
| `02_image_to_3d_trellis2.json` | 8 | API graph. Background removal plus TRELLIS.2, picked image to a textured GLB. This is the branch that made the model above. |
| `03_image_to_3d_hunyuan3d_fallback.json` | 23 | API graph. Hunyuan3D mesh, delight, six-camera multiview paint, and texture bake. |
| `seed_to_3d.png` | | Figure: one prompt, four seeds, one pick, one 3D model. |
| `seed_sweep.png` | | Figure: the four-seed sweep for three prompts. |
| `screenshot.png` | | Figure: the laid-out TRELLIS.2 graph on the ComfyUI canvas, with an anatomical heart. |
| `docs/SPEC.md` | | Requirements and how changes ship. |

The numbered files are the API versions the backend sends. They load into ComfyUI too, but ComfyUI lays them out automatically.

## How it works

The HollowNest backend builds each graph in Python and posts it to the ComfyUI API. That backend lives with the site, not in this repo.

- **Step 1, reference image.** `01_text_to_image_zimage_seed_sweep.json` loads `z_image_turbo_bf16`, encodes the prompt with `TextEncodeZImageOmni`, and samples a 1024 by 1024 latent with `KSampler` at 8 steps, CFG 1.0, euler, simple scheduler. The site runs this graph once per seed, 42 through 45.
- **Step 2, the user picks one.** The four images land under `refs/<prompt-slug>_seed<N>` in the ComfyUI output directory. Only the picked one goes to 3D.
- **Step 3, main branch.** `02_image_to_3d_trellis2.json` loads `picked_seed.png`, removes the background with the `u2net` rembg session, and feeds image plus mask into `Trellis2GetConditioning` on a black background. `Trellis2ImageToShape` and `Trellis2ShapeToTexturedMesh` run 12 steps per stage at guidance 7.5, seed 42. `Trellis2ExportGLB` decimates to 200,000 faces, bakes 2048 px textures, remeshes, and writes to `3D/hollownest_job`.
- **Step 3, fallback branch.** `03_image_to_3d_hunyuan3d_fallback.json` pads the image to 518 px, cuts it out with `TransparentBGSession+`, and composites it onto a flat grey canvas. `Hy3DGenerateMesh` runs 50 steps at guidance 5.5, the VAE decodes at octree 512, and post-processing caps the mesh at 50,000 faces. `Hy3DDelightImage` strips lighting, six cameras render normals and positions at 1024 px, `Hy3DSampleMultiView` paints 25 steps per view, and the bake plus vertex inpaint produce a 2048 px texture. `SaveGLB` writes to `3D/hollownest_job`.
- **Which branch runs.** TRELLIS.2 is the default. Hunyuan3D runs only if the TRELLIS.2 nodes are missing on the server.
- **The laid-out graph.** `hollownest-trellis2-workflow.json` uses the same TRELLIS.2 settings as graph 02. Its input image is already cut out, so the mask comes from the alpha channel through `InvertMask`. The site version adds the rembg nodes instead, because uploaded photos still have backgrounds. It ends in a `Preview3D` node and writes to `3D/Hollownest`.

![The TRELLIS.2 graph in ComfyUI](screenshot.png)

**Why rembg and not the built-in remover.** TRELLIS.2's own background remover threw data type errors on the DGX's ARM chip. The graphs use the ComfyUI_essentials rembg nodes instead.

**A real run from the live site**, shown in the figure at the top:

1. **Prompt:** "an orange shoe."
2. **Four seeds:** two lace-up shoes, an ankle bootie, and a pull-tab rain boot.
3. **The pick:** seed 44, the rain boot.
4. **The 3D model:** HollowNest job `3f100f63a2e0`, 188,518 faces, 2048 px PBR textures, rendered in Blender from three angles.

`seed_sweep.png` shows the same four-seed sweep for three prompts. The fire hydrant stays consistent across seeds. The shoe changes completely.

## Configuration

There are no environment variables and no config files. Everything is a node input inside the JSON. These are the inputs the backend rewrites per job.

| Graph | Node | Input | Required? | What it does |
|---|---|---|---|---|
| 01 | `TextEncodeZImageOmni` | `prompt` | Yes | User prompt plus the fixed style suffix. |
| 01 | `KSampler` | `seed` | Yes | One of 42, 43, 44, 45 on the site. The file ships with 44. |
| 01 | `SaveImage` | `filename_prefix` | Yes | `refs/<prompt-slug>_seed<N>`. |
| 02, 03 | `LoadImage` | `image` | Yes | Filename of the picked image in the ComfyUI input directory. Ships as `picked_seed.png`. |
| 02 | `Trellis2ExportGLB` | `filename_prefix` | Yes | Where the GLB lands. Ships as `3D/hollownest_job`. |
| 03 | `SaveGLB` | `filename_prefix` | Yes | Same prefix as graph 02. |
| 02 | `Trellis2ImageToShape`, `Trellis2ShapeToTexturedMesh` | `seed` | No | Fixed at 42. Change for a different roll of the same image. |
| 03 | `Hy3DGenerateMesh`, `Hy3DDelightImage`, `Hy3DSampleMultiView` | `seed` | No | Fixed at 42. |

Node pack dependencies:

- **[ComfyUI-TRELLIS2](https://github.com/PozzettiAndrea/ComfyUI-TRELLIS2)** for the main 3D graph.
- **[ComfyUI_essentials](https://github.com/cubiq/ComfyUI_essentials)** for background removal and resizing.
- **[ComfyUI-Hunyuan3DWrapper](https://github.com/kijai/ComfyUI-Hunyuan3DWrapper)** for the fallback graph.

## Deploy / distribute

The graphs run inside ComfyUI on the Office of Research NVIDIA DGX Spark. There is no build or deploy step for this repo. Copy the JSON files to wherever the backend reads them, or load them in the ComfyUI browser UI. The live site at the link above is a separate Vercel deployment and is not in this repo.

## Related

- **Live site:** https://hollownest-one.vercel.app/generate
- **ComfyUI-TRELLIS2:** https://github.com/PozzettiAndrea/ComfyUI-TRELLIS2
- **ComfyUI_essentials:** https://github.com/cubiq/ComfyUI_essentials
- **ComfyUI-Hunyuan3DWrapper:** https://github.com/kijai/ComfyUI-Hunyuan3DWrapper

## Spec

See [docs/SPEC.md](docs/SPEC.md) for requirements and how changes ship.
