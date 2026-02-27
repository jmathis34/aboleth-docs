# Language & Translation

Aboleth STT supports all languages recognized by the underlying Whisper model. You can lock recognition to a specific language for faster, more accurate results, or use auto-detection to let Whisper identify the language from the audio.

Translation from any supported language to English is also available with multilingual models.

---

## Setting the Language

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);

    // Auto-detect language from audio (default)
    STT->SetLanguage("auto");

    // Lock to a specific language
    STT->SetLanguage("en");   // English
    STT->SetLanguage("de");   // German
    STT->SetLanguage("ja");   // Japanese
    STT->SetLanguage("es");   // Spanish

    // Check current language
    FString Current = STT->GetLanguage();  // Returns "auto", "en", etc.
    ```

=== "Blueprint"

    Call **Set Language** on the Subsystem with a language code string (`"auto"`, `"en"`, `"de"`, etc.). Use **Get Language** to read the current setting.

!!! tip "Lock the language when you know it"
    Setting a specific language code (e.g. `"en"`) instead of `"auto"` skips the language detection step and produces faster, more accurate results. Use `"auto"` only when the input language is genuinely unknown.

### API Reference

| Function | Description |
|----------|-------------|
| `SetLanguage(FString)` | Set recognition language (`"auto"` or a language code) |
| `GetLanguage()` | Get the current language code |
| `GetAvailableLanguages()` | Returns all language codes supported by Whisper (static) |

### Listing Available Languages

Call `GetAvailableLanguages()` to retrieve the full list of supported language codes at runtime:

=== "C++"

    ```cpp
    TArray<FString> Languages = UAbolethSTTSubsystem::GetAvailableLanguages();
    for (const FString& Code : Languages)
    {
        UE_LOG(LogTemp, Log, TEXT("Supported: %s"), *Code);
    }
    ```

=== "Blueprint"

    Call **Get Available Languages** (static) to get an array of language code strings.

---

## Translation to English

Whisper can translate non-English speech directly to English during transcription. This runs as part of the same inference pass -- no separate translation step is needed.

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);

    // Enable translation (speak German, get English text)
    STT->SetTranslateToEnglish(true);

    // Check if translation is enabled
    bool bTranslating = STT->GetTranslateToEnglish();
    ```

=== "Blueprint"

    Call **Set Translate to English** (`true`) on the Subsystem. Use **Get Translate to English** to check the current state.

| Function | Description |
|----------|-------------|
| `SetTranslateToEnglish(bool)` | Enable or disable translation to English |
| `GetTranslateToEnglish()` | Check if translation is active |

!!! warning "Requires a multilingual model"
    Translation only works with **multilingual** Whisper models (e.g. `ggml-large-v3-turbo-q5_0.bin`). English-only models (filenames ending in `.en`) do not support translation and will ignore this setting.

---

## Project Settings

Language defaults are configured in **Project Settings > Aboleth Speech-to-Text > Language**:

| Setting | Default | Description |
|---------|---------|-------------|
| Language | `auto` | Language code for recognition, or `"auto"` for auto-detection |
| Translate to English | `false` | Translate non-English speech to English during transcription |
