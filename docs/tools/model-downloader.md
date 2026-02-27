# Model Downloader

The Model Downloader fetches Whisper GGUF models directly from HuggingFace into your project's `Plugins/AbolethSTT/Models/` directory.

**Access via:**

**Project Settings > Plugins > Aboleth Speech-to-Text > Model Management**

---

## Recommended Models

| Model | Size | Speed | Accuracy | Notes |
|-------|------|-------|----------|-------|
| Tiny English | ~75 MB | Fastest | Low | Fastest model, lowest accuracy. Good for prototyping or low-spec hardware |
| Large V3 Turbo Q5 | ~1.1 GB | Fast | Excellent | **Recommended** -- best balance of size, speed, and accuracy |
| Large V3 Turbo Q8 | ~1.6 GB | Fast | Excellent | Near-lossless quantization, slightly larger than Q5 |
| Large V3 Turbo | ~3.1 GB | Fast | Excellent | Full-precision Turbo, multilingual |

!!! tip "Recommended Model"
    **`ggml-large-v3-turbo-q5_0.bin`** is the best general-purpose choice. It delivers excellent accuracy at one-third the file size of the full-precision Turbo model, with minimal latency overhead from quantization.

---

## Turbo Architecture

The Large V3 Turbo models use a distilled architecture that is significantly faster than the older Small and Medium models despite being larger in file size. This is because the Turbo encoder is shallower and more efficient -- it was distilled from the full Large V3 model to retain accuracy while reducing compute.

Quantized variants (Q5, Q8) add negligible latency compared to full precision while reducing file size by up to 3x.

---

## VRAM Usage

VRAM consumption during inference approximately equals the model's file size on disk.

| Model | Approximate VRAM |
|-------|------------------|
| Tiny English | ~75 MB |
| Large V3 Turbo Q5 | ~1.1 GB |
| Large V3 Turbo Q8 | ~1.6 GB |
| Large V3 Turbo | ~3.1 GB |

!!! warning
    Choose a model that fits within your target hardware's available VRAM. If running alongside other GPU-intensive systems (rendering, other ML models), account for their VRAM usage as well.

---

## Download & Storage

Models are downloaded from HuggingFace and stored in:

```
Plugins/AbolethSTT/Models/
```

The **Silero VAD model** (`ggml-silero-v6.2.0.bin`) is also available through the downloader, though it ships bundled with the plugin by default.

!!! note
    The Model Downloader requires an internet connection. For air-gapped environments, download models manually from HuggingFace and place them in the `Models/` directory.
