# **Real-Time Voice/Vocal Element Detector: Mobile SPA Design (v2.0)**
*Last Updated: July 23, 2026*
*Author: Transparency-X (Declan)*
*Version: 2.0*

---

## **📌 Document Changelog (v1.0 → v2.0)**
| **Section**               | **Changes**                                                                                     |
|---------------------------|-----------------------------------------------------------------------------------------------|
| **Objective**             | Clarified focus on **mobile SPA** and **real-time constraints**.                              |
| **Architecture**          | Added **mobile-specific optimizations** (AudioWorklet, Web Workers, WASM).                  |
| **Core Detection**        | Prioritized **DSP + lightweight neural models** for MVP1.                                    |
| **MVP Definitions**       | Split into **MVP1 (Core Voice Detection)** and **MVP2 (Enhanced Vocal Elements)**.          |
| **Technical Stack**       | Updated to **ONNX Runtime Web, TensorFlow.js, WebGL, WASM**.                                 |
| **Performance Targets**   | Added **mobile-specific benchmarks** (latency, CPU, memory).                                |
| **Privacy & Compliance**  | Explicit **GDPR/CCPA** considerations and **local-only processing**.                       |
| **UI/UX**                 | Simplified for mobile; added **spectrogram (WebGL)** and **adaptive UX**.                   |
| **Risk Mitigation**       | Added **battery drain, permission issues, and model size** risks.                            |
| **Development Roadmap**   | Structured into **phases with timelines**.                                                  |

---

## **🎯 1. Objective**

### **1.1 Purpose**
Develop a **real-time, mobile-first Single-Page Application (SPA)** that detects and classifies **voice and vocal elements** in live audio streams. The system must:
- Run **locally in the browser** (no cloud dependency).
- Support **low-latency processing** (<250ms for MVP1, <300ms for MVP2).
- Work on **mid-range smartphones** (e.g., iPhone SE, Samsung Galaxy A50).
- Preserve **privacy** (no audio storage or uploads without consent).
- Detect the following vocal elements:
  1. **Normal spoken voice**
  2. **Singing or sustained vocalization**
  3. **Whispered or breathy voice**
  4. **Muffled voice** (e.g., through walls, masks, or low-quality mics)
  5. **Digital/synthetic voice** (e.g., TTS, vocoders, compressed audio)
  6. **Partial vocal elements** (e.g., vowels, humming, sibilants, plosives)

### **1.2 Key Constraints**
| **Constraint**            | **Target**                                                                                     |
|---------------------------|-----------------------------------------------------------------------------------------------|
| **Latency**               | <250ms (MVP1), <300ms (MVP2)                                                                   |
| **CPU Usage**             | <20% (MVP1), <30% (MVP2)                                                                       |
| **Memory Usage**          | <100MB (MVP1), <200MB (MVP2)                                                                   |
| **Battery Impact**        | Minimal (adaptive duty cycling)                                                              |
| **Offline Support**       | Full (PWA caching for MVP2)                                                                   |
| **False Positive Rate**   | <5% (MVP1), <3% (MVP2)                                                                         |
| **Voice Recall**          | >90% (MVP1), >95% (MVP2)                                                                       |

---

## **🏗️ 2. Recommended Architecture (Mobile SPA)**

### **2.1 High-Level Pipeline**
```
User Grants Microphone Permission
         |
         v
[Audio Capture: MediaDevices API]
         |
         v
[AudioWorklet Processor: Low-Latency Audio Chunking]
         |
         v
[Ring Buffer: Store Recent Audio (200-500ms)]
         |
         +---> [Web Worker 1: DSP Feature Extraction]
         |        - Energy, ZCR, Spectral Centroid
         |        - Pitch (YIN/MCLeod), Harmonicity
         |        - Formants (LPC/WASM)
         |
         +---> [Web Worker 2: Neural Model Inference]
         |        - Silero VAD (ONNX Runtime Web)
         |        - YAMNet (TensorFlow.js) for MVP2
         |
         v
[Fusion Engine: Rule-Based (MVP1) or ML (MVP2)]
         |
         v
[Temporal State Machine: Smooth Onset/Offset]
         |
         v
[UI Thread: Render Results + Visualizations]
```

