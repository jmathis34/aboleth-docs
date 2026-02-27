# Subsystem API

`UAbolethSTTSubsystem` is a **GameInstanceSubsystem** that serves as the central interface for all speech-to-text operations. It manages the full STT pipeline including model loading, microphone capture, VAD, and transcription.

## Accessing the Subsystem

=== "C++"

    ```cpp
    #include "AbolethSTTSubsystem.h"

    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(WorldContextObject);
    if (STT && STT->IsSTTReady())
    {
        STT->StartListening();
    }
    ```

=== "Blueprint"

    Use the **Get Subsystem** node with class `AbolethSTTSubsystem`. Since this is a
    `GameInstanceSubsystem`, it is available from any world context.

---

## Core Lifecycle

Functions for loading, starting, and stopping the STT system.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `LoadSTTSystem` | BlueprintCallable | `bool` | Loads the Whisper model and initializes the full pipeline. Returns `true` on success. |
| `UnloadSTTSystem` | BlueprintCallable | `void` | Tears down the pipeline and releases all model resources. |
| `IsSTTReady` | BlueprintPure | `bool` | Returns `true` if the model is loaded and the pipeline is ready to accept audio. |
| `StartListening` | BlueprintCallable | `void` | Opens the microphone and begins VAD-driven capture. |
| `StopListening` | BlueprintCallable | `void` | Stops microphone capture and halts processing. |
| `IsMicrophoneActive` | BlueprintPure | `bool` | Returns `true` if the microphone is currently capturing audio. |
| `ForceResetProcessing` | BlueprintCallable | `void` | Aborts any in-flight transcription and resets the pipeline to `Idle`. |

---

## State Queries

Read-only queries for inspecting the current state of the STT pipeline.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `GetPipelineState` | BlueprintPure | `EAbolethPipelineState` | Current pipeline state (Idle, SpeechDetected, Processing, etc.). |
| `IsSpeechDetected` | BlueprintPure | `bool` | Returns `true` if the VAD is currently detecting speech. |
| `IsProcessingSTT` | BlueprintPure | `bool` | Returns `true` if a transcription pass is in progress. |
| `GetCurrentRMSLevel` | BlueprintPure | `float` | Current audio RMS level. **Metering only** --- not used for VAD gating. |
| `GetCurrentVADProbability` | BlueprintPure | `float` | Raw Silero VAD probability (0.0--1.0) for the most recent window. |
| `GetCurrentUtteranceDuration` | BlueprintPure | `float` | Duration in seconds of the current utterance being accumulated. |
| `GetTimeSinceLastSpeech` | BlueprintPure | `float` | Seconds elapsed since the last detected speech frame. |
| `GetAccumulatedSampleCount` | BlueprintPure | `int32` | Number of audio samples buffered for the current utterance. |
| `GetLastTranscribedText` | BlueprintPure | `FString` | The most recent final transcription result. |

!!! info "RMS vs VAD"
    `GetCurrentRMSLevel` is provided for audio-level UI display (volume meters, etc.). Speech detection is handled entirely by the Silero VAD model via `GetCurrentVADProbability`.

---

## Audio Status

Query the state of the audio capture hardware and buffers.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `GetMicrophoneStatus` | BlueprintPure | `FAudioCaptureStatus` | Full capture status including audio levels, queue depth, and pipeline state. |
| `GetActiveMic` | BlueprintPure | `UMicCaptureComponent*` | Pointer to the currently active microphone capture component. |
| `GetUnifiedAudioBuffer` | BlueprintPure | `UAbolethUnifiedAudioBuffer*` | Access the shared audio ring buffer used across the pipeline. |

---

## Manual Processing

Manually trigger transcription outside the standard VAD-driven pipeline.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `ProcessUtteranceAsync` | BlueprintCallable | `void` | Submits the current audio buffer for async transcription. Results arrive via `OnUtteranceProcessed`. |
| `ProcessUtteranceImmediate` | BlueprintCallable | `FString` | **Blocking.** Transcribes the current buffer and returns the result immediately. Use with caution on the game thread. |
| `ResetVADState` | BlueprintCallable | `void` | Clears accumulated VAD state and audio buffers without stopping capture. |

!!! warning "Blocking Call"
    `ProcessUtteranceImmediate` blocks the calling thread until transcription completes. Prefer `ProcessUtteranceAsync` for gameplay code.

---

## Language

Configure the target language for transcription and translation.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `SetLanguage` | BlueprintCallable | `void` | Set the source language code (e.g., `"en"`, `"ja"`, `"de"`). Use `"auto"` for automatic detection. |
| `GetLanguage` | BlueprintPure | `FString` | Returns the currently configured language code. |
| `SetTranslateToEnglish` | BlueprintCallable | `void` | Enable or disable Whisper's built-in translation-to-English mode. |
| `GetTranslateToEnglish` | BlueprintPure | `bool` | Returns `true` if translation to English is active. |
| `GetAvailableLanguages` | Static | `TArray<FString>` | Returns all language codes supported by the loaded Whisper model. |

---

## Runtime Settings

