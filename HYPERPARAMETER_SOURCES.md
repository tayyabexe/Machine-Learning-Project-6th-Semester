# Hyperparameter Sources (Decision Tree; ANN section further below)

Maintained file for the report's "Models & Experimental Setup" section. Updated 2026-11-21: each dataset now has its own sourced hyperparameters. The previous shared values are kept at the bottom for history.

## Current values per dataset and notebook

| Dataset | Notebook (target) | criterion | splitter | max_depth | min_samples_split | min_samples_leaf | random_state | Source |
|---|---|---|---|---|---|---|---|---|
| TON_IoT | `dt_label.ipynb` (label, binary) | gini | best | None | 2 | 1 | 42 | [S1] |
| TON_IoT | `dt_type.ipynb` (type, 10 classes) | gini | best | None | 2 | 1 | 42 | [S1] |
| Edge-IIoT | `dt_label.ipynb` (Attack_label, binary) | gini | best | None | 2 | 1 | 42 | [S3], context [S2] |
| Edge-IIoT | `dt_type.ipynb` (Attack_type, 15 classes) | gini | best | None | 2 | 1 | 42 | [S3], context [S2] |
| X-IIoTID | `dt_class3.ipynb` (binary) | gini | best | None | 2 | 1 | 42 | [S4] |
| X-IIoTID | `dt_class2.ipynb` (10 classes) | gini | best | None | 2 | 1 | 42 | [S4] |
| X-IIoTID | `dt_class1.ipynb` (19 classes) | gini | best | None | 2 | 1 | 42 | [S4] |

`random_state=42` is our fixed seed for all models (also used in [S1]).

### What each source actually states
Legend: **bold** = written explicitly · *(default)* = the paper says "default parameters", so the scikit-learn default applies · — = not mentioned.

| Source | criterion | splitter | max_depth | min_samples_split | min_samples_leaf | random_state |
|---|---|---|---|---|---|---|
| [S1] Dharini et al. 2026 (TON_IoT) | **gini** | — | **None** | **2** | **1** | **42** |
| [S3] Kuo & Wu 2026 (Edge-IIoT; tree on CICIoT2023) | **gini** | **best** | **None** | **2** | 1 *(default)* | — |
| [S4] Al-Hawawreh et al. 2022 (X-IIoTID) | gini *(default)* | best *(default)* | None *(default)* | 2 *(default)* | 1 *(default)* | — |
| [S2] Ferrag et al. 2022 (Edge-IIoT dataset paper) | — | — | — | — | — | — |

## Sources and exact evidence

**[S1] TON_IoT: verified from the full text (PDF read).**
N. Dharini, V. S. Janani, J. Katiravan, "Efficient detection of intrusions in TON-IoT dataset using hybrid feature selection approach," *Scientific Reports*, vol. 16, art. 7763, 2026. doi:10.1038/s41598-026-37834-y
- Quote: "The Decision Tree model was trained with default lightweight hyperparameters (gini criterion, no fixed maximum depth, minimum split of two samples, and one sample per leaf)".
- Quote: "70% of the data was used for training and 30% for testing, with a fixed random seed (random_state = 42)".
- Their DT results: full ToN-IoT accuracy 1.0 (binary, Table 12); reduced 5-feature set 0.986 binary, 0.972 multi-class.
- Caveat: their Table 10 also lists "Reduced Pruning" and "No. of decision trees created 1702", which do not match scikit-learn. We follow the text sentence above.

**[S2] Edge-IIoT: dataset paper (verified, PDF read). No values published.**
M. A. Ferrag, O. Friha, D. Hamouda, L. Maglaras, H. Janicke, "Edge-IIoTset: A New Comprehensive Realistic Cyber Security Dataset of IoT and IIoT Applications for Centralized and Federated Learning," *IEEE Access*, vol. 10, pp. 40281-40306, 2022. doi:10.1109/ACCESS.2022.3165809
- Quote: "hyper-parameter tuning using Grid Search with stratified cross-validation". The final DT values are **not reported**.
- Their DT results: 2-class 99.98%, 15-class 67.11% accuracy.

