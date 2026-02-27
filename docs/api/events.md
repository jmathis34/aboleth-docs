# Events & Delegates

All Aboleth STT events are declared as `DECLARE_DYNAMIC_MULTICAST_DELEGATE` and exposed as `BlueprintAssignable` properties. They are available on the **subsystem**, **Listener Actor**, and **Listener Component**.

---

## Transcription Events

Core events for receiving transcription results.

| Event | Parameters | Description |
|---|---|---|
| `OnUtteranceProcessed` | `FString TranscribedText` | **The main event.** Fires when a complete utterance has been transcribed. This is the final, confirmed text after VAD detects silence and the full audio segment is processed. |
| `OnSTTProcessingStarted` | `float UtteranceDuration` | Fires when audio is submitted to Whisper for transcription. `UtteranceDuration` is the length of the audio segment in seconds. |
| `OnSTTProcessingFailed` | `FString ErrorMessage` | Fires if transcription fails. Contains a human-readable error description. |

---

## Streaming Events

Events for real-time streaming transcription. Only fire when streaming is enabled.

| Event | Parameters | Description |
|---|---|---|
| `OnTranscriptionUpdated` | `FString CommittedText`, `FString TentativeText`, `bool bTranscriptionFinished` | **Ideal for live UI.** Fires on every streaming snapshot. `CommittedText` contains words confirmed by Local Agreement. `TentativeText` contains unconfirmed words from the latest pass. `bTranscriptionFinished` is `true` on the final update. |
| `OnStreamingTextCommitted` | `FString CommittedText`, `FString AccumulatedText` | Fires when new words are confirmed by Local Agreement. `CommittedText` is the newly confirmed segment. `AccumulatedText` is all confirmed text so far. |

!!! tip "Choosing a Streaming Event"
    Use `OnTranscriptionUpdated` for subtitle-style UI where you want to show both stable and in-progress text. Use `OnStreamingTextCommitted` when you only need confirmed words (e.g., feeding into a command parser).

---

## VAD Events

Events driven by the Silero Voice Activity Detection model.

| Event | Parameters | Description |
|---|---|---|
| `OnSpeechDetected` | `float InitialRMS` | Fires when the VAD transitions from silence to speech. `InitialRMS` is the audio level at the moment of detection (metering value, not a VAD input). |
| `OnSilenceDetected` | --- | Fires when the VAD transitions from speech to silence after the configured minimum silence duration. |
| `OnVADUpdate` | `float CurrentRMS`, `bool bIsSpeech` | Periodic update with current audio level and VAD speech state. Useful for driving visual indicators. |
| `OnVADProbability` | `float Probability`, `bool bIsSpeech` | Raw Silero VAD output. Fires approximately 31 times per second (one per 512-sample window at 16kHz). `Probability` is the speech probability (0.0--1.0). |
| `OnUtteranceReady` | `int32 SampleCount` | Fires when a complete utterance has been captured and is ready for transcription. `SampleCount` is the total number of audio samples in the utterance. |

---

## System Events

Lifecycle and device-change notifications.

| Event | Parameters | Description |
|---|---|---|
| `OnSTTSystemLoaded` | `bool bSuccess` | Fires after `LoadSTTSystem` completes. `bSuccess` indicates whether the model loaded successfully. |
| `OnPipelineStateChanged` | `EAbolethPipelineState OldState`, `EAbolethPipelineState NewState` | Fires on every pipeline state transition. See [Structs & Enums](structs-enums.md#eabolethpipelinestate) for state values. |
| `OnAudioDeviceChanged` | `FAudioDeviceInfo NewDevice` | Fires when the active audio input device changes (via `SetAudioDeviceByIndex` or hardware hot-plug). |
| `OnAudioDevicesRefreshed` | --- | Fires after `RefreshAudioDevices` completes enumeration. |

---

## Blueprint Binding

To bind events in Blueprint:

1. Place an **AbolethListenerActor** in the level or add an **AbolethListenerComponent** to an actor.
2. Select the actor/component and open the **Details** panel.
3. Scroll to the **Events** section.
4. Click the **+** button next to the event you want to handle (e.g., `OnUtteranceProcessed`).
5. The Event Graph opens with a new event node. Connect your logic to it.
6. For `OnTranscriptionUpdated`, wire `CommittedText` and `TentativeText` into a **Text Block** or subtitle widget for live display.

---

## C++ Binding

### Basic Transcription

```cpp
void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
    if (!STT) return;

    STT->OnUtteranceProcessed.AddDynamic(this, &AMyActor::HandleUtterance);
    STT->OnSpeechDetected.AddDynamic(this, &AMyActor::HandleSpeechStart);
    STT->OnSilenceDetected.AddDynamic(this, &AMyActor::HandleSilenceStart);
    STT->OnPipelineStateChanged.AddDynamic(this, &AMyActor::HandleStateChange);

    STT->LoadSTTSystem();
    STT->StartListening();
}

void AMyActor::HandleUtterance(const FString& TranscribedText)
{
    UE_LOG(LogTemp, Log, TEXT("Player said: %s"), *TranscribedText);
    // Feed into dialogue system, command parser, etc.
}

void AMyActor::HandleSpeechStart(float InitialRMS)
{
    UE_LOG(LogTemp, Log, TEXT("Speech started (RMS: %.3f)"), InitialRMS);
    // Show "listening" indicator in UI
}

void AMyActor::HandleSilenceStart()
{
    UE_LOG(LogTemp, Log, TEXT("Silence detected"));
    // Hide "listening" indicator
}

void AMyActor::HandleStateChange(EAbolethPipelineState OldState,
                                  EAbolethPipelineState NewState)
{
    UE_LOG(LogTemp, Log, TEXT("Pipeline: %s -> %s"),
        *UEnum::GetValueAsString(OldState),
        *UEnum::GetValueAsString(NewState));
}
```

### Live Streaming Text

```cpp
void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
    if (!STT) return;

    STT->OnTranscriptionUpdated.AddDynamic(this, &AMyActor::HandleLiveText);

    STT->SetStreamingEnabled(true);
    STT->LoadSTTSystem();
    STT->StartListening();
}

void AMyActor::HandleLiveText(const FString& CommittedText,
                               const FString& TentativeText,
                               bool bTranscriptionFinished)
{
    // CommittedText  = confirmed words (stable, won't change)
    // TentativeText  = unconfirmed words (may change on next pass)

    FString DisplayText = CommittedText;
    if (!TentativeText.IsEmpty())
    {
        // Show tentative text in a different style (e.g., grey/italic)
        DisplayText += TEXT(" ") + TentativeText;
    }

    SubtitleWidget->SetText(FText::FromString(DisplayText));

    if (bTranscriptionFinished)
    {
        // Final result — tentative text is empty, committed is complete
        UE_LOG(LogTemp, Log, TEXT("Final: %s"), *CommittedText);
    }
}
```

!!! info "Delegate Lifetime"
    Delegates bound via `AddDynamic` are automatically cleaned up when the bound object is destroyed. No manual unbinding is required in most cases.
