# Setup Methods

Aboleth STT provides three ways to integrate, from simplest to most flexible.

---

## AAbolethListenerActor

A standalone actor you drop into any level. Owns its own `MicCaptureComponent`. Auto-activates on BeginPlay by default.

**When to use:** Prototyping, simple voice command systems, one-actor-per-level setups.

---

## UAbolethListenerComponent

An `ActorComponent` you attach to any existing actor. Functionally identical to the Listener Actor but lives on your PlayerCharacter, HUD, or whatever you need.

**When to use:** Most gameplay scenarios. Attach to your character or game manager.

---

## UAbolethSTTSubsystem (Direct)

The `GameInstanceSubsystem` that owns the entire pipeline. The Listener Actor and Component are convenience wrappers around this. Use directly for maximum control.

**When to use:** Custom audio pipelines, multiple microphone management, non-standard architectures.

---

## Comparison

All three expose the same events and runtime API. The Actor and Component relay everything from the subsystem.

| Method | Setup Effort | Best For |
|--------|-------------|----------|
| Listener Actor | Drag & drop | Prototyping, simple setups |
| Listener Component | Add Component | Most gameplay scenarios |
| Subsystem Direct | Manual C++ | Full control, custom pipelines |