**[S3] Edge-IIoT: values used. Verified from the full text (PDF downloaded by Tayyab, 2026-11-21).**
C.-W. Kuo, C.-X. Wu, "A Lightweight Two-Stage Intrusion Detection Framework Optimized for Edge-Based IoT Environments," *Computers, Materials & Continua*, vol. 87, no. 3, art. 53, 2026. doi:10.32604/cmc.2026.076767
- Quote (Section 3.3, Stage 1): "The Stage 1 Decision Tree was implemented using Scikit-learn with default hyperparameters ... criterion ('gini') ... max_depth (None): Allows unrestricted tree growth ... min_samples_split (2) & splitter ('best')".
- Their split: 80% train / 20% test, preprocessing fitted on the training set only (same as ours).
- **Caveat:** this Decision Tree was trained on **CICIoT2023** (binary DDoS vs non-DDoS, 99.99% accuracy). Edge-IIoTset was used only to check how well the whole two-stage framework generalizes (accuracy 94.73%, Table 14). The values are therefore not Edge-IIoT-specific. They are the scikit-learn defaults, the same as [S1] and [S4]. Say this in the report.

**[S4] X-IIoTID: verified from the full text (PDF read, authors' copy from the dataset's GitHub).**
M. Al-Hawawreh, E. Sitnikova, N. Aboutorab, "X-IIoTID: A Connectivity-Agnostic and Device-Agnostic Intrusion Data Set for Industrial Internet of Things," *IEEE Internet of Things Journal*, vol. 9, no. 5, pp. 3962-3977, 2022. doi:10.1109/JIOT.2021.3102056
- Quote: "we use the default parameters of the algorithms ... Our experiments are conducted using the Python language (Scikit[-learn])". This means the scikit-learn `DecisionTreeClassifier` defaults (gini, best, None, 2, 1).
- Their DT results (5-fold CV): binary 99.54%, 10-class 99.49% (macro F1 97.27%), 19-class 99.45% (macro F1 93.80%).
- Bonus for the ANN (same paper): DNN with 3 hidden layers of 200 neurons, ReLU, sigmoid (binary) / softmax (multi-class), batch 250, epochs 10, optimizer RMSprop.

**scikit-learn documentation**, `DecisionTreeClassifier`: https://scikit-learn.org/stable/modules/generated/sklearn.tree.DecisionTreeClassifier.html

## Notes for the report
- All three datasets end up with the unrestricted (full-depth) tree, because the dataset papers ([S1], [S4]) used library defaults. Say this plainly.
- Values were taken from papers, not tuned on our data. The stratified 5-fold validation shows stability; the test set is used once.
- Our split (80/20, then 5-fold on the 80%) differs from [S1] (70/30) and [S4] (5-fold CV on all data), so the numbers are not exactly comparable.
- A full-depth tree can overfit. Compare train and test accuracy (Q15) and report it.

---

# ANN (MLP) hyperparameters (added 2026-11-21)

Implementation: scikit-learn 1.8 `MLPClassifier` (TensorFlow/PyTorch are not installed). For every model: `StandardScaler` fitted on the training data only (inside each fold too), `shuffle=True`, `early_stopping=False`, `random_state=42`. The output layer is chosen by scikit-learn: one logistic unit for binary targets (equivalent to a 2-unit softmax/sigmoid), softmax for multiclass; loss = cross-entropy.

| Dataset | Notebooks | hidden_layer_sizes | activation | solver | learning_rate_init | batch_size | max_iter (epochs) | alpha (L2) | n_iter_no_change | Source |
|---|---|---|---|---|---|---|---|---|---|---|
| TON_IoT | `ann_label`, `ann_type` | (128, 64, 32) | relu | adam | 0.001 | 256 | 50 | 0.0001 *(default)* | 10 | [A1] |
| Edge-IIoT | `ann_label`, `ann_type` | (90, 90) | relu | adam | 0.001 *(default)* | 800 | 25 | 0.0001 *(default)* | 25 (always run 25) | [A2] |
| X-IIoTID | `ann_class3`, `ann_class2`, `ann_class1` | (200, 200, 200) | relu | adam (paper: RMSprop) | 0.001 *(default)* | 250 | 10 | 0.0 | 10 (default) | [A3] |

*(default)* = not stated in the paper, scikit-learn/Keras default used.

