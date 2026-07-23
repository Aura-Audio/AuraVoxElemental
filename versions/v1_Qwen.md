## Real-Time Voice/Vocal Element Detector: Pre-Code Design

Below is a formal design for a Python script and/or a single-page web application that detects live audio containing voice or vocal elements. The design is intended to handle normal speech, singing, muffled voices, whispered voices, and digital or synthetic vocal sounds. It also identifies acoustic elements that are characteristic of human vocal production.

---

# 1. Objective

The system should analyze a live audio stream and determine, in real time, whether the audio contains voice or vocal elements.

The system should detect:

1. Normal spoken voice.
2. Singing or sustained vocalization.
3. Whispered or breathy voice.
4. Muffled voice, such as voice through a wall, fabric, mask, low-quality microphone, or distant capture.
5. Digital, compressed, robotic, synthetic, or text-to-speech-like voice.
6. Partial vocal elements, such as vowels, humming, syllables, sibilants, or vocal bursts, even when full intelligible speech is not present.

The system should also report which vocal-specific acoustic features are present.

---

# 2. Recommended Architecture

A robust design should not rely on a single detector. Instead, it should use an ensemble of:

1. **Neural voice/activity detection**
   - Detects likely speech or vocal sound using pretrained audio classifiers or voice activity models.

2. **Digital signal processing evidence**
   - Detects physical properties unique or highly characteristic of vocal production, such as pitch, harmonics, formants, and syllabic modulation.

3. **Temporal smoothing and state logic**
   - Prevents flickering detections and confirms that vocal evidence persists over time.

4. **Explainable feature reporting**
   - Outputs not only “voice detected” but also the detected vocal elements, such as fundamental frequency, harmonicity, formants, sibilance, and modulation.

This hybrid approach is preferable because neural models are good at recognizing speech-like patterns, while DSP features provide interpretable evidence that a sound is produced by a vocal source.

---

# 3. High-Level System Pipeline

The processing pipeline should be organized as follows:

```text
Audio Input
    |
    v
Capture and Buffering
    |
    v
Preprocessing
    |
    v
Frame-Based Feature Extraction
    |
    +---> Neural Voice/Vocal Classifier
    |
    +---> Pitch and Harmonicity Analyzer
    |
    +---> Formant / Spectral Envelope Analyzer
    |
    +---> Temporal Modulation Analyzer
    |
    +---> Noise / Muffling / Digital Artifact Analyzer
    |
    v
Evidence Fusion
    |
    v
Temporal State Machine
    |
    v
Real-Time Output / Visualization
```

---

# 4. Audio Input Options

## 4.1 Python Script Input Sources

The Python version may support:

1. Microphone input.
2. System audio loopback, where supported.
3. Audio file input for testing.
4. Streaming input from a browser or network source.

For live microphone capture, a low-latency audio library such as PortAudio-compatible tooling should be used.

For system audio:

- Windows: WASAPI loopback.
- macOS: virtual audio device, such as BlackHole or similar.
- Linux: PulseAudio or PipeWire monitor source.

## 4.2 Web App Input Sources

The single-page web app may support:

1. Microphone capture using browser media APIs.
2. Tab or system audio capture where permitted by the browser and operating system.
3. Local audio file playback for testing.

The web app should process audio locally whenever possible to preserve privacy and reduce latency.

---

# 5. Audio Preprocessing

All incoming audio should be normalized into a consistent analysis format.

Recommended settings:

| Parameter | Recommended Value |
|---|---:|
| Channels | Mono |
| Sample rate | 16 kHz for speech models; optionally 22.05 kHz or 24 kHz for richer spectral analysis |
| Bit depth | 32-bit floating point internally |
| Frame length | 20 to 30 milliseconds |
| Hop length | 10 milliseconds |
| Analysis window | 200 to 500 milliseconds for stable detection |
| Target latency | 100 to 250 milliseconds |

Preprocessing steps:

1. Convert stereo to mono.
2. Resample to the target sample rate.
3. Remove DC offset.
4. Apply a gentle high-pass filter to remove very low-frequency rumble, for example below 60 to 80 Hz.
5. Maintain a rolling buffer of recent audio.
6. Optionally estimate ambient noise floor for adaptive thresholding.

---

# 6. Core Detection Strategy

The detector should combine several independent sources of evidence.

## 6.1 Neural Voice/Vocal Likelihood

Use one or more pretrained neural models to estimate the probability that the current audio segment contains voice or vocal sound.

Possible model categories:

