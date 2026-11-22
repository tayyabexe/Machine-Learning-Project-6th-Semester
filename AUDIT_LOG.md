# Audit Log: Decision Tree on TON_IoT (train_test_network.csv)
Owner: Tayyab (Decision Trees + ANN). Notebook: scripts/script1.ipynb
Rule: after every step, add the decision, the reason, and the exact figures. Never overwrite old entries; add a new one that references the old one.

## Fixed project decisions
| # | Decision | Reason | Source |
|---|----------|--------|--------|
| D1 | Two Decision Tree models: `label` (binary) and `type` (10-class) | Results.xlsx reports both | Results.xlsx |
| D2 | ~~80/20 split, no cross-validation~~ SUPERSEDED by D10 | Team changed plan | You |
| D3 | Same split and preprocessing for all models | Fair comparison rule | Assignment03 PDF |
| D4 | Tune only on validation folds taken from the training data, never on test | Fair comparison rule | Assignment03 PDF |
| D5 | Tuning is manual (loops written by hand), no GridSearch | Your instruction | You |
| D6 | Drop `src_ip`, `dst_ip`, `src_port`, `dst_port` | Identifiers cause overfitting | Proposal, You |
| D7 | Latency = predict() time on test set / n test rows, in ms per sample | Matches the sheet's per-sample style; same for ANN | Agreed in chat |
| D8 | Metrics: Accuracy, Precision, Recall, F1, ROC-AUC, Log-Loss, FPR, MCC, Latency, TP/TN/FP/FN; macro-average for `type` | Results.xlsx | Results.xlsx |
| D10 | **Split: 20% test (stratified), then stratified 5-fold on the other 80%. Per fold: 64% train, 16% validation. Whole team agreed.** | Team decision; replaces D2 | Teammates |
| D11 | Decision Tree hyperparameters fixed from papers: gini, best, max_depth=5, min_samples_split=5, min_samples_leaf=1, random_state=42 (see HYPERPARAMETER_SOURCES.md). Replaces manual tuning (D5) for the main result; the 5 folds now measure stability, not tuning | Team-supplied paper values | You |
| D12 | Stratify the test split on `type` in both notebooks so label and type models use identical rows. Encoders fit on the training set only | Fair comparison, no leakage | Me, agreed by building |
| D9 | Writing into Results.xlsx: decide later | Your instruction | You |

## Open decisions (recommended option first)
| # | Question | Recommendation | Status |
|---|----------|----------------|--------|
| O1 | Sparse columns | Drop columns >95% empty, keep and encode the rest | Pending your check of Q5 |
| O2 | Duplicates | Now unclear, see Step 1 figures: removing them deletes 55% of rows and shrinks ransomware to ~260. Follow the teammates' choice | Pending (switch `DEDUPLICATE` in the notebooks) |
| O3 | max_depth=5 underfits `type` | Report as is, optionally add a labelled deeper-tree variant | Pending your decision |

## Dataset facts (measured on the raw file, before any cleaning)
- Shape: **211,043 rows x 44 columns** (42 features + `label` + `type`)
- `label`: 1 = 161,043 (76.3%), 0 = 50,000 (23.7%)
- `type`: normal 50,000; backdoor, ddos, dos, injection, password, ransomware, scanning, xss 20,000 each; **mitm only 1,043** (rare class; expect it to be the hardest)
- Duplicate rows: **20,569** (9.7%)
- Columns >99% empty (`-`): all `http_*`, `ssl_*`, `weird_*`
- Partly empty: `dns_query` and `dns_AA/RD/RA/rejected` 83.5%, `service` 62.6%
- Categorical cardinality: `proto` 3, `service` 9, `conn_state` 13
- dtypes: 27 text, 16 int, 1 float

## Reference results already in Results.xlsx (Dataset1, label / type)
| Model | Accuracy | F1 | MCC | Latency |
|-------|----------|----|-----|---------|
| KNN (k=5) | 0.9969 / 0.9700 | 0.9980 / 0.9319 | 0.9913 / 0.9657 | 0.3305 / 0.3280 |
| Random Forest | 0.9992 / 0.9871 | 0.9995 / 0.9651 | 0.9976 / 0.9853 | 0.0023 / 0.0076 |
| XGBoost | 0.9993 / 0.9883 | 0.9995 / 0.9617 | 0.9979 / 0.9866 | 0.0008 / 0.0048 |

## Change log
- Change 1: team moved from plain 80/20 (no CV) to 20% test + stratified 5-fold on the 80% (64/16/20 per fold). D2 superseded by D10. Tuning (D5) stays manual but now averages over 5 folds.
- Change 2 (2026-11-20): Decision Tree notebooks stripped of code; they now hold only questions + 'what to do' with empty code cells. The user types the code manually; reference code is given in chat.

## Step log
(Add entries below in this format.)