**[A1] TON_IoT: verified from the full text (MDPI PDF via mdpi-res.com mirror).**
A. M. H. Othman, M. F. Ab Razak, A. Firdaus, H. Tahaei, M. Gheisari, "A Hybrid CNN–LSTM Model for IoT Intrusion Detection: A Robustness Analysis Across Datasets," *Future Internet*, vol. 18, no. 7, art. 345, 2026. doi:10.3390/fi18070345
- Same file as ours: Table 2 "ToN_IoT Benign 50,000 | Attack 161,043 | 44 features" (binary label only).
- Table 4: "Hidden Layers … DNN 128–64–32, MLP 64–32"; "Activation ReLU"; "Output Activation Sigmoid"; "Loss Function BCEWithLogitsLoss"; "Optimizer Adam"; "Learning Rate 0.001"; "Batch Size 256"; "Epochs 50".
- Text: "Training runs up to 50 epochs with early stopping on validation loss … (patience = 10)". "No hyperparameter tuning techniques, such as grid search and random search, were used."
- Their results (Table 7): MLP acc 99.61%, DNN 99.77% (binary). 80/20 stratified split, but with deduplication, GWO feature selection (30 features) and SMOTE, which we do not use.
- **Caveats:** (1) batch size is 256 in Table 4 / Section 3.6 but 128 in Section 3.7; we use 256 (the table). (2) No L2 value given. (3) Early stopping mapped to scikit-learn's training-loss rule (`n_iter_no_change=10`), not validation loss with best-weight restore. (4) **The paper reports the binary label only; the same settings are used for the 10-class `type` (our extension, decided by Tayyab 2026-11-21).**
- Not openable (paywalled / repository offline): Booij et al. 2022 IEEE IoTJ doi:10.1109/JIOT.2021.3085194; Moustafa 2021 Sustainable Cities and Society doi:10.1016/j.scs.2021.102994 (TON_IoT authors).

**[A2] Edge-IIoT: verified from the full text (IEEE Xplore PDF, Table 9 read as an image on page 20).**
Ferrag et al. 2022, IEEE Access 10:40281-40306, doi:10.1109/ACCESS.2022.3165809 (same as [S2]).
- Table 9 "Settings for deep learning classifier": centralized batch size 800, total epochs 25; hidden layers 2; hidden nodes 90; regularization L2; activation ReLu; classification function softmax; optimizer adam.
- Not given: learning rate, L2 strength, dropout, loss. The text's "shallow DNN with only 47 trainable parameters" contradicts Table 9 (2 × 90 nodes gives thousands of weights).
- Their DNN results: 2-class 99.99%, 6-class 96.01%, 15-class 94.67%, trained on `DNN-EdgeIIoT-dataset.csv` (~2.2M rows) with SMOTE, not on our `ML-EdgeIIoT-dataset.csv`. This explains part of the gap to our results.

**[A3] X-IIoTID: verified from the full text** (same paper as [S4], Al-Hawawreh et al. 2022, doi:10.1109/JIOT.2021.3102056).
- DNN: 3 hidden layers of 200 neurons, ReLU, sigmoid (binary) / softmax (multi-class), batch 250, 10 epochs, optimizer RMSprop.
- **Caveat:** RMSprop is not available in scikit-learn; Adam is used (Adam = RMSprop + momentum). Learning rate not stated → 0.001. No L2 stated → `alpha=0.0`.

Backup sources checked but not used (Edge-IIoT): Samin et al. 2024 PeerJ CS doi:10.7717/peerj-cs.2276 (128-64 ReLU, no optimizer/batch/epochs); Idouglid et al. 2025 Informatica 49:57-68 doi:10.31449/inf.v49i33.6944 (64-32-16, Adam, lr 0.001, 100 epochs, batch 32; weaker venue, inconsistent table).

---

## History: previous shared values (superseded 2026-11-21)
| Hyperparameter | Value | Evidence (supplied by the team, not verified) |
|---|---|---|
| criterion | gini | Li et al. (2024), Table 4 |
| splitter | best | Li et al. (2024), Table 4 |
| max_depth | 5 | Sharma et al. (2025), Algorithm 1 |
| min_samples_split | 5 | Sharma et al. (2025), Algorithm 1 |
| min_samples_leaf | 1 | scikit-learn default |
| random_state | 42 | Li et al. (2024), Table 4 |

These values underfitted every multiclass target: TON_IoT type accuracy 0.7959, Edge-IIoT Attack_type 0.4529, X-IIoTID class1 macro-F1 0.3292.
