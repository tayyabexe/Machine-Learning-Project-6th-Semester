# ANN (MLP): Status Update (all 3 datasets)

**From:** Tayyab · **Date:** 22 Nov 2026 · **Models:** ANN (multilayer perceptron), 7 models in total (one per target column)

---

## 1. Summary

- All **7 ANN models** are trained and tested (the X-IIoTID ones on a second laptop, because they take about 12 min each).
- Hyperparameters come from **one published paper per dataset**, the same approach as for the Decision Trees.
- The **ANN is below the Decision Tree on every model**:
  - **Small gap:** TON_IoT `label` (0.9923 vs 0.9981) and X-IIoTID (about 0.989 vs 0.995 on all three targets).
  - **Large gap:** TON_IoT `type` (0.82 vs 0.98) and both Edge-IIoT targets.
- **TON_IoT `type` and both Edge-IIoT targets underfit**: the epoch counts given in the papers are too few for our setup. Nothing was tuned on the test set.

---

## 2. Models

| Dataset | Notebook | Target | Type | Status |
|---|---|---|---|---|
| TON_IoT | `ann_label.ipynb` | `label` | Binary | ✅ Run |
| TON_IoT | `ann_type.ipynb` | `type` | 10 classes | ✅ Run |
| Edge-IIoT | `ann_label.ipynb` | `Attack_label` | Binary | ✅ Run |
| Edge-IIoT | `ann_type.ipynb` | `Attack_type` | 15 classes | ✅ Run |
| X-IIoTID | `ann_class3.ipynb` | `class3` | Binary | ✅ Run (second laptop) |
| X-IIoTID | `ann_class2.ipynb` | `class2` | 10 classes | ✅ Run (second laptop) |
| X-IIoTID | `ann_class1.ipynb` | `class1` | 19 classes | ✅ Run (second laptop) |

Notebooks are in `scripts/ANN/<Dataset>/`. They use scikit-learn's `MLPClassifier`, because TensorFlow and PyTorch are not installed.

---

## 3. Hyperparameters and sources