### Step 1: Complete Decision Tree notebooks built and test-run (2026-11-20)
- What we did: wrote `scripts/Decision Trees/TON_IoT/dt_label.ipynb` and `dt_type.ipynb` (cleaning, split, encoding, 5-fold validation, final model, all metrics, plots). Executed end to end to confirm they run; numbers below come from that run (not yet run by you in Jupyter).
- Cleaning figures: 16 columns >95% empty dropped, 4 identifier columns dropped, 24 columns left. Duplicates now: 117,159 (55.5%). Rows after deduplication: 93,884 (ransomware 260, mitm 1,039, normal 23,879).
- Split (DEDUPLICATE=False): 168,834 train / 42,209 test; 52 features after encoding.
- Results, DEDUPLICATE=False (paper hyperparameters, test set):
  | Target | Accuracy | Precision | Recall | F1 | ROC-AUC | Log-Loss | FPR | MCC | Latency ms/sample |
  |---|---|---|---|---|---|---|---|---|---|
  | label | 0.9914 | 0.9926 | 0.9962 | 0.9944 | 0.9964 | 0.0354 | 0.0240 | 0.9762 | 0.000286 |
  | type (macro) | 0.7959 | 0.7976 | 0.7221 | 0.7205 | 0.9663 | 0.5305 | 0.0232 | 0.7790 | 0.000637 |
  - label confusion: TP 32,087, TN 9,760, FP 240, FN 122. type macro: TP 3,359.5, TN 37,126.7, FP 861.4, FN 861.4.
  - label 5-fold validation: accuracy 0.9911 +/- 0.0008, F1 0.9942 +/- 0.0005. Train/test accuracy gap -0.0001.
  - type: train acc 0.7939 vs test 0.7959 (underfitting). Per class: ddos recall 0.25, xss precision 0.37, mitm recall 0.00 (209 test rows), ransomware recall 0.61.
- Results, DEDUPLICATE=True: label accuracy 0.9802, F1 0.9869, MCC 0.9476, FPR 0.0657 (test TP 13,944, TN 4,462, FP 314, FN 57). type accuracy 0.8261, macro-F1 0.5491, MCC 0.7945.
- Decision and why: kept `DEDUPLICATE=False` as the default until teammates answer (O2). `max_depth=5` kept because it is the agreed paper value, but it clearly underfits `type` (O3).
- Effect on later steps: ANN must use the same split (stratify on `type`, random_state 42), the same 5 folds, and the same DEDUPLICATE choice. Teammates' sheet rows show 36,719 test rows for the label confusion counts (KNN), which differs from our 42,209 (or 18,777 if deduplicated); ask what they did.


### Step N: <title> (date)
- What we did:
- Figures:
- Decision and why:
- Effect on later steps:

## Step 2: notebooks completed and executed (2026-11-21)
- Both `dt_label.ipynb` and `dt_type.ipynb` now contain full code and saved outputs (DEDUPLICATE=False, paper hyperparameters).
- Fixes to the user's own label code: added `log_loss` to imports; `df.drop_duplicates` -> `df.drop_duplicates().reset_index(drop=True)`; replaced the unfinished encoding cell (`one_cols +`) with the full encoding cell.
- Figures reproduced exactly: label acc 0.9914 / F1 0.9944 / MCC 0.9762 / FPR 0.0240 (TP 32,087, TN 9,760, FP 240, FN 122); type acc 0.7959 / macro-F1 0.7205 / MCC 0.7790 / FPR 0.0232.
- 5-fold validation: label acc 0.9911 +/- 0.0008, F1 0.9942 +/- 0.0005; type acc 0.8076 +/- 0.0318, macro-F1 0.7324 +/- 0.0320.
- Depth 5; leaves: label 27, type 29. Train acc = test acc in both (label 0.9914; type 0.7939 vs 0.7959), so no overfitting; type underfits (ddos recall 0.25, mitm 0).

## Step 3: Edge-IIoTset Decision Tree notebooks (2026-11-21)
Location: `scripts/Decision Trees/Edge-IIoT/dt_label.ipynb` (Attack_label), `dt_type.ipynb` (Attack_type). Built and executed by Claude; same split/folds/hyperparameters/metrics as TON_IoT.

**Dataset facts (raw):** 157,800 rows x 63 columns. Attack_label 1 = 133,499, 0 = 24,301 (84.6/15.4). Attack_type: 15 classes; Normal 24,301; most attacks ~10-14.5k; rare: MITM 1,214, Fingerprinting 1,001. No NaN: an unused field is stored as 0 (text columns as both "0" and "0.0").