1. **Voice Activity Detection model**
   - Good for speech presence.
   - Effective for normal spoken voice.
   - May need supplementation for singing, humming, whispering, or heavily processed vocals.

2. **Audio event classifier**
   - Can detect broader classes such as:
     - Speech
     - Singing
     - Human voice
     - Male speech
     - Female speech
     - Child speech
     - Whisper
     - Yell
     - Laughter
     - Vocal percussion
     - Synthetic voice, if trained or partially trained for it

3. **Custom vocal detector**
   - A small convolutional or recurrent model trained on voice versus non-voice audio.
   - Can be trained to recognize muffled, distant, compressed, and synthetic vocal audio.

The neural component should output one or more probabilities, such as:

```text
speech_probability
singing_probability
vocal_probability
synthetic_voice_probability
```

---

## 6.2 Pitch and Harmonicity Evidence

Human voiced speech and singing usually contain a fundamental frequency and a harmonic series.

Features to extract:

1. Fundamental frequency, often called F0.
2. Voicing probability.
3. Harmonic-to-noise ratio.
4. Harmonic stack strength.
5. Pitch stability.
6. Pitch contour.

Typical voiced human pitch ranges:

| Voice Type | Approximate F0 Range |
|---|---:|
| Adult male speech | 80 to 180 Hz |
| Adult female speech | 160 to 300 Hz |
| Child speech | 250 to 400 Hz or higher |
| Singing | Wider range, potentially 60 Hz to 1 kHz or more |

Pitch evidence is strong for:

- Vowels.
- Humming.
- Sustained singing.
- Voiced consonants.
- Some synthetic voices.

Pitch evidence may be weak or absent for:

- Whispering.
- Unvoiced consonants.
- Breath sounds.
- Heavily muffled audio.
- Percussive vocal sounds.

Therefore, pitch should be used as supporting evidence, not as the only criterion.

---

## 6.3 Formant and Vocal-Tract Resonance Evidence

Formants are resonant frequency bands produced by the shape of the human vocal tract. They are one of the strongest indicators of vocal sound.

Features to extract:

1. First formant, F1.
2. Second formant, F2.
3. Third formant, F3.
4. Formant bandwidth.
5. Formant stability and movement.
6. Spectral envelope shape.

Formant evidence is useful because many non-vocal sounds do not exhibit the same moving resonance patterns found in speech and singing.

Formant tracking can be performed using:

- Linear predictive coding.
- Spectral envelope estimation.
- Cepstral analysis.
- Established speech analysis algorithms.

Formants may be degraded in muffled audio, but lower formants often remain partially detectable even when high frequencies are attenuated.

---

## 6.4 Spectral Shape Evidence

Voice has characteristic spectral properties.

Relevant features:

1. Spectral centroid.
2. Spectral bandwidth.
3. Spectral flatness.
4. Spectral roll-off.
5. Spectral tilt.
6. Low-frequency to high-frequency energy ratio.
7. Band-limited energy concentration.
8. Presence of vowel-like spectral peaks.

Voice often has:

- Strong low-to-mid frequency energy.
- Harmonic peaks.
- Formant-like spectral envelope peaks.
- Less flat noise than many environmental sounds.
- More structured spectrum than wind, traffic, or applause.

However, music and some synthetic sounds can also have structured spectra, so spectral shape should be combined with pitch, formant, and temporal evidence.

---

## 6.5 Temporal Modulation Evidence

Speech and singing have characteristic timing patterns.

Important temporal features:

1. Syllabic modulation rate.
2. Amplitude envelope fluctuations.
3. Rhythm regularity.
4. Onset and offset patterns.
5. Voiced-unvoiced transitions.
6. Silence gaps between phrases.
7. Speech-rate estimate.

Speech commonly exhibits amplitude modulation in the approximate range of 2 to 8 Hz, corresponding to syllable rhythm.

Singing may have slower or more sustained modulation, with vibrato around approximately 4 to 7 Hz in sustained notes.

Temporal modulation helps distinguish voice from steady noise, machinery, wind, and many musical textures.

---

## 6.6 Phoneme-Like Element Detection

The system can identify elements associated with vocal articulation.

These elements do not require full speech recognition. They only require detection of broad acoustic classes.

Potential elements:

