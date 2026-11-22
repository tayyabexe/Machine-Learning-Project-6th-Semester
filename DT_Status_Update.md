# Decision Tree: Status Update (all 3 datasets)

**From:** Tayyab · **Date:** 21 Nov 2026 · **Models:** Decision Tree, 7 models in total (one per target column)

---

## 1. Summary

- All **7 Decision Tree models** are trained and tested on TON_IoT, Edge-IIoT and X-IIoTID.
- The first hyperparameters (`max_depth=5`) were **too shallow**: multiclass accuracy was as low as **0.45** on Edge-IIoT.
- I replaced them with **dataset-specific values from published papers** and retrained everything.
- Accuracy is now **0.96 to 0.998** on every model except Edge-IIoT multiclass, which went from **0.45 to 0.86**.

---

## 2. Models

| Dataset | Notebook | Target | Type |
|---|---|---|---|
| TON_IoT | `dt_label.ipynb` | `label` | Binary |
| TON_IoT | `dt_type.ipynb` | `type` | 10 classes |
| Edge-IIoT | `dt_label.ipynb` | `Attack_label` | Binary |
| Edge-IIoT | `dt_type.ipynb` | `Attack_type` | 15 classes |
| X-IIoTID | `dt_class3.ipynb` | `class3` | Binary |
| X-IIoTID | `dt_class2.ipynb` | `class2` | 10 classes |
| X-IIoTID | `dt_class1.ipynb` | `class1` | 19 classes |

Notebooks are in `scripts/Decision Trees/<Dataset>/`.

---

## 3. Why the hyperparameters changed

**Old values (first run, used for all datasets):**
```python
criterion="gini", splitter="best", max_depth=5, min_samples_split=5, min_samples_leaf=1, random_state=42
```
A tree of depth 5 can have at most 32 leaves, which is too few for 10 to 19 classes. Train and test accuracy were both low and almost equal, which means the model was **underfitting**. For example:
- On Edge-IIoT, 6 of the 15 attack types were never predicted.
- On X-IIoTID `class1`, 12 of the 19 classes had zero recall.

**New values (from papers, same for all 7 models):**
```python
criterion="gini", splitter="best", max_depth=None, min_samples_split=2, min_samples_leaf=1, random_state=42
```
`max_depth=None` means the tree grows until its leaves are pure.

---

## 4. Sources

