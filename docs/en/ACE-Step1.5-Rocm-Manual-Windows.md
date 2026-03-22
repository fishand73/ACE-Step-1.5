# ACE-Step 1.5 – AMD ROCm Setup Manual for Windows

Tested configuration: Windows 11, AMD ROCm 7.2, Python 3.12, RX 7000 / RX 6000 series.

> For Linux ROCm setup, see [ACE-Step1.5-Rocm-Manual-Linux.md](ACE-Step1.5-Rocm-Manual-Linux.md).

---

## Requirements

| Component | Requirement |
|-----------|-------------|
| OS | Windows 11 (22H2 or later recommended) |
| Python | **3.12 only** — AMD officially provides ROCm wheels for Python 3.12 on Windows |
| GPU | AMD RX 7000 series (RDNA3) or RX 6000 series (RDNA2) |
| AMD Driver | 26.1.1 or later |
| Disk space | ~15 GB (ROCm SDK + models) |
| Git | Any recent version |

---

## Step 1 – Verify Python 3.12

```cmd
python --version
```

Expected output: `Python 3.12.x`. If you have a different version, download Python 3.12 from
<https://www.python.org/downloads/> and install it.

---

## Step 2 – Clone the Repository

```cmd
git clone https://github.com/ACE-Step/ACE-Step-1.5.git
cd ACE-Step-1.5
```

---

## Step 3 – Create a Dedicated Virtual Environment

Using a separate `venv_rocm` environment prevents conflicts with CUDA PyTorch wheels that
`uv sync` would otherwise install.

```cmd
python -m venv venv_rocm
venv_rocm\Scripts\activate
```

Verify the active Python version:

```cmd
python --version
```

---

## Step 4 – Install ROCm SDK and PyTorch

Follow the instructions inside `requirements-rocm.txt` (shown below for reference).

### 4a. Install the ROCm SDK wheels

```cmd
pip install --no-cache-dir ^
  https://repo.radeon.com/rocm/windows/rocm-rel-7.2/rocm_sdk_core-7.2.0.dev0-py3-none-win_amd64.whl ^
  https://repo.radeon.com/rocm/windows/rocm-rel-7.2/rocm_sdk_devel-7.2.0.dev0-py3-none-win_amd64.whl ^
  https://repo.radeon.com/rocm/windows/rocm-rel-7.2/rocm_sdk_libraries_custom-7.2.0.dev0-py3-none-win_amd64.whl ^
  https://repo.radeon.com/rocm/windows/rocm-rel-7.2/rocm-7.2.0.dev0.tar.gz
```

### 4b. Install PyTorch for ROCm

```cmd
pip install --no-cache-dir ^
  https://repo.radeon.com/rocm/windows/rocm-rel-7.2/torch-2.9.1+rocmsdk20260116-cp312-cp312-win_amd64.whl ^
  https://repo.radeon.com/rocm/windows/rocm-rel-7.2/torchaudio-2.9.1+rocmsdk20260116-cp312-cp312-win_amd64.whl ^
  https://repo.radeon.com/rocm/windows/rocm-rel-7.2/torchvision-0.24.1+rocmsdk20260116-cp312-cp312-win_amd64.whl
```

### 4c. Install ACE-Step dependencies

```cmd
pip install -r requirements-rocm.txt
```

> **Note:** `torchao` and `torchcodec` are excluded from `requirements-rocm.txt` because they
> require CUDA. ACE-Step automatically uses `soundfile` for audio I/O on ROCm, providing full
> functionality without `torchcodec`.

---

## Step 5 – Verify GPU Detection

```cmd
python -c "import torch; print('CUDA available:', torch.cuda.is_available()); print('GPU:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'None'); print('HIP version:', getattr(torch.version, 'hip', 'N/A'))"
```

Expected output (example for RX 7900 XTX):

```
CUDA available: True
GPU: AMD Radeon RX 7900 XTX
HIP version: 6.2.41133-dd7f95766
```