| Vocal Element | Description | Detection Evidence |
|---|---|---|
| Voiced vowel | Sustained vocalic sound | F0, harmonics, stable formants |
| Humming | Closed-mouth vocalization | Strong F0, low high-frequency energy |
| Voiced consonant | Sounds such as /b/, /d/, /g/, /v/, /z/ | Pitch plus transient or noise components |
| Unvoiced consonant | Sounds such as /s/, /f/, /sh/ | High-frequency noise, low pitch evidence |
| Sibilance | Sharp /s/ or /sh/ sounds | Energy in 4 to 10 kHz range, high spectral flatness in that band |
| Plosive burst | Sudden release in /p/, /t/, /k/ | Short broadband burst followed by vowel or silence |
| Whisper | Breathy voiceless speech | Speech-like modulation without stable F0 |
| Breath | Inhalation or exhalation | Noise-like, low harmonic content |
| Vocal fry | Low creaky vibration | Irregular low-frequency pulsing |
| Falsetto | High-pitched vocal register | High F0, often weaker harmonics |
| Vibrato | Pitch modulation in singing | Periodic F0 modulation around 4 to 7 Hz |
| Vocal shout/yell | High-energy vocalization | High intensity, strong harmonics, wide bandwidth |

---

# 7. Handling Muffled Voices

Muffled voices may lack high-frequency detail but often preserve some low-frequency pitch and formant information.

The detector should not require high-frequency speech content.

Muffled-voice strategy:

1. Emphasize lower frequency bands, for example below 2 to 3 kHz.
2. Use pitch detection robust to low-bandwidth audio.
3. Track lower formants, especially F1 and possibly F2.
4. Look for speech-like amplitude modulation.
5. Detect harmonic structure even when weak.
6. Compare against non-vocal low-frequency noise using harmonicity and modulation.
7. Estimate a muffling index.

Possible muffling indicators:

- Reduced energy above 3 or 4 kHz.
- Low spectral centroid.
- Narrow spectral bandwidth.
- Preserved pitch or formant movement.
- Speech-like temporal modulation.
- Low intelligibility but high vocal likelihood.

The output may include:

```text
muffled_voice_likelihood
muffling_index
estimated_bandwidth_limit
```

---

# 8. Handling Digital, Compressed, or Synthetic Voices

Digital voices may include:

1. Compressed telephony audio.
2. VoIP audio.
3. Robotically processed voice.
4. Text-to-speech output.
5. Vocoder or voice-changer output.
6. Heavily quantized or low-bitrate voice.

The system should detect these as vocal if they retain voice-like structure.

Digital-voice strategy:

1. Detect codec-like bandwidth limits.
2. Detect unnatural spectral cutoffs.
3. Detect quantization or compression artifacts.
4. Detect overly stable or mechanically quantized pitch.
5. Detect synthetic harmonic structure.
6. Detect unnatural formant transitions.
7. Preserve voice detection even when naturalness is low.

Possible digital indicators:

- Abrupt high-frequency cutoff at common codec limits.
- Repeated spectral patterns.
- Robotic pitch steps.
- Reduced micro-prosody.
- Artificially clean harmonic stack.
- Unnatural noise suppression artifacts.
- Packet loss or dropout patterns.

The output may include:

```text
digital_voice_likelihood
synthetic_voice_likelihood
codec_artifact_index
robotic_pitch_index
```

Important: synthetic or digital voice detection should be treated as an optional classification layer. The primary goal is to detect vocal elements, not necessarily to prove whether the voice is human.

---

# 9. Evidence Fusion Design

The system should combine multiple evidence channels into a single voice decision.

## 9.1 Frame-Level Evidence

For each analysis frame or short time window, compute:

```text
neural_vocal_score
pitch_score
harmonicity_score
formant_score
modulation_score
spectral_voice_score
muffled_voice_score
digital_voice_score
noise_penalty
```

## 9.2 Fusion Options

There are two practical fusion approaches.

### Option A: Rule-Based Weighted Score

A transparent weighted score can be used:

```text
voice_score =
    w1 * neural_vocal_score
  + w2 * pitch_score
  + w3 * harmonicity_score
  + w4 * formant_score
  + w5 * modulation_score
  + w6 * spectral_voice_score
  - w7 * noise_penalty
```

Advantages:

- Explainable.
- Easy to tune.
- Does not require labeled training data for the fusion stage.

Disadvantages:

- Requires careful threshold selection.
- May be less accurate in complex acoustic environments.

### Option B: Learned Fusion Classifier

Use a small machine-learning classifier, such as gradient boosting, logistic regression, or a small temporal neural network, to combine features.

Input features:

- Neural model probabilities.
- Pitch features.
- Harmonicity features.
- Formant features.
- Spectral features.
- Modulation features.
- Noise and artifact features.

