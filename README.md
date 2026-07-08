# Network Intrusion Detection System (NIDS) — ML-based

A machine-learning pipeline that analyzes network traffic and classifies it as
**Normal** or one of four attack categories: **DoS**, **Probe**, **R2L**, **U2R**.
Built on the industry-standard **NSL-KDD** dataset (the cleaned-up successor to
KDD Cup 1999), which is already included in `data/` — no manual download needed.

```
Normal traffic  ──┐
DoS attacks     ──┤
Probe attacks   ──┼──►  Feature engineering ──► Random Forest / Gradient Boosting ──► Alert / Classification
R2L attacks     ──┤
U2R attacks     ──┘
```

## What's included

```
nids_project/
├── data/
│   ├── KDDTrain+.txt        # 125,973 labeled training connections
│   └── KDDTest+.txt         # 22,544 labeled test connections (includes novel attack subtypes)
├── models/                  # created after training: scaler, encoders, trained model
├── outputs/                 # created after training/eval: reports, confusion matrix
├── src/
│   ├── config.py            # column names + attack-category mapping (DoS/Probe/R2L/U2R)
│   ├── preprocess.py         # loading, cleaning, encoding, scaling
│   ├── train.py              # trains Random Forest and/or Gradient Boosting
│   ├── evaluate.py           # classification report + confusion matrix plot
│   └── predict.py            # run the trained model on new traffic (CSV or demo)
├── requirements.txt
└── README.md
```

## Quick start

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Train the model (takes ~30 seconds on a laptop CPU)
cd src
python train.py                # Random Forest (default, best all-round choice)
# or: python train.py --model gb     (Gradient Boosting)
# or: python train.py --model both   (train & compare both, keeps the better one)

# 3. Evaluate — prints metrics and saves a confusion matrix image
python evaluate.py

# 4. Classify new traffic
python predict.py --demo                     # quick sanity check on sample test rows
python predict.py --input /path/to/traffic.csv
```

After training you'll have, in `models/`:
- `best_model.pkl` – the production classifier
- `scaler.pkl`, `label_encoder.pkl`, `feature_names.pkl` – needed to transform new data identically

And in `outputs/`:
- `random_forest_report.txt` (and/or `gradient_boosting_report.txt`)
- `confusion_matrix.png`

## How it works

1. **Data**: Each row is a network connection record with 41 features — things
   like `duration`, `protocol_type`, `src_bytes`, `dst_bytes`, `count`,
   `serror_rate`, `same_srv_rate`, etc. — plus a fine-grained attack label
   (e.g. `neptune`, `smurf`, `satan`, `guess_passwd`, `rootkit`, or `normal`).
2. **Category mapping** (`config.py`): the 39 specific attack labels are
   grouped into the 4 standard categories used in intrusion-detection
   research:
   - **DoS** – Denial of Service (neptune, smurf, back, teardrop, ...)
   - **Probe** – surveillance/scanning (nmap, portsweep, satan, ...)
   - **R2L** – Remote-to-Local, unauthorized access from a remote machine
     (guess_passwd, ftp_write, warezmaster, ...)
   - **U2R** – User-to-Root, privilege escalation (buffer_overflow, rootkit, ...)
3. **Preprocessing**: categorical fields (`protocol_type`, `service`, `flag`)
   are one-hot encoded; numeric fields are standardized (zero mean, unit
   variance) with `StandardScaler`.
4. **Model**: a `RandomForestClassifier` (200 trees, class-balanced) by
   default. A `GradientBoostingClassifier` is available for comparison.
   Random Forest is used because it handles the mixed numeric/categorical
   feature space well, trains fast, and gives interpretable feature
   importances — useful for a security analyst who wants to know *why*
   something was flagged.
5. **Evaluation**: accuracy, per-class precision/recall/F1, macro-F1, and a
   confusion matrix.

## A note on realistic accuracy expectations

On the standard `KDDTest+` split you should see roughly:
- Overall accuracy: **~73–78%**
- DoS / Probe / Normal: strong precision and recall
- **R2L and U2R recall will look low** (often under 10%)

This is not a bug — it's a well-documented property of NSL-KDD: the official
test set intentionally includes attack **subtypes that never appear in
training**, specifically to test generalization to unseen attacks. Papers
in this space almost always report the same pattern. If you want higher
R2L/U2R recall for a demo, you can do a random train/test split from the
combined data instead of using the official split (see "Extending" below).

## Extending this project

- **Use your own traffic**: export live/captured traffic (e.g. from Zeek/Bro,
  CICFlowMeter, or Wireshark+feature extraction) into a CSV with the same 41
  columns and run `predict.py --input yourfile.csv`.
- **Try other models**: XGBoost, LightGBM, or a small neural network (Keras/
  PyTorch) can be dropped into `train.py` following the same pattern.
- **Binary mode**: for a simpler "attack vs. normal" alarm, collapse
  `attack_category` to two classes before encoding.
- **Real-time deployment**: wrap `predict.py`'s logic in a small Flask/FastAPI
  service that accepts a JSON feature vector and returns a classification —
  useful for plugging into a SIEM or alerting pipeline.
- **Feature selection**: `train.py` prints the top 15 most important
  features after training — useful for building a lighter-weight model for
  high-throughput environments.

## Dataset citation

NSL-KDD: Tavallaee, M., Bagheri, E., Lu, W., & Ghorbani, A. (2009). "A
Detailed Analysis of the KDD CUP 99 Data Set." Submitted to Second IEEE
Symposium on Computational Intelligence for Security and Defense
Applications (CISDA).
