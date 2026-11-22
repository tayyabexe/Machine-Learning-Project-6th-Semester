<div align="center">

<img src="../assets/divider.svg" alt="" width="100%"/>

# 📊 Graphs

**75 PNG files: 49 taken straight from the executed notebooks, plus 26 versions made for the group report.**

</div>

## 1. Folder layout

```text
Graphs/
├── Decision Trees/<Dataset>/dt_<target>/
└── ANN/<Dataset>/ann_<target>/
```

`<Dataset>` is `TON_IoT`, `Edge-IIoT` or `X-IIoTID`.

## 2. File names

| # | File | 🌳 DT | 🧠 ANN | What it shows |
|:-:|---|:-:|:-:|---|
| 1 | `01_class_distribution.png` | ✅ | ✅ | How many rows each class has (shows the imbalance) |
| 2 | `02_confusion_matrix_and_roc.png` | ✅ | ✅ | **Binary targets:** confusion matrix (counts) and ROC curve with AUC |
| 2 | `02_confusion_matrix.png` | ✅ | ✅ | **Multiclass targets:** row-normalised confusion matrix (the diagonal is each class's recall) |
| 3 | `03_feature_importance.png` | ✅ | ➖ | Top 10 features the tree relies on |
| 4 | `04_tree_top_levels.png` | ✅ | ➖ | The first 3 levels of the tree |
| 3 | `03_loss_curve.png` | ➖ | ✅ | Training loss per epoch (still falling at the end = underfitting) |

**Report versions**

| File | Model | What it is |
|---|---|---|
| `05_report_confusion_matrix.png`, `06_report_roc_curve.png` | DT (all 7) | Larger versions redrawn from a re-run that reproduced the saved results exactly |
| `04_report_confusion_matrix.png`, `05_report_roc_curve.png` | ANN TON_IoT and Edge-IIoT | Larger versions redrawn from a re-run that reproduced the saved results exactly |
| `04_report_confusion_matrix.png`, `05_report_roc_curve.png` | ANN `class3` | Made from the saved results and figure of the reported model |
| `05_report_roc_curve_rerun.png` | ANN `class2`, `class1` | **From a re-run on another laptop that did not reproduce the reported model exactly.** Macro AUC 0.999716 and 0.999609, against 0.999606 and 0.999601 in the results table |

For ANN `class2` and `class1`, the report uses the original `02_confusion_matrix.png`.

## 3. Quick links

| Dataset | 🌳 Decision Tree | 🧠 ANN |
|---|---|---|
| 🌐 TON_IoT | [`dt_label`](Decision%20Trees/TON_IoT/dt_label) · [`dt_type`](Decision%20Trees/TON_IoT/dt_type) | [`ann_label`](ANN/TON_IoT/ann_label) · [`ann_type`](ANN/TON_IoT/ann_type) |
| 🏭 Edge-IIoTset | [`dt_label`](Decision%20Trees/Edge-IIoT/dt_label) · [`dt_type`](Decision%20Trees/Edge-IIoT/dt_type) | [`ann_label`](ANN/Edge-IIoT/ann_label) · [`ann_type`](ANN/Edge-IIoT/ann_type) |
| ⚙️ X-IIoTID | [`dt_class3`](Decision%20Trees/X-IIoTID/dt_class3) · [`dt_class2`](Decision%20Trees/X-IIoTID/dt_class2) · [`dt_class1`](Decision%20Trees/X-IIoTID/dt_class1) | [`ann_class3`](ANN/X-IIoTID/ann_class3) · [`ann_class2`](ANN/X-IIoTID/ann_class2) · [`ann_class1`](ANN/X-IIoTID/ann_class1) |

<div align="center"><img src="../assets/divider.svg" alt="" width="100%"/></div>