| Dataset | Paper | Hidden layers | Optimizer | Learning rate | Batch | Epochs | Verified? |
|---|---|---|---|---|---|---|---|
| **TON_IoT** | A. M. H. Othman, M. F. Ab Razak, A. Firdaus, H. Tahaei, M. Gheisari, "A Hybrid CNN–LSTM Model for IoT Intrusion Detection: A Robustness Analysis Across Datasets," *Future Internet*, 18(7):345, 2026. [doi:10.3390/fi18070345](https://doi.org/10.3390/fi18070345) | 128-64-32 | Adam | 0.001 | 256 | 50 (early-stop patience 10) | ✅ Full PDF. ⚠️ Binary `label` only; the same settings are used for `type` |
| **Edge-IIoT** | M. A. Ferrag, O. Friha, D. Hamouda, L. Maglaras, H. Janicke, "Edge-IIoTset: A New Comprehensive Realistic Cyber Security Dataset of IoT and IIoT Applications for Centralized and Federated Learning," *IEEE Access*, 10:40281–40306, 2022. [doi:10.1109/ACCESS.2022.3165809](https://doi.org/10.1109/ACCESS.2022.3165809) | 90-90 | Adam (+ L2) | 0.001 *(default)* | 800 | 25 | ✅ Full PDF, Table 9 (dataset authors' paper) |
| **X-IIoTID** | M. Al-Hawawreh, E. Sitnikova, N. Aboutorab, "X-IIoTID: A Connectivity-Agnostic and Device-Agnostic Intrusion Data Set for Industrial Internet of Things," *IEEE Internet of Things Journal*, 9(5):3962–3977, 2022. [doi:10.1109/JIOT.2021.3102056](https://doi.org/10.1109/JIOT.2021.3102056) | 200-200-200 | RMSprop → **Adam** | 0.001 *(default)* | 250 | 10 | ✅ Full PDF (dataset authors' paper) |

**All models:**
- ReLU activation.
- Output layer: sigmoid for binary targets, softmax for multiclass (scikit-learn chooses it automatically).
- `random_state=42`.
- *(default)* = not given in the paper, so the library default is used.

**Differences from the papers:**
- **X-IIoTID:** scikit-learn has no RMSprop, so Adam is used instead.
- **L2 strength:** no paper gives a value, so the scikit-learn default (0.0001) is used. X-IIoTID uses none, since its paper states none.
- **TON_IoT early stopping:** the paper stops on validation loss. scikit-learn's rule watches the training loss instead.

Full quotes are in `HYPERPARAMETER_SOURCES.md` (ANN section).

---

## 4. Results (test set, 20%)

Multiclass metrics are **macro-averaged**.

| Dataset | Model | **ANN Acc** | Precision | Recall | F1 | ROC-AUC | FPR | MCC | DT Acc |
|---|---|---|---|---|---|---|---|---|---|
| TON_IoT | label | **0.9923** | 0.9933 | 0.9966 | 0.9949 | 0.9982 | 0.0217 | 0.9786 | 0.9981 |
| TON_IoT | type | **0.8232** | 0.7788 | 0.7558 | 0.7501 | 0.9912 | 0.0196 | 0.8046 | 0.9766 |
| Edge-IIoT | Attack_label | **0.9098** | 0.9101 | 0.9913 | 0.9490 | 0.9198 | 0.5383 | 0.6083 | 0.9623 |
| Edge-IIoT | Attack_type | **0.6645** | 0.7053 | 0.6137 | 0.6204 | 0.9476 | 0.0248 | 0.6366 | 0.8573 |
| X-IIoTID | class3 | **0.9899** | 0.9950 | 0.9841 | 0.9895 | 0.9994 | 0.0047 | 0.9798 | 0.9958 |
| X-IIoTID | class2 | **0.9887** | 0.9763 | 0.9491 | 0.9618 | 0.9996 | 0.0019 | 0.9833 | 0.9953 |
| X-IIoTID | class1 | **0.9886** | 0.9735 | 0.9166 | 0.9411 | 0.9996 | 0.0011 | 0.9835 | 0.9953 |

**Training and prediction time:**

| Model | Training time | Inference latency |
|---|---|---|
| TON_IoT label / type | 50 s / 58 s | 0.0022 / 0.0025 ms per sample |
| Edge-IIoT Attack_label / Attack_type | 11 s / 12 s | 0.0018 / 0.0017 ms per sample |
| X-IIoTID class3 / class2 / class1 | 116 s / 131 s / 141 s | 0.0026 / 0.0032 / 0.0028 ms per sample |

The Decision Tree trains faster (about 1 s on TON_IoT and Edge-IIoT, 30-46 s on X-IIoTID) and predicts faster (0.0003 to 0.0011 ms per sample). The X-IIoTID ANN times were measured on the second laptop, so they are only roughly comparable. Log-Loss, TP/TN/FP/FN and test time are in each notebook's results table and in `Results.xlsx`.

---

## 5. Points to note

1. **The multiclass models underfit.** Train and test accuracy are almost the same (TON `type` 0.824 / 0.823; Edge `Attack_type` 0.662 / 0.665), and the training loss was still falling at the last epoch.
2. **TON_IoT `type`:**
   - `backdoor` is almost never detected (recall 0.02); 98% of it is predicted as `dos`.
   - `mitm` recall is 0.36.
   - The 5 folds vary a lot (accuracy 0.88 ± 0.05).
3. **Edge-IIoT `Attack_label` has a high false-positive rate** (0.54): more than half of the normal traffic is flagged as attack. The cause is likely class imbalance (85% of rows are attacks).
4. **Edge-IIoT is far below the paper** (99.99% / 94.67%). The paper trained its DNN on a larger file (`DNN-EdgeIIoT`, about 2.2M rows) with SMOTE. We kept the same data and preprocessing as every other model.
5. **X-IIoTID works well** (about 0.989 on all three targets, no over- or underfitting). Accuracy is slightly below the DT, but the ANN has a better Log-Loss (0.03 vs 0.15-0.17) and ROC-AUC (0.9994+), meaning its probabilities are better calibrated. Weak classes: `MitM` (F1 0.73, only 23 test rows), `fuzzing` (recall 0.65) and `TCP Relay` (recall 0.70) in class1; `C&C` (F1 0.83) in class2.
6. **Our numbers are not exactly comparable with the papers**, because the splits and preprocessing differ (the papers used SMOTE, feature selection and deduplication).

The full list of problems is in `ANN_Issues.md`.

---

## 6. Common setup (same as the Decision Trees)

- **Split:** 80% train / 20% test, stratified on the finest label, so the ANN and the DT test on exactly the same rows.
- **Validation:** stratified 5-fold on the 80% (64% train / 16% validation per fold). Used to check stability only; no tuning.
- **Preprocessing:** the same cleaning and one-hot encoding as the DT.
- **Scaling:** added `StandardScaler`, fitted on the training data only and refitted inside each fold. Neural networks need it; trees don't.
- **Duplicates:** kept (`DEDUPLICATE = False`).
- **Latency:** `predict()` time on the whole test set ÷ number of test rows (ms per sample).
- **Graphs:** class distribution, confusion matrix (+ ROC for binary), training loss curve. Saved in `Graphs/ANN/` and on Drive.

---

## 7. Questions for the team

1. Should we **remove duplicate rows**? This matters most for TON_IoT (55%).
2. Can we report an **extra, clearly labelled ANN variant with more epochs**, tuned on the validation folds only, to show the underfitting can be fixed? The main result would stay paper-based.
3. Should **SMOTE or class weights** be used for *all* models to handle the imbalance (Edge-IIoT FPR 0.54)? It only stays fair if every algorithm uses it.
4. **Results.xlsx:** I replaced the earlier ANN values in the TON_IoT `label` row and the X-IIoTID rows with these results (the old X-IIoTID values are kept in `Results_backup_before_xiiot_ann.xlsx`). Who filled the earlier values, and with which settings?
