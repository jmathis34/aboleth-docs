# Listener Component

`UAbolethListenerComponent` provides the same STT functionality as the [Listener Actor](listener-actor.md), but as an `UActorComponent` that can be attached to any actor in your game.

## Adding the Component

=== "Blueprint"

    1. Select any actor in the level or open a Blueprint actor.
    2. Click **Add Component** in the Details or Components panel.
    3. Search for **Aboleth STT Listener** and add it.
    4. The component auto-activates with the owning actor.

=== "C++"

    ```cpp
    // In your actor's constructor
    AMyCharacter::AMyCharacter()
    {
        ListenerComp = CreateDefaultSubobject<UAbolethListenerComponent>(TEXT("STTListener"));
    }

    // In BeginPlay — bind events
    void AMyCharacter::BeginPlay()
    {
        Super::BeginPlay();

        ListenerComp->OnUtteranceProcessed.AddDynamic(
            this, &AMyCharacter::HandleSpeech);

        ListenerComp->OnTranscriptionUpdated.AddDynamic(
            this, &AMyCharacter::HandleLiveText);
    }
    ```

---

## Configuration Properties

Same override properties as the Listener Actor. Configure per-instance in the Details panel.

| Property | Type | Default | Description |
|---|---|---|---|
| `bAutoActivate` | `bool` | `true` | Inherited from `UActorComponent`. Activates and begins listening when the owning actor starts. |
| `bOverrideVADThreshold` | `bool` | `false` | When `true`, use `VADThresholdOverride` instead of the project-wide setting. |
| `VADThresholdOverride` | `float` | `0.5` | Per-instance VAD speech probability threshold (0.0--1.0). Only applied when `bOverrideVADThreshold` is `true`. |
| `bOverrideAudioDevice` | `bool` | `false` | When `true`, use `AudioDeviceIndexOverride` instead of the system default. |
| `AudioDeviceIndexOverride` | `int32` | `-1` | Index of the audio input device to use. `-1` uses the system default. Only applied when `bOverrideAudioDevice` is `true`. |
| `bOverrideCaptureMode` | `bool` | `false` | When `true`, use `CaptureModeOverride` instead of the project-wide setting. |
| `CaptureModeOverride` | `EAbolethCaptureMode` | `VADAutomatic` | Capture mode for this instance. Only applied when `bOverrideCaptureMode` is `true`. |

---

## Functions

| Function | Specifier | Return | Description |
|---|---|---|---|
| `Activate` | BlueprintCallable | `bool` | Loads the STT system (if needed), opens the mic, and begins listening. Returns `true` on success. |
| `Deactivate` | BlueprintCallable | `void` | Stops listening and releases the microphone. |
| `IsActive` | BlueprintPure | `bool` | Returns `true` if the component is currently listening. |
| `GetMicComponent` | BlueprintPure | `UMicCaptureComponent*` | Returns the internal microphone capture component. |
| `StartManualCapture` | BlueprintCallable | `void` | Begin push-to-talk recording. Only valid when capture mode is `PushToTalk`. |
| `StopManualCapture` | BlueprintCallable | `void` | End push-to-talk recording and submit audio for transcription. |
| `IsManualCaptureActive` | BlueprintPure | `bool` | Returns `true` if a manual capture session is in progress. |
| `SetCaptureMode` | BlueprintCallable | `void` | Switch between `VADAutomatic` and `PushToTalk` at runtime. |
| `GetCaptureMode` | BlueprintPure | `EAbolethCaptureMode` | Returns the active capture mode. |

---

## Events

The Listener Component exposes all standard events (see [Events & Delegates](events.md)) plus an additional streaming convenience delegate:

| Event | Parameters | Description |
|---|---|---|
| `OnTranscriptionUpdated` | `FString CommittedText`, `FString TentativeText`, `bool bTranscriptionFinished` | Fires during streaming with both confirmed and tentative text. Ideal for driving live subtitle UI directly from the component. |

All other events (`OnUtteranceProcessed`, `OnSpeechDetected`, `OnSilenceDetected`, etc.) are identical to the Listener Actor and subsystem.

---

## Component vs Actor

| | Listener Actor | Listener Component |
|---|---|---|
| **Use case** | Standalone, drop into a level | Attach to an existing actor (player, NPC, UI manager) |
| **Placement** | Drag into the level from Content Browser | Add via **Add Component** in the Details panel |
| **Auto-activate** | `bAutoActivate` property (default `true`) | Inherits `bAutoActivate` from `UActorComponent` (default `true`) |
| **Streaming event** | Via subsystem binding | `OnTranscriptionUpdated` exposed directly |
| **Typical owner** | Level, GameMode | PlayerCharacter, PlayerController, HUD |

---

## Notes

- **One active listener at a time.** The underlying subsystem is a singleton --- only one Listener Component or Listener Actor should be active per game instance.
- `GetSTTSubsystem()` is **C++ only**. In Blueprint, use the engine **Get Subsystem** node.
- `bAutoActivate` is inherited from `UActorComponent` and defaults to `true`. Set it to `false` if you want to activate the component manually via `Activate()`.
