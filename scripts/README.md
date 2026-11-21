<div align="center">

<img src="../assets/divider.svg" alt="" width="100%"/>

# 📓 Notebooks

**14 Jupyter notebooks: one Decision Tree and one ANN for each of the 7 targets.**

</div>

## 1. How each notebook is organised

1. Every notebook is a list of **questions (Q1, Q2, ...)**, each followed by short code cells.
2. Code is explained with **comments**; the last line of a cell shows the result.
3. All notebooks are **saved with their outputs**, so you can read the results without running anything.
4. The datasets are loaded from the project root with `../../../<file>.csv`. See the main [README](../README.md#8-how-to-run).

## 2. All notebooks

| # | Dataset | Target | Classes | 🌳 Decision Tree | 🧠 ANN | DT accuracy | ANN accuracy |
|:-:|---|---|:-:|---|---|:-:|:-:|
| 1 | 🌐 TON_IoT | `label` | <kbd>2</kbd> | [`dt_label.ipynb`](Decision%20Trees/TON_IoT/dt_label.ipynb) | [`ann_label.ipynb`](ANN/TON_IoT/ann_label.ipynb) | 0.9981 | 0.9923 |
| 2 | 🌐 TON_IoT | `type` | <kbd>10</kbd> | [`dt_type.ipynb`](Decision%20Trees/TON_IoT/dt_type.ipynb) | [`ann_type.ipynb`](ANN/TON_IoT/ann_type.ipynb) | 0.9766 | 0.8232 |
| 3 | 🏭 Edge-IIoTset | `Attack_label` | <kbd>2</kbd> | [`dt_label.ipynb`](Decision%20Trees/Edge-IIoT/dt_label.ipynb) | [`ann_label.ipynb`](ANN/Edge-IIoT/ann_label.ipynb) | 0.9623 | 0.9098 |
| 4 | 🏭 Edge-IIoTset | `Attack_type` | <kbd>15</kbd> | [`dt_type.ipynb`](Decision%20Trees/Edge-IIoT/dt_type.ipynb) | [`ann_type.ipynb`](ANN/Edge-IIoT/ann_type.ipynb) | 0.8573 | 0.6645 |
| 5 | ⚙️ X-IIoTID | `class3` | <kbd>2</kbd> | [`dt_class3.ipynb`](Decision%20Trees/X-IIoTID/dt_class3.ipynb) | [`ann_class3.ipynb`](ANN/X-IIoTID/ann_class3.ipynb) | 0.9958 | 0.9899 |
| 6 | ⚙️ X-IIoTID | `class2` | <kbd>10</kbd> | [`dt_class2.ipynb`](Decision%20Trees/X-IIoTID/dt_class2.ipynb) | [`ann_class2.ipynb`](ANN/X-IIoTID/ann_class2.ipynb) | 0.9953 | 0.9887 |
| 7 | ⚙️ X-IIoTID | `class1` | <kbd>19</kbd> | [`dt_class1.ipynb`](Decision%20Trees/X-IIoTID/dt_class1.ipynb) | [`ann_class1.ipynb`](ANN/X-IIoTID/ann_class1.ipynb) | 0.9953 | 0.9886 |

## 3. What differs between the two model types

| Part | 🌳 Decision Tree | 🧠 ANN |
|---|---|---|
| Scaling | none (trees split on thresholds) | `StandardScaler`, fitted on training data only |
| Hyperparameters | same for all datasets (scikit-learn defaults, from papers) | one architecture per dataset (from papers) |
| Extra graphs | top 10 feature importances, first 3 tree levels | training loss per epoch |

<div align="center"><img src="../assets/divider.svg" alt="" width="100%"/></div>