**Preprocessing decisions (dataset-specific, consistent with D-rules for TON_IoT):**
- E1. The ">95% empty" rule is NOT applied: zero here means "protocol not used", a real signal (e.g. MQTT fields). Only useless columns dropped.
- E2. Dropped 8 constant columns (icmp.unused, http.tls_port, dns.qry.type, dns.retransmit_request_in, mqtt.msg_decoded_as, mbtcp.len, mbtcp.trans_id, mbtcp.unit_id).
- E3. Dropped identifiers: frame.time, icmp.transmit_timestamp, ip.src_host, ip.dst_host, arp.src.proto_ipv4, arp.dst.proto_ipv4, tcp.srcport, tcp.dstport, udp.port.
- E4. Dropped raw free-text content: http.file_data, http.request.full_uri, http.request.uri.query, tcp.options, tcp.payload, mqtt.msg. (E3+E4 match the Edge-IIoTset authors' recommended drop list.)
- E5. Result 157,800 x 40 (38 features). Duplicates 5,604 (3.6%); DEDUPLICATE=False as TON_IoT.
- E6. One-hot for 7 text columns ("0" unified with "0.0"): http.request.method, http.referer, http.request.version, dns.qry.name.len, mqtt.conack.flags, mqtt.protoname, mqtt.topic -> 59 features. Train 126,240 / test 31,560 (stratified on Attack_type).

**Results (test set, paper hyperparameters):**
| Target | Acc | Prec | Rec | F1 | ROC-AUC | Log-Loss | FPR | MCC | Latency ms/sample |
|---|---|---|---|---|---|---|---|---|---|
| Attack_label | 0.9646 | 0.9598 | 1.0000 | 0.9795 | 0.9577 | 0.1165 | 0.2300 | 0.8597 | ~0.00038 |
| Attack_type (macro) | 0.4529 | 0.5236 | 0.3225 | 0.3185 | 0.7619 | 1.5847 | 0.0431 | 0.4794 | ~0.00047 |
- Label confusion: TP 26,700, TN 3,742, FP 1,118, FN 0. 5-fold: acc 0.9674 +/- 0.0012, F1 0.9811 +/- 0.0007. Depth 5, 10 leaves, train 0.374 s.
- Type: macro TP 953.0, TN 28,305.0, FP 1,151.0, FN 1,151.0. 5-fold: acc 0.4533 +/- 0.0009, macro-F1 0.3184 +/- 0.0011. Depth 5, 12 leaves, train 0.609 s. Train acc 0.4534 = test 0.4529: severe underfitting. Recall 0 for Backdoor, Fingerprinting, MITM, Ransomware, SQL_injection, Uploading; everything unknown falls into Normal (Normal precision 0.22).
- Label FPR 0.23 is high: 23% of normal traffic flagged as attack.

**Depth check (O3 evidence; validation fold 1 of the 80% only, test set NOT used, not in the notebooks):** max_depth 5 -> acc 0.4537 / macro-F1 0.3192; 10 -> 0.7135 / 0.5986; 15 -> 0.8572 / 0.8281; 20 -> 0.8760 / 0.8480; None -> 0.8567 / 0.8177. Strengthens O3: max_depth=5 is not suitable for multiclass targets.

## Step 4: X-IIoTID Decision Tree notebooks (2026-11-21)
Location: `scripts/Decision Trees/X-IIoTID/dt_class3.ipynb` (binary), `dt_class2.ipynb` (10 classes), `dt_class1.ipynb` (19 classes). Built and executed by Claude. Markdown cells hold questions only (user request: no hints); Edge-IIoT notebooks' markdown was cleaned the same way.

**Dataset facts:** 820,834 rows x 68 columns. class3: Normal 421,417 / Attack 399,417. class2: 10 classes (crypto-ransomware 458 rarest). class1: 19 classes (Fake_notification 28, MitM 117 rarest). Missing marked as "-" (and "?" in Protocol); after conversion: byte/rate columns 27.27% missing, packet columns 18.61%, Duration 9.23%, system-stat columns 0.06%. Broken cells ("#DIV/0!", "excel", "aza") coerced to NaN.

**Preprocessing:** X1 replace "-"/"?" with NaN, numeric columns via to_numeric(errors="coerce"). X2 drop constant Bad_checksum, is_SYN_with_RST. X3 drop identifiers Date, Timestamp, Scr_IP, Scr_port, Des_IP, Des_port -> 820,834 x 60 (57 features). X4 duplicates 119,685 (14.6%), DEDUPLICATE=False. X5 split 80/20 stratified on class1 for all three notebooks (656,667 / 164,167). X6 text NaN -> "missing", one-hot Protocol, Service, anomaly_alert; numeric NaN -> -1 (no negatives in the data) -> 78 features.

**Results (test set, paper hyperparameters max_depth=5):**
| Target | Acc | Prec | Rec | F1 | ROC-AUC | Log-Loss | FPR | MCC | 5-fold acc / F1 |
|---|---|---|---|---|---|---|---|---|---|
| class3 | 0.9465 | 0.9931 | 0.8964 | 0.9423 | 0.9505 | 0.1830 | 0.0059 | 0.8968 | 0.9470+/-0.0008 / 0.9428+/-0.0009 |
| class2 (macro) | 0.8884 | 0.7200 | 0.5845 | 0.6334 | 0.9414 | 0.3741 | 0.0211 | 0.8335 | 0.8887+/-0.0009 / 0.5890+/-0.0314 |
| class1 (macro) | 0.8916 | 0.3219 | 0.3378 | 0.3292 | 0.9048 | 0.3923 | 0.0101 | 0.8407 | 0.8923+/-0.0003 / 0.3296+/-0.0003 |
- class3 confusion: TP 71,607, TN 83,785, FP 498, FN 8,277. Leaves 32, train ~10 s, latency ~0.00033 ms/sample.
- class2 macro TP 14,583.9 / TN 145,917.5 / FP 1,832.8 / FN 1,832.8; Tampering recall 0. 23 leaves.
- class1 macro TP 7,703.6 / TN 154,589.8 / FP 936.8 / FN 936.8; 12 of 19 classes have recall 0. 23 leaves.
- Train acc = test acc for all three: underfitting again at max_depth=5 on multiclass (O3).

## Step 5: dataset-specific hyperparameters from papers, all 7 DT models retrained (2026-11-21)
Supersedes D-hyperparameters (max_depth=5, min_samples_split=5). New values for all 7 notebooks: gini, best, max_depth=None, min_samples_split=2, min_samples_leaf=1, random_state=42. Sources in HYPERPARAMETER_SOURCES.md: TON_IoT [S1] Dharini et al. 2026 Sci Rep (verified); X-IIoTID [S4] Al-Hawawreh et al. 2022 IEEE IoTJ, "default parameters" (verified); Edge-IIoT [S3] CMC 2026 two-stage IDS (NOT verified, full text blocked) + [S2] Ferrag 2022 (values not published).

| Dataset / target | Acc | Prec | Rec | F1 | ROC-AUC | Log-Loss | FPR | MCC | Latency ms | Train/Test acc | Depth/Leaves | Old acc (depth 5) |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| TON_IoT label | 0.9981 | 0.9986 | 0.9989 | 0.9988 | 0.9972 | 0.0668 | 0.0044 | 0.9948 | 0.00034 | 0.9998/0.9981 | 31/431 | 0.9914 |
| TON_IoT type (macro) | 0.9766 | 0.9479 | 0.9516 | 0.9495 | 0.9800 | 0.5103 | 0.0026 | 0.9732 | 0.00110 | 0.9886/0.9766 | 56/2464 | 0.7959 |
| Edge-IIoT Attack_label | 0.9623 | 0.9764 | 0.9791 | 0.9777 | 0.9245 | 1.3254 | 0.1298 | 0.8544 | 0.00106 | 0.9984/0.9623 | 74/3978 | 0.9646 |
| Edge-IIoT Attack_type (macro) | 0.8573 | 0.8161 | 0.8425 | 0.8201 | 0.9282 | 4.5661 | 0.0103 | 0.8447 | 0.00072 | 0.9796/0.8573 | 94/13933 | 0.4529 |
| X-IIoTID class3 | 0.9958 | 0.9954 | 0.9959 | 0.9957 | 0.9958 | 0.1526 | 0.0044 | 0.9915 | 0.00109 | 1.0/0.9958 | 38/2130 | 0.9465 |
| X-IIoTID class2 (macro) | 0.9953 | 0.9791 | 0.9821 | 0.9806 | 0.9907 | 0.1680 | 0.0007 | 0.9931 | 0.00091 | 1.0/0.9953 | 43/2429 | 0.8884 |
| X-IIoTID class1 (macro) | 0.9953 | 0.9612 | 0.9656 | 0.9633 | 0.9826 | 0.1699 | 0.0004 | 0.9932 | 0.00095 | 1.0/0.9953 | 43/2600 | 0.8916 |

Confusion: TON label TP 32,173 TN 9,956 FP 44 FN 36; TON type macro TP 4,122.2 TN 37,889.4 FP/FN 98.7; Edge label TP 26,141 TN 4,229 FP 631 FN 559; Edge type macro TP 1,803.7 TN 29,155.7 FP/FN 300.3; X class3 TP 79,557 TN 83,915 FP 368 FN 327; X class2 macro TP 16,340.2 TN 147,673.8 FP/FN 76.5; X class1 macro TP 8,599.6 TN 155,485.8 FP/FN 40.8.
5-fold val acc: TON label 0.9977+/-0.0003, TON type 0.9751+/-0.0009, Edge label 0.9624+/-0.0013, Edge type 0.8564+/-0.0022, X3 0.9955+/-0.0001, X2 0.9953+/-0.0002, X1 0.9950+/-0.0001.
Notes: Edge-IIoT label got slightly worse (0.9646 -> 0.9623) and overfits (train 0.9984); Edge Log-Loss is high because full trees give 0/1 probabilities.

## Step 6: Edge-IIoT source verified (2026-11-21)
Tayyab downloaded the paper PDF (`A_Lightweight_Two-Stage_Intrusion_Detection_Framew.pdf`, project root). Confirmed: Kuo & Wu (2026), CMC 87(3):53, doi:10.32604/cmc.2026.076767; Stage-1 DT = scikit-learn defaults (gini, max_depth None, min_samples_split 2, splitter best), 80/20 split. Caveat: their DT was trained on CICIoT2023 (DDoS vs non-DDoS); Edge-IIoTset was only used to test the whole framework (94.73%). Hyperparameter values unchanged, so no retraining; only the comments in the Edge-IIoT notebooks and the source files were updated.

## Step 7: Decision Tree values written into the team results sheet (local copy) + graphs (2026-11-21)
- Downloaded the team's `Results.xlsx` from Drive (owner: teammate; file id 1cNzcOBdrcVs-JKfjXrk5jMlKpcqRNCk3). Layout now has Train Time and Test Time columns. Saved filled copy as `Results.xlsx` (project root) and `result.csv`. The Drive file itself was NOT changed.
- Filled Decision Trees rows: TON_IoT rows 7/18, Edge-IIoT rows 32/43, X-IIoTID rows 57/68. All metric columns filled from the Step 5 notebook outputs (6-decimal display precision).
- Test Time = inference latency x test rows / 1000 (s): the same predict() timing the notebooks measure, derived rather than printed separately.
- Graphs (4 per model: class distribution, confusion matrix (+ROC for binary), feature importance, top 3 tree levels) saved to `Graphs/Decision Trees/<Dataset>/<model>/` and uploaded to Drive `ML Project Datasets/<Dataset>/<model>/`.

## Step 8: tree plots fixed + CLAUDE.md rewritten for the ANN session (2026-11-21)
- Q18 tree cell in all 7 DT notebooks: impurity=False, figsize (24, 9), the "samples"/"value" lines removed from the boxes (they overlapped). All 7 notebooks re-executed; accuracies unchanged (0.998105, 0.976616, 0.962294, 0.857288, 0.995767, 0.995340, 0.995279).
- New `04_tree_top_levels.png` saved locally and uploaded to Drive; the old Drive images were renamed "old_04_tree_top_levels (overlapping).png" (not deleted, awaiting the user's OK).
- CLAUDE.md rewritten: common pipeline, dataset-specific cleaning, DT results, ANN plan, Results-sheet layout, Drive folder IDs, notebook style.
- User decision (2026-11-21): the ANN values already in the team sheet (TON_IoT label row, X-IIoTID row) are to be overwritten with our own ANN results once the ANN notebooks are done (local copy).

## Step 9: ANN (MLPClassifier) notebooks, TON_IoT + Edge-IIoT executed, X-IIoTID code ready (2026-11-21/22)
- Library: scikit-learn 1.8 `MLPClassifier` (no TensorFlow/PyTorch installed). Same cleaning, split, encoding, 5 folds, metrics and latency method as the DT notebooks; added `StandardScaler` fitted on the training data only (refitted inside each fold). Plots: class distribution, confusion matrix (+ROC for binary), training loss curve. DEDUPLICATE=False.
- Hyperparameters from papers (details and quotes in HYPERPARAMETER_SOURCES.md, ANN section):
  - TON_IoT [A1] Othman et al. 2026 Future Internet: (128,64,32), relu, adam, lr 0.001, batch 256, 50 epochs, n_iter_no_change=10. Paper is binary-only; same settings used for `type` (Tayyab's decision).
  - Edge-IIoT [A2] Ferrag et al. 2022 Table 9: (90,90), relu, adam, batch 800, 25 epochs, L2 (alpha 1e-4 default; lr 0.001 default).
  - X-IIoTID [A3] Al-Hawawreh et al. 2022: (200,200,200), relu, batch 250, 10 epochs; RMSprop -> Adam (not in sklearn); alpha 0.
- Results (test set):

| Dataset / target | Acc | Prec | Rec | F1 | ROC-AUC | Log-Loss | FPR | MCC | Latency ms | Train s | Test s | Train/Test acc | 5-fold acc | DT acc |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| TON_IoT label | 0.992277 | 0.993285 | 0.996616 | 0.994948 | 0.998204 | 0.033308 | 0.021700 | 0.978584 | 0.002211 | 49.96 | 0.0933 | 0.9927/0.9923 | 0.9919+/-0.0007 | 0.998105 |
| TON_IoT type (macro) | 0.823237 | 0.778800 | 0.755834 | 0.750091 | 0.991213 | 0.363979 | 0.019598 | 0.804636 | 0.002483 | 57.53 | 0.1048 | 0.8239/0.8232 | 0.8793+/-0.0497 | 0.976616 |
| Edge Attack_label | 0.909791 | 0.910057 | 0.991348 | 0.948965 | 0.919831 | 0.216262 | 0.538272 | 0.608308 | 0.001792 | 11.46 | 0.0565 | 0.9125/0.9098 | 0.9120+/-0.0011 | 0.962294 |
| Edge Attack_type (macro) | 0.664480 | 0.705320 | 0.613702 | 0.620449 | 0.947619 | 0.879621 | 0.024845 | 0.636588 | 0.001723 | 12.09 | 0.0544 | 0.6619/0.6645 | 0.6603+/-0.0056 | 0.857288 |

- Confusion: TON label TP 32,100 TN 9,783 FP 217 FN 109; TON type macro TP 3,474.8 TN 37,242.0 FP/FN 746.1; Edge label TP 26,469 TN 2,244 FP 2,616 FN 231; Edge type macro TP 1,398.07 TN 28,750.07 FP/FN 705.93. Test rows 42,209 (TON) and 31,560 (Edge), features 52 / 59, same as DT.
- Findings: all multiclass ANNs underfit (train acc = test acc, loss still falling at the last epoch; all TON fits ran the full 50 epochs, early stop never triggered). TON type: backdoor recall 0.02 (98% predicted as dos), mitm recall 0.36; two of five folds much worse (0.8429, 0.8100), so results depend on initialisation. Edge label FPR 0.54 (most normal traffic flagged as attack); Edge type weak on XSS/Password/Uploading/SQL_injection. Edge results far below the paper (99.99% / 94.67%), which used the larger DNN-EdgeIIoT file + SMOTE. Nothing tuned on the test set.
- X-IIoTID: `ann_class3/2/1.ipynb` written with full code, NOT executed here (full run ~40-45 min on this laptop; Tayyab will run them on another laptop). Smoke test with 1 epoch (temporary copies, not saved): class3 and class1 run end to end without errors.
- Results.xlsx (local copy) ANN rows filled: TON_IoT rows 6/17 (row 6 overwrote the teammate's earlier ANN label values, per the user's decision), Edge rows 31/42. X-IIoTID rows 56/67 still hold the teammate's values until our run. `result.csv` did not exist; recreated from Results.xlsx.
- Graphs: `Graphs/ANN/<Dataset>/ann_<target>/` (01_class_distribution, 02_confusion_matrix[_and_roc], 03_loss_curve) for TON_IoT and Edge-IIoT; uploaded to Drive in new `ann_label` / `ann_type` subfolders of the TON_IoT and Edge-IIoT folders.

## Step 10: X-IIoTID ANN executed on the second laptop + merged into the main copy (2026-11-22)
- Source of this step: the second laptop's project copy (`ML Project Datasets (1).zip`, Downloads), whose AUDIT_LOG called it "Step 9" (renumbered here because Step 9 already exists in this copy). Only X-IIoTID files were taken from the zip: the zip's TON_IoT `ann_type.ipynb` was an old unexecuted version (22 KB) and was NOT copied.
- Environment (second laptop): Python 3.12.3, scikit-learn 1.8.0, pandas 3.1.1, i5-11400H; nbclient, in place. Notebook code verified identical to this copy's `ann_class3/2/1.ipynb` (only outputs added). No error outputs.
- Hyperparameters [A3]: (200,200,200), relu, adam (paper: RMSprop), lr 0.001, batch 250, max_iter 10, alpha 0, random_state 42. 78 features, 656,667 / 164,167 rows. 96,401 weights (class3).
- Run times (whole notebook): class3 640 s, class2 711 s, class1 717 s.

| Model | Acc | Prec | Rec | F1 | ROC-AUC | Log-Loss | FPR | MCC | Latency ms | Train s | Test s | Train/Test acc | 5-fold acc | DT acc |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| class3 | 0.989864 | 0.995013 | 0.984102 | 0.989527 | 0.999426 | 0.027427 | 0.004675 | 0.979763 | 0.002566 | 116.31 | 0.4212 | 0.9910/0.9899 | 0.9893+/-0.0005 | 0.995767 |
| class2 (macro) | 0.988743 | 0.976341 | 0.949069 | 0.961758 | 0.999606 | 0.031598 | 0.001862 | 0.983274 | 0.003204 | 131.05 | 0.5260 | 0.9900/0.9887 | 0.9885+/-0.0001 | 0.995340 |
| class1 (macro) | 0.988579 | 0.973459 | 0.916642 | 0.941070 | 0.999601 | 0.033512 | 0.001116 | 0.983507 | 0.002846 | 141.03 | 0.4672 | 0.9897/0.9886 | 0.9882+/-0.0011 | 0.995279 |

- Confusion: class3 TP 78,614 TN 83,889 FP 394 FN 1,270; class2 macro TP 16,231.9 TN 147,565.5 FP/FN 184.8; class1 macro TP 8,541.68 TN 155,427.95 FP/FN 98.68. Macro-F1 5-fold: class2 0.9599+/-0.0028, class1 0.9248+/-0.0100.
- Weak classes: class2 C&C F1 0.83, Exploitation recall 0.83; class1 MitM F1 0.73 (23 test rows), fuzzing recall 0.65, TCP Relay recall 0.70, Discovering_resources recall 0.75.
- Versus DT: ANN accuracy and F1 lower on all three, but better Log-Loss (0.027-0.034 vs 0.15-0.17) and ROC-AUC (0.9994-0.9996 vs 0.983-0.996). No overfitting or strong underfitting (train ~ test, both ~0.99). ANN training 116-141 s on the second laptop vs DT 30-46 s on this PC (different machines, so the times are only roughly comparable).
- Merge into this copy: copied the 3 executed notebooks, `Graphs/ANN/X-IIoTID/` (9 PNGs) and `Results_backup_before_xiiot_ann.xlsx` (teammate's earlier X-IIoTID ANN values). Rows 56/67 of `Results.xlsx` written from the notebook outputs; they match the second laptop's sheet exactly, and every other row matches too except our TON/Edge ANN rows. `result.csv` regenerated. All 6 ANN rows are now filled.
- Drive: the 9 graphs were already uploaded from the second laptop (X-IIoTID folder, subfolders `ann_class3` 1EFIvFsYuZ4IieQ9QDvSHFYY8GkjXhyXM, `ann_class2` 1SM_50CXSChtP19tsllHUaJBggPSaM0kJ, `ann_class1` 1w1T8rezlxoJUQF1h5OyBDp2IwKkF4Fbo); verified 3 files each, nothing re-uploaded.
- Note: the project folder also holds a nested `ML Project Datasets/` folder and `d--Machine-Learning-...zip` (created 2026-11-22 01:23-01:25, an older snapshot with unexecuted X-IIoTID notebooks). Not used; left in place pending the user's OK to delete.
- Follow-up (2026-11-22): renamed the X-IIoTID class3 graph `02_confusion_matrix_roc.png` -> `02_confusion_matrix_and_roc.png` locally and on Drive (file 1zfB3hBT-fuzEE5Gw7L8_ANrg-Oikdg9E), to match the other binary folders.
- Full verification (2026-11-22): 14 notebooks (7 DT + 7 ANN) present, executed, 0 error outputs, results table present; ANN graph files byte-identical to the notebook outputs; 14 local graph folders complete (DT 4 files, ANN 3 files, standard names); 14 Drive folders complete (DT 4 files + the kept `old_04_tree_top_levels (overlapping).png`, ANN 3 files); Results.xlsx ANN + DT rows fully filled for all 3 datasets; `result.csv` identical to Results.xlsx.

## Step 11: leftovers removed + GitHub repository published (2026-11-22)
- Deleted (user's OK): nested `ML Project Datasets/` snapshot folder and `d--Machine-Learning-...zip` in the project root, and `Downloads/ML Project Datasets (1).zip` (all X-IIoTID content had already been merged in Step 10). `Downloads/ML Project Datasets (1)/` and `Downloads/ML Project Datasets.zip` kept, per the user.
- User decision: no need to strip the student ID from the notebooks or the Drive IDs from this log before publishing.
- Added `README.md` (project overview, datasets with official links, pipeline diagram, hyperparameter sources, DT vs ANN results for all 7 targets, graph gallery, project structure, how to run, limitations, references), animated SVG banners in `assets/` (header, stats, divider, footer: grey→blue→amber colour cycling, falling autumn leaves), `requirements.txt` (scikit-learn 1.8.0, pandas 3.0, numpy 2.4, matplotlib 3.11, seaborn 0.13.2, openpyxl 3.1.5), `.gitignore`, `.gitattributes`.
- Not published (in `.gitignore`): the 3 dataset CSVs (X-IIoTID is 339 MB, over GitHub's 100 MB limit), zips, `CLAUDE.md` + `.claude/` (private working notes), `Documentations/` and the downloaded paper PDF (course and third-party material), `scripts/script1.ipynb` (scratch).
- Pushed to https://github.com/tayyabexe/Machine-Learning-Project-6th-Semester (public, was empty), branch `main`.
- Follow-up (2026-11-22): README redesigned with more animated SVG components in `assets/`: `ticker.svg` (live prediction ticker), `datasets.svg` (dataset cards with animated attack/normal bars), `pipeline.svg` (6 steps with moving data packets), `ann_architectures.svg` (layer diagrams with flowing signals), `results_chart.svg` (DT vs ANN accuracy bars, generated from the results; colours #3b82f6 / #d97706 passed the dataviz palette validator on the dark surface). Tables upgraded with coloured score badges, winner markers, gap column, icons, plus MCC / Log-Loss and timing tables. Added `scripts/README.md`, `Graphs/README.md` and `CITATION.cff`. All 87 local README links verified.
- 2026-11-22: created `Hyperparameter_Reference.pdf` (5 pages, A4): all 14 models with dataset, target, algorithm, every hyperparameter value, its source paper and whether each value is from the paper, a default, adapted, or our choice; full references. Built from HTML via Edge print-to-PDF. Not pushed to GitHub (user's request).

## Step 12: number audit, stale figures fixed, group document filled (2026-11-22)
- Cause of error: Step 7 filled the sheet, Step 8 re-executed the 7 DT notebooks (accuracies compared, timings not), and the sheet was never updated. The "Full verification" in Step 10 only checked that cells were filled and that the CSV equalled the XLSX; it did not compare values with the notebooks. That entry is superseded by this one.
- Audit of 210 DT/ANN cells against saved notebook outputs: 189 matched, 21 stale (DT latency, train time, test time for all 7 models). All 21 corrected in Results.xlsx and result.csv; re-audit 210/210.
- The DT latencies in the Step 5 table are superseded: TON 0.000412/0.000321; Edge 0.000369/0.000450; X 0.000933/0.000427/0.001243 ms. Train: 1.137182/0.577558; 0.664619/0.969402; 38.617534/28.045988/45.740922 s.
- Also corrected locally, not pushed: README timing table and speed ratios (3-100x train, 2-8x predict); timing ranges in ANN_Status_Update.md and DT_Status_Update.md; Decision_Tree_Code_Guide.md still held the max_depth=5 settings and results.
- 49/49 local graph files are identical to their notebook outputs (14 folders: DT 4 each, ANN 3 each).
- Reproduction on this PC: 11/14 models exact. X-IIoTID ANN class3/2/1 did not reproduce (here accuracy 0.990028/0.989121/0.988085; saved 0.989864/0.988743/0.988579); the saved values remain the official ones.
- Group document (Drive 179x93eW1f_Gihll0dRedn1_inCZ2IUbc): new version uploaded 13:40 to the same file on the user's request, built on the 12:33 version. Added 210 values, 26 graphs, 6 hyperparameter cells, references [8]-[10], 1 note. Live download SHA-256 0c67598e0ace99fb1ad13202a9f3d0ee85f3f12a69cd96f56d4770fccc420dd8 = verified file; 929 other table cells unchanged. Empty: ANN X-IIoTID class2 and class1 ROC (never saved). Backups in Documentations/Google_Doc_Update/.
- Open: "no tuning on test / test used exactly once" is inaccurate for the DT (max_depth=5 replaced after test results were seen, Steps 1-5). Appears in Hyperparameter_Reference.pdf, README.md, HYPERPARAMETER_SOURCES.md, DT_Status_Update.md; wording fix awaits the user's OK.

## Step 13: tuning claim corrected, ROC search, push (2026-11-22)
- The Step 12 open item is closed on the user's OK: README.md (+ its two SVGs), HYPERPARAMETER_SOURCES.md, DT_Status_Update.md, Decision_Tree_Code_Guide.md and the local Hyperparameter_Reference.pdf now state that the DT depth was revised after test results were seen. The notebooks' "test set used once" headings are not edited yet (7 DT notebooks are inaccurate on this).
- ROC for ANN X-IIoTID class2/class1: searched the project, the two older copies, git history, the two Drive model folders and a Drive title search (first 50 matches). No ROC of the reported models was found. A teammate's ANN curves exist but belong to another model (163,315 test rows and class2 accuracy 0.986358, against 164,167 and 0.988743; 5 of the classes drawn). Predictions from today's reproduction run also differ (macro AUC 0.999716 / 0.999609, against 0.999606 / 0.999601). User decided: no re-run. Both cells stay empty until the user chooses.
- Pushed the Step 12 and 13 corrections on the user's OK. The PDF stays out of the repository (user's earlier request).

## Step 14: labelled reproduction ROC curves + DT notebook wording (2026-11-22)
- Supersedes the Step 13 "both cells stay empty". The user chose the predictions saved by the 12:38-12:47 reproduction run. ANN X-IIoTID class2/class1 ROC cells in the group document now hold those curves, each titled "Reproduction run, not the reported model" with both macro AUCs (reproduction 0.999716 / 0.999609; reported 0.999606 / 0.999601). No training. All table numbers and confusion matrices remain those of the reported models.
- Uploaded 14:00 as one new version of the same Drive file: 5,438,509 bytes, SHA-256 e8957ea2d83bc74221004734caee9c343b6659d785052c55f40b2e789d052f34. Only those 2 table cells differ from the 13:40 version. The local copy was replaced; earlier Drive versions are retained. Page layout was not rendered or visually checked (user asked to skip it).
- 7 DT notebooks, markdown only, on the user's OK (dt_label included): section 9 headings now say the test set was used more than once. TON dt_label/dt_type plan lines and stale depth-5 text (dt_label Q2, dt_type Q15, both Q22) were corrected as well. Code cells and outputs unchanged. Not committed or pushed.
- Hyperparameter_Reference.pdf still says "Every source was read in full"; only the relevant sections were checked. Not changed (awaiting the user's answer).

## Step 15: ROC title removed, report graphs saved to Drive (2026-11-22)
- Supersedes the Step 14 title. On the user's request "Reproduction run, not the reported model" was removed from the 2 ROC titles. The disclosure is kept in each legend: "Re-run macro AUC 0.999716 (table: 0.999606)" and "Re-run macro AUC 0.999609 (table: 0.999601)". Document uploaded 14:09 to the same file: 5,427,151 bytes, SHA-256 87a81fe3a465367fc04bc2362fa05b77510a9b447ce38fcb87c7aed48eb7bd0c, verified by download.
- 26 report graphs saved in Graphs/ and uploaded to the 14 Drive model folders (ANN 04/05_report_*, DT 05/06_report_*; the 2 reproduction curves end in _rerun). The other 2 of the 28 report graphs are byte-identical to existing 02_confusion_matrix.png files (ANN class2/class1). The 49 original graphs were kept unchanged.
- Error during upload: the first 26 uploads were converted to Google Docs (disableConversionToGoogleType was missing). Re-uploaded as PNG; the 26 wrong copies were moved, not deleted, to the My Drive folder 14Swv0sXDEO_6_SP_GlZciqeWRj5x-5ee until the user decides.

## Step 16: outdated tree plots removed from Drive (2026-11-22)
- On the user's request, 5 remaining `old_04_tree_top_levels (overlapping).png` files (Edge-IIoT dt_label/dt_type, X-IIoTID dt_class3/2/1) were moved to Drive Trash. The 2 TON_IoT ones were already in Trash (not trashed by this session). Verified: no active old_04 file remains; each of the 7 DT folders holds 6 PNGs including the corrected 04_tree_top_levels.png. This supersedes the "kept old_04" note in Step 10. No local copy existed.

## Step 17: final check, PDF sentence, push (2026-11-22)
- On the user's OK: Hyperparameter_Reference.pdf now says "The relevant sections of each source were checked" (and the column reads "Checked in full text"). Not published (earlier request). The Steps 14-16 changes are pushed with Graphs/README.md updated for the 26 report versions.
- Re-check: Results 210/210; live group document (SHA-256 87a81fe3...) 210/210 numbers, 28/28 pictures, 929 other cells unchanged; 14 notebooks with 0 error outputs and no code/output changes.
- Open in the group document (not edited): 12 pages against a limit of 8; no SVM and no X-IIoTID KNN; the DT test-set reuse is not disclosed in it; unfinished discussion, limitations and conclusion; sample counts, placeholder authors and several teammates' values need correction.
