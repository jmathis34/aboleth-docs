# Architecture

## Pipeline Overview

```
Microphone (48kHz stereo)
    |
    v
MicCaptureComponent
    |  (float32 → mono downmix → resample 48k→16k → int16)
    |  (software mic gain applied here)
    v
UnifiedAudioBuffer (30-second ring buffer, int16, 16kHz)
    |
    +──> AbolethVADComponent (Silero LSTM streaming inference)
    |        |
    |        |  512-sample windows (32ms at 16kHz)
    |        |  LSTM state persists across calls
    |        |
    |        |  Onset:  probability > threshold for min_speech_duration_ms
    |        |  Offset: probability < threshold for min_silence_duration_ms
    |        |
    |        v
    |    [Speech segment boundaries identified]
    |
    +──> ExtractRange() pulls audio from ring buffer
    |    (with speech_pad_ms reach-back for context)
    |
    +──> [If Streaming Enabled] Growing-window passes every N ms
    |    Local Agreement (n=2) confirms words across consecutive passes
    |    OnTranscriptionUpdated fires with committed + tentative text
    |
    v
WhisperWorker (background thread)
    |
    |  whisper_full() with integrated Silero VAD
    |  GPU inference via CUDA or Vulkan
    |  Optional beam search decoding (final pass)
    |
    v
Transcribed text → Game thread queue → OnUtteranceProcessed delegate
```

---

## Silero VAD

Voice activity detection is handled entirely by **Silero VAD**, a neural network (LSTM) that runs through whisper.cpp's GGML inference engine. There is no RMS pre-filter for speech detection — Silero is the sole speech detector.

!!! info "RMS is for display only"
    RMS levels are computed in `MicCaptureComponent` and exposed via `FAudioLevelInfo` for **display and metering purposes only** (e.g. driving a voice activity meter in your UI). They do not gate or influence VAD decisions.

The Silero model uses a streaming LSTM architecture:

- **512-sample windows** (32ms at 16kHz), ~31 probability readings per second
- **LSTM hidden/cell state persists** across calls, building temporal context
- **State reset on speech offset** so the next utterance starts clean
- **SoftReset()** clears duration counters but preserves LSTM state for warm onset detection between utterances

---

## Pipeline State Machine

```
Uninitialized  ──>  Idle              (system loaded)
Idle           ──>  Accumulating      (Silero onset confirmed)
Accumulating   ──>  Processing        (silence detected, segment extracted)
Accumulating   ──>  Idle              (segment discarded — ring buffer overrun)
Processing     ──>  Idle              (transcription complete)
Processing     ──>  Error             (timeout / exception)
Error          ──>  Idle              (recovery)
Any            ──>  Uninitialized     (system unload)
```

| State | Description |
|-------|-------------|
| `Uninitialized` | System not yet initialized or model not loaded |
| `Idle` | System loaded, microphone active, listening for speech |
| `SpeechDetected` | Silero VAD has detected potential speech onset, accumulating evidence |
| `Accumulating` | Confirmed speech, actively accumulating audio into current segment |
| `Processing` | Audio segment being processed by Whisper on a background thread |
| `Error` | Recoverable error; can be reset to Idle |

Transitions fire `OnPipelineStateChanged` for UI/logic binding. The state machine validates all transitions — invalid transitions are rejected and logged.
