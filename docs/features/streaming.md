# Streaming Transcription

Aboleth STT supports live word-by-word transcription while the user is still speaking. Words appear in near-real-time, confirmed through a **Local Agreement (n=2)** algorithm that ensures stability -- words must appear in the same position across two consecutive Whisper passes before they are committed.

Streaming is **enabled by default** and works in both VAD Automatic and Push-to-Talk capture modes.

---

## How It Works

```
Speech onset detected
    |
    v
Streaming window opens (ring buffer position recorded)
    |
    |  Every StreamingChunkIntervalMs...
    |
    v
Growing audio window sent to Whisper (with word timestamps)
    |
    v
Local Agreement compares current pass words to previous pass
    |
    +---> Words at same position in both passes --> Committed
    |         fires OnStreamingTextCommitted
    |         appears in OnTranscriptionUpdated CommittedText
    |
    +---> Words only in current pass --> Tentative
              appears in OnTranscriptionUpdated TentativeText
              (ghost/preview -- not yet confirmed)
    |
    v
Speech ends --> final clean pass
    |
    v
Remaining tentative words flushed as committed
OnTranscriptionUpdated fires with bTranscriptionFinished = true
```

### Step by step

1. **Speech onset** -- Silero VAD detects speech and the streaming window opens at the current ring buffer position.
2. **Periodic passes** -- Every `StreamingChunkIntervalMs` (default 1000 ms), the growing audio window from onset to the current position is extracted and sent to Whisper with word-level timestamps enabled.
3. **Local Agreement (n=2)** -- The current pass's word list is compared to the previous pass. A word is committed only when it appears at the same index in *both* passes. This prevents hallucinated or unstable words from reaching your UI.
4. **Committed words** fire `OnStreamingTextCommitted` and appear in the `CommittedText` parameter of `OnTranscriptionUpdated`.
5. **Tentative words** (present in the current pass but not yet confirmed) appear in `TentativeText` -- use these for ghost/preview text in your UI.
6. **Speech ends** -- A final clean Whisper pass runs over the complete utterance. Any remaining tentative words are flushed as committed. `OnTranscriptionUpdated` fires one last time with `bTranscriptionFinished = true` and an empty `TentativeText`.

---

## Buffer Trimming

For long utterances, the streaming audio window would grow unbounded, making each Whisper pass progressively slower. Aboleth STT handles this automatically:

- When the streaming window exceeds `BufferTrimmingSec` (default 6 seconds), the buffer is trimmed at the nearest Whisper segment boundary.
- Committed text from the trimmed region becomes the `initial_prompt` for subsequent Whisper passes, preserving linguistic continuity across the trim point.
- This keeps streaming pass latency bounded regardless of how long the user speaks.

!!! info "Buffer trimming is transparent"
    Trimming happens internally. Your delegate callbacks receive the full accumulated text as if no trimming occurred -- `CommittedText` always contains the complete transcription from the start of the utterance.

---

## Configuration

Streaming is enabled by default. Use these functions to adjust behavior at runtime:

| Function | Description |
|----------|-------------|
| `SetStreamingEnabled(bool)` | Toggle streaming on or off |
| `IsStreamingEnabled()` | Check current streaming state |
| `SetStreamingChunkIntervalMs(int32)` | Set the pass interval (50--2000 ms) |
| `GetStreamingChunkIntervalMs()` | Get current interval |
| `GetStreamingPassCount()` | Number of passes fired in the current/last utterance |
| `GetStreamingAccumulatedText()` | Full running text (prior + current committed) |

!!! tip "Choosing a chunk interval"
    Lower intervals (e.g. 300 ms) give faster word confirmation but increase GPU load. Higher intervals (e.g. 1000 ms) are gentler on the GPU but words appear in less frequent bursts. **1000 ms** is a good default for most hardware. Adjust based on your target GPU and whether other inference workloads (like LLM) share the device.

---

## Delegates

### OnTranscriptionUpdated

The primary delegate for live transcription UI. Fires on every streaming pass *and* on the final utterance result, whether or not streaming is enabled.

```
OnTranscriptionUpdated(FString CommittedText, FString TentativeText, bool bTranscriptionFinished)
```

| Parameter | Description |
|-----------|-------------|
| `CommittedText` | Words confirmed by Local Agreement (stable, will not change) |
| `TentativeText` | Words from the latest pass not yet confirmed (may change or disappear) |
| `bTranscriptionFinished` | `true` on the final result after speech ends. `TentativeText` is empty. |

### OnStreamingTextCommitted

Fires each time new words are confirmed by Local Agreement. Useful if you only care about confirmed text and want to ignore tentative previews.

```
OnStreamingTextCommitted(FString CommittedText, FString AccumulatedText)
```

| Parameter | Description |
|-----------|-------------|
| `CommittedText` | The word(s) just confirmed in this pass |
| `AccumulatedText` | Full accumulated text from the start of the utterance |

---

## Live UI Example (C++)

Bind `OnTranscriptionUpdated` to display committed text normally and tentative text as grey/italic preview. Clear the tentative text when the transcription finishes.

=== "C++"

    ```cpp
    void AMyHUD::BeginPlay()
    {
        Super::BeginPlay();

        UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
        STT->OnTranscriptionUpdated.AddDynamic(this, &AMyHUD::HandleTranscriptionUpdated);
    }

    void AMyHUD::HandleTranscriptionUpdated(
        const FString& CommittedText,
        const FString& TentativeText,
        bool bTranscriptionFinished)
    {
        // Committed text -- final, stable words
        CommittedTextBlock->SetText(FText::FromString(CommittedText));

        if (bTranscriptionFinished)
        {
            // Utterance complete -- clear tentative, use full text
            TentativeTextBlock->SetText(FText::GetEmpty());
            ProcessFinalTranscription(CommittedText);
        }
        else
        {
            // Still speaking -- show tentative as grey/italic preview
            TentativeTextBlock->SetText(FText::FromString(TentativeText));
        }
    }
    ```

=== "Blueprint"

    Bind `On Transcription Updated` on the Listener Component or Subsystem. Use `Committed Text` for a stable text block and `Tentative Text` for a greyed-out preview. Check `bTranscription Finished` to detect when the utterance is complete.

---

## Project Settings

These defaults are configured in **Project Settings > Aboleth Speech-to-Text > Streaming**:

| Setting | Default | Description |
|---------|---------|-------------|
| Enable Streaming | `true` | Master toggle for streaming transcription |
| Streaming Chunk Interval (ms) | `1000` | How often to send audio to Whisper during speech |
| Buffer Trimming Threshold (seconds) | `6` | Trim the streaming window when it exceeds this duration |
