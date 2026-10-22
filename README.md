# TRELLIS.2 on an 8 GB GPU: Install and Test Guide

Target: Windows 10/11, NVIDIA RTX 50 series (Blackwell), 8 GB VRAM.
Recommended route: **trellis.cpp + q4 GGUF weights**. Alternative: ComfyUI-Trellis2.

> **Note:** 8 GB is not an officially tested setup. Microsoft's official pipeline needs 24 GB, and trellis.cpp only documents the 1024 cascade on a 16 GB card.
> The q4 weight set is about 6.5 GB, split into one file per stage. The largest pipeline stage (`shape_dec` / `tex_dec`) is 0.85 GB and the largest file overall is `birefnet.gguf` at 0.88 GB, so each stage likely fits. For comparison, the full-precision flow files are 2.59 GB each. No published 8 GB benchmark exists yet, so step 5 below is your real test.

---

## Route A: trellis.cpp (recommended)

### 1. Check the driver

Install the latest NVIDIA Game Ready or Studio driver, then run:

```powershell
nvidia-smi
```

Expected: your GPU listed, about 8 GB memory, CUDA Version 12.8 or higher (the minimum for Blackwell).

### 2. Install trellis.cpp

Pick one:

**Option 1: installer (CLI + Trellis Studio desktop app)**

PowerShell (not admin):

```powershell
irm https://raw.githubusercontent.com/pwilkin/trellis.cpp/main/install/install.ps1 | iex
```

- Detects your GPU runtime (CUDA / ROCm / Vulkan) and picks the matching build
- Also installs **Trellis Studio**, a desktop app, if you prefer a UI over the CLI
- Downloads the full-precision weights (about 16.5 GB). The README does not say this step can be skipped. After step 3, point Studio at the q4 folder in **Settings > models directory**, or pass `--models` on the CLI
- Note the folder where `trellis-cli.exe` is installed

**Option 2: CLI only (no 16.5 GB download)**