Output:

```text
voice_probability
voice_class
confidence
```

Advantages:

- Better adaptation to complex conditions.
- Can learn interactions between features.

Disadvantages:
- Requires labeled training data.
- Requires validation and calibration.

A recommended approach is to begin with rule-based fusion for the prototype, then replace it with a learned fusion model once enough labeled data is available.

---

# 10. Temporal State Machine

Real-time detection should avoid rapid toggling between “voice” and “no voice.”

A state machine should be used.

Possible states:

```text
NON_VOICE
POSSIBLE_VOICE
VOICE_ACTIVE
VOICE_DECAY
```

Example state logic:

1. Enter `POSSIBLE_VOICE` when the fused voice score exceeds an onset threshold for a short minimum duration.
2. Enter `VOICE_ACTIVE` when vocal evidence remains stable for 150 to 300 milliseconds.
3. Enter `VOICE_DECAY` when the score drops below an offset threshold.
4. Return to `NON_VOICE` if the low score persists for 300 to 800 milliseconds.

Recommended hysteresis:

| Parameter | Suggested Range |
|---|---:|
| Onset threshold | 0.55 to 0.70 |
| Offset threshold | 0.30 to 0.45 |
| Minimum voice duration | 150 to 300 ms |
| Maximum gap tolerance | 300 to 800 ms |

These values should be configurable.

---

# 11. Output Design

The system should produce real-time structured output.

## 11.1 Primary Output

At each update interval, output:

```text
timestamp
voice_present
voice_confidence
voice_state
voice_type
```

Example voice types:

```text
none
speech
singing
whisper
humming
muffled_speech
digital_speech
synthetic_voice
vocal_sound
uncertain
```

## 11.2 Vocal Element Output

The system should also report detected vocal elements.

Example fields:

```text
f0_hz
f0_confidence
voiced_probability
harmonicity
formant_f1_hz
formant_f2_hz
formant_f3_hz
formant_confidence
spectral_centroid_hz
spectral_tilt
amplitude_modulation_rate_hz
syllabic_rhythm_score
sibilance_score
plosive_burst_score
whisper_score
breath_score
nasality_estimate
vibrato_score
muffling_index
digital_artifact_index
synthetic_voice_score
```

## 11.3 Example Structured Event

A conceptual output event could look like this:

```json
{
  "timestamp": "2026-07-23T14:05:11.230Z",
  "voice_present": true,
  "voice_state": "VOICE_ACTIVE",
  "voice_confidence": 0.91,
  "voice_type": "muffled_speech",
  "elements": {
    "f0_hz": 132.4,
    "f0_confidence": 0.72,
    "voiced_probability": 0.81,
    "harmonicity": 0.64,
    "formant_f1_hz": 480,
    "formant_f2_hz": 1180,
    "formant_f3_hz": 2450,
    "formant_confidence": 0.58,
    "spectral_centroid_hz": 910,
    "amplitude_modulation_rate_hz": 4.3,
    "syllabic_rhythm_score": 0.77,
    "sibilance_score": 0.21,
    "plosive_burst_score": 0.12,
    "whisper_score": 0.08,
    "breath_score": 0.19,
    "muffling_index": 0.74,
    "digital_artifact_index": 0.31,
    "synthetic_voice_score": 0.12
  }
}
```

---

# 12. Python Script Design

The Python implementation should be modular and suitable for real-time console operation or a local web interface.

## 12.1 Module Breakdown

### AudioInput

Responsibilities:

- Open audio stream.
- Capture microphone or loopback audio.
- Provide fixed-size audio chunks.
- Handle device errors and sample-rate conversion.

### Preprocessor

Responsibilities:

- Convert to mono.
- Resample.
- Remove DC offset.
- Apply high-pass filtering.
- Maintain rolling audio buffer.

### FeatureExtractor

Responsibilities:

- Compute short-time energy.
- Compute spectral features.
- Compute modulation features.
- Compute pitch and harmonicity features.
- Compute formant-related features.

### NeuralClassifier

Responsibilities:

- Run voice activity detection.
- Run vocal/speech audio classification.
- Optionally run synthetic voice detection.
- Provide frame or segment probabilities.

### VocalEvidenceAnalyzer

Responsibilities:

- Combine pitch, formant, harmonicity, and modulation evidence.
- Detect vocal elements such as vowels, sibilants, plosives, whispers, and humming.

### FusionEngine

Responsibilities:

