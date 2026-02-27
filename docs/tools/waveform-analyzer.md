# Silero Waveform Analyzer

An interactive visualization tool for inspecting Silero VAD probability logs. Use it to fine-tune VAD thresholds and diagnose speech detection behavior.

<div style="margin: 1em 0;">
  <a href="../SileroWaveformAnalyzer.html" target="_blank" class="md-button md-button--primary">
    Open Full-Screen Analyzer
  </a>
</div>

---

## How to Use

1. **Enable Probability Logging** in Project Settings:

    **Project Settings > Plugins > Aboleth Speech-to-Text > Debug > Enable Probability Logging**

2. **Run your game** and speak into the microphone. The plugin writes per-frame VAD data to:

    ```
    Plugins/AbolethSTT/Tools/silero_probabilities.csv
    ```

3. **Open the analyzer** using the button above (or the embedded view below) and drag your CSV file onto it.

---

## Embedded Analyzer

<iframe src="../SileroWaveformAnalyzer.html" style="width: 100%; height: 600px; border: 1px solid #333; border-radius: 8px;" allowfullscreen></iframe>

---

## What You Can See

| Element | Description |
|---------|-------------|
| **Green waveform** | Raw Silero speech probability (0.0 -- 1.0) over time |
| **Green shaded regions** | Periods where `is_speech = true` |
| **Orange dashed line** | Onset threshold -- probability above this triggers speech detection |
| **Yellow dashed line** | Dip threshold -- probability below this during silence helps end detection |
| **Pink vertical lines** | Speech onset and offset events |
| **Blue dotted lines** | Streaming inference passes |
| **Teal pills (WORDS track)** | Committed words positioned at their audio timestamps |
| **Purple pills (FINAL track)** | Final transcription results |

## Interactive Controls

- **Onset Threshold** -- Drag to visualize how different thresholds would affect speech detection
- **Dip Threshold** -- Adjust the silence threshold for offset behavior
- **Zoom** -- Zoom into dense regions. Scroll horizontally when zoomed
- **Smoothing** -- Apply a moving average to reduce noise in the probability curve

!!! tip "Hover for Details"
    Hover over any point on the waveform to see exact timestamp, probability, and speech state. Hover over committed words in the WORDS track to see audio position, duration, and commit latency.
