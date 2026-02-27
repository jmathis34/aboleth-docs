# Project Settings

All Aboleth STT settings are configured in **Project Settings > Plugins > Aboleth Speech-to-Text**. These map to `DefaultGame.ini` under the section `[/Script/AbolethSTT.WhisperDeveloperSettings]`.

!!! tip "Runtime changes"
    Many of these settings can also be changed at runtime without reloading the model. See [Runtime Settings API](runtime-settings.md) for the full list of setters and getters.

---

## Model

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `ModelFilename` | String | `ggml-large-v3-turbo-q5_0.bin` | Active Whisper model filename. Resolved relative to the `Plugins/AbolethSTT/Models/` directory. |
| `VADModelFilename` | String | `ggml-silero-v6.2.0.bin` | Silero VAD model filename. **Read-only** — ships with the plugin. |

!!! note
    Changing `ModelFilename` requires reloading the STT system (`LoadSTTSystem`). Use the [Model Downloader](../tools/model-downloader.md) to fetch additional models.

---

## Silero VAD

These settings map directly to `whisper_vad_params` in the whisper.cpp backend. They control when speech is detected and how segment boundaries are drawn.

| Setting | Type | Default | Range | Description |
|---------|------|---------|-------|-------------|
| `threshold` | Float | `0.5` | 0.0 -- 1.0 | Speech probability threshold. Higher values require more confidence before triggering onset. |
| `min_speech_duration_ms` | Int | `250` | 0 -- 5000 | Minimum consecutive milliseconds above threshold before speech onset is confirmed. Prevents transient noise from triggering detection. |
| `min_silence_duration_ms` | Int | `100` | 0 -- 5000 | Minimum consecutive milliseconds below threshold before speech offset is confirmed. Higher values tolerate longer pauses mid-sentence. |
| `max_speech_duration_s` | Float | `0.0` | 0.0 -- 3600.0 | Maximum duration of a single speech segment before forcing a new one. `0.0` disables the limit. |
| `speech_pad_ms` | Int | `30` | 0 -- 1000 | Padding added before and after detected speech. The onset reach-back captures audio slightly before the VAD triggered. |
| `samples_overlap` | Float | `0.1` | 0.0 -- 2.0 | Overlap in seconds when copying speech audio between segments. Ensures no audio is lost at segment boundaries. |

!!! info "Tuning VAD"
    See the [Tuning Guide](tuning-guide.md) for practical advice on adjusting these values for different environments and use cases.

---

## Performance

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `NumThreads` | Int | `4` | Number of CPU threads used by whisper.cpp. Only affects CPU-bound work — GPU inference is not thread-limited. |
| `GPUBackend` | Enum | `Auto` | GPU acceleration backend. `Auto` selects CUDA if available, then Vulkan, then CPU. Options: `Auto`, `CUDA`, `Vulkan`. |
| `bFlashAttention` | Bool | `true` | Enables Flash Attention for faster encoder inference. Recommended on for all GPU backends. |
| `bDynamicAudioContext` | Bool | `false` | Scales the encoder window to match actual audio length instead of always processing 30 seconds. Reduces latency on short utterances. |
| `MinAudioContext` | Int | `512` | Floor value for dynamic audio context. Must be a power of 2. Prevents the encoder window from becoming too small on very short audio. |

!!! warning "MinAudioContext"
    Setting `MinAudioContext` below 512 may cause artifacts with Whisper models. The default of 512 is recommended for all supported models.

---

## Language

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `Language` | String | `"auto"` | Language code for transcription (e.g. `"en"`, `"es"`, `"ja"`). `"auto"` enables automatic language detection per utterance. |
| `bTranslateToEnglish` | Bool | `false` | When enabled, translates non-English speech to English. Uses Whisper's built-in translation capability. |

---

## Audio

| Setting | Type | Default | Range | Description |
|---------|------|---------|-------|-------------|
| `InputGainDb` | Float | `0.0` | 0.0 -- 20.0 | Software microphone gain in decibels. `0` is passthrough (no amplification). `6` dB is approximately 2x gain. `12` dB is approximately 4x gain. |
| `TargetSampleRate` | Int | `16000` | -- | Target sample rate for Whisper inference. **Read-only** — Whisper requires 16kHz mono audio. |

!!! tip
    If your microphone input is too quiet, increase `InputGainDb` before adjusting VAD thresholds. A gain of 6 dB doubles the signal amplitude and often resolves low-sensitivity issues without changing detection behavior.

---

## Streaming

| Setting | Type | Default | Range | Description |
|---------|------|---------|-------|-------------|
| `bEnableStreaming` | Bool | `true` | -- | Enables Local Agreement streaming. When on, partial transcription results are emitted while the user is still speaking. |
| `StreamingChunkIntervalMs` | Int | `1000` | 50 -- 2000 | Interval in milliseconds between streaming inference passes. Lower values give faster feedback but increase GPU load. |
| `BufferTrimmingSec` | Int | `6` | 3 -- 30 | When the accumulated audio buffer exceeds this length in seconds, earlier audio is trimmed. Keeps memory and inference time bounded during long utterances. |

---

## Beam Search

Beam search is disabled by default. It improves accuracy on the final transcription pass at the cost of additional compute.

| Setting | Type | Default | Range | Description |
|---------|------|---------|-------|-------------|
| `bEnableBeamSearch` | Bool | `false` | -- | Enables beam search decoding on the final (post-utterance) inference pass. |
| `BeamSize` | Int | `5` | 1 -- 10 | Number of beams to maintain during search. Higher values explore more hypotheses. |
| `LengthPenalty` | Float | `-1.0` | -1.0 -- 3.0 | Penalty applied to longer sequences. `-1.0` disables the penalty. Values above 1.0 favor longer transcriptions. |
| `bBeamSearchDuringStreaming` | Bool | `false` | -- | Enables beam search during streaming passes. Significantly increases streaming latency. |
| `bEnableBeamSearchGate` | Bool | `true` | -- | Enables adaptive beam dropout. Beams that fall too far behind the leader are pruned early to save compute. |
| `BeamSearchGateMs` | Int | `250` | 50 -- 2000 | Time threshold in milliseconds for the beam search gate. Beams not contributing within this window are dropped. |

!!! note
    Enabling beam search during streaming (`bBeamSearchDuringStreaming`) is generally not recommended. The latency cost outweighs the accuracy gain for real-time use cases. Reserve beam search for the final pass where accuracy matters most.

---

## Debug

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `bEnableDebugLogging` | Bool | `false` | Logs per-utterance diagnostics including timing, segment lengths, and VAD decisions. Output goes to the `LogAbolethSTT` log category. |
| `bEnableProbabilityLogging` | Bool | `false` | Writes per-frame speech probability values to a CSV file for offline analysis. Useful for tuning VAD thresholds against specific audio environments. |

!!! info
    For more on debugging tools, see [Debug & Profiling](../tools/debug.md).
