# Runtime Settings API

All runtime setters take effect **immediately** without reloading the Whisper model or restarting the microphone. These are available on the `UAbolethSTTSubsystem` and are exposed to both C++ and Blueprint.

!!! tip "When to use runtime settings"
    Runtime settings are ideal for options menus, accessibility features, or adapting to changing conditions mid-session. For example, you might expose VAD threshold and mic gain as player-facing sliders.

---

## VAD

| Setter | Getter | Type | Description |
|--------|--------|------|-------------|
| `SetVADThreshold(float)` | `GetVADThreshold()` | Float | Speech probability threshold (0.0 -- 1.0). |

??? example "Adjusting VAD sensitivity from Blueprint"
    Call **Set VAD Threshold** on the subsystem node. A value of `0.7` works well for noisy environments; `0.4` is better for quiet rooms where short utterances need to be captured.

`ReloadVADSettings()` pushes all VAD-related project settings (threshold, durations, padding) to the active Silero instance in a single call. Use this after modifying multiple VAD parameters in `DefaultGame.ini` or through a settings UI that writes to the developer settings object directly.

---

## Audio

| Setter | Getter | Type | Description |
|--------|--------|------|-------------|
| `SetMicGainDb(float)` | `GetMicGainDb()` | Float | Software microphone gain in decibels (0.0 -- 20.0). |

!!! note
    Gain is applied in the audio pipeline before VAD processing. Increasing gain amplifies both speech and noise, so you may need to raise the VAD threshold if you increase gain significantly.

---

## Language

| Setter | Getter | Type | Description |
|--------|--------|------|-------------|
| `SetLanguage(FString)` | `GetLanguage()` | String | Language code (e.g. `"en"`, `"ja"`, `"auto"`). |
| `SetTranslateToEnglish(bool)` | `GetTranslateToEnglish()` | Bool | Enable or disable translation to English. |

---

## Capture Mode

| Setter | Getter | Type | Description |
|--------|--------|------|-------------|
| `SetCaptureMode(EAbolethCaptureMode)` | `GetCaptureMode()` | Enum | Switch between `VoiceActivity` (auto-detect speech) and `PushToTalk` (manual trigger). |

---

## Streaming

| Setter | Getter | Type | Description |
|--------|--------|------|-------------|
| `SetStreamingEnabled(bool)` | `IsStreamingEnabled()` | Bool | Toggle Local Agreement streaming on or off. |
| `SetStreamingChunkIntervalMs(int)` | `GetStreamingChunkIntervalMs()` | Int | Interval between streaming passes in milliseconds (50 -- 2000). |

---

## Beam Search

| Setter | Getter | Type | Description |
|--------|--------|------|-------------|
| `SetBeamSearchEnabled(bool)` | `IsBeamSearchEnabled()` | Bool | Toggle beam search on the final pass. |
| `SetBeamSize(int)` | `GetBeamSize()` | Int | Number of beams (1 -- 10). |
| `SetLengthPenalty(float)` | `GetLengthPenalty()` | Float | Length penalty (-1.0 to disable, up to 3.0). |
| `SetBeamSearchDuringStreaming(bool)` | `IsBeamSearchDuringStreaming()` | Bool | Toggle beam search during streaming passes. |
| `SetBeamSearchGateEnabled(bool)` | `IsBeamSearchGateEnabled()` | Bool | Toggle adaptive beam dropout. |
| `SetBeamSearchGateMs(int)` | `GetBeamSearchGateMs()` | Int | Gate threshold in milliseconds (50 -- 2000). |

---

## Debug

| Setter | Getter | Type | Description |
|--------|--------|------|-------------|
| `SetDebugLogging(bool)` | `IsDebugLoggingEnabled()` | Bool | Toggle per-utterance diagnostic logging. |

---

## Example: Options Menu

=== "C++"

    ```cpp
    void UOptionsWidget::ApplyVoiceSettings()
    {
        UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);

        STT->SetVADThreshold(VADThresholdSlider->GetValue());
        STT->SetMicGainDb(MicGainSlider->GetValue());
        STT->SetLanguage(LanguageComboBox->GetSelectedOption());
        STT->SetStreamingEnabled(StreamingToggle->IsChecked());
    }
    ```

=== "Blueprint"

    Get the subsystem via **Get Subsystem** (GameInstance > AbolethSTTSubsystem), then wire your UI widget values into the corresponding **Set** nodes. All changes apply instantly.