- Combine neural and DSP evidence.
- Produce a unified voice score.
- Apply thresholds and calibration.

### StateMachine

Responsibilities:

- Track voice onset and offset.
- Apply temporal smoothing.
- Maintain current voice state.

### Reporter

Responsibilities:

- Emit JSON lines.
- Write logs.
- Send data to a user interface.
- Optionally record annotated audio segments.

### UserInterface

Optional components:

- Command-line output.
- Local web dashboard.
- Real-time spectrogram.
- Confidence meters.
- Detected element indicators.

## 12.2 Suggested Python Technology Stack

Possible libraries and components:

| Function | Possible Tools |
|---|---|
| Audio capture | PortAudio-compatible audio libraries |
| Numerical processing | NumPy, SciPy |
| Audio features | librosa, torchaudio, or custom DSP |
| Voice activity detection | Silero VAD or similar ONNX model |
| Audio classification | YAMNet, PANNs, or another AudioSet-pretrained model |
| Pitch tracking | pYIN, CREPE, FCPE, or similar |
| Formant tracking | LPC-based tracking or speech analysis libraries |
| Machine learning runtime | ONNX Runtime or PyTorch |
| Optional UI | FastAPI with WebSocket, or a desktop GUI |
| Optional visualization | Browser frontend, Matplotlib, or WebGL spectrogram |

## 12.3 Python Processing Model

The Python script should use a real-time-safe architecture:

1. Audio callback captures chunks and places them in a queue.
2. Worker thread processes audio chunks.
3. Feature extraction and model inference run in the worker thread.
4. UI or reporter consumes detection results.
5. The audio callback should avoid heavy computation.

This prevents audio dropouts and keeps latency predictable.

---

# 13. Single-Page Web App Design

The web app should ideally perform processing locally in the browser.

## 13.1 Browser Architecture

```text
User Grants Audio Permission
        |
        v
Audio Capture
        |
        v
AudioWorklet Processor
        |
        v
Ring Buffer
        |
        v
Web Worker
        |
        +---> Feature Extraction
        |
        +---> ONNX / TensorFlow.js Model Inference
        |
        +---> Pitch / Harmonicity / Modulation Analysis
        |
        v
Fusion Engine
        |
        v
UI State
        |
        v
Visualization and Event Output
```

## 13.2 Web Components

### AudioCapture

Responsibilities:

- Request microphone or audio-sharing permission.
- Create an audio context.
- Route audio into an AudioWorklet.
- Handle sample-rate conversion if needed.

### AudioWorkletProcessor

Responsibilities:

- Receive raw audio samples in real time.
- Transfer chunks to a worker thread.
- Avoid blocking the audio rendering thread.

### RingBuffer

Responsibilities:

- Store recent audio samples.
- Provide overlapping analysis windows.
- Allow stable pitch and formant analysis.

### InferenceWorker

Responsibilities:

- Run neural models.
- Run DSP feature extraction.
- Perform fusion.
- Send results to the main thread.

### FeatureExtractorWasm

Optional but recommended.

Responsibilities:

- Compute FFTs.
- Compute mel spectrograms.
- Compute pitch candidates.
- Compute spectral features.
- Improve performance compared with pure JavaScript DSP.

### ModelRuntime

Possible runtimes:

- ONNX Runtime Web.
- TensorFlow.js.
- WebGPU-accelerated inference where available.
- WASM-optimized inference for smaller models.

Possible models:

- Silero VAD exported to ONNX.
- YAMNet or a smaller vocal classifier exported to ONNX or TensorFlow.js format.
- Custom lightweight vocal detector.

### Visualization

The UI should display:

1. Live input level.
2. Voice confidence meter.
3. Voice state indicator.
4. Detected voice type.
5. Real-time spectrogram.
6. Pitch overlay.
7. Formant overlay, optional.
8. Detected element indicators.
9. Muffling index.
10. Digital artifact index.
11. Event log.
12. Export controls.

## 13.3 Web App Privacy Considerations

The app should:

- Process audio locally when possible.
- Avoid uploading audio unless the user explicitly enables server processing.
- Clearly indicate when listening is active.
- Request permission only when needed.
- Provide a visible stop button.
- Avoid storing audio without consent.

---

# 14. Model Strategy

## 14.1 Minimum Viable Model Set

For an initial version, use:

1. A voice activity detection model.
2. A general audio classifier with vocal classes.
3. DSP-based pitch and harmonicity analysis.

This provides a strong baseline without requiring immediate custom training.

## 14.2 Recommended Classes

