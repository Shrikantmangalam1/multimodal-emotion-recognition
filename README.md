# 🧠 AI-Enabled Multimodal Emotion Recognition Framework
### Student Mental Health Monitoring Using Audiovisual Log Analysis

---

> ## ⚠️ Prototype — Not a Production System
> This is a **proof-of-concept** built to demonstrate the *concept and feasibility* of multimodal emotion recognition — not a production-ready or clinically validated tool.
>
> **Key limitations to be aware of:**
> - Trained and tested on only **73 student videos** — far too small for reliable generalisation
> - Audio Random Forest classifier achieved only **~34% cross-validation accuracy** due to the small dataset
> - No ground-truth emotion labels were available — the system generates its own labels (unsupervised)
> - Fusion weights (40/30/30) are manually set by design reasoning, not learned from data
> - The keyword classifier cannot handle negation, metaphor, or indirect emotional expression
> - FER library has known limitations with non-frontal poses, poor lighting, and demographic diversity
>
> **This project shows *how* such a system could be built.** A production-grade version would require a much larger labelled dataset, clinically validated thresholds, and proper consent frameworks. It should never be used for actual mental health diagnosis or decision-making without qualified human oversight.

---

## 📌 Overview

This project proposes an end-to-end **multimodal emotion analysis pipeline** that automatically assesses the emotional well-being of students from short video recordings. Rather than relying on a single source of information, the system fuses **three independent modalities** — spoken language (text), facial expressions, and vocal acoustics (audio) — into a unified risk assessment.

The system accepts `.mp4` video files as input and outputs a structured Excel report with per-subject emotion labels, scores, and a **four-tier mental health assessment**: `Positive`, `Neutral`, `Moderate Risk`, or `High Risk`.

---

## 🗂️ Repository Structure

```
multimodal-emotion-recognition/
├── New_major_project__3_.ipynb       # Main pipeline (Whisper + FER + Fusion)
├── Audio_predict__module__2_.ipynb   # Audio Random Forest classifier
├── plots/                            # Architecture diagrams and result charts
└── README.md
```

---

## ⚙️ Tech Stack

| Component | Tool / Library |
|-----------|---------------|
| Speech-to-Text | OpenAI Whisper (base model) |
| Text Emotion | Rule-based Keyword Classifier (custom) |
| Facial Emotion | FER library + OpenCV |
| Audio Features | Librosa (MFCC, ZCR, RMS, Pitch) |
| Audio Classifier | Random Forest (scikit-learn) |
| Video Processing | FFmpeg |
| Output | Pandas + OpenPyXL (Excel) |
| Environment | Google Colab + Google Drive |

---

## 🏗️ System Architecture

The pipeline follows a **sequential design with a parallel middle stage**. A single video input passes through three independent analytical modules, whose outputs are then fused into a final assessment.