| Dataset | Paper | What it says | Verified? |
|---|---|---|---|
| **TON_IoT** | N. Dharini, V. S. Janani, J. Katiravan, "Efficient detection of intrusions in TON-IoT dataset using hybrid feature selection approach," *Scientific Reports*, 16:7763, 2026. [doi:10.1038/s41598-026-37834-y](https://doi.org/10.1038/s41598-026-37834-y) | "gini criterion, no fixed maximum depth, minimum split of two samples, and one sample per leaf"; random_state = 42 | ✅ Read the full PDF |
| **X-IIoTID** | M. Al-Hawawreh, E. Sitnikova, N. Aboutorab, "X-IIoTID: A Connectivity-Agnostic and Device-Agnostic Intrusion Data Set for Industrial Internet of Things," *IEEE Internet of Things Journal*, 9(5):3962–3977, 2022. [doi:10.1109/JIOT.2021.3102056](https://doi.org/10.1109/JIOT.2021.3102056) | "we use the default parameters of the algorithms" (scikit-learn), which are the values above | ✅ Read the full PDF (dataset authors' paper) |
| **Edge-IIoT** | C.-W. Kuo, C.-X. Wu, "A Lightweight Two-Stage Intrusion Detection Framework Optimized for Edge-Based IoT Environments," *Computers, Materials & Continua*, 87(3):53, 2026. [doi:10.32604/cmc.2026.076767](https://doi.org/10.32604/cmc.2026.076767) | "Decision Tree was implemented using Scikit-learn with default hyperparameters": gini, max_depth None, min_samples_split 2, splitter "best" | ✅ Read the full PDF. ⚠️ Their tree was trained on CICIoT2023; Edge-IIoTset was only used to test the full framework (94.73%) |

The Edge-IIoT dataset paper itself (Ferrag et al., *IEEE Access* 10:40281, 2022, [doi:10.1109/ACCESS.2022.3165809](https://doi.org/10.1109/ACCESS.2022.3165809)) tuned its Decision Tree with grid search but **did not publish the final values**. So no paper gives Edge-IIoT-specific Decision Tree values. All three sources use the scikit-learn defaults.

### Hyperparameter values per source

Legend: **bold** = written explicitly in the paper · *(default)* = the paper says "default parameters" without listing this value, so the scikit-learn default applies · — = not mentioned.

| Source | Dataset | criterion | splitter | max_depth | min_samples_split | min_samples_leaf | random_state |
|---|---|---|---|---|---|---|---|
| Dharini et al. (2026), *Sci. Rep.* | TON_IoT | **gini** | — | **None** ("no fixed maximum depth") | **2** | **1** | **42** |
| Kuo & Wu (2026), *CMC* | Edge-IIoT (tree trained on CICIoT2023) | **gini** | **best** | **None** | **2** | 1 *(default)* | — |
| Al-Hawawreh et al. (2022), *IEEE IoT J.* | X-IIoTID | gini *(default)* | best *(default)* | None *(default)* | 2 *(default)* | 1 *(default)* | — |
| Ferrag et al. (2022), *IEEE Access* | Edge-IIoT (dataset paper) | — | — | — | — | — | — |
| scikit-learn defaults (reference) | — | gini | best | None | 2 | 1 | None |
| **What we used (all 7 models)** | all | **gini** | **best** | **None** | **2** | **1** | **42** |

`random_state=42` is our own fixed seed for every model, so the results are reproducible. It matches Dharini et al.

---

## 5. Results (test set, 20%)

Multiclass metrics are **macro-averaged**.

| Dataset | Model | Old Acc (depth 5) | **New Acc** | Precision | Recall | F1 | ROC-AUC | FPR | MCC |
|---|---|---|---|---|---|---|---|---|---|
| TON_IoT | label | 0.9914 | **0.9981** | 0.9986 | 0.9989 | 0.9988 | 0.9972 | 0.0044 | 0.9948 |
| TON_IoT | type | 0.7959 | **0.9766** | 0.9479 | 0.9516 | 0.9495 | 0.9800 | 0.0026 | 0.9732 |
| Edge-IIoT | Attack_label | 0.9646 | **0.9623** | 0.9764 | 0.9791 | 0.9777 | 0.9245 | 0.1298 | 0.8544 |
| Edge-IIoT | Attack_type | 0.4529 | **0.8573** | 0.8161 | 0.8425 | 0.8201 | 0.9282 | 0.0103 | 0.8447 |
| X-IIoTID | class3 | 0.9465 | **0.9958** | 0.9954 | 0.9959 | 0.9957 | 0.9958 | 0.0044 | 0.9915 |
| X-IIoTID | class2 | 0.8884 | **0.9953** | 0.9791 | 0.9821 | 0.9806 | 0.9907 | 0.0007 | 0.9931 |
| X-IIoTID | class1 | 0.8916 | **0.9953** | 0.9612 | 0.9656 | 0.9633 | 0.9826 | 0.0004 | 0.9932 |

Inference latency is about **0.0003 to 0.0012 ms per sample** for all models. Log-Loss, TP/TN/FP/FN and training time are in each notebook's results table (Q19).

---

## 6. Points to note

1. **Edge-IIoT `Attack_label` got slightly worse** (0.9646 → 0.9623) and now **overfits**: train accuracy is 0.9984 and test is 0.9623. Its FPR is high (0.13).
2. **Edge-IIoT `Attack_type` is still the weakest** at 0.86. For comparison, the dataset paper's own Decision Tree got only **67.11%** on 15 classes.
3. **High Log-Loss on Edge-IIoT** (1.33 and 4.57): a full-depth tree gives 0/1 probabilities, so every wrong prediction costs a lot.
4. **Our results are not exactly comparable with the papers**, because our split is different (80/20 + stratified 5-fold). TON_IoT used 70/30, and X-IIoTID used 5-fold CV on all the data.

---

## 7. Common setup (same for all models)

- **Split:** 80% train / 20% test, stratified on the finest label (`type`, `Attack_type`, `class1`), so all models of a dataset test on the same rows.
- **Validation:** stratified 5-fold on the 80% (64% train / 16% validation per fold). Used to check stability only; no parameter search was run. The first Decision Tree setting (max_depth=5) was replaced by the papers' library defaults after its test results showed underfitting, so the test set influenced that one choice and was used more than once. The Decision Tree test scores may therefore be slightly optimistic.
- **Dropped columns:** IPs, ports, timestamps, constant columns, and raw text/payload columns. TON_IoT also drops columns that are more than 95% empty.
- **Encoding:** one-hot for categorical columns, fitted on the training set only; no scaling (trees don't need it).
- **Duplicates:** kept for now (`DEDUPLICATE = False`). Duplicate share: TON_IoT 55%, Edge-IIoT 3.6%, X-IIoTID 14.6%.
- **Latency:** `predict()` time on the whole test set ÷ number of test rows (ms per sample).

---

## 8. Questions for the team

1. Should we **remove duplicate rows**? This matters most for TON_IoT (55% duplicates).
2. Are you also using **dataset-specific hyperparameters from papers** for your models, so the comparison is fair?
3. If anyone finds a paper with **Edge-IIoT-specific** Decision Tree values (ideally for 15-class `Attack_type`), please share it. The current source used a different dataset for its tree.
4. Our TON_IoT test set has **42,209 rows**. The KNN row in `Results.xlsx` sums to 36,719. Which split did you use?
