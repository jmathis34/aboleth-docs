# Debug & Profiling

Aboleth STT includes built-in diagnostics for logging, probability analysis, and UE5 stat profiling.

---

## Debug Logging

Debug logging is **gated behind a flag** — system lifecycle logs (model load, backend selection, microphone open/close) are always shown regardless of this setting.

**Enable in Project Settings:**

**Project Settings > Plugins > Aboleth Speech-to-Text > Debug > Enable Debug Logging**

**Enable at runtime (C++):**

```cpp
UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
STT->SetDebugLogging(true);
```

When enabled, the following diagnostics are written to the `LogAbolethSTT` log category:

| Log Output | Description |
|------------|-------------|
| Audio level analysis | Per-frame RMS and peak amplitude of captured audio |
| Sample counts | Number of samples extracted from the ring buffer per segment |
| Whisper inference time | Wall-clock duration of each `whisper_full()` call |
| Realtime factor | Ratio of inference time to audio duration (< 1.0 means faster than real-time) |
| Segment count | Number of text segments returned by the decoder |
| Pipeline latency | End-to-end time from VAD offset to transcription result |
| Ring buffer extraction details | Start/end positions, sample count, and any wraparound behavior |

!!! tip
    Enable debug logging when tuning VAD parameters or diagnosing latency issues. Disable it in shipping builds to avoid log spam.

---

## Probability Logging

Probability logging writes raw per-frame Silero VAD output to a CSV file for offline analysis.

**Enable in Project Settings:**

**Project Settings > Plugins > Aboleth Speech-to-Text > Debug > Enable Probability Logging**

**Output file:**

```
Plugins/AbolethSTT/Tools/silero_probabilities.csv
```

The CSV contains timestamped speech probability values for every 512-sample VAD window processed during the session.

### Silero Waveform Analyzer

The plugin ships with a standalone visualization tool for inspecting probability logs:

```
Plugins/AbolethSTT/Tools/SileroWaveformAnalyzer.html
```

This HTML file is included with the plugin — no additional download required. Open it in any browser, load the CSV, and the analyzer renders an interactive waveform showing speech probability over time. Use it to:

- Visualize where speech onsets and offsets are detected
- Identify false triggers or missed speech segments
- Compare probability curves across different VAD threshold settings

!!! info
    The analyzer runs entirely in the browser with no server or dependencies. It ships with the plugin at the path above — just open the HTML file and drag in your CSV.

---

## UE5 Stat Profiling

Aboleth STT registers a custom stat group for real-time profiling through the Unreal console.

**Console command:**

```
stat AbolethSTT
```

### Available Stats

| Stat | Type | Description |
|------|------|-------------|
| Whisper Inference | Cycle | Time spent in `whisper_full()` per inference pass |
| VAD Processing | Cycle | Time spent in Silero VAD probability computation |
| Audio Callback | Cycle | Time spent in the microphone capture callback |
| Pipeline State | Int | Current pipeline state enum value |
| GPU Backend | Int | Active backend: `1` = CUDA, `2` = Vulkan, `3` = Both, `0` = None |
| Audio Queue Samples | Int | Number of samples currently queued for processing |
| Last Inference Ms | Float | Wall-clock milliseconds of the most recent inference pass |
| Last Audio Duration Ms | Float | Duration in milliseconds of the audio segment sent to Whisper |
| Streaming Pass Count | Int | Total number of streaming inference passes since listening started |
| VAD Probability | Float | Most recent Silero speech probability (0.0 -- 1.0) |
| RMS Level | Float | Current RMS audio level from the microphone |
| Silero Probability | Float | Raw Silero model output before threshold comparison |

!!! note
    Cycle stats measure wall-clock time and appear in the standard UE5 stat overlay. Use them alongside `stat GPU` and `stat Threading` to identify bottlenecks in the full pipeline.

!!! warning
    `stat AbolethSTT` is a development tool. The stat group is compiled out in Shipping builds.
