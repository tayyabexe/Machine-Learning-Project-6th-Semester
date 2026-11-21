# ANN (MLP): Issues and Limitations

Owner: Tayyab (Decision Trees + ANN). Last updated: 2026-11-22.
Use this file for the report's "Limitations" / "Discussion" sections. Full figures are in `AUDIT_LOG.md` (Step 9); sources and quotes are in `HYPERPARAMETER_SOURCES.md` (ANN section).

---

## 1. Results summary (test set)

| Model | ANN Accuracy | ANN F1 | ANN MCC | DT Accuracy | Paper's result |
|---|---|---|---|---|---|
| TON_IoT `label` | 0.9923 | 0.9949 | 0.9786 | 0.9981 | 99.77% (DNN, binary) |
| TON_IoT `type` (macro) | 0.8232 | 0.7501 | 0.8046 | 0.9766 | not reported |
| Edge-IIoT `Attack_label` | 0.9098 | 0.9490 | 0.6083 | 0.9623 | 99.99% |
| Edge-IIoT `Attack_type` (macro) | 0.6645 | 0.6204 | 0.6366 | 0.8573 | 94.67% |
| X-IIoTID `class3` | 0.9899 | 0.9895 | 0.9798 | 0.9958 | 99%+ (paper) |
| X-IIoTID `class2` (macro) | 0.9887 | 0.9618 | 0.9833 | 0.9953 | 99%+ (paper) |
| X-IIoTID `class1` (macro) | 0.9886 | 0.9411 | 0.9835 | 0.9953 | 99%+ (paper) |

The ANN is below the Decision Tree on every model. X-IIoTID is close (about 0.6 percentage points), and its Log-Loss and ROC-AUC are better than the DT's. Nothing was tuned on the test set; the numbers are reported as they came out.

---

## 2. Model behaviour problems

### 2.1 Multiclass models underfit
- **Evidence:** train accuracy is about equal to test accuracy, and both are low:
  - TON `type`: 0.8239 vs 0.8232
  - Edge `Attack_type`: 0.6619 vs 0.6645
- The training loss was still falling at the last epoch:
  - TON `type`: 0.79 → 0.35 over 50 epochs
  - Edge `Attack_type`: 1.586 → 0.883 over 25 epochs, still dropping about 0.003 per epoch
- **Cause:** the epoch counts taken from the papers (50 and 25) are too few for our setup. The TON paper's settings were made for a binary task only.
- **Early stopping never triggered:** in TON_IoT, all 12 fits ran the full 50 epochs, so `n_iter_no_change=10` had no effect.

### 2.2 TON_IoT `type`: weak classes
| Class | Recall | Where the errors go |
|---|---|---|
| backdoor | 0.02 | 98% predicted as `dos` (so `dos` precision falls to 0.45) |
| mitm (only 209 test rows) | 0.36 | 38% → `xss`, 14% → `normal` |
| password | 0.74 | 23% → `backdoor` |
| scanning | 0.81 | 14% → `dos` |

### 2.3 TON_IoT `type`: unstable folds
- Fold accuracy is 0.8793 ± 0.0497. Two folds (0.8429 and 0.8100) are much worse than the other three (about 0.914).
- Whether the network learns to separate backdoor from dos seems to depend on the random weight initialisation.

### 2.4 Edge-IIoT `Attack_label`: very high false-positive rate
- **FPR 0.538:** 2,616 of 4,860 normal test rows are flagged as attack, so MCC is only 0.61.
- Attack recall is very high (0.991), so the model is biased towards "attack".
- **Likely cause:** class imbalance. About 85% of the rows are attacks.

### 2.5 Edge-IIoT `Attack_type`: weak classes
| Class | Precision / Recall |
|---|---|
| XSS | 0.48 / 0.09 |
| Password | 0.65 / 0.20 |
| Uploading | 0.51 / 0.22 |
| SQL_injection | 0.35 / 0.33 |
| MITM | 1.00 / 0.37 |
| Normal | 0.50 / 0.77 (confused with the web attacks) |

Strong classes: DDoS_ICMP and DDoS_UDP (F1 1.00), Vulnerability_scanner 0.88, Backdoor 0.85.

### 2.6 X-IIoTID: small, rare classes are the weak spot
- No over- or underfitting: train and test accuracy are both about 0.99 (class3 0.9910 / 0.9899, class2 0.9900 / 0.9887, class1 0.9897 / 0.9886).

| Target | Class | Problem |
|---|---|---|
| class1 | MitM | F1 0.73 (only 23 test rows) |
| class1 | fuzzing | recall 0.65 |
| class1 | TCP Relay | recall 0.70 |
| class1 | Discovering_resources | recall 0.75 |
| class2 | C&C | F1 0.83 |
| class2 | Exploitation | recall 0.83 |

- Macro recall falls as the number of classes grows: 0.984 (2 classes) → 0.949 (10) → 0.917 (19).
- The class1 folds are less stable (macro-F1 0.9248 ± 0.0100) than the class3 and class2 folds.

---

## 3. Gaps between our setup and the source papers

### 3.1 Library limitation (scikit-learn `MLPClassifier`)
- TensorFlow and PyTorch are not installed (Python 3.14), so we used `MLPClassifier`. It has:
  - no RMSprop optimizer
  - no dropout
  - no validation-loss early stopping with best-weight restore
