# Structs & Enums

Core data types used throughout the Aboleth STT API.

---

## Enums

### EAbolethPipelineState

Current state of the STT processing pipeline. Reported by `GetPipelineState()` and the `OnPipelineStateChanged` delegate.

| Value | Description |
|---|---|
| `Uninitialized` | The STT system has not been loaded. Call `LoadSTTSystem()` to initialize. |
| `Idle` | System is loaded and listening, but no speech is detected. |
| `SpeechDetected` | The VAD has detected speech. Audio is being monitored for sustained voice activity. |
| `Accumulating` | Speech is confirmed and audio samples are being buffered for transcription. |
| `Processing` | Audio has been submitted to Whisper and transcription is in progress. |
| `Error` | A pipeline error occurred. Check logs or `OnSTTProcessingFailed` for details. |

```
Uninitialized ──LoadSTTSystem()──► Idle
                                     │
                              VAD speech ──► SpeechDetected
                                                  │
                                        sustained ──► Accumulating
                                                          │
                                               silence ──► Processing
                                                              │
                                                     result ──► Idle
```

---

### EAbolethCaptureMode

Determines how audio capture is initiated.

| Value | Description |
|---|---|
| `VADAutomatic` | The Silero VAD automatically detects speech and silence. No user input required. |
| `PushToTalk` | Audio is only captured between `StartManualCapture()` and `StopManualCapture()` calls. |

---

### EAbolethGPUBackend

GPU acceleration backend for Whisper inference.

| Value | Description |
|---|---|
| `Auto` | Automatically selects the best available backend. Prefers CUDA if available, falls back to Vulkan. |
| `CUDA` | Use NVIDIA CUDA for GPU acceleration. Requires an NVIDIA GPU with CUDA support. |
| `Vulkan` | Use Vulkan for GPU acceleration. Works on NVIDIA, AMD, and Intel GPUs with Vulkan drivers. |

---

## Structs

### FAudioLevelInfo

Real-time audio level metrics from the microphone. Used for visual metering --- **not** for VAD decisions.

| Field | Type | Description |
|---|---|---|
| `CurrentAmplitude` | `float` | Instantaneous peak amplitude of the current audio frame (0.0--1.0). |
| `PeakAmplitude` | `float` | Highest amplitude observed since the last reset. Useful for peak-hold meters. |
| `RMSLevel` | `float` | Root-mean-square level of the current frame. Represents average loudness. |
| `bIsSilent` | `bool` | `true` if the current frame is below the silence threshold. Based on amplitude, not VAD. |

!!! info "Metering vs VAD"
    `FAudioLevelInfo` is computed in `MicCaptureComponent` for audio level display purposes only. Speech detection is handled entirely by the Silero VAD model, which operates independently of these amplitude metrics.

---

### FAudioDeviceInfo

Describes an audio input device detected by the system.

| Field | Type | Description |
|---|---|---|
| `DeviceName` | `FString` | Human-readable device name (e.g., `"Blue Yeti Stereo Microphone"`). |
| `DeviceId` | `FString` | Platform-specific unique device identifier. |
| `DeviceIndex` | `int32` | Index in the device list. Pass to `SetAudioDeviceByIndex()`. |
| `InputChannels` | `int32` | Number of input channels supported by the device. |
| `PreferredSampleRate` | `int32` | The device's preferred sample rate in Hz (e.g., `44100`, `48000`). |
| `bSupportsHardwareAEC` | `bool` | `true` if the device supports hardware acoustic echo cancellation. |

---

### FAudioCaptureStatus

Snapshot of the full audio capture state. Returned by `GetMicrophoneStatus()`.

| Field | Type | Description |
|---|---|---|
| `AudioLevels` | `FAudioLevelInfo` | Current audio level metrics. |
| `bIsCapturing` | `bool` | `true` if the microphone is actively capturing audio. |
| `AudioQueueSamples` | `int32` | Number of audio samples currently buffered in the processing queue. |
| `bIsProcessingSTT` | `bool` | `true` if a transcription pass is currently in progress. |
| `PipelineState` | `EAbolethPipelineState` | Current pipeline state at the time of the query. |

---

### FWhisperWordResult

A single word from Whisper's word-level timestamp output. Returned when word timestamps are enabled.

| Field | Type | Description |
|---|---|---|
| `Text` | `FString` | The transcribed word text. |
| `T0` | `int32` | Start time of the word in **centiseconds** (1/100th of a second) from the beginning of the audio segment. |
| `T1` | `int32` | End time of the word in **centiseconds** from the beginning of the audio segment. |
| `Probability` | `float` | Whisper's confidence score for this word (0.0--1.0). |

??? example "Converting Centiseconds to Seconds"
    ```cpp
    float StartSeconds = WordResult.T0 / 100.0f;
    float EndSeconds   = WordResult.T1 / 100.0f;
    float Duration     = EndSeconds - StartSeconds;
    ```
