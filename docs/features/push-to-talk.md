# Push-to-Talk

Aboleth STT supports a manual **Push-to-Talk (PTT)** capture mode alongside the default VAD-automatic mode. In PTT mode, you control exactly when speech capture starts and stops -- typically bound to a key press and release.

---

## Setup

Switch to Push-to-Talk mode, then call `StartManualCapture()` / `StopManualCapture()` on key press/release:

=== "C++"

    ```cpp
    // During initialization
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
    STT->SetCaptureMode(EAbolethCaptureMode::PushToTalk);

    // On key press
    void AMyCharacter::OnPTTPressed()
    {
        UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
        STT->StartManualCapture();
    }

    // On key release
    void AMyCharacter::OnPTTReleased()
    {
        UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);
        STT->StopManualCapture();
    }
    ```

=== "Blueprint"

    1. Call **Set Capture Mode** on the Subsystem (or Listener Component) and select **Push-to-Talk**.
    2. Bind your PTT key's **Pressed** event to **Start Manual Capture**.
    3. Bind your PTT key's **Released** event to **Stop Manual Capture**.
    4. Bind **On Utterance Processed** (or **On Transcription Updated** if streaming) to receive the transcribed text.

---

## Listener Component / Actor Relay

The Listener Component and Listener Actor both expose PTT relay functions, so you can call them directly on the component without reaching for the subsystem:

| Function | Description |
|----------|-------------|
| `StartManualCapture()` | Begin PTT capture (relays to subsystem) |
| `StopManualCapture()` | End PTT capture (relays to subsystem) |
| `IsManualCaptureActive()` | Returns `true` while a PTT capture is in progress |
| `SetCaptureMode(EAbolethCaptureMode)` | Switch between VAD Automatic and Push-to-Talk |
| `GetCaptureMode()` | Get the current capture mode |

---

## Behavior Details

### The microphone is always recording

The microphone and ring buffer run continuously regardless of capture mode. PTT does not start or stop the microphone -- it simply marks **segment boundaries** within the ongoing audio stream. This means:

- There is no microphone startup latency when the user presses the PTT key.
- The ring buffer already contains recent audio, so the segment start can reach back slightly for natural speech onset.

### VAD still runs during PTT

Silero VAD continues to evaluate speech probability every frame in PTT mode. However, it does not control segment boundaries -- that is entirely manual. The VAD probability is still broadcast via `OnVADProbability` so you can use it for UI indicators (e.g. a voice activity meter) even in PTT mode.

### Streaming works during PTT

If streaming is enabled, it activates during PTT captures. Words are confirmed via Local Agreement just as in VAD-automatic mode. `OnTranscriptionUpdated` fires with committed and tentative text throughout the PTT capture.

### Deferred capture during Processing

If the user presses the PTT key while the pipeline is in the **Processing** state (i.e. a previous utterance is still being transcribed), the capture request is deferred:

- `StartManualCapture()` records the ring buffer position and queues the capture.
- When Processing completes and the pipeline returns to Idle, the deferred capture begins automatically from the recorded position.
- If **both** `StartManualCapture()` and `StopManualCapture()` are called during Processing, the entire segment is queued and processed as soon as the current transcription finishes.

!!! info "No audio is lost"
    Because the ring buffer is always recording, audio spoken during the Processing state is not lost. The deferred capture extracts from the correct ring buffer positions even if there was a delay before processing began.

### Minimum duration

If the captured PTT segment is shorter than the minimum viable duration, it is silently discarded to avoid sending noise or accidental key taps to Whisper.

---

## Capture Modes

| Mode | Enum Value | Behavior |
|------|-----------|----------|
| VAD Automatic | `EAbolethCaptureMode::VADAutomatic` | Silero VAD detects speech onset/offset automatically |
| Push-to-Talk | `EAbolethCaptureMode::PushToTalk` | Manual start/stop via `StartManualCapture()` / `StopManualCapture()` |

Switch modes at any time with `SetCaptureMode()`. The change takes effect immediately.