If `CUDA available: False`, see the [Troubleshooting](#troubleshooting) section below.

---

## Step 6 – Launch ACE-Step

Use the provided ROCm launcher, which automatically sets all required environment variables:

```cmd
start_gradio_ui_rocm.bat
```

Or for the REST API server:

```cmd
start_api_server_rocm.bat
```

The launcher sets these environment variables automatically:

| Variable | Value | Purpose |
|----------|-------|---------|
| `ACESTEP_LM_BACKEND` | `pt` | Use PyTorch backend (bypasses nano-vllm flash_attn dependency) |
| `HSA_OVERRIDE_GFX_VERSION` | `11.0.0` | RDNA3 GPU architecture hint (adjust for your GPU — see below) |
| `TORCH_COMPILE_BACKEND` | `eager` | Disable Triton compiler (not available on ROCm Windows) |
| `MIOPEN_FIND_MODE` | `FAST` | Fast kernel selection (avoids minutes-long hangs on first VAE decode) |
| `TOKENIZERS_PARALLELISM` | `false` | Suppress HuggingFace tokenizer warnings |

---

## HSA_OVERRIDE_GFX_VERSION by GPU

For RDNA3 GPUs the GFX version must be set manually. The launcher defaults to `11.0.0` (RX 7900
XT/XTX). Edit `start_gradio_ui_rocm.bat` and change the value if you have a different GPU:

| GPU | GFX version | `HSA_OVERRIDE_GFX_VERSION` |
|-----|-------------|---------------------------|
| RX 7900 XT / RX 7900 XTX / RX 9070 XT | gfx1100 | `11.0.0` |
| RX 7800 XT / RX 7700 XT | gfx1101 | `11.0.1` |
| RX 7600 | gfx1102 | `11.0.2` |
| RX 6900 XT / RX 6800 XT / RX 6800 (RDNA2) | gfx1030 | not needed (auto-detected) |

To change the value, open `start_gradio_ui_rocm.bat` in a text editor and update this line:

```batch
set HSA_OVERRIDE_GFX_VERSION=11.0.0
```

---

## Accessing the Web UI

Once the launcher is running, open your browser at:

```
http://127.0.0.1:7860
```

On first launch, models are downloaded automatically from HuggingFace (or ModelScope if
configured). This may take several minutes depending on your internet connection.

### Recommended settings for ROCm

In the Gradio UI:

- **LM Backend**: `pt` (PyTorch) — already set by the launcher
- **INT8 Quantization**: disabled — `torchao` is not available on ROCm Windows
- **CPU Offload**: enable for GPUs with ≤ 20 GB VRAM (default for 4B LM model)

---

## Customising the Launcher

All configurable options are defined as variables at the top of `start_gradio_ui_rocm.bat`.
You can also create a `.env` file in the project root to persist settings across updates:

```batch
REM .env example
PORT=7860
LANGUAGE=zh
ACESTEP_LM_MODEL_PATH=acestep-5Hz-lm-1.7B
ACESTEP_DOWNLOAD_SOURCE=modelscope
```

See the full list of supported `.env` variables in [INSTALL.md](INSTALL.md#environment-variables-env).

---

## Troubleshooting

### GPU not detected (`CUDA available: False`)

1. Run the diagnostic script:

   ```cmd
   python scripts\check_gpu.py
   ```

2. Confirm your AMD driver version is 26.1.1 or later:
   - Open **Device Manager → Display adapters → AMD Radeon RX …**
   - Check driver version in the **Driver** tab

3. Ensure you are using the `venv_rocm` virtual environment, **not** the standard one (which
   contains CUDA PyTorch):

   ```cmd
   venv_rocm\Scripts\activate
   python -c "import torch; print(torch.__version__)"
   ```

   The version string should contain `+rocmsdk` (e.g. `2.9.1+rocmsdk20260116`).

4. Set `HSA_OVERRIDE_GFX_VERSION` for your GPU (see table above) before importing torch:

   ```cmd
   set HSA_OVERRIDE_GFX_VERSION=11.0.0
   python -c "import torch; print(torch.cuda.is_available())"
   ```

### First-run VAE decode hangs for minutes

This is a known MIOpen kernel benchmarking issue. The launcher sets `MIOPEN_FIND_MODE=FAST`
automatically. If you are launching manually, add:

```cmd
set MIOPEN_FIND_MODE=FAST
```

After the first run, MIOpen caches the selected kernels and subsequent runs start normally.

### `flash_attn` import error

The PyTorch backend (`--backend pt`) is used by default on ROCm. If you see a `flash_attn`
import error, ensure `ACESTEP_LM_BACKEND=pt` is set:

```cmd
set ACESTEP_LM_BACKEND=pt
```

The ROCm launcher sets this automatically.

### Out-of-memory errors (OOM)

- Enable CPU offload in the UI or add `--offload_to_cpu true` to the launcher command.
- Use a smaller LM model (e.g. `acestep-5Hz-lm-0.6B` instead of `4B`).
- GPU tier thresholds and model recommendations:

  | VRAM | Recommended LM | Offload needed? |
  |------|---------------|-----------------|
  | ≤ 6 GB | None (DiT only) | Yes |
  | 6–8 GB | 0.6B | Yes |
  | 8–16 GB | 0.6B / 1.7B | Yes |
  | 16–24 GB | 1.7B | Optional |
  | ≥ 24 GB | 4B | No |

---

## Manual Launch (without the `.bat` launcher)

If you prefer to launch from a plain command prompt, set all required variables manually:

```cmd
venv_rocm\Scripts\activate

set ACESTEP_LM_BACKEND=pt
set HSA_OVERRIDE_GFX_VERSION=11.0.0
set TORCH_COMPILE_BACKEND=eager
set MIOPEN_FIND_MODE=FAST
set TOKENIZERS_PARALLELISM=false

python acestep\acestep_v15_pipeline.py --port 7860 --backend pt --offload_to_cpu true
```

---

## Related Resources

- [`requirements-rocm.txt`](../../requirements-rocm.txt) — Full Windows ROCm dependency list with installation steps
- [GPU Compatibility Guide](GPU_COMPATIBILITY.md) — VRAM tiers, duration limits, batch sizes
- [GPU Troubleshooting Guide](GPU_TROUBLESHOOTING.md) — Advanced diagnostics
- [Linux ROCm Manual](ACE-Step1.5-Rocm-Manual-Linux.md) — Linux setup (RDNA4, cachy-os)
- [INSTALL.md](INSTALL.md) — Full installation guide for all platforms