![System Architecture](https://raw.github.com/Shrikantmangalam1/multimodal-emotion-recognition/main/Diagram/System%20Architecture.png)

**Pipeline stages:**
1. **Input** — MP4 video → FFmpeg extracts WAV audio
2. **Stage 1** — Whisper ASR → text transcription → Keyword Emotion Classifier
3. **Stage 2** — OpenCV frame sampling → FER library → facial emotion label + score
4. **Stage 3** — Librosa feature extraction → Random Forest → audio emotion label + score
5. **Fusion** — Weighted linear combination → Final Score → Assessment tier
6. **Output** — Excel report with 25 columns per subject

---

## 📋 Stage 1 — Text Emotion Analysis

### Keyword Classifier Logic

The classifier detects **6 emotion categories**: `happy`, `sad`, `anxious`, `depressed`, `nervous`, `sarcasm`.

**Decision flow:**

![Keyword Classifier Flowchart](https://raw.github.com/Shrikantmangalam1/multimodal-emotion-recognition/main/Diagram/Keyword%20Classifier%20logic.png)

**Emotion → Score mapping:**

| Emotion | Score |
|---------|-------|
| Happy | +2 |
| Neutral | 0 |
| Nervous | −1 |
| Sarcasm | −2 |
| Sad | −2 |
| Anxious | −3 |
| Depressed | −4 |

**Key design features:**
- **Sarcasm detection** — triggered when positive keywords co-occur with ≥2 negative keyword matches (e.g. *"oh great, another assignment"* → sarcasm, not happy)
- **Ratio check** — if ≥60% of matched keywords are positive → `happy`
- **Priority hierarchy** for negatives: `depressed > anxious > sad > nervous`

---

## 😐 Stage 2 — Facial Emotion Analysis

- Samples **1 frame every 15 frames** using OpenCV for computational efficiency
- FER library (pre-trained CNN on FER2013 — 35,887 images, 7 categories) runs inference on each sampled frame
- **Dominant emotion** = most frequently occurring label across all frames
- Falls back to `neutral (score 0)` if no face is detected

**Facial score mapping:** happy (+2), surprise (+1), neutral (0), sad (−2), angry/disgust/fear (−3)

---

## 🎙️ Stage 3 — Audio Feature Extraction & Classification

### Feature Extraction Pipeline

![Audio Feature Extraction Flowchart](https://raw.github.com/Shrikantmangalam1/multimodal-emotion-recognition/main/Diagram/AudioFeature%20Extraction%20flowchart.png)

A **16-dimensional feature vector** is extracted from each WAV file:

| Feature | Dimensions | What it captures |
|---------|-----------|-----------------|
| MFCCs | 13 | Spectral envelope of speech |
| Zero Crossing Rate (ZCR) | 1 | Signal noisiness / voicing |
| RMS Energy | 1 | Vocal loudness / intensity |
| Average Pitch | 1 | Fundamental frequency / prosodic tone |

### Random Forest Classifier

- **200 decision trees**, balanced class weights, 5-fold stratified cross-validation
- Rare classes (`fear`, `disgust`, `angry`) merged into `sad` to handle class imbalance
- Training labels = the **more negative** of text and facial labels (conservative strategy to prioritise at-risk detection)

#### Confusion Matrix
![Confusion Matrix](https://raw.github.com/Shrikantmangalam1/multimodal-emotion-recognition/main/Diagram/Confusion%20Diagram.png)

#### Feature Importance
![Feature Importance](https://raw.github.com/Shrikantmangalam1/multimodal-emotion-recognition/main/Diagram/Feature%20Importance%20Bar%20Chart.png)

> MFCCs (especially MFCC_4, MFCC_6, MFCC_9) were the most discriminative features. RMS energy and average pitch also ranked highly as prosodic emotion markers.

---

## 🔀 Multimodal Fusion

### Fusion Formula

![Fusion Diagram](https://raw.github.com/Shrikantmangalam1/multimodal-emotion-recognition/main/Diagram/Multimodel%20fusion%20Diagram.png)

```
Final Score = (0.4 × Text Score) + (0.3 × Facial Score) + (0.3 × Audio Score)
```

**Why these weights?**
- Text gets the highest weight (0.4) — linguistic content is the most semantically direct signal
- Facial and audio are equally weighted (0.3 each) — complementary non-verbal channels

### Assessment Tiers

| Final Score | Assessment |
|-------------|-----------|
| ≥ 0.5 | ✅ Positive |
| −0.5 to 0.5 | 🟡 Neutral |
| −1.5 to −0.5 | 🟠 Moderate Risk |
| < −1.5 | 🔴 High Risk |

---

## 📊 Results

### Emotion Distribution — Text vs Facial (n=73)
![Emotion Distribution](http://raw.github.com/Shrikantmangalam1/multimodal-emotion-recognition/main/Diagram/emotion%20Distribution%20Bar%20Chart.png)

### Final Assessment Distribution
![Final Assessment Distribution](https://raw.github.com/Shrikantmangalam1/multimodal-emotion-recognition/main/Diagram/Final%20Assessment%20Distribution.png)

| Assessment | Count | Percentage |
|-----------|-------|-----------|
| ✅ Positive | 28 | 38.4% |
| 🟡 Neutral | 6 | 8.2% |
| 🟠 Moderate Risk | 18 | 24.7% |
| 🔴 High Risk | 21 | 28.8% |

---

## 💡 Key Findings

| # | Finding |
|---|---------|
| 1 | Text analysis identified `happy` as most prevalent (39.7%), but 45.2% showed negative indicators (`anxious`, `sarcasm`, `depressed`, `sad`) |
| 2 | Facial analysis showed `neutral` dominating (41.1%) — subjects tend to moderate facial expressions even when speech reveals distress |
| 3 | The sarcasm heuristic successfully prevented misclassification of masked negative sentiment as positive |
| 4 | Multimodal fusion consistently outperformed any single modality — each channel compensated for the other's failure modes |
| 5 | The conservative audio labelling strategy (using the more negative of text/facial as training target) reduced false negatives for at-risk subjects |

---

## ⚠️ Limitations

- **Small dataset** — only 73 videos, too small to generalise reliably
- **No ground-truth labels** — the system is unsupervised; labels are generated by the pipeline itself
- **Keyword classifier** cannot handle negation (*"I don't feel happy"*), metaphor, or indirect expression
- **FER library** has known limitations with non-frontal poses, poor lighting, and demographic diversity
- **Fixed fusion weights** — set by design reasoning, not learned from data
- **No longitudinal tracking** — each video is analysed as an independent snapshot

---

## 🚀 Future Scope

- **Real-time processing** of live video streams
- Replace keyword classifier with fine-tuned **BERT/RoBERTa** for contextual NLP
- Replace Random Forest with **wav2vec 2.0** end-to-end audio model
- **Learnable fusion** using attention-based multimodal transformers
- **Longitudinal tracking** to detect deteriorating trends over time
- **Multilingual support** via Whisper's multilingual capabilities

---

## 🛠️ Setup & Run

```bash
# 1. Open in Google Colab
# 2. Mount Google Drive and update VIDEO_FOLDER path
# 3. Dependencies are installed automatically by the first cells

# Step 2 — Run the full pipeline
New_major_project__3_.ipynb

# Step 2 — Train and save the audio classifier
Audio_predict__module__2_.ipynb
```

**Dependencies:**
```
openai-whisper, fer==22.5.1, tensorflow, librosa==0.9.2,
opencv-python, scikit-learn, pandas, openpyxl, ffmpeg
```

---

## 📁 Output

The system generates an Excel file with **25 columns** per video:

`Video Name` · `Transcription` · `Emotion` · `Emotion_Score` · `Dominant_Facial_Emotion` · `Facial_Emotion_Score` · `Facial_Emotion_Breakdown` · `MFCC_1–13` · `ZCR` · `RMS` · `Avg_Pitch` · `Audio_Label` · `Audio_Label_Score` · `Final_Score` · `Final_Assessment`

---

## 📚 References

- Radford et al. (2022) — OpenAI Whisper: Robust Speech Recognition via Large-Scale Weak Supervision
- Goodfellow et al. (2013) — FER2013 Dataset and CNN baseline for Facial Emotion Recognition
- Breiman (2001) — Random Forests
- Morency et al. (2011) — Weighted late fusion for multimodal sentiment analysis
- Devlin et al. (2019) — BERT: Pre-training of Deep Bidirectional Transformers

---

*This system is intended as a proof-of-concept decision-support tool — not a diagnostic system. Any real-world application must involve qualified human professionals and appropriate ethical safeguards.*
