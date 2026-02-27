# Listener Actor

`AAbolethListenerActor` is a ready-made actor that provides zero-configuration speech-to-text when placed in a level. It wraps the full STT pipeline --- microphone capture, VAD, and transcription --- so you can start receiving speech events without writing any setup code.

## Quick Start

1. Drag **AbolethListenerActor** from the Content Browser into your level.
2. Press **Play**.
3. The actor auto-activates, loads the STT system, and begins listening.

All transcription events are exposed as **BlueprintAssignable** delegates directly on the actor.

---

## Configuration Properties

Override these in the Details panel to customize behavior per-instance.

| Property | Type | Default | Description |
|---|---|---|---|
| `bAutoActivate` | `bool` | `true` | Automatically activate and begin listening on `BeginPlay`. |
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
| `Deactivate` | BlueprintCallable | `void` | Stops listening and releases the microphone. Does **not** unload the model. |
| `IsActive` | BlueprintPure | `bool` | Returns `true` if the actor is currently listening. |
| `GetMicComponent` | BlueprintPure | `UMicCaptureComponent*` | Returns the internal microphone capture component. |
| `StartManualCapture` | BlueprintCallable | `void` | Begin push-to-talk recording. Only valid when capture mode is `PushToTalk`. |
| `StopManualCapture` | BlueprintCallable | `void` | End push-to-talk recording and submit audio for transcription. |
| `IsManualCaptureActive` | BlueprintPure | `bool` | Returns `true` if a manual capture session is in progress. |
| `SetCaptureMode` | BlueprintCallable | `void` | Switch between `VADAutomatic` and `PushToTalk` at runtime. |
| `GetCaptureMode` | BlueprintPure | `EAbolethCaptureMode` | Returns the active capture mode. |

---

## Events

All events are exposed as **BlueprintAssignable** delegates. Bind them in the Details panel or via `AddDynamic` in C++.

The Listener Actor exposes the same event set as the subsystem. See [Events & Delegates](events.md) for the full delegate reference including parameter signatures.

---

## Usage Examples

=== "Blueprint"

    1. Place an **AbolethListenerActor** in the level.
    2. Select it and open the **Details** panel.
    3. Scroll to the **Events** section.
    4. Click **+** next to **OnUtteranceProcessed** and bind it to your handler.
    5. The actor auto-activates on play --- no further setup required.

=== "C++"

    ```cpp
    // In your level setup or game mode
    AAbolethListenerActor* Listener = GetWorld()->SpawnActor<AAbolethListenerActor>();

    Listener->OnUtteranceProcessed.AddDynamic(this, &AMyGameMode::HandleSpeech);
    Listener->OnSpeechDetected.AddDynamic(this, &AMyGameMode::HandleSpeechStart);
    Listener->OnSilenceDetected.AddDynamic(this, &AMyGameMode::HandleSilence);

    // Auto-activates by default, or call manually:
    Listener->Activate();
    ```

---

## Notes

- **One listener at a time.** Only one Listener Actor (or Listener Component) should be active per game instance. The underlying subsystem is a singleton.
- `GetSTTSubsystem()` is **C++ only**. For Blueprint access to the subsystem, use the engine **Get Subsystem** node.
- Deactivating the actor stops capture but leaves the Whisper model loaded. Call `UnloadSTTSystem` on the subsystem to fully release resources.
