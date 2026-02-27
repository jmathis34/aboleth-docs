# Quick Start

Three ways to get speech-to-text running, from simplest to most flexible.

---

## Option A: Drop-in Actor (Fastest)

1. Download a Whisper model via **Project Settings > Aboleth Speech-to-Text > Model Management**.
2. Drag `AbolethSTTListenerActor` into your level.
3. In the Details panel, bind `OnUtteranceProcessed` to receive transcribed text.
4. Play. Talk. Text appears.

No Blueprint wiring required. The actor auto-loads the Whisper model, opens the microphone, and starts listening on BeginPlay.

---

## Option B: Listener Component

1. Open any Blueprint actor (e.g. your PlayerCharacter).
2. Add Component > **Aboleth STT Listener**.
3. Bind `OnUtteranceProcessed` on the component.
4. Play.

Same behavior as the actor, but attached to an existing actor instead of standalone.

---

## Option C: Subsystem Direct (Full Control)

=== "C++"

    ```cpp
    // Get the GameInstanceSubsystem
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);

    // Load Whisper model + Silero VAD
    STT->LoadSTTSystem();

    // Start microphone and listening
    STT->StartListening();

    // Bind the transcription event
    STT->OnUtteranceProcessed.AddDynamic(this, &AMyActor::OnSpeechRecognized);
    ```

    ```cpp
    void AMyActor::OnSpeechRecognized(const FString& Text)
    {
        UE_LOG(LogTemp, Log, TEXT("Player said: %s"), *Text);
    }
    ```

=== "Blueprint"

    Use the built-in **Get Subsystem** node (GameInstance > AbolethSTTSubsystem) to access the subsystem, then call `Load STT System`, `Start Listening`, and bind `On Utterance Processed`.

---

## What's Next?

- [Architecture](architecture.md) — Understand how the pipeline works
- [Setup Methods](setup-methods.md) — Choose the right integration method
- [Project Settings](../configuration/project-settings.md) — Configure models, VAD, GPU, and more
- [Streaming](../features/streaming.md) — Enable live word-by-word transcription