If training a custom model, use classes such as:

```text
non_voice
speech
singing
whisper
humming
muffled_voice
digital_or_synthetic_voice
vocal_sound_other
```

Alternatively, use multi-label outputs:

```text
voice_present
speech
singing
whisper
muffled
digital
synthetic
```

Multi-label design is useful because a sound can be both singing and digital, or speech and muffled.

## 14.3 Training Data Sources

Potential data sources:

1. Public speech corpora.
2. Multilingual speech datasets.
3. Singing datasets.
4. Music with vocals.
5. Instrumental music.
6. Environmental noise datasets.
7. Sound event datasets.
8. Synthetic speech samples.
9. Text-to-speech samples.
10. Low-quality telephony audio.
11. Muffled recordings.
12. Room impulse responses.

## 14.4 Data Augmentation

To improve robustness, augment training data with:

1. Additive noise.
2. Reverberation.
3. Room simulation.
4. Low-pass filtering.
5. Bandwidth limitation.
6. Microphone distortion.
7. Codec compression.
8. Bitrate reduction.
9. Packet loss simulation.
10. Pitch shifting.
11. Time stretching.
12. Robotic or vocoder effects.
13. Whisper simulation.
14. Muffling filters.
15. Far-field capture simulation.

Augmentation is essential for detecting muffled and digital voices.

---

# 15. Feature Set Specification

The following feature set is recommended for the fusion engine.

## 15.1 Energy and Time Features

```text
rms_energy
peak_level
zero_crossing_rate
envelope_slope
silence_ratio
burst_count
amplitude_modulation_2_8_hz
amplitude_modulation_4_7_hz
```

## 15.2 Spectral Features

```text
spectral_centroid
spectral_bandwidth
spectral_rolloff
spectral_flatness
spectral_flux
spectral_tilt
low_high_energy_ratio
band_energy_80_300_hz
band_energy_300_2000_hz
band_energy_2000_4000_hz
band_energy_4000_8000_hz
```

## 15.3 Pitch and Harmonicity Features

```text
f0_hz
f0_confidence
voiced_probability
harmonic_to_noise_ratio
harmonic_stack_strength
pitch_stability
pitch_contour_slope
pitch_modulation_rate
pitch_modulation_depth
```

## 15.4 Formant and Vocal-Tract Features

```text
formant_f1
formant_f2
formant_f3
formant_bandwidths
formant_stability
formant_movement_rate
lpc_residual_sharpness
spectral_envelope_peakiness
```

## 15.5 Speech-Rhythm Features

```text
syllable_rate_estimate
voiced_segment_duration
unvoiced_segment_duration
pause_pattern_score
rhythm_regularity
onset_density
```

## 15.6 Robustness and Artifact Features

```text
estimated_bandwidth_limit
high_frequency_loss
muffling_index
noise_floor
snr_estimate
codec_artifact_index
spectral_discontinuity
quantization_noise_index
robotic_pitch_index
dropout_index
```

---

# 16. Detection Rules for Specific Vocal Elements

## 16.1 Normal Speech

Indicators:

- Neural speech probability high.
- F0 present during voiced segments.
- Formants detectable.
- Syllabic modulation around 2 to 8 Hz.
- Alternation between voiced, unvoiced, and silence.

## 16.2 Singing

Indicators:

- Sustained pitch.
- Stable harmonic stack.
- Longer voiced segments.
- Vibrato may be present.
- Broader pitch range than speech.
- Neural singing probability may be high.

## 16.3 Whisper

Indicators:

- Speech-like spectral envelope.
- Speech-like modulation.
- Weak or absent F0.
- Higher noise component.
- Sibilance may be prominent.
- Neural whisper or speech probability may still be elevated.

## 16.4 Humming

Indicators:

- Strong F0.
- Strong harmonics.
- Low high-frequency energy.
- Weak consonantal transitions.
- Sustained pitch.

## 16.5 Muffled Voice

Indicators:

- Reduced high-frequency energy.
- Low spectral centroid.
- Limited bandwidth.
- Preserved low-frequency pitch.
- Preserved speech modulation.
- Partial formant evidence.
- Neural voice probability may remain moderate or high.

## 16.6 Digital or Synthetic Voice

Indicators:

- Voice-like pitch and formants.
- Unnatural pitch stability or quantization.
- Codec bandwidth limits.
- Compression artifacts.
- Robotic spectral texture.
- Neural synthetic voice probability, if available.

## 16.7 Vocal Percussion or Non-Linguistic Vocal Sounds