- **X-IIoTID:** the paper uses **RMSprop**. We use **Adam**, the closest available optimizer (Adam = RMSprop + momentum).
- **TON_IoT:** the paper uses early stopping on *validation loss* with patience 10. We use scikit-learn's *training-loss* rule (`n_iter_no_change=10`), which avoids carving an extra validation split out of the training data.

### 3.2 Values missing from the papers (defaults used)
| Dataset | Missing value | What we used |
|---|---|---|
| TON_IoT | L2 strength | alpha = 0.0001 (sklearn default) |
| Edge-IIoT | Learning rate | 0.001 (Adam default) |
| Edge-IIoT | L2 strength (paper only says "L2") | alpha = 0.0001 (sklearn default) |
| X-IIoTID | Learning rate | 0.001 (default) |
| X-IIoTID | L2 | none (alpha = 0), since none is stated |

### 3.3 TON_IoT `type` has no source of its own
- The TON paper (Othman et al. 2026) reports the binary `label` only. The same settings are used for the 10-class `type`. This is our own extension (Tayyab's decision, 2026-11-21).
- The TON_IoT authors' own papers could not be opened (paywalled / repository offline):
  - Booij et al. 2022, IEEE IoTJ, doi:10.1109/JIOT.2021.3085194
  - Moustafa 2021, Sustainable Cities and Society, doi:10.1016/j.scs.2021.102994

### 3.4 Inconsistencies inside the papers
- **TON_IoT (Othman 2026):** batch size is 256 in Table 4 / Section 3.6 but 128 in Section 3.7. We use 256 (the table).
- **Edge-IIoT (Ferrag 2022):** the text says "a shallow DNN with only 47 trainable parameters". That contradicts Table 9: 2 hidden layers × 90 nodes gives thousands of weights (ours: 13,681).

### 3.5 Different data and preprocessing from the papers
The assignment rule is the same split and preprocessing for all models, so we did **not** copy each paper's preprocessing. This explains part of the gap to the papers' results.

| Paper | What they did that we did not |
|---|---|
| TON_IoT (Othman 2026) | Removed duplicates, GWO feature selection (30 features), SMOTE on the training set |
| Edge-IIoT (Ferrag 2022) | Trained the DNN on the larger `DNN-EdgeIIoT-dataset.csv` (about 2.2M rows) instead of `ML-EdgeIIoT-dataset.csv`; removed duplicates; SMOTE |
| X-IIoTID (Al-Hawawreh 2022) | 5-fold CV on all the data (no separate test set) |

### 3.6 X-IIoTID missing values
- Numeric NaN is filled with -1 (the same as for the trees), then standardised. For a neural network, a median fill plus a missing-indicator column would be an alternative.
- We kept -1 so that all models use identical preprocessing.

---

## 4. Process and housekeeping issues

| Issue | What happened | Status |
|---|---|---|
| X-IIoTID run time | A full run takes about 40-45 min on the main PC (3 notebooks × 6 fits) | Run on a second laptop instead (640 / 711 / 717 s per notebook) and merged back. Its training times come from a different machine, so they are only roughly comparable with the other models |
| Merging the second laptop's copy | The second laptop's zip also held an old, unexecuted TON_IoT `ann_type.ipynb` | Only the X-IIoTID files were taken; the executed TON_IoT notebook was kept |
| Leftover copies in the project | A nested `ML Project Datasets/` folder and `d--Machine-Learning-...zip` (an older snapshot) sit in the project root | Not used; waiting for Tayyab's OK to delete |
| Graph name mismatch | The second laptop saved the X-IIoTID class3 confusion graph as `02_confusion_matrix_roc.png`; the other binary folders use `02_confusion_matrix_and_roc.png` | Fixed: renamed locally and on Drive (2026-11-22) |
| `result.csv` missing | CLAUDE.md said it existed in the root; it did not | Recreated from Results.xlsx |
| Results.xlsx locked | The file was open in Excel, so the first save failed | Fixed after the file was closed. Keep it closed while results are written |
| Teammate ANN values overwritten | Row 6 (TON_IoT label) held a teammate's ANN values | Overwritten with ours, as decided by Tayyab. X-IIoTID rows 56/67 will be overwritten after the run |
| Hidden warnings | `ConvergenceWarning` is filtered in the notebooks, because the fixed epoch counts always trigger it. Given 2.1, the warning was correct | Stated here for transparency |
| Notebook style (TON `ann_label`) | Tayyab's hand-written cleaning cells (Q3-Q9) were copied as they are, including print statements and the "Classs" typo | Left unchanged on purpose |
| Leftover browser files | A research agent left a `.playwright-mcp` folder (browser logs) in the project | Deleted |
| Missing cell IDs | The X-IIoTID notebooks had no cell IDs (nbformat warning) | Fixed (normalised and validated) |

---

## 5. Open questions for the team
1. Should duplicates be removed (`DEDUPLICATE`)? This matters most for TON_IoT (55% duplicates).
2. Are we allowed to report an extra, clearly labelled ANN variant with more epochs, tuned on the validation folds only, to show the underfitting is fixable? The main result would stay paper-based.
3. Should SMOTE or class weights be considered for all models to address the imbalance seen in Edge-IIoT (FPR 0.54)? It would have to apply to every algorithm to keep the comparison fair.
