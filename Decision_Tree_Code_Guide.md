# Decision Tree Code Guide: TON_IoT (`train_test_network.csv`)

A step-by-step reference for the two notebooks in `scripts/Decision Trees/TON_IoT/`:

- `dt_label.ipynb` predicts **`label`** (binary: 0 = normal, 1 = attack)
- `dt_type.ipynb` predicts **`type`** (10 classes: normal, backdoor, ddos, dos, injection, mitm, password, ransomware, scanning, xss)

The notebooks contain only the questions and a "what to do" hint. Type the code yourself, one cell per question, and run each cell before moving on. The cell numbers (Q1, Q2, ...) match the question numbers in the notebooks.

This code was run end to end once before being written here. Your numbers should match the "What you should see" values to about 3 decimals.

---

## Contents

1. [The plan in one picture](#1-the-plan-in-one-picture)
2. [Part 1: both notebooks (setup, cleaning, split)](#2-part-1-both-notebooks-setup-cleaning-split)
3. [Part 2: the `label` notebook (model and metrics)](#3-part-2-the-label-notebook-model-and-metrics)
4. [Part 3: the `type` notebook (only what differs)](#4-part-3-the-type-notebook-only-what-differs)
5. [What you should see](#5-what-you-should-see)
6. [Decisions still open](#6-decisions-still-open)

---

## 1. The plan in one picture

```
All data (211,043 rows)
│
├── 20% TEST set ........ locked away, used ONCE at the end (Q13 onward)
│
└── 80% TRAINING set
    └── Stratified 5-fold (Q11). Each round:
        ├── 64% of all data → train a tree
        └── 16% of all data → validate it
        (5 rounds, so every row is used for validation once)

Final model (Q12) = train on the full 80%, then test once on the 20%.
```

Fixed decisions behind the code:

| Decision | Value | Why |
|---|---|---|
| Split | 80/20, stratified on `type` | Same rows in both notebooks and all models, so the comparison is fair |
| Validation | Stratified 5-fold on the 80% | Team decision. Shows how stable the model is |
| Hyperparameters | gini, best, `max_depth=5`, `min_samples_split=5`, `min_samples_leaf=1`, `random_state=42` | Taken from papers (see `HYPERPARAMETER_SOURCES.md`), so nothing is tuned on our data |
| Dropped columns | `src_ip`, `dst_ip`, `src_port`, `dst_port`, plus columns more than 95% empty | Identifiers cause overfitting; empty columns carry no information |
| Scaling | None | Trees compare values to thresholds, so scale doesn't matter (the ANN will need scaling) |
| Latency | `predict()` time on the test set / number of test rows, in ms per sample | Same method for every model |

---

## 2. Part 1: both notebooks (setup, cleaning, split)

Steps Q1 to Q8 and Q10 are identical in `dt_label.ipynb` and `dt_type.ipynb`. Q9 differs (shown for `label` here, for `type` in Part 3).

### Q1: Imports

**What it does:** loads the tools. pandas holds the tables, sklearn provides the tree and the metrics, matplotlib and seaborn draw the plots. `RANDOM_STATE = 42` makes every random choice repeatable, so you get the same results each run.

```python
import time
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.model_selection import train_test_split, StratifiedKFold
from sklearn.preprocessing import LabelEncoder          # needed in the `type` notebook only
from sklearn.tree import DecisionTreeClassifier, plot_tree
from sklearn.metrics import (accuracy_score, precision_score, recall_score, f1_score,
                             roc_auc_score, roc_curve, log_loss, matthews_corrcoef,
                             confusion_matrix, classification_report)

RANDOM_STATE = 42
DATA_PATH = "../../../train_test_network.csv"
```

The path goes up three folders because the notebook lives in `scripts/Decision Trees/TON_IoT/`.

### Q2: Settings

**What it does:** keeps the tree's settings in one dictionary `HP`, so you never retype them. `DEDUPLICATE` is a switch for the duplicate-rows decision (see [section 6](#6-decisions-still-open)).

```python
HP = dict(criterion="gini", splitter="best", max_depth=5,
          min_samples_split=5, min_samples_leaf=1, random_state=RANDOM_STATE)
DEDUPLICATE = False
```

### Q3: Load the data

**What it does:** reads the CSV. `encoding="utf-8-sig"` removes an invisible character at the start of the file; without it the first column would be misnamed.

```python
df = pd.read_csv(DATA_PATH, encoding="utf-8-sig")
print("Rows, columns:", df.shape)
```

Expect `(211043, 44)`.

### Q4: Look at the data

**What it does:** shows the first rows, the column types, summary statistics, and how many rows each class has. Check whether the classes are balanced and which is rarest. (In the `type` notebook, replace `"label"` with `"type"`.)

```python
display(df.head())
df.info()
display(df.describe())
print(df["label"].value_counts())
print(df["label"].value_counts(normalize=True).round(4))
df["label"].value_counts().plot(kind="bar", title="Class distribution: label")
plt.show()
```

### Q5: Missing values

**What it does:** this dataset writes "missing" as `-`. The code turns every `-` into a real blank (NaN), then lists the percentage missing per column.

```python
df = df.replace("-", np.nan)
missing_pct = (df.isna().mean() * 100).sort_values(ascending=False)
display(missing_pct[missing_pct > 0].round(2))
```

### Q6: Drop mostly empty columns

**What it does:** a column that is over 95% blank gives the model almost nothing to learn from. This drops them.

```python
sparse_cols = missing_pct[missing_pct > 95].index.tolist()
print("Dropping", len(sparse_cols), "columns:", sparse_cols)
df = df.drop(columns=sparse_cols)
print("Shape now:", df.shape)
```

Expect 16 columns dropped (all the `http_*`, `ssl_*` and `weird_*` ones). The shape becomes `(211043, 28)`.

### Q7: Drop identifier columns

**What it does:** IPs and ports say who is talking, not how an attack behaves. A tree that memorises them would not work on a different network.

```python
df = df.drop(columns=["src_ip", "dst_ip", "src_port", "dst_port"])
print("Shape now:", df.shape)
```

The shape becomes `(211043, 24)`.

### Q8: Duplicate rows

**What it does:** counts identical rows. With the switch on, it removes them. Duplicates are a problem because the same row can land in both train and test, so the model "sees" test rows during training and the scores look better than they are.

```python
n_dups = df.duplicated().sum()
print("Duplicate rows:", n_dups, f"({n_dups / len(df):.1%})")
print("Rows before:", len(df))
if DEDUPLICATE:
    df = df.drop_duplicates().reset_index(drop=True)
print("Rows after: ", len(df))
print(df["type"].value_counts())
```

Expect about 117,159 duplicates (55%). If removed, 93,884 rows remain and `ransomware` falls from 20,000 to about 260. Run once with `DEDUPLICATE = False` and once with `True`, and write both results in `AUDIT_LOG.md`.

### Q9: Split the data (`label` notebook)

**What it does:**
1. `X` is the inputs. Both `label` and `type` are removed from it, so the model can't see the answer (that would be leakage).
2. `y` is the answer we want to predict.
3. The data is split 80/20. `stratify=df["type"]` keeps the class proportions the same in train and test, and makes both notebooks use the exact same rows.

```python
X = df.drop(columns=["label", "type"])
y = df["label"]

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.20, stratify=df["type"], random_state=RANDOM_STATE)

print("Train:", X_train.shape, " Test:", X_test.shape)
print(y_train.value_counts(normalize=True).round(4))
print(y_test.value_counts(normalize=True).round(4))
```

Expect 168,834 training rows and 42,209 test rows (with duplicates kept). The class shares should be nearly identical in train and test (about 76.3% attack).

### Q10: Encode text columns

**What it does:** the tree needs numbers, so the text columns are converted:
1. Missing text becomes its own category, `"missing"`.
2. Columns with few values (`proto`, `service`, `conn_state`, `dns_AA`, `dns_RD`, `dns_RA`, `dns_rejected`) are one-hot encoded: one yes/no column per value.
3. `dns_query` has 700+ different values, so one-hot would add too many columns. It is replaced by how common each value is (frequency encoding).

Everything is learned from the training data only and then applied to the test data. Learning from the full data would let test information leak into training. `reindex` makes the test columns match the training columns exactly.

```python
onehot_cols = ["proto", "service", "conn_state", "dns_AA", "dns_RD", "dns_RA", "dns_rejected"]
X_train = X_train.copy()
X_test = X_test.copy()

for c in onehot_cols + ["dns_query"]:
    X_train[c] = X_train[c].fillna("missing")
    X_test[c] = X_test[c].fillna("missing")

freq_map = X_train["dns_query"].value_counts(normalize=True)
X_train["dns_query"] = X_train["dns_query"].map(freq_map)
X_test["dns_query"] = X_test["dns_query"].map(freq_map).fillna(0)   # unseen in training -> 0

X_train = pd.get_dummies(X_train, columns=onehot_cols).astype(float)
X_test = pd.get_dummies(X_test, columns=onehot_cols).reindex(columns=X_train.columns, fill_value=0).astype(float)

print("Features:", X_train.shape[1])
print("NaN left?", X_train.isna().any().any(), X_test.isna().any().any())
```

Expect 52 features and `False False` for NaN.

---

## 3. Part 2: the `label` notebook (model and metrics)

### Q11: Validation with stratified 5-fold

**What it does:** splits the 80% training data into 5 parts. Each round trains a new tree on 4 parts (64% of all data) and scores it on the fifth (16%). The test set is not touched.
- The **mean** score tells you how good the tree is.
- The **standard deviation** tells you how stable it is across different slices of data. A small value means the model doesn't depend on lucky data.

```python
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=RANDOM_STATE)

rows = []
for i, (tr, va) in enumerate(skf.split(X_train, y_train), start=1):
    m = DecisionTreeClassifier(**HP)
    m.fit(X_train.iloc[tr], y_train.iloc[tr])
    pred = m.predict(X_train.iloc[va])
    rows.append({"fold": i, "n_train": len(tr), "n_val": len(va),
                 "val_accuracy": accuracy_score(y_train.iloc[va], pred),
                 "val_F1": f1_score(y_train.iloc[va], pred)})
fold_df = pd.DataFrame(rows)
display(fold_df.round(4))
print("Mean accuracy: %.4f +/- %.4f" % (fold_df.val_accuracy.mean(), fold_df.val_accuracy.std()))
print("Mean F1:       %.4f +/- %.4f" % (fold_df.val_F1.mean(), fold_df.val_F1.std()))
```

`DecisionTreeClassifier(**HP)` unpacks the dictionary into the tree's settings.

### Q12: Train the final model

**What it does:** trains one tree on the full 80% and times it. This is the model whose test score you report.

```python
model = DecisionTreeClassifier(**HP)
t0 = time.perf_counter()
model.fit(X_train, y_train)
train_time = time.perf_counter() - t0
print("Training time: %.3f s" % train_time)
print("Depth:", model.get_depth(), " Leaves:", model.get_n_leaves())
```

### Q13: Predict on the test set and time it

**What it does:** this is the first use of the test set. Latency is the time to predict the whole test set divided by the number of rows, in ms per sample. `predict_proba` gives the class probabilities, which ROC-AUC and Log-Loss need.

```python
t0 = time.perf_counter()
y_pred = model.predict(X_test)
latency_ms = (time.perf_counter() - t0) / len(X_test) * 1000
y_proba = model.predict_proba(X_test)
print("Latency: %.6f ms/sample" % latency_ms)
```

Time only `predict()`, not the preprocessing.

### Q14: Metrics

**What it does:** computes everything `Results.xlsx` asks for.

| Metric | Plain meaning |
|---|---|
| Accuracy | Share of all predictions that are right |
| Precision | Of everything flagged as an attack, how much really was |
| Recall | Of all real attacks, how many were caught |
| F1 | Balance of precision and recall |
| ROC-AUC | How well the model separates the two classes at any threshold |
| Log-Loss | How confident and correct the probabilities are (lower is better) |
| FPR | Share of normal traffic wrongly flagged as an attack |
| MCC | One score that stays honest when classes are unequal |
| TP, TN, FP, FN | Counts of attacks caught, normals passed, false alarms, missed attacks |

```python
accuracy  = accuracy_score(y_test, y_pred)
precision = precision_score(y_test, y_pred)
recall    = recall_score(y_test, y_pred)
f1        = f1_score(y_test, y_pred)
roc_auc   = roc_auc_score(y_test, y_proba[:, 1])
logloss   = log_loss(y_test, y_proba)
mcc       = matthews_corrcoef(y_test, y_pred)

tn, fp, fn, tp = confusion_matrix(y_test, y_pred).ravel()
fpr = fp / (fp + tn)

print(f"Accuracy {accuracy:.4f} | Precision {precision:.4f} | Recall {recall:.4f} | F1 {f1:.4f}")
print(f"ROC-AUC {roc_auc:.4f} | Log-Loss {logloss:.4f} | FPR {fpr:.4f} | MCC {mcc:.4f}")
print(f"TP {tp} | TN {tn} | FP {fp} | FN {fn}")
```

`y_proba[:, 1]` is the probability of class 1 (attack).

### Q15: Overfitting check

**What it does:** compares the training score with the test score.
- Train far above test: the tree memorised the training data (overfitting).
- Both low: the tree is too simple (underfitting).
- Both close and high: good.

```python
train_acc = accuracy_score(y_train, model.predict(X_train))
test_acc = accuracy_score(y_test, y_pred)
print(f"Train {train_acc:.4f} | Test {test_acc:.4f} | Gap {train_acc - test_acc:+.4f}")
```

### Q16: Plots

**What it does:** left, the confusion matrix (the diagonal is correct predictions). Right, the ROC curve: the closer it hugs the top-left corner, the better.

```python
fig, ax = plt.subplots(1, 2, figsize=(12, 4.5))
sns.heatmap(confusion_matrix(y_test, y_pred), annot=True, fmt="d", cmap="Blues", ax=ax[0],
            xticklabels=["normal (0)", "attack (1)"], yticklabels=["normal (0)", "attack (1)"])
ax[0].set(title="Confusion matrix (test set)", xlabel="Predicted", ylabel="Actual")

fpr_c, tpr_c, _ = roc_curve(y_test, y_proba[:, 1])
ax[1].plot(fpr_c, tpr_c, label=f"AUC = {roc_auc:.4f}")
ax[1].plot([0, 1], [0, 1], "k--", alpha=0.4)
ax[1].set(title="ROC curve", xlabel="False Positive Rate", ylabel="True Positive Rate")
ax[1].legend()
plt.tight_layout()
plt.show()
```

### Q17: Feature importance

**What it does:** shows which columns the tree relied on most when deciding.

```python
importances = pd.Series(model.feature_importances_, index=X_train.columns).sort_values(ascending=False)
display(importances.head(10).round(4))
importances.head(10).sort_values().plot(kind="barh", title="Top 10 feature importances")
plt.tight_layout()
plt.show()
```

### Q18: Draw the tree

**What it does:** draws the first 3 levels, so you can read the first decisions the tree makes and check they make sense for network attacks.

```python
plt.figure(figsize=(20, 8))
plot_tree(model, feature_names=list(X_train.columns), class_names=["normal", "attack"],
          filled=True, max_depth=3, fontsize=8)
plt.show()
```

### Q19: Results table

**What it does:** collects every number in one table, ready to copy into `AUDIT_LOG.md`. Check that TP + TN + FP + FN equals the number of test rows.

```python
results = pd.DataFrame([{
    "Accuracy": accuracy, "Precision": precision, "Recall": recall, "F1-Score": f1,
    "ROC-AUC": roc_auc, "Log-Loss": logloss, "FPR": fpr, "MCC": mcc,
    "Inference Latency (ms/sample)": latency_ms, "Training time (s)": train_time,
    "TP": tp, "TN": tn, "FP": fp, "FN": fn, "Test rows": len(y_test),
}], index=["Decision Tree - label"])
display(results.T)
```

---

## 4. Part 3: the `type` notebook (only what differs)

Q1 to Q3, Q5 to Q8, Q10, Q12, Q13, Q15 and Q17 are identical to the `label` notebook. Everything below differs.

**Q4:** use `"type"` instead of `"label"`.

### Q9: Split (type)

**What it does:** the target is text, so `LabelEncoder` turns it into numbers 0 to 9. `class_names` remembers which number is which class.

```python
le = LabelEncoder()
y = pd.Series(le.fit_transform(df["type"]), index=df.index)
class_names = list(le.classes_)
print(dict(enumerate(class_names)))

X = df.drop(columns=["label", "type"])
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.20, stratify=df["type"], random_state=RANDOM_STATE)
```

### Q11: Validation folds (type)

**What it does:** the same as the `label` version, except F1 is a **macro average**: each class counts equally, so rare classes can't be ignored.

```python
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=RANDOM_STATE)

rows = []
for i, (tr, va) in enumerate(skf.split(X_train, y_train), start=1):
    m = DecisionTreeClassifier(**HP)
    m.fit(X_train.iloc[tr], y_train.iloc[tr])
    pred = m.predict(X_train.iloc[va])
    rows.append({"fold": i, "n_train": len(tr), "n_val": len(va),
                 "val_accuracy": accuracy_score(y_train.iloc[va], pred),
                 "val_macro_F1": f1_score(y_train.iloc[va], pred, average="macro")})
fold_df = pd.DataFrame(rows)
display(fold_df.round(4))
print("Mean accuracy: %.4f +/- %.4f" % (fold_df.val_accuracy.mean(), fold_df.val_accuracy.std()))
print("Mean macro-F1: %.4f +/- %.4f" % (fold_df.val_macro_F1.mean(), fold_df.val_macro_F1.std()))
```

### Q14: Metrics (type)

**What it does:** with 10 classes, each metric is computed per class and then averaged (macro).
- ROC-AUC uses "one-vs-rest": each class against all the others.
- TP/TN/FP/FN are also counted one class versus the rest, then averaged. That is why the sheet shows fractional values for `type`.
- `classification_report` lists each class separately. This is where you will see which classes the tree fails on.

```python
labels = np.arange(len(class_names))
accuracy  = accuracy_score(y_test, y_pred)
precision = precision_score(y_test, y_pred, average="macro", zero_division=0)
recall    = recall_score(y_test, y_pred, average="macro", zero_division=0)
f1        = f1_score(y_test, y_pred, average="macro", zero_division=0)
roc_auc   = roc_auc_score(y_test, y_proba, multi_class="ovr", average="macro", labels=labels)
logloss   = log_loss(y_test, y_proba, labels=labels)
mcc       = matthews_corrcoef(y_test, y_pred)

cm = confusion_matrix(y_test, y_pred, labels=labels)
TP = np.diag(cm)                       # correct predictions per class
FP = cm.sum(axis=0) - TP               # predicted as this class, but wrong
FN = cm.sum(axis=1) - TP               # really this class, but missed
TN = cm.sum() - (TP + FP + FN)         # everything else
fpr = np.mean(FP / (FP + TN))

print(f"Accuracy {accuracy:.4f} | Precision {precision:.4f} | Recall {recall:.4f} | F1 {f1:.4f}  (macro)")
print(f"ROC-AUC {roc_auc:.4f} | Log-Loss {logloss:.4f} | FPR {fpr:.4f} | MCC {mcc:.4f}")
print(f"Macro TP {TP.mean():.2f} | TN {TN.mean():.2f} | FP {FP.mean():.2f} | FN {FN.mean():.2f}")
print(classification_report(y_test, y_pred, target_names=class_names, zero_division=0))
```

### Q16: Plots (type)

**What it does:** one 10×10 confusion matrix. Each row is divided by its total, so a rare class like `mitm` is as readable as a big one.

```python
cm_norm = confusion_matrix(y_test, y_pred, labels=labels, normalize="true")
plt.figure(figsize=(9, 7))
sns.heatmap(cm_norm, annot=True, fmt=".2f", cmap="Blues",
            xticklabels=class_names, yticklabels=class_names)
plt.title("Confusion matrix, row-normalised (test set)")
plt.xlabel("Predicted")
plt.ylabel("Actual")
plt.tight_layout()
plt.show()
```

### Q18: Draw the tree (type)

**What it does:** the same as the `label` version, but with the real class names.

```python
plt.figure(figsize=(22, 9))
plot_tree(model, feature_names=list(X_train.columns), class_names=class_names,
          filled=True, max_depth=3, fontsize=8)
plt.show()
```

### Q19: Results table (type)

**What it does:** the same table, but with the macro TP/TN/FP/FN values and a different index name. Replace the confusion entries and the index:

```python
"TP": TP.mean(), "TN": TN.mean(), "FP": FP.mean(), "FN": FN.mean(),
```
```python
index=["Decision Tree - type (macro)"]
```

---

## 5. What you should see

All figures are on the test set, with paper hyperparameters and `DEDUPLICATE = False` unless noted.

| Target | Accuracy | Precision | Recall | F1 | ROC-AUC | Log-Loss | FPR | MCC |
|---|---|---|---|---|---|---|---|---|
| `label` | 0.9914 | 0.9926 | 0.9962 | 0.9944 | 0.9964 | 0.0354 | 0.0240 | 0.9762 |
| `type` (macro) | 0.7959 | 0.7976 | 0.7221 | 0.7205 | 0.9663 | 0.5305 | 0.0232 | 0.7790 |

- `label`: TP 32,087, TN 9,760, FP 240, FN 122, 42,209 test rows. Five-fold validation accuracy 0.9911 ± 0.0008. Training time about 0.5 s (varies by machine).
- `type`: macro TP 3,359.5, TN 37,126.7, FP 861.4, FN 861.4.
- With `DEDUPLICATE = True`: `label` accuracy 0.9802, `type` accuracy 0.8261 but macro-F1 only 0.5491.

**Why `type` is low:** `max_depth=5` allows at most 32 leaves, which is too few for 10 classes. The tree underfits (train accuracy 0.794, test 0.796). In the per-class report, `ddos` recall is only 0.25 and `mitm` (209 test rows) is never predicted. `label` is fine at this depth. Treat this as a finding to discuss in the report, and decide whether to add a clearly labelled deeper-tree comparison.

---

## 6. Decisions still open

| # | Question | Where it matters |
|---|---|---|
| O2 | Remove duplicates before the split? (55% of rows are duplicates; removing them shrinks `ransomware` to about 260 rows) | `DEDUPLICATE` switch in Q2. Ask teammates what they did |
| O3 | Add a deeper-tree variant for `type`? | Report discussion; keep the paper-value result as the main one |
| – | Test-row count in `Results.xlsx` | The KNN label row sums to 36,719 test rows, but ours is 42,209 (or 18,777 if deduplicated). Ask teammates how they split |

Related files in the project root: `AUDIT_LOG.md` (decisions and figures), `HYPERPARAMETER_SOURCES.md` (paper sources for the report), `CLAUDE.md` (context for a new session).
