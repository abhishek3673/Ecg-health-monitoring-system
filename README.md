# Real-Time Single-Lead ECG Monitoring System

> End-to-end arrhythmia detection using AD8232 + ESP32 + ML classifiers with IoT dashboard and Telegram alerts.

---

## Overview

A full-stack cardiac monitoring system that acquires single-lead ECG signals via hardware, classifies beats in real-time using machine learning, and streams results to a web dashboard with instant Telegram notifications.

```
AD8232 → ESP32 → Python Backend → Firebase → Web Dashboard
                      ↓                            ↓
               ML Classifier              Telegram Alerts
```

**Classification:** AAMI standard beat types — N (Normal), S (Supraventricular), V (Ventricular Ectopic)  
**Dataset:** MIT-BIH Arrhythmia Database (PhysioNet)  
**End-to-end latency:** ~22–33 ms (ADC → filter → ML → Firebase)

---

## Project Structure

```
ecg_project/
├── esp32/
│   ├── ecg_monitor.ino       # ESP32 firmware (Pan-Tompkins + BPM + Firebase)
│   └── config.h              # Wi-Fi, Firebase, pin configuration
│
├── phase4_backend/
│   ├── realtime_backend.py   # Python real-time pipeline
│   ├── requirements.txt      # Python dependencies
│   ├── firebase_key.json     # Firebase Admin SDK key (keep private)
│   ├── ecg_rf_model.pkl      # Random Forest model (from Phase 2)
│   ├── ecg_svm_model.pkl     # SVM model (from Phase 2)
│   ├── ecg_cnn_model.h5      # 1D-CNN model (from Phase 2)
│   └── ecg_scaler.pkl        # Feature scaler (from Phase 2)
│
└── phase5_dashboard/
    └── index.html            # Real-time web dashboard (Firebase + Chart.js)
```

---

## Hardware

| Component | Role |
|-----------|------|
| AD8232 | Single-lead ECG analog front-end |
| ESP32 Dev Module | ADC sampling @ 360 Hz, Wi-Fi, local Pan-Tompkins |
| Electrodes (3) | RA, LA, RL placement |
| Buzzer + LEDs | Local real-time alert |

**Wiring:**

| AD8232 Pin | ESP32 GPIO |
|------------|------------|
| OUTPUT | GPIO 34 |
| LO+ | GPIO 32 |
| LO- | GPIO 33 |
| 3.3V | 3.3V |
| GND | GND |

---

## ML Pipeline (Phase 2)

Three models trained on MIT-BIH Arrhythmia Database with patient-wise train/test split (records 100–119 train, 200–234 test).

### Results

| Model | Class | Precision | Recall | F1 | AUC |
|-------|-------|-----------|--------|----|-----|
| Random Forest | N | 0.898 | 0.993 | 0.943 | — |
| Random Forest | S | 0.184 | 0.005 | 0.011 | — |
| Random Forest | V | 0.851 | 0.433 | 0.574 | — |
| SVM | N | 0.895 | 0.915 | 0.905 | — |
| SVM | S | 0.314 | 0.081 | 0.128 | — |
| SVM | V | 0.332 | 0.375 | 0.352 | — |
| **1D-CNN** | **N** | **0.927** | **0.779** | **0.846** | **0.790** |
| **1D-CNN** | **S** | 0.024 | 0.066 | 0.036 | 0.598 |
| **1D-CNN** | **V** | **0.517** | **0.842** | **0.641** | **0.959** |

**Deployment model:** Ensemble vote (RF + SVM + CNN). CNN selected as primary — highest V-class recall (0.842) and AUC (0.959), the clinically critical arrhythmia class.

> S-class underperformance is consistent with published literature on single-lead ECG classification due to morphological similarity with N-class beats.

---

## System Latency (Paper Table)

| Stage | Latency |
|-------|---------|
| Signal Filtering | ~8–10 ms |
| Pan-Tompkins | ~2–3 ms |
| ML Inference (Ensemble) | ~12–20 ms |
| Firebase Push (async) | ~50–150 ms* |
| **End-to-End** | **~22–33 ms** |

*Firebase push is asynchronous — does not block real-time processing.

---

## Setup & Run

### 1. ESP32 Firmware

1. Open `esp32/config.h` — fill in Wi-Fi credentials, Firebase host, Telegram token
2. Flash `ecg_monitor.ino` via Arduino IDE
3. Board: `ESP32 Dev Module` | CPU: `240 MHz` | Baud: `921600`

### 2. Python Backend

```bash
cd phase4_backend
py -3.11 -m pip install pyserial scipy numpy firebase-admin python-telegram-bot joblib tensorflow requests PyWavelets scikit-learn
py -3.11 realtime_backend.py
```

> Requires **Python 3.11** — TensorFlow does not support Python 3.12+

Edit `SERIAL_PORT` in `realtime_backend.py` to match your COM port (check Device Manager).

### 3. Web Dashboard

Open `phase5_dashboard/index.html` in Chrome.  
Replace `messagingSenderId` and `appId` in the file with values from Firebase Console → Project Settings → General.

### 4. Telegram Bot

1. Message `@BotFather` → `/newbot` → copy token
2. Message `@userinfobot` → copy Chat ID
3. Paste both into `config.h` (ESP32) and `realtime_backend.py`

---

## Alert Logic

| Condition | Alert |
|-----------|-------|
| V-class beat detected | Immediate Telegram alert |
| 3+ consecutive abnormal beats | Immediate Telegram alert |
| BPM > 110 for 10+ minutes | "High heart rate detected for prolonged period of time" |
| BPM < 45 for 10+ minutes | "Low heart rate detected for prolonged period of time" |
| Same alert type | 10-minute cooldown between repeats |

---

## Dependencies

- **Firmware:** Arduino ESP32 core, WiFi.h, HTTPClient.h (built-in)
- **Python:** pyserial, scipy, numpy, tensorflow, scikit-learn, firebase-admin, python-telegram-bot, PyWavelets, joblib
- **Dashboard:** Firebase JS SDK v9, Chart.js v4

---

## References

- Moody GB, Mark RG. The MIT-BIH Arrhythmia Database. *IEEE Eng Med Biol* 2001.
- Pan J, Tompkins WJ. A real-time QRS detection algorithm. *IEEE Trans Biomed Eng* 1985.
- ANSI/AAMI EC57:2012 — Testing and reporting performance results of cardiac rhythm and ST segment measurement algorithms.
