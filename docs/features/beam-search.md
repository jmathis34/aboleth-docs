# Beam Search

Aboleth STT supports **beam search decoding** as an alternative to Whisper's default greedy decoding. Beam search explores multiple candidate transcriptions in parallel and selects the highest-scoring sequence, which can improve accuracy on ambiguous or noisy audio at the cost of additional computation.

---

## Greedy vs Beam Search

| Strategy | How it works | Trade-off |
|----------|-------------|-----------|
| **Greedy** | Picks the single most probable token at each step | Fast. Good enough for clear speech. |
| **Beam Search** | Maintains N candidate sequences (beams) and scores them | More accurate on ambiguous audio, but ~1.5--2x slower per pass. |

By default, Aboleth STT uses greedy decoding for all passes. When beam search is enabled, it applies to the **final Whisper pass** at the end of an utterance (and to non-streaming single-utterance processing). Streaming passes continue to use fast greedy decoding unless explicitly overridden.

---

## Configuration

### Enabling Beam Search

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);

    // Enable beam search for the final pass
    STT->SetBeamSearchEnabled(true);

    // Set beam size (1-10, default 5)
    STT->SetBeamSize(5);

    // Length penalty (-1.0 = disabled, >0 = penalize short sequences)
    STT->SetLengthPenalty(-1.0f);
    ```

=== "Blueprint"

    Call **Set Beam Search Enabled** (`true`) on the Subsystem. Optionally call **Set Beam Size** and **Set Length Penalty** to tune behavior.

### Runtime API

| Function | Description |
|----------|-------------|
| `SetBeamSearchEnabled(bool)` | Enable or disable beam search on the final pass |
| `IsBeamSearchEnabled()` | Check if beam search is enabled |
| `SetBeamSize(int32)` | Set number of beams (1--10) |
| `GetBeamSize()` | Get current beam size |
| `SetLengthPenalty(float)` | Set length penalty (-1.0 disables, >0 penalizes short sequences) |
| `GetLengthPenalty()` | Get current length penalty |

### Length Penalty

The length penalty controls Whisper's preference for shorter vs longer sequences during beam scoring:

| Value | Effect |
|-------|--------|
| `-1.0` | Disabled (default) -- no length preference |
| `0.0` | No penalty applied |
| `0.6 -- 1.0` | Encourages longer, more complete transcriptions |
| `> 1.0` | Strongly penalizes short sequences |

---

## Beam Search Gate (Adaptive Dropout)

Beam search can cause noticeable latency on slower GPUs, especially with longer audio. The **Beam Search Gate** prevents this by automatically dropping to greedy decoding when a beam search pass exceeds a configurable time budget.

### How it works

1. A beam search pass runs and its inference time is measured.
2. If the pass exceeds `BeamSearchGateMs` (default 250 ms), the gate trips.
3. The **next** pass falls back to greedy decoding (fast, no lag).
4. After the greedy pass completes, beam search is retried.
5. If the retry stays within budget, beam search continues. If it exceeds the gate again, the cycle repeats.

This creates an adaptive loop: beam search is used whenever the GPU can handle it, and greedy is used as a fallback when it cannot.

### Configuration

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);

    // Enable the gate (default: true when beam search is on)
    STT->SetBeamSearchGateEnabled(true);

    // Set the time budget (50-2000 ms, default 250)
    STT->SetBeamSearchGateMs(250);
    ```

=== "Blueprint"

    Call **Set Beam Search Gate Enabled** (`true`) and **Set Beam Search Gate Ms** (`250`) on the Subsystem.

| Function | Description |
|----------|-------------|
| `SetBeamSearchGateEnabled(bool)` | Enable or disable adaptive dropout |
| `IsBeamSearchGateEnabled()` | Check if the gate is active |
| `SetBeamSearchGateMs(int32)` | Set the time budget threshold (50--2000 ms) |
| `GetBeamSearchGateMs()` | Get current gate threshold |

!!! tip "Choosing a gate threshold"
    **250 ms** is a good starting point for mid-range GPUs (RTX 3060/4060 class). If you see frequent dropout in the logs, increase the threshold or consider disabling beam search during streaming. On high-end GPUs (RTX 4080/4090), you can safely lower the threshold or disable the gate entirely.

---

## Beam Search During Streaming

By default, streaming passes use greedy decoding for speed. If your GPU has headroom, you can enable beam search on streaming passes as well for more accurate intermediate results.

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
    STT->SetBeamSearchDuringStreaming(true);
    ```

=== "Blueprint"

    Call **Set Beam Search During Streaming** (`true`) on the Subsystem.

| Function | Description |
|----------|-------------|
| `SetBeamSearchDuringStreaming(bool)` | Enable or disable beam search on streaming passes |
| `IsBeamSearchDuringStreaming()` | Check current setting |

!!! warning "Performance impact"
    Beam search during streaming increases each streaming pass by approximately **1.5--2x**. On most GPUs this means streaming passes take longer than the chunk interval, causing passes to back up. Only enable this if your GPU can complete a beam search pass well within your configured `StreamingChunkIntervalMs`. The Beam Search Gate still applies during streaming, so the system will automatically fall back to greedy if passes exceed the gate threshold.

---

## Project Settings

These defaults are configured in **Project Settings > Aboleth Speech-to-Text > Beam Search**:

| Setting | Default | Description |
|---------|---------|-------------|
| Enable Beam Search (Final Pass) | `false` | Use beam search on the final pass and non-streaming processing |
| Beam Size | `5` | Number of candidate sequences to explore |
| Length Penalty | `-1.0` | Sequence length scoring (-1.0 = disabled) |
| Beam Search During Streaming | `false` | Also use beam search on intermediate streaming passes |
| Enable Beam Search Gate | `true` | Adaptive dropout when beam search exceeds time budget |
| Beam Search Gate (ms) | `250` | Maximum allowed beam search inference time before dropout |