Indicators:

- Short bursts.
- Broadband noise or pitch pulses.
- Human vocal spectral envelope.
- Lack of sustained speech rhythm.
- Neural vocal probability may be moderate.

---

# 17. User Interface Design

## 17.1 Main Controls

The interface should include:

1. Start listening button.
2. Stop listening button.
3. Input device selector.
4. Sensitivity control.
5. Noise threshold control.
6. Detection mode selector.
7. Export button.
8. Calibration button.

Detection modes could include:

```text
Speech only
Singing and speech
Any vocal sound
High sensitivity
Low false-alarm mode
```

## 17.2 Live Indicators

Display:

1. Input level meter.
2. Noise floor estimate.
3. Voice confidence meter.
4. Current voice state.
5. Detected voice type.
6. Pitch readout.
7. Harmonicity meter.
8. Formant confidence meter.
9. Muffling index.
10. Digital artifact index.

## 17.3 Visualizations

Recommended visualizations:

1. Scrolling waveform.
2. Real-time spectrogram.
3. Pitch contour overlay.
4. Formant tracks overlay.
5. Confidence timeline.
6. Detected event markers.

## 17.4 Event Log

The event log should show:

```text
start_time
end_time
duration
voice_type
confidence
detected_elements
```

Users should be able to export:

1. JSON event log.
2. CSV summary.
3. Annotated timestamps.
4. Optional audio clips, with consent.

---

# 18. Calibration Design

Because microphones and environments vary, the system should support calibration.

Calibration modes:

1. Silence calibration.
2. Ambient noise calibration.
3. Voice calibration.
4. Automatic adaptive calibration.

Calibration should estimate:

- Noise floor.
- Typical microphone frequency response.
- Input gain.
- Background noise type.
- Threshold offsets.

A simple calibration flow:

1. User clicks “Calibrate silence.”
2. System records ambient noise for several seconds.
3. System estimates noise profile.
4. User optionally speaks a sample sentence.
5. System adjusts sensitivity and voice thresholds.

---

# 19. Performance Requirements

Target performance for a modern laptop or recent smartphone:

| Metric | Target |
|---|---:|
| End-to-end latency | 100 to 250 ms |
| Frame hop | 10 ms |
| Decision update rate | 10 to 20 Hz |
| CPU usage | Low enough for sustained background use |
| Memory usage | Moderate, preferably under 500 MB for Python prototype |
| Web inference | Preferably under 300 ms total latency |
| False positives | Low in common noise conditions |
| Recall for clear speech | Very high |
| Recall for muffled speech | Moderate to high, depending on audibility |

If real-time performance is insufficient, the system can reduce model size, increase hop length, or run inference every second or third frame.

---

# 20. Evaluation Plan

The system should be evaluated using multiple datasets and conditions.

## 20.1 Test Categories

1. Clean speech.
2. Noisy speech.
3. Reverberant speech.
4. Muffled speech.
5. Whispered speech.
6. Singing.
7. Humming.
8. Synthetic speech.
9. Compressed telephony speech.
10. Music with vocals.
11. Instrumental music.
12. Environmental noise.
13. Machinery noise.
14. Animal sounds.
15. Voice-like non-human sounds.

## 20.2 Metrics

Use:

1. Frame-level precision.
2. Frame-level recall.
3. Segment-level precision.
4. Segment-level recall.
5. F1 score.
6. ROC curve.
7. Detection latency.
8. False alarm rate per hour.
9. Mean time to detect voice onset.
10. Mean time to clear voice offset.
11. Accuracy by voice type.
12. Accuracy under muffling levels.
13. Accuracy under codec conditions.

## 20.3 Regression Tests

Maintain a test suite containing:

- Known speech samples.
- Known non-voice samples.
- Difficult false-positive cases.
- Muffled samples.
- Synthetic voice samples.
- Music with vocals.
- Instrumental music.
- Noise-only samples.

---

# 21. Edge Cases and Failure Modes

The design should account for the following cases.

## 21.1 Music with Vocals

The system should detect vocal elements even when accompanied by instruments.

Strategy:

- Use vocal event classification.
- Use pitch and harmonic tracking.
- Look for vocal formant structure.
- Use source separation optionally, if performance allows.

## 21.2 Instrumental Music Mimicking Voice

Some instruments can imitate voice.

Strategy:

- Require formant-like movement.
- Require speech-like modulation.
- Require harmonic and pitch patterns consistent with vocal tract behavior.