Download the Windows CUDA build from the [Releases page](https://github.com/pwilkin/trellis.cpp/releases) and unzip it to `C:\trellis`. The `cuda12` variant is for Pascal/Volta cards only, so don't pick it for an RTX 50.

### 3. Download only the q4 weights (about 6.5 GB)

```powershell
pip install -U "huggingface_hub[cli]"
hf download ilintar/trellis2-gguf --include "q4/*" --local-dir C:\trellis\models
```

Files in `C:\trellis\models\q4`:

| File | Size |
|---|---|
| dinov3.gguf | 0.17 GB |
| ss_flow.gguf | 0.73 GB |
| ss_dec.gguf | 0.15 GB |
| shape_flow_512.gguf | 0.73 GB |
| shape_flow_1024.gguf | 0.73 GB |
| shape_dec.gguf | 0.85 GB |
| tex_flow_512.gguf | 0.73 GB |
| tex_flow_1024.gguf | 0.73 GB |
| tex_dec.gguf | 0.85 GB |
| birefnet.gguf (background removal, not quantized) | 0.88 GB |
| **Total** | **6.55 GB** |

### 4. Prepare a test image

- One object, centered
- Plain or transparent background
- About 1024 x 1024 px
- A PNG with transparency is best: it skips background removal and saves VRAM

Save it as `C:\trellis\test.png`.

### 5. First test (lightest settings)

```powershell
cd C:\trellis
mkdir out
trellis-cli.exe test.png out\test_512.glb --models models\q4 --res 512 --faces 100000 --atlas 1024 --seed 42 --require-gpu --verbose
```

`--require-gpu` makes the run fail instead of falling back to the very slow, RAM-hungry CPU path, so you know the GPU actually did the work.

Watch VRAM in a second window:

```powershell
nvidia-smi -l 1
```

### 6. Check the result

Open `out\test_512.glb` in any of:

- Windows 3D Viewer
- https://gltf-viewer.donmccurdy.com
- Blender (File > Import > glTF 2.0)

### 7. Step up, one change at a time

| Test | Change | Effect |
|---|---|---|
| More detail | `--steps 20` | Same VRAM, slower |
| Higher resolution | `--res 1024 --faces 200000 --atlas 2048` | May fit with q4. If it runs out of memory, go back to 512 |
| Geometry only | `--no-texture` | Fallback if texture baking runs out of memory |
| Save VRAM | `--bg-removal threshold` | Skips loading BiRefNet (0.88 GB) |
| Faster UV unwrap | `--box-uv` | Box projection instead of xatlas. Faster, looser texture packing |

### CLI flags reference

Run `trellis-cli.exe --help` for the full list.

| Flag | Meaning | Default |
|---|---|---|
| `--res 512\|1024\|1536` | Geometry resolution (512 = light path, no cascade) | 1024 |
| `--steps N` | Flow sampler steps | 12 |
| `--faces N` | QEM decimation face target (min 1000) | 300K at 1024/1536, 150K at 512 |
| `--decim GRID` | Legacy grid decimation (`0` = full mesh). Can't combine with `--faces` | |
| `--atlas PX` | UV texture atlas size | 2048 at 1024, 1024 at 512 |
| `--box-uv` | Box-projection UVs instead of xatlas | off |
| `--seed N` | Random seed | random |
| `--bg-removal threshold\|birefnet` | Background removal method | auto (keeps alpha if present, else BiRefNet) |
| `--models DIR` | Folder with GGUF weights | |
| `--model trellis\|pixal3d` | Which weight family `--models` holds | trellis |
| `--no-texture` | Skip texture generation | |
| `--backend NAME` | Force a backend (e.g. `Vulkan`, `CPU`) | auto |
| `--require-gpu` | Fail instead of falling back to CPU | off |
| `--threads N` | CPU threads | all hardware threads |
| `--verbose` | Graph timings and progress logs | |

---

## If you run out of memory

1. Close Chrome and other GPU apps (WebGL tabs use VRAM)
2. Add `--bg-removal threshold`, or use a PNG with transparency
3. Add `--no-texture` and texture the model somewhere else
4. Last resort: `--backend Vulkan` (slower, handles memory differently)

---

## Route B: ComfyUI-Trellis2 (more setup)

Use this only if you need to chain TRELLIS.2 with other ComfyUI nodes.

1. Install **ComfyUI Portable** with Python 3.11 and Torch 2.7.0 cu128. This is the author's only tested setup, and Blackwell needs cu128 or later. Torch 2.8.0 wheels also ship in `Torch280\`, but they are untested
2. Clone the node into `ComfyUI\custom_nodes`:

   ```powershell
   git clone https://github.com/visualbruno/ComfyUI-Trellis2
   ```

3. Install the wheels from `custom_nodes\ComfyUI-Trellis2\wheels\Windows\Torch270\`, in this order:
   1. cumesh
   2. nvdiffrast
   3. nvdiffrec_render
   4. flex_gemm
   5. o_voxel

   ```powershell
   python_embeded\python.exe -m pip install <wheel>.whl
   ```

4. Install the requirements:

   ```powershell
   python_embeded\python.exe -m pip install -r ComfyUI\custom_nodes\ComfyUI-Trellis2\requirements.txt
   ```

5. Request access to the gated model `facebook/dinov3-vitl16-pretrain-lvd1689m` on Hugging Face, then clone it into:

   ```
   ComfyUI\models\facebook\dinov3-vitl16-pretrain-lvd1689m
   ```

6. Load a workflow from `example_workflows`. For 8 GB, start with the settings below. The node supports all of them, but the author publishes no VRAM figures, so these are a starting point rather than documented guidance:
   - FP8 model
   - `sdpa` attention backend
   - Tiled decoder
   - Resolution 512

---

## What to expect

- Speed: the trellis.cpp benchmark on an RTX 5060 Ti (CUDA, 1024, 12 steps, including model load) is **3:16** for a light input and **7:23** for a heavy one. At 512 on 8 GB, expect a few minutes per model
- Quality: 512 is fine for previews and game props. Use the Hugging Face Space or a rented GPU for 1536

## Sources

- trellis.cpp: https://github.com/pwilkin/trellis.cpp
- trellis.cpp wiki: https://github.com/pwilkin/trellis.cpp/wiki
- GGUF weights: https://huggingface.co/ilintar/trellis2-gguf
- ComfyUI-Trellis2: https://github.com/visualbruno/ComfyUI-Trellis2
- TRELLIS 2 ComfyUI guide: https://trellis2.app/blog/trellis-2-comfyui
- Official model card: https://huggingface.co/microsoft/TRELLIS.2-4B
