# Requirements & Dependencies

Everything Aboleth STT needs to run, from hardware to bundled libraries.

---

## GPU Requirement

Aboleth STT requires GPU acceleration for real-time inference.

| Backend | Requirements | Notes |
|---------|-------------|-------|
| CUDA | NVIDIA GPU, Compute Capability 5.0+ | Preferred -- fastest inference path |
| Vulkan | NVIDIA, AMD, or Intel GPU with Vulkan support | Cross-vendor fallback |

The **Auto** backend setting (default) selects CUDA when an NVIDIA GPU is detected and falls back to Vulkan otherwise.

!!! tip
    CUDA consistently outperforms Vulkan on NVIDIA hardware. If you have an NVIDIA GPU, let Auto select CUDA or set it explicitly in [Project Settings](configuration/project-settings.md).

---

## Platform

| Requirement | Value |
|-------------|-------|
| Engine | Unreal Engine 5.7 |
| Platform | Windows Win64 |
| CPU | AVX instruction set required |
| C++ Standard | C++20 |

!!! warning "AVX Requirement"
    whisper.cpp and GGML require AVX support. CPUs without AVX (some older or low-power processors) will fail at runtime. All modern desktop and laptop CPUs from 2011 onward support AVX.

---

## Bundled Dependencies

Aboleth STT ships with all required libraries. No external installs or SDKs are needed.

| Dependency | Purpose |
|------------|---------|
| **whisper.cpp** | Speech-to-text inference engine |
| **ggml** | Tensor library powering CPU, CUDA, and Vulkan compute |
| **Silero VAD v6.2.0** | Neural voice activity detection (runs via GGML -- no ONNX runtime) |

### CUDA DLLs

The following NVIDIA DLLs are bundled for CUDA backend support:

| DLL | Purpose |
|-----|---------|
| `cublas64_12.dll` | CUDA Basic Linear Algebra Subprograms |
| `cublasLt64_12.dll` | CUDA BLAS lightweight runtime |
| `cudart64_12.dll` | CUDA runtime library |

### Vulkan

| Library | Purpose |
|---------|---------|
| `ggml-vulkan` | GGML Vulkan compute backend |

!!! info
    Vulkan support uses the system's Vulkan driver. No additional Vulkan SDK installation is required -- the driver bundled with your GPU's display driver is sufficient.

---

## Unreal Engine Module Dependencies

The plugin depends on the following UE modules:

| Module | Purpose |
|--------|---------|
| `AudioCapture` | Microphone input access |
| `Synthesis` | Audio processing utilities |
| `AudioMixer` | Audio mixing and routing |
| `AudioCaptureCore` | Platform audio capture abstraction |
| `SignalProcessing` | DSP and resampling |
| `AudioSynesthesia` | Audio analysis features |
| `AudioExtensions` | Extended audio API support |
| `DeveloperSettings` | Project Settings integration |

---

## Model Files

Whisper and Silero models are stored in:

```
Plugins/AbolethSTT/Models/
```

Use the [Model Downloader](tools/model-downloader.md) to fetch models directly from HuggingFace, or download them manually and place them in the `Models/` directory.

**Packaged builds** stage model files to:

```
$(TargetOutputDir)/Models/
```

!!! note
    Models are not included with the plugin download. You must download at least one Whisper model before first use. The Silero VAD model (`ggml-silero-v6.2.0.bin`) ships with the plugin.

---

## DLL Loading

Core DLLs are loaded at module startup during `FAbolethSTTModule::StartupModule()`. GPU backend DLLs (CUDA, Vulkan) are loaded conditionally based on the selected backend and available hardware.

**Packaged builds** stage all DLLs to:

```
$(TargetOutputDir)
```

!!! warning
    If CUDA DLLs fail to load (e.g. missing NVIDIA driver), the system falls back to Vulkan automatically when using the Auto backend. If both GPU backends fail, the plugin will not function -- CPU-only inference is not supported.