### **2.2 Mobile-Specific Optimizations**
| **Component**          | **Optimization**                                                                               |
|------------------------|-----------------------------------------------------------------------------------------------|
| **Audio Capture**      | Use `AudioWorklet` (not deprecated `ScriptProcessorNode`).                                   |
| **Sample Rate**        | Downsample to **16kHz** (or 8kHz for VAD-only).                                                |
| **Frame Size**         | 20–50ms frames, 10ms hop.                                                                       |
| **Neural Models**      | **Quantized ONNX/TensorFlow.js** (8-bit) + **WebGL acceleration**.                            |
| **DSP**               | Offload to **WASM** (e.g., Essentia.js for FFT/LPC).                                           |
| **UI Rendering**       | Use **WebGL** (regl/three.js) for spectrogram.                                                |
| **Background Processing** | Pause when app is backgrounded (use `Page Visibility API`).                              |
| **Battery Savings**    | Reduce sample rate when battery is low.                                                      |

---

## **🔊 3. Audio Input (Mobile SPA)**

### **3.1 Supported Sources**
| **Source**               | **Implementation**                                                                             | **MVP1** | **MVP2** |
|--------------------------|-----------------------------------------------------------------------------------------------|----------|----------|
| Microphone               | `navigator.mediaDevices.getUserMedia({ audio: true })`                                      | ✅        | ✅        |
| Local Audio File         | `<input type="file" accept="audio/*">` + `AudioContext`                                      | ✅        | ✅        |
| System Audio (Loopback)  | **Not supported in browsers** (use native app for MVP3).                                     | ❌        | ❌        |

### **3.2 Preprocessing**
| **Parameter**            | **Value**                                                                                      | **Notes**                                  |
|--------------------------|-----------------------------------------------------------------------------------------------|--------------------------------------------|
| Channels                 | Mono                                                                                           | Convert stereo to mono.                   |
| Sample Rate              | 16kHz (MVP1), 22.05kHz (MVP2)                                                                  | Trade-off between accuracy and performance. |
| Bit Depth                | 32-bit float (internal)                                                                         |                                            |
| Frame Length             | 20–30ms                                                                                         |                                            |
| Hop Length               | 10ms                                                                                           |                                            |
| High-Pass Filter         | 60–80Hz cutoff                                                                                 | Remove rumble.                             |
| DC Offset Removal        | Yes                                                                                           |                                            |
| Rolling Buffer           | 200–500ms                                                                                       | For stable feature extraction.            |

---

## **🎛️ 4. Core Detection Strategy**

### **4.1 Hybrid Approach**
The system combines **neural models** (for pattern recognition) and **DSP** (for interpretability).

| **Method**               | **Role**                                                                                       | **MVP1** | **MVP2** |
|--------------------------|-----------------------------------------------------------------------------------------------|----------|----------|
| Neural VAD               | Detect speech/voice presence (Silero VAD).                                                     | ✅        | ✅        |
| Vocal Classifier         | Classify voice type (speech/singing/whisper/muffled/synthetic).                                | ❌        | ✅        |
| Pitch Detection          | Fundamental frequency (F0) tracking.                                                           | ✅        | ✅        |
| Harmonicity Analysis     | Harmonic-to-Noise Ratio (HNR).                                                                | ❌        | ✅        |
| Formant Tracking         | F1–F3 tracking (LPC-based).                                                                    | ❌        | ✅        |
| Spectral Features        | Centroid, bandwidth, rolloff, flatness.                                                        | ✅        | ✅        |
| Temporal Modulation      | Syllabic rhythm (2–8Hz), vibrato (4–7Hz).                                                       | ❌        | ✅        |
| Muffling Detection       | High-frequency energy loss, spectral tilt.                                                     | ❌        | ✅        |
| Digital Artifact Detection | Codec cutoffs, quantization noise.                                                           | ❌        | ✅        |

### **4.2 Detection Rules by Vocal Element**

#### **Normal Speech**
- **Neural**: High speech probability (Silero VAD > 0.7).
- **DSP**: F0 present (80–300Hz), formants detectable (F1: 200–1000Hz, F2: 500–3000Hz).
- **Temporal**: Syllabic modulation (2–8Hz).

#### **Singing**
- **Neural**: High singing probability (YAMNet).
- **DSP**: Sustained F0 (60Hz–1kHz), stable harmonics, vibrato (4–7Hz modulation).
- **Temporal**: Longer voiced segments (>500ms).

#### **Whisper**
- **Neural**: Moderate speech probability (Silero VAD: 0.4–0.6).
- **DSP**: Weak/absent F0, high noise component, sibilance (4–10kHz energy).
- **Temporal**: Speech-like modulation (2–8Hz).

