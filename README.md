# Low-Resource Thai Profanity Detection and Censoring in Audio

Senior project at Assumption University of Thailand (team of 3). The system finds Thai profanity directly in the audio waveform and bleeps it out of uploaded audio or video, without transcribing the speech first.

No public Thai profanity dataset exists, so the team collected and hand-labelled its own.

## What's here

| Folder | Contents |
|---|---|
| `audio_processing/` | Audio conversion and background noise removal |
| `dataset_creation/` | Scripts that build and check the labelled dataset |
| `scripts/` | Fine-tuning experiments and windowed evaluation |
| `server/` | Flask API (`POST /api/censor`) that returns the bleeped file |
| `sr1-web/` | React + Vite upload page for the API |
| `evaluation_results/`, `csv/`, `plots/` | Results and labels from earlier experiment runs |

The audio dataset and trained models are not in this repo.

## Approach

- **Data:** 2,019 instances from 251 audio files (social media, YouTube, voice recordings), labelled by hand in Audacity. Clean speech vastly outweighs profanity.
- **Model:** fine-tuned [`airesearch/wav2vec2-large-xlsr-53-th`](https://huggingface.co/airesearch/wav2vec2-large-xlsr-53-th) with a classification head and class-weighted cross-entropy.
- **Preprocessing:** 16 kHz mono, pre-emphasis, spectral subtraction, Hamming window, RMS and z-score normalisation.
- **Augmentation:** noise, time stretching, pitch shifting, volume scaling, mix-cut.

## Results

These are the final numbers from the project's research paper. The files in `evaluation_results/` come from earlier runs on a smaller dataset, so they differ.

| Setup | Binary accuracy | Precision | Recall | F1 |
|---|---:|---:|---:|---:|
| Rectangular window (baseline) | 51.69% | 46.30% | 3.55% | 6.80% |
| Hamming window | 62.54% | 62.12% | 29.08% | 39.61% |
| FFT enhancement | 52.44% | 61.54% | 5.67% | 10.39% |
| Hamming + FFT | 50.55% | 50.00% | 1.42% | 2.76% |
| Class-weighted + Hamming | 63.58% | 67.19% | 30.50% | 41.95% |
| **Class-weighted + Hamming + mix-cut** | **66.43%** | 49.39% | 42.55% | **45.57%** |

**Real-world performance is much lower.** We also tested 80+ window and stride settings and added IoU-based timing evaluation. Measured against the original word-level labels, F1 falls to **6.5%**, or **3.9%** with IoU at a 0.5 threshold. Window-level scores overstate performance. The main causes are class imbalance, short words that sound alike, and detections that split one word into fragments. The system is not ready for unattended censoring.

## Running it

You need your own trained model at `server/models/CW_ham` and a `server/beep.wav`, since neither is committed.

```bash
pip install -r requirements.txt --extra-index-url https://download.pytorch.org/whl/cu126
pip install flask flask-cors pydub "moviepy<2" noisereduce  # used by the server, not in requirements.txt
cd server && python app.py                                  # http://localhost:8080

cd sr1-web && pnpm install && pnpm dev                      # upload page
```
