# Tuning Guide

Practical solutions for common speech detection issues. Each section describes a symptom, explains why it happens, and gives specific settings to fix it.

!!! tip "Start with defaults"
    The default settings work well for a typical desktop microphone in a quiet room. Only adjust these if you are experiencing a specific problem.

---

## Too much background noise triggers detection

**Symptom:** The system fires transcription events when nobody is speaking — fans, music, keyboard clicks, or ambient noise cause false positives.

**Why:** The VAD threshold is too low for the noise floor in the environment.

**Fix:**

- Increase `threshold` from `0.5` to **0.6 -- 0.7**. This requires higher neural network confidence before onset is triggered.
- Increase `min_speech_duration_ms` from `250` to **400 -- 600**. This forces the VAD to see sustained speech-like audio before committing, filtering out brief noise bursts.

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
    STT->SetVADThreshold(0.7f);
    ```

=== "Blueprint"

    Call **Set VAD Threshold** on the subsystem with a value of `0.7`.

=== "DefaultGame.ini"

    ```ini
    [/Script/AbolethSTT.WhisperDeveloperSettings]
    threshold=0.7
    min_speech_duration_ms=500
    ```

---

## It cuts off my sentences

**Symptom:** The system triggers offset (end of speech) too early. Words at the end of sentences are lost, especially during natural pauses mid-thought.

**Why:** The `min_silence_duration_ms` is too short. Even a brief pause between words or clauses causes the VAD to finalize the segment.

**Fix:**

- Increase `min_silence_duration_ms` from `100` to **300 -- 500**. This gives the speaker more room to pause without ending the segment.

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
    // Allow up to 400ms of silence before finalizing
    STT->ReloadVADSettings(); // after updating the developer settings object
    ```

=== "DefaultGame.ini"

    ```ini
    [/Script/AbolethSTT.WhisperDeveloperSettings]
    min_silence_duration_ms=400
    ```

!!! warning
    Setting `min_silence_duration_ms` above 800ms can make the system feel sluggish — the player has to wait a noticeable amount of time after finishing a sentence before the transcription appears. Values between 300 and 500 are a good balance.

---

## Short words like "yes" or "no" get dropped

**Symptom:** Very short utterances are ignored. The player says "yes" and nothing happens.

**Why:** The `min_speech_duration_ms` is longer than the utterance itself. A quick "yes" might only be 150 -- 200ms of voiced audio, which falls below the default 250ms onset threshold.

**Fix:**

- Lower `min_speech_duration_ms` from `250` to **100 -- 150**. This allows shorter bursts of speech to register.
- Optionally lower `threshold` to **0.4** if the short words are also quiet.

=== "DefaultGame.ini"

    ```ini
    [/Script/AbolethSTT.WhisperDeveloperSettings]
    threshold=0.4
    min_speech_duration_ms=100
    ```

!!! note "Trade-off"
    Lowering both `threshold` and `min_speech_duration_ms` makes the system more sensitive to all sounds, not just speech. If you are in a noisy environment, you may need to compensate by increasing `InputGainDb` instead of lowering the threshold — a louder speech signal is easier for Silero to distinguish from noise.

---

## My microphone is too quiet

**Symptom:** The system detects speech inconsistently or not at all. The audio level meter shows very low values. Quiet speakers or low-gain microphones are particularly affected.

**Why:** The raw microphone signal is too weak for the VAD to confidently distinguish speech from silence.

**Fix:**

- Increase `InputGainDb`. Start with **6.0** (approximately 2x amplification). If still too quiet, try **12.0** (approximately 4x).

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
    STT->SetMicGainDb(6.0f);
    ```

=== "Blueprint"

    Call **Set Mic Gain Db** on the subsystem with a value of `6.0`.

=== "DefaultGame.ini"

    ```ini
    [/Script/AbolethSTT.WhisperDeveloperSettings]
    InputGainDb=6.0
    ```

!!! tip "Gain before threshold"
    Always try increasing gain before lowering the VAD threshold. Amplifying the signal raises speech above the noise floor, giving the VAD a cleaner decision boundary. Lowering the threshold instead makes the VAD more sensitive to everything, including noise.

---

## Streaming text appears too slowly

**Symptom:** When streaming is enabled, the live transcription updates feel laggy. There is a noticeable delay between speaking and seeing partial results.

**Why:** The `StreamingChunkIntervalMs` determines how often a streaming inference pass runs. The default of 1000ms means partial results update at most once per second.

**Fix:**

- Lower `StreamingChunkIntervalMs` from `1000` to **500**. This doubles the update rate.
- Going below 300ms is possible but increases GPU utilization. On lower-end hardware, this can cause frame drops.

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
    STT->SetStreamingChunkIntervalMs(500);
    ```

=== "Blueprint"

    Call **Set Streaming Chunk Interval Ms** on the subsystem with a value of `500`.

=== "DefaultGame.ini"

    ```ini
    [/Script/AbolethSTT.WhisperDeveloperSettings]
    StreamingChunkIntervalMs=500
    ```

!!! info "GPU budget"
    Each streaming pass runs a full Whisper encoder + decoder inference on the accumulated audio. At 500ms intervals, you get two passes per second. Monitor your frame times with [Debug & Profiling](../tools/debug.md) to ensure the GPU budget stays comfortable for your target hardware.

---

## Recommended Presets

These are starting points. Adjust to taste based on your hardware and environment.

=== "Quiet Room (Desktop Mic)"

    | Setting | Value |
    |---------|-------|
    | `threshold` | `0.5` |
    | `min_speech_duration_ms` | `250` |
    | `min_silence_duration_ms` | `100` |
    | `InputGainDb` | `0.0` |
    | `StreamingChunkIntervalMs` | `1000` |

=== "Noisy Environment"

    | Setting | Value |
    |---------|-------|
    | `threshold` | `0.7` |
    | `min_speech_duration_ms` | `500` |
    | `min_silence_duration_ms` | `200` |
    | `InputGainDb` | `0.0` |
    | `StreamingChunkIntervalMs` | `1000` |

=== "Voice Commands (Short Utterances)"

    | Setting | Value |
    |---------|-------|
    | `threshold` | `0.4` |
    | `min_speech_duration_ms` | `100` |
    | `min_silence_duration_ms` | `300` |
    | `InputGainDb` | `6.0` |
    | `StreamingChunkIntervalMs` | `500` |