#### **Muffled Voice**
- **Neural**: Moderate voice probability (Silero VAD: 0.5–0.7).
- **DSP**: Reduced high-frequency energy (>3kHz), low spectral centroid, preserved low-F0.
- **Temporal**: Speech-like amplitude modulation.

#### **Digital/Synthetic Voice**
- **Neural**: High synthetic voice probability (custom ONNX model).
- **DSP**: Unnatural pitch stability, codec cutoffs (e.g., 4kHz), quantization noise.

---

## **🔗 5. Evidence Fusion**

### **5.1 MVP1: Rule-Based Weighted Score**
```
voice_score = (
    0.6 * neural_vad_score +
    0.2 * pitch_score +
    0.1 * spectral_voice_score +
    0.1 * energy_score
) - noise_penalty
```
- **Thresholds**:
  - Onset: `voice_score > 0.6` for 150ms.
  - Offset: `voice_score < 0.4` for 300ms.

### **5.2 MVP2: Learned Fusion (ML)**
- **Input Features**: Neural probabilities, pitch, harmonicity, formants, spectral features, modulation.
- **Model**: Tiny **logistic regression** or **gradient boosting** (ONNX/WASM).
- **Output**: `voice_probability`, `voice_type`, `confidence`.

---

## **⏳ 6. Temporal State Machine**

### **6.1 States**
```
NON_VOICE → POSSIBLE_VOICE → VOICE_ACTIVE → VOICE_DECAY → NON_VOICE
```

### **6.2 Transitions**
| **From**          | **To**               | **Condition**                                                                               |
|-------------------|----------------------|-------------------------------------------------------------------------------------------|
| NON_VOICE         | POSSIBLE_VOICE       | `voice_score > 0.55` for 150ms.                                                             |
| POSSIBLE_VOICE    | VOICE_ACTIVE         | `voice_score > 0.6` for 300ms.                                                             |
| VOICE_ACTIVE      | VOICE_DECAY         | `voice_score < 0.45` for 100ms.                                                             |
| VOICE_DECAY       | NON_VOICE            | `voice_score < 0.4` for 500ms.                                                             |

---

## **📤 7. Output Design**

### **7.1 MVP1 Output (JSON)**
```json
{
  "timestamp": "2026-07-23T14:05:11.230Z",
  "voice_present": true,
  "voice_confidence": 0.85,
  "voice_state": "VOICE_ACTIVE",
  "f0_hz": 132.4,
  "f0_confidence": 0.72,
  "spectral_centroid_hz": 910,
  "rms_energy": 0.45
}
```