Adjust VAD sensitivity, microphone gain, and debug options at runtime.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `SetVADThreshold` | BlueprintCallable | `void` | Set the Silero VAD speech probability threshold (0.0--1.0). Default: `0.5`. |
| `GetVADThreshold` | BlueprintPure | `float` | Returns the current VAD threshold. |
| `SetMicGainDb` | BlueprintCallable | `void` | Set microphone gain in decibels. `0.0` = unity gain. |
| `GetMicGainDb` | BlueprintPure | `float` | Returns the current microphone gain in dB. |
| `ReloadVADSettings` | BlueprintCallable | `void` | Re-reads VAD parameters from project settings and applies them. |
| `SetDebugLogging` | BlueprintCallable | `void` | Enable or disable verbose STT debug logging at runtime. |
| `IsDebugLoggingEnabled` | BlueprintPure | `bool` | Returns `true` if debug logging is active. |

---

## Capture Mode

Switch between VAD-driven automatic capture and push-to-talk.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `SetCaptureMode` | BlueprintCallable | `void` | Switch between `VADAutomatic` and `PushToTalk`. |
| `GetCaptureMode` | BlueprintPure | `EAbolethCaptureMode` | Returns the active capture mode. |
| `StartManualCapture` | BlueprintCallable | `void` | Begin recording (push-to-talk). Only valid when capture mode is `PushToTalk`. |
| `StopManualCapture` | BlueprintCallable | `void` | End recording and submit audio for transcription. |
| `IsManualCaptureActive` | BlueprintPure | `bool` | Returns `true` if a manual capture session is in progress. |

---

## Streaming

Control real-time streaming transcription with Local Agreement confirmation.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `SetStreamingEnabled` | BlueprintCallable | `void` | Enable or disable streaming transcription. |
| `IsStreamingEnabled` | BlueprintPure | `bool` | Returns `true` if streaming is active. |
| `SetStreamingChunkIntervalMs` | BlueprintCallable | `void` | Set the interval between streaming snapshot passes in milliseconds. |
| `GetStreamingChunkIntervalMs` | BlueprintPure | `int32` | Returns the current streaming chunk interval. |
| `GetStreamingPassCount` | BlueprintPure | `int32` | Number of streaming passes executed for the current utterance. |
| `GetStreamingAccumulatedText` | BlueprintPure | `FString` | Returns all confirmed (committed) text from the current streaming session. |

!!! tip "Local Agreement"
    Streaming uses Local Agreement with n=2 to confirm words. Only words that appear consistently across two consecutive snapshot passes are committed, reducing hallucination in partial results.

---

## Beam Search

Configure beam search decoding for higher-quality transcription at the cost of latency.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `SetBeamSearchEnabled` | BlueprintCallable | `void` | Enable or disable beam search decoding. |
| `IsBeamSearchEnabled` | BlueprintPure | `bool` | Returns `true` if beam search is active. |
| `SetBeamSize` | BlueprintCallable | `void` | Set the number of beams (e.g., 5). Higher values improve quality but increase latency. |
| `GetBeamSize` | BlueprintPure | `int32` | Returns the current beam size. |
| `SetLengthPenalty` | BlueprintCallable | `void` | Set the length penalty factor for beam search scoring. |
| `GetLengthPenalty` | BlueprintPure | `float` | Returns the current length penalty. |
| `SetBeamSearchDuringStreaming` | BlueprintCallable | `void` | Allow beam search to run during streaming snapshot passes. |
| `IsBeamSearchDuringStreaming` | BlueprintPure | `bool` | Returns `true` if beam search is used during streaming. |
| `SetBeamSearchGateEnabled` | BlueprintCallable | `void` | Enable duration-based gating --- beam search only activates after a minimum utterance length. |
| `IsBeamSearchGateEnabled` | BlueprintPure | `bool` | Returns `true` if the beam search gate is active. |
| `SetBeamSearchGateMs` | BlueprintCallable | `void` | Minimum utterance duration in milliseconds before beam search activates. |
| `GetBeamSearchGateMs` | BlueprintPure | `int32` | Returns the current beam search gate threshold. |

---

## Audio Devices

Enumerate and select audio input devices at runtime.

| Function | Specifier | Return | Description |
|---|---|---|---|
| `RefreshAudioDevices` | BlueprintCallable | `void` | Re-enumerates available audio input devices. Fires `OnAudioDevicesRefreshed`. |
| `GetAvailableAudioDevices` | BlueprintPure | `TArray<FAudioDeviceInfo>` | Returns all detected audio input devices. |
| `SetAudioDeviceByIndex` | BlueprintCallable | `void` | Switch to a specific device by its index in the device list. |
| `UseDefaultAudioDevice` | BlueprintCallable | `void` | Revert to the system default audio input device. |
| `GetSelectedDeviceIndex` | BlueprintPure | `int32` | Returns the index of the currently selected device. `-1` for system default. |
| `GetSelectedDeviceInfo` | BlueprintPure | `FAudioDeviceInfo` | Returns full info for the currently selected audio device. |

---

## Static Helpers (C++ Only)

| Function | Scope | Return | Description |
|---|---|---|---|
| `GetSTTSubsystem` | Static | `UAbolethSTTSubsystem*` | Retrieves the subsystem from any world context object. Returns `nullptr` if unavailable. |

!!! note "Blueprint Access"
    `GetSTTSubsystem` is **C++ only**. In Blueprint, use the engine-provided **Get Subsystem** node and select `AbolethSTTSubsystem` as the class. This works because `UAbolethSTTSubsystem` is a `UGameInstanceSubsystem`.
