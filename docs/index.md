# Aboleth STT

**GPU-accelerated speech-to-text for Unreal Engine 5.7**

Aboleth STT brings real-time speech recognition to your Unreal Engine project. Powered by whisper.cpp with Silero VAD, streaming transcription, and CUDA/Vulkan GPU inference — everything runs locally on the player's hardware. No cloud APIs, no per-minute billing, no internet required.

---

## Highlights

- :material-gpu: **GPU-Accelerated** — CUDA (NVIDIA) and Vulkan (AMD/Intel/NVIDIA) backends
- :material-microphone: **Silero VAD** — Neural voice activity detection with streaming LSTM
- :material-text-box-outline: **Streaming Transcription** — See text appear word-by-word as the player speaks
- :material-gamepad-variant: **Push-to-Talk** — VAD automatic or manual capture modes
- :material-translate: **99 Languages** — Auto-detect or force a specific language, with translation to English
- :material-tune: **Runtime Tunable** — Every setting adjustable from Blueprint or C++ without reloading
- :material-download: **In-Editor Model Downloader** — One-click download from HuggingFace in Project Settings
- :material-state-machine: **Formal Pipeline State Machine** — Clean state transitions with full event coverage

---

## Get Started in 60 Seconds

1. **Download a model** — Project Settings > Aboleth Speech-to-Text > Model Management
2. **Drop an actor** — Drag `AbolethSTTListenerActor` into your level
3. **Bind one event** — `OnUtteranceProcessed` gives you the transcribed text
4. **Play** — Talk into your mic. Text appears.

[Quick Start Guide :material-arrow-right:](getting-started/quick-start.md){ .md-button .md-button--primary }
[API Reference :material-arrow-right:](api/subsystem.md){ .md-button }

---

## How It Works

```
Microphone → MicCapture → Ring Buffer → Silero VAD → Whisper (GPU) → Text
                                            ↓
                                    [Streaming passes]
                                    Local Agreement (n=2)
                                    Word-by-word output
```

Aboleth STT captures audio from the microphone, resamples to 16kHz mono, and writes to a 30-second ring buffer. Silero VAD (streaming LSTM) detects speech onset and offset. When speech ends, the audio segment is sent to Whisper on a background thread for GPU inference. With streaming enabled, growing audio windows are sent to Whisper during speech for near-real-time word-by-word output.

[Architecture Details :material-arrow-right:](getting-started/architecture.md){ .md-button }