## 21.3 Wind and Noise

Wind can cause low-frequency energy and amplitude modulation.

Strategy:

- Check spectral flatness.
- Check harmonicity.
- Check formant evidence.
- Use high-pass filtering.
- Use noise classification.

## 21.4 Machinery

Machinery may have stable pitch-like tones.

Strategy:

- Detect excessive pitch stability.
- Detect lack of formant movement.
- Detect lack of syllabic rhythm.
- Detect mechanical harmonic patterns.

## 21.5 Very Muffled Voice

If almost all high-frequency information is lost, detection becomes difficult.

Strategy:

- Rely on low-frequency pitch.
- Rely on rhythm.
- Rely on partial formants.
- Report low confidence if evidence is weak.

## 21.6 Whisper

Whisper lacks pitch.

Strategy:

- Rely on spectral envelope.
- Rely on speech rhythm.
- Rely on sibilance and breath patterns.
- Use neural whisper detection if available.

---

# 22. Optional Advanced Enhancements

These can be added after the core system works.

## 22.1 Source Separation

A lightweight vocal separation model can isolate vocal components from music or noise.

Advantages:

- Better detection in music.
- Better pitch and formant tracking.

Disadvantages:

- Higher CPU/GPU cost.
- Higher latency.

## 22.2 Speaker Diarization

This would identify when different voices are present.

Not required for basic voice detection, but useful for multi-speaker environments.

## 22.3 Language-Independent Phoneme Features

The system could detect broad phonetic categories without performing speech recognition.

Possible categories:

- Vowel-like.
- Nasal-like.
- Fricative-like.
- Plosive-like.
- Silence.
- Noise.

This improves explainability without requiring transcription.

## 22.4 Synthetic Voice Classification

A dedicated model could classify whether a voice is likely synthetic.

This is challenging and should be treated as probabilistic, not definitive.

---

# 23. Recommended Minimum Viable Product

The first version should be simple but effective.

## MVP Features

1. Live microphone input.
2. 16 kHz mono preprocessing.
3. Neural voice probability.
4. Pitch and harmonicity detection.
5. Basic spectral features.
6. Simple fusion score.
7. Temporal smoothing.
8. Console or web dashboard output.
9. JSON event log.
10. Basic sensitivity control.

## MVP Output

```text
voice_present
voice_confidence
f0_hz
harmonicity
spectral_centroid
modulation_score
muffling_index
```

This MVP can later be expanded with formants, singing detection, synthetic voice detection, and advanced visualization.

---

# 24. Recommended Development Phases

## Phase 1: Prototype

- Audio capture.
- Frame extraction.
- Energy and spectral features.
- Pitch detection.
- Simple threshold-based voice detection.

## Phase 2: Neural Detection

- Add voice activity model.
- Add vocal sound classifier.
- Combine neural and DSP evidence.
- Add temporal smoothing.

## Phase 3: Vocal Element Analysis

- Add formant tracking.
- Add harmonicity metrics.
- Add sibilance and plosive detection.
- Add whisper and humming detection.

## Phase 4: Robustness

- Add muffled voice detection.
- Add digital artifact detection.
- Add calibration.
- Add noise adaptation.

## Phase 5: Web App

- Port processing to browser.
- Use AudioWorklet and Web Worker.
- Use ONNX or TensorFlow.js inference.
- Build real-time visualization.

## Phase 6: Evaluation and Tuning

- Test across datasets.
- Measure latency and false alarms.
- Tune thresholds.
- Optionally train a custom fusion model.

---

# 25. Final Recommended Design Summary

The recommended system is a real-time hybrid voice/vocal detector with the following characteristics:

1. It captures live audio from a microphone, system audio, or browser audio source.
2. It converts audio to mono 16 kHz frames.
3. It runs a neural voice or vocal classifier.
4. It extracts pitch, harmonicity, formants, spectral shape, and modulation features.
5. It identifies vocal elements such as voiced vowels, humming, sibilance, plosives, whisper, breath, and vibrato.
6. It detects muffled voice by emphasizing low-frequency pitch, partial formants, and speech rhythm.
7. It detects digital or synthetic voice by recognizing codec artifacts, unnatural pitch behavior, and artificial spectral structure.
8. It fuses all evidence into a confidence score.
9. It applies temporal smoothing to produce stable real-time decisions.
10. It outputs structured detection results and visual indicators.

This design provides a practical path from a Python prototype to a browser-based single-page application while preserving explainability, real-time performance, and robustness to difficult voice conditions.