### **7.2 MVP2 Output (JSON)**
```json
{
  "timestamp": "2026-07-23T14:05:11.230Z",
  "voice_present": true,
  "voice_confidence": 0.91,
  "voice_state": "VOICE_ACTIVE",
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

## **📱 8. Mobile SPA Implementation**

### **8.1 Tech Stack**
| **Component**          | **Technology**                          | **Notes**                                  |
|------------------------|----------------------------------------|--------------------------------------------|
| **Audio Capture**      | Web Audio API (`AudioWorklet`)         | Low-latency, non-blocking.                 |
| **DSP**               | WASM (Essentia.js) or Custom JS         | Offload heavy computation.                |
| **Neural Models**      | ONNX Runtime Web, TensorFlow.js        | Quantized models for performance.          |
| **Fusion Logic**       | Web Worker (Rule-Based or ML)           | Avoid UI thread blocking.                 |
| **UI Framework**       | React/Vue                               | Lightweight, mobile-friendly.              |
| **Visualization**      | WebGL (regl/three.js)                   | For spectrogram and overlays.              |
| **Build Tool**         | Vite                                    | Fast dev and production builds.            |
| **PWA Support**        | Workbox                                | For offline caching (MVP2).                |

### **8.2 File Structure**
```
/real-time-voice-detector
├── /public
│   ├── index.html
│   └── manifest.json (PWA)
├── /src
│   ├── /audio
│   │   ├── AudioCapture.js       # AudioWorklet setup
│   │   ├── RingBuffer.js         # Audio buffering
│   │   └── Preprocessor.js       # Resampling, filtering
│   ├── /dsp
│   │   ├── FeatureExtractor.js    # Energy, ZCR, spectral features
│   │   ├── PitchDetector.js      # YIN/MCLeod in WASM/JS
│   │   └── FormantTracker.js     # LPC in WASM
│   ├── /models
│   │   ├── silero_vad.onnx        # Quantized VAD model
│   │   └── yamnet.json            # TensorFlow.js vocal classifier
│   ├── /fusion
│   │   ├── RuleBasedFusion.js    # MVP1 fusion
│   │   └── MlFusion.js           # MVP2 fusion (ONNX/WASM)
│   ├── /state
│   │   └── StateMachine.js        # Temporal smoothing
│   ├── /ui
│   │   ├── App.jsx               # Main UI
│   │   ├── ConfidenceMeter.jsx   # Voice confidence display
│   │   ├── Spectrogram.jsx       # WebGL spectrogram
│   │   └── VoiceState.jsx        # State indicator
│   ├── /workers
│   │   ├── DspWorker.js          # Web Worker for DSP
│   │   └── ModelWorker.js        # Web Worker for neural models
│   └── main.js                   # Entry point
├── package.json
└── vite.config.js
```

---

## **🎨 9. User Interface (Mobile SPA)**

### **9.1 MVP1 UI Components**
| **Component**          | **Description**                                                                               | **Implementation**                          |
|------------------------|-----------------------------------------------------------------------------------------------|--------------------------------------------|
| Microphone Button      | Start/stop audio capture.                                                                     | `<button>` with `getUserMedia`.            |
| Input Level Meter      | Show microphone volume.                                                                      | `AnalyserNode` + Canvas.                   |
| Voice Confidence Meter | Gauge/bar showing `voice_confidence`.                                                        | SVG/Canvas.                                |
| Voice State Indicator  | "Listening..." / "Voice Detected" / "No Voice".                                               | Text + color coding.                       |
| Sensitivity Slider     | Adjust detection threshold.                                                                  | `<input type="range">`.                   |

### **9.2 MVP2 UI Components (Additional)**
| **Component**          | **Description**                                                                               | **Implementation**                          |
|------------------------|-----------------------------------------------------------------------------------------------|--------------------------------------------|
| Spectrogram            | Real-time frequency analysis.                                                                 | WebGL (regl).                              |
| Pitch Contour          | Overlay F0 on spectrogram.                                                                    | WebGL line rendering.                      |
| Voice Type Indicator   | "Speech" / "Singing" / "Whisper" / "Muffled" / "Synthetic".                                   | Text + icons.                              |
| Vocal Elements Panel   | Show scores for F0, formants, sibilance, etc.                                                 | Collapsible sidebar.                       |
| Export Button          | Download JSON/CSV logs.                                                                       | `Blob` + `URL.createObjectURL`.            |
| Calibration Button     | Auto-calibrate noise floor.                                                                  | 5-second silence recording.                |

### **9.3 UI Mockup (Text-Based)**
```
+-------------------------------------+
|  [🎤 Start Listening]                |
+-------------------------------------+
|  [■■■■■■■■□□□□]  Input Level       |
+-------------------------------------+
|  Voice Confidence: [■■■■■□□□□] 85%  |
+-------------------------------------+
|  Status: VOICE ACTIVE                |
|  Type: Muffled Speech                |
+-------------------------------------+
|  [Spectrogram (WebGL)]               |
|  (Pitch contour overlay)             |
+-------------------------------------+
|  [Export Logs] [Calibrate]           |
+-------------------------------------+
```

---

## **⚙️ 10. MVP Feature Lists**

### **10.1 MVP1: Core Voice Detection**
**Goal**: *Detect voice presence in real time with low latency and minimal false positives.*

| **ID** | **Feature**                          | **Priority** | **Status** | **Notes**                                  |
|--------|--------------------------------------|--------------|------------|--------------------------------------------|
| 1      | Microphone capture                   | ⭐⭐⭐        | Required   | `getUserMedia` + `AudioWorklet`.           |
| 2      | Audio preprocessing (16kHz mono)     | ⭐⭐⭐        | Required   | Resampling, DC offset, high-pass filter.   |
| 3      | DSP: Energy, ZCR, spectral centroid   | ⭐⭐⭐        | Required   | `AnalyserNode` + custom JS.                |
| 4      | Pitch detection (YIN)                | ⭐⭐⭐        | Required   | WASM or JS.                                |
| 5      | Neural VAD (Silero ONNX)             | ⭐⭐⭐        | Required   | Quantized model.                           |
| 6      | Rule-based fusion                    | ⭐⭐⭐        | Required   | Weighted score.                           |
| 7      | Temporal state machine               | ⭐⭐⭐        | Required   | 3-state smoothing.                         |
| 8      | JSON output (voice_present, confidence) | ⭐⭐⭐      | Required   | Emit via WebSocket or DOM.                |
| 9      | UI: Confidence meter                  | ⭐⭐          | Required   | SVG/Canvas.                                |
| 10     | UI: Voice state indicator             | ⭐⭐          | Required   | Text + color.                              |
| 11     | UI: Input level meter                 | ⭐⭐          | Required   | `AnalyserNode` + Canvas.                   |
| 12     | Calibration (noise floor)            | ⭐⭐          | Optional   | Auto-calibrate on startup.                 |
| 13     | Sensitivity slider                    | ⭐           | Optional   | Adjust thresholds.                        |

---

### **10.2 MVP2: Enhanced Vocal Element Detection**
**Goal**: *Classify vocal elements and provide explainable features.*

| **ID** | **Feature**                          | **Priority** | **Status** | **Notes**                                  |
|--------|--------------------------------------|--------------|------------|--------------------------------------------|
| 1      | All MVP1 features                    | ⭐⭐⭐        | Required   |                                            |
| 2      | Neural vocal classifier (YAMNet)     | ⭐⭐⭐        | Required   | TensorFlow.js or ONNX.                     |
| 3      | DSP: Harmonicity (HNR)               | ⭐⭐⭐        | Required   |                                            |
| 4      | DSP: Formant tracking (F1–F3)        | ⭐⭐⭐        | Required   | LPC in WASM.                               |
| 5      | DSP: Temporal modulation             | ⭐⭐          | Required   | Syllabic rhythm, vibrato.                  |
| 6      | DSP: Muffling index                  | ⭐⭐          | Required   | High-frequency energy loss.                |
| 7      | DSP: Sibilance score                 | ⭐⭐          | Optional   | 4–10kHz energy.                            |
| 8      | DSP: Plosive burst detection         | ⭐           | Optional   |                                            |
| 9      | Learned fusion (ML)                  | ⭐⭐          | Optional   | Tiny ONNX model.                           |
| 10     | JSON output: Voice type + elements   | ⭐⭐⭐        | Required   | Extend MVP1 output.                        |
| 11     | UI: Spectrogram (WebGL)              | ⭐⭐          | Required   | regl/three.js.                             |
| 12     | UI: Pitch contour overlay            | ⭐⭐          | Required   | WebGL line.                                |
| 13     | UI: Voice type indicator             | ⭐⭐          | Required   | "Speech" / "Singing" / etc.                |
| 14     | UI: Vocal elements panel             | ⭐⭐          | Optional   | Collapsible sidebar.                       |
| 15     | UI: Export logs (JSON/CSV)           | ⭐⭐          | Optional   | `Blob` + download.                         |
| 16     | PWA: Offline support                 | ⭐⭐          | Required   | Workbox caching.                           |
| 17     | Adaptive thresholds                 | ⭐⭐          | Optional   | Adjust based on noise.                     |
| 18     | Background noise suppression         | ⭐           | Optional   | Wiener filter in WASM.                     |

---

## **🚀 11. Development Phases**

### **Phase 1: MVP1 (4–6 weeks)**
| **Week** | **Tasks**                                                                                     |
|----------|-----------------------------------------------------------------------------------------------|
| 1        | Set up **AudioWorklet + Web Worker** pipeline. Test latency.                                |
| 2        | Implement **DSP features** (energy, ZCR, spectral centroid).                                |
| 3        | Integrate **Silero VAD (ONNX)**. Test on real devices.                                      |
| 4        | Add **pitch detection (YIN)** and **rule-based fusion**.                                   |
| 5        | Build **MVP1 UI** (confidence meter, state indicator, input level).                        |
| 6        | Optimize performance, test edge cases, fix bugs.                                           |

### **Phase 2: MVP2 (6–8 weeks)**
| **Week** | **Tasks**                                                                                     |
|----------|-----------------------------------------------------------------------------------------------|
| 1–2     | Add **formant tracking (LPC in WASM)** and **harmonicity (HNR)**.                           |
| 3        | Integrate **YAMNet (TensorFlow.js)** for vocal classification.                             |
| 4        | Implement **spectrogram (WebGL)** and **pitch contour overlay**.                             |
| 5        | Add **muffling index, sibilance score**, and **export logs**.                                |
| 6        | Optimize **model quantization** and **PWA caching**.                                         |
| 7–8     | Test **edge cases** (muffled, synthetic voices) and **polish UI**.                          |

### **Phase 3: Post-MVP2 (Optional)**
- **Source separation** (MediaPipe Audio).
- **Speaker diarization** (for multi-speaker).
- **Synthetic voice classification** (custom ONNX model).
- **Native wrappers** (Capacitor/Cordova for app stores).

---

## **⚠️ 12. Risks & Mitigation**

| **Risk**                          | **Likelihood** | **Impact** | **Mitigation**                                  |
|-----------------------------------|----------------|------------|-----------------------------------------------|
| Neural models too slow on mobile  | High           | High       | Use **quantized models** + **WebGL acceleration**. |
| DSP too slow in JS                | Medium         | High       | Offload to **WASM** (Essentia.js).               |
| Microphone permission issues      | High           | High       | **Educate users** (clear prompts).              |
| Battery drain                     | High           | Medium     | **Throttle processing** in background.          |
| False positives in noisy envs    | Medium         | Medium     | **Adaptive thresholds** + **noise suppression**. |
| Offline models too large          | Medium         | Medium     | **Lazy-load models** or **stream from CDN**.   |
| Formant tracking inaccurate       | Medium         | Medium     | Use **simplified LPC** or **fallback to spectral features**. |

---

## **📊 13. Evaluation Plan**

### **13.1 Test Datasets**
| **Category**               | **Source**                                                                                     | **MVP1** | **MVP2** |
|---------------------------|-----------------------------------------------------------------------------------------------|----------|----------|
| Clean speech              | LibriSpeech, Common Voice                                                                     | ✅        | ✅        |
| Noisy speech              | CHiME, DSP Challenge                                                                           | ✅        | ✅        |
| Muffled speech             | Custom recordings (through fabric, walls)                                                     | ❌        | ✅        |
| Whispered speech           | Custom recordings                                                                             | ❌        | ✅        |
| Singing                   | GTZAN, MedleyDB                                                                               | ❌        | ✅        |
| Synthetic voice           | TTS samples (e.g., Google TTS, Amazon Polly)                                                  | ❌        | ✅        |
| Compressed audio           | Telephony (8kHz), VoIP (Opus)                                                                 | ❌        | ✅        |
| Environmental noise        | ESC-50, UrbanSound                                                                             | ✅        | ✅        |
| Instrumental music        | GTZAN, MedleyDB                                                                               | ✅        | ✅        |

### **13.2 Metrics**
| **Metric**                     | **Target (MVP1)** | **Target (MVP2)** | **Measurement**                     |
|--------------------------------|--------------------|--------------------|------------------------------------|
| Frame-level precision          | >85%               | >90%               | Test on 1-hour datasets.           |
| Frame-level recall             | >90%               | >95%               |                                    |
| Segment-level F1               | >85%               | >90%               |                                    |
| False positive rate (per hour) | <5                 | <3                 | Test with noise-only samples.     |
| Latency (end-to-end)            | <250ms             | <300ms             | Chrome DevTools.                   |
| CPU usage                        | <20%               | <30%               | Chrome DevTools.                   |
| Memory usage                     | <100MB             | <200MB             | Chrome DevTools.                   |
| Battery impact                   | Minimal            | Low                | Manual testing.                    |

---

## **🔐 14. Privacy & Compliance**

### **14.1 Data Handling**
- **No audio storage**: Process audio in real time; discard after analysis.
- **No cloud uploads**: All processing happens **locally in the browser**.
- **Explicit consent**: Request microphone permission **only when needed** (e.g., on button click).
- **User control**: Provide a **visible stop button** and **clear indicators** when listening is active.

### **14.2 GDPR/CCPA Compliance**
- **Transparency**: Clearly explain in the UI why microphone access is needed.
- **Data minimization**: Only collect **aggregated metrics** (e.g., `voice_present: true`), not raw audio.
- **Right to erase**: Allow users to **clear logs** and **revoke microphone access**.
- **Privacy policy**: Include a **link to a privacy policy** explaining data usage.

---

## **📚 15. References & Dependencies**

### **15.1 Libraries**
| **Library**               | **Purpose**                          | **License**       | **Notes**                                  |
|---------------------------|--------------------------------------|-------------------|--------------------------------------------|
| ONNX Runtime Web          | Run ONNX models in browser           | MIT               | For Silero VAD.                           |
| TensorFlow.js             | Run TensorFlow models in browser     | Apache 2.0        | For YAMNet.                               |
| Essentia.js               | DSP in WASM                          | AGPL-3.0          | For LPC, FFT.                              |
| regl                      | WebGL functional rendering           | MIT               | For spectrogram.                          |
| Workbox                   | PWA caching                           | MIT               | For offline support (MVP2).                |
| Vite                      | Build tool                           | MIT               | Fast dev/prod builds.                     |

### **15.2 Models**
| **Model**               | **Purpose**                          | **Size**       | **Source**                                  |
|--------------------------|--------------------------------------|----------------|--------------------------------------------|
| Silero VAD (ONNX)        | Voice Activity Detection             | ~1MB (quantized) | [GitHub](https://github.com/snakers4/silero-vad) |
| YAMNet (TensorFlow.js)   | Audio Classification                 | ~10MB           | [TF Hub](https://tfhub.dev/google/yamnet/1) |

### **15.3 Datasets**
| **Dataset**               | **Purpose**                          | **License**       | **Link**                                  |
|---------------------------|--------------------------------------|-------------------|--------------------------------------------|
| LibriSpeech               | Clean speech                         | CC-BY-4.0         | [Website](https://www.librispeech.org/)    |
| Common Voice              | Multilingual speech                  | CC0               | [Website](https://commonvoice.mozilla.org/) |
| CHiME                     | Noisy speech                         | Creative Commons  | [Website](http://spandh.dcs.shef.ac.uk/chime_challenge/) |
| ESC-50                    | Environmental sounds                  | CC-BY-4.0         | [Website](https://github.com/karolpiczak/ESC-50) |
| GTZAN                     | Music (with/without vocals)          | Academic Use     | [Website](http://opihi.cs.uvic.ca/sound/music.php) |

---

## **📝 16. Open Questions & Next Steps**

### **16.1 Decisions Needed**
1. **Neural Model Choice for MVP1**:
   - Silero VAD (ONNX) vs. custom lightweight model?
   - **Recommendation**: Start with **Silero VAD (ONNX)** for MVP1.

2. **DSP Library for MVP1**:
   - Custom JS vs. WASM (Essentia.js)?
   - **Recommendation**: **Custom JS** for MVP1 (simpler), then **WASM** for MVP2.

3. **UI Framework**:
   - React vs. Vue vs. Vanilla JS?
   - **Recommendation**: **React** (familiar, mobile-friendly).

4. **PWA for MVP1?**:
   - Add offline support in MVP1 or defer to MVP2?
   - **Recommendation**: Defer to **MVP2**.

### **16.2 Immediate Next Steps**
1. **Prototype the Audio Pipeline**:
   - Build a **minimal AudioWorklet + Web Worker** demo.
   - Test latency on **real mobile devices**.
   - Example: [AudioWorklet Demo](https://googlechrome.github.io/web-audio-samples/audio-worklet/design-pattern/).

2. **Benchmark Models**:
   - Test **Silero VAD (ONNX)** and **YAMNet (TensorFlow.js)** on mobile.
   - Measure **latency, CPU usage, and memory footprint**.

3. **Design UI Mockups**:
   - Create **Figma/Excalidraw** mockups for MVP1 UI.
   - Focus on **confidence meter, state indicator, input level**.

4. **Set Up Development Environment**:
   - Initialize **Vite + React** project.
   - Add **ONNX Runtime Web** and **TensorFlow.js** dependencies.

5. **Stakeholder Review**:
   - Present this document to **Transparency-X** for feedback.
   - Align on **MVP1 scope, timelines, and success metrics**.

---

## **🏁 Conclusion**
This **v2.0 design** provides a **mobile-first, privacy-centric, and scalable** approach to building a **Real-Time Voice/Vocal Element Detector** as a **Single-Page Application**. By prioritizing **MVP1 for core voice detection** and **MVP2 for enhanced vocal analysis**, we ensure a **phased, risk-managed** development process that aligns with **Transparency-X’s goals** of **high-impact productivity** and **real-time monitoring**.

**Next**: Begin **MVP1 prototyping** (AudioWorklet + Silero VAD) and **validate performance on mobile devices**.
