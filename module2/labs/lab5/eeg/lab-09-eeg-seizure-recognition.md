# Lab 9: Building ML models for EEG seizure recognition

**AI in Healthcare · Unit II: Clinical applications of AI**

**Duration:** about 120 minutes


## Problem statement

An **electroencephalogram (EEG)** records the brain's electrical activity as a voltage signal over time. Neurologists review long EEG recordings to find periods of **seizure activity**. That review is slow, so researchers study whether a model could flag short EEG segments that a clinician should look at first.

In this lab, you will build and evaluate machine-learning models that take **one short EEG segment** and predict whether it contains seizure activity.

| Item | Definition for this lab |
|---|---|
| Clinical motivation | Help prioritise EEG segments for clinician review |
| Computational task | Binary classification of one segment |
| Model input | `X1` to `X178`: 178 ordered signal values from one segment |
| Target | `1` = seizure activity (original `y = 1`); `0` = non-seizure (original `y = 2, 3, 4, 5`) |
| Unit of analysis | One segment row, **not** one patient |
| Intended user | A clinician reviewing EEG, who makes the final decision |

This is an educational exercise. Your model is not a diagnostic tool and must not be described as ready for clinical use.

## Learning objectives

By the end of the lab, you should be able to:

1. Explain what one row, one recording, and one person mean in this dataset.
2. Construct a binary target from a multi-class label and justify the choice.
3. Split the data so segments from the same recording do not leak between training and test sets.
4. Train and compare a baseline model with at least two stronger models.
5. Evaluate models with sensitivity, specificity, positive predictive value (PPV), negative predictive value (NPV), ROC-AUC, and calibration, not only accuracy.
6. Choose a decision threshold and explain its clinical trade-off.
7. Describe the limitations that stop these results from proving clinical usefulness.

## Prerequisites

- Python, pandas, NumPy, scikit-learn, and matplotlib.
- Train/validation/test splits, logistic regression, tree ensembles, and confusion matrices.
- The companion EEG lecture notebook, for EEG vocabulary and the meaning of each label.

No medical training is assumed.

## Dataset

**Source:** [Epileptic Seizure Recognition (Kaggle)](https://www.kaggle.com/datasets/harunshimanto/epileptic-seizure-recognition). This is a reshaped CSV version of the Bonn EEG dataset described by [Andrzejak et al. (2001)](https://doi.org/10.1103/PhysRevE.64.061907). Check the licence and terms shown on the Kaggle page before redistributing the file.

**Size:** one CSV file of about 7 MB, with 11,500 rows and 180 columns. A CPU laptop is enough for this lab.

**Access:** download it from Kaggle (a free account is required), or use the course copy at `module2/lectures/data/Epileptic Seizure Recognition.csv`. Do not commit the CSV to your submission.

| Column | Meaning |
|---|---|
| `Unnamed` | Segment identifier, for example `X21.V1.791`. Do **not** use it as a model feature. |
| `X1` … `X178` | 178 ordered signal values from one segment, about 1 second of EEG |
| `y` | Original label: 1 = seizure activity; 2 = recorded from the tumour area; 3 = recorded from a healthy brain area in a patient with a tumour; 4 = healthy volunteer, eyes closed; 5 = healthy volunteer, eyes open |

### How the rows were created

The source data contains **500 single-channel EEG recordings**, 100 for each label. Each recording lasted about 23.6 seconds. The CSV cuts each recording into **23 chunks** of 178 values:

`500 recordings × 23 chunks = 11,500 rows`

In the identifier `X21.V1.791`, the part before the first dot (`X21`) is the chunk number. The rest (`V1.791`) is shared by all 23 chunks of one recording. In the course copy of the file, this gives exactly 500 groups of 23 rows, and every group has a single label. Treat this as a **recording ID** for splitting.

The original study recorded these 500 segments from a small number of people (healthy volunteers and patients). The CSV contains no patient identifier, so even a recording-level split cannot guarantee that the same person is absent from both training and test data.

## Tasks

### Part A: Load, check, and define the target (20 min)

1. Load the CSV. Report its shape, the number of missing values, and the count of each original label.
2. Extract the recording ID from `Unnamed`. Confirm that there are 500 recordings, 23 rows per recording, and one label per recording.
3. Create the binary target `seizure`. Report the number and percentage of positive rows.
4. Plot at least two example segments from each original label.

**Questions**

- What is one row, one recording, and one person in this dataset?
- What accuracy would a model get if it always predicted "non-seizure"? Why does this make accuracy a weak headline metric here?
- The dataset has 20% seizure rows. Why does that not tell us how common seizures are in real EEG review?

### Part B: Split without leakage (15 min)

1. Split **by recording ID** into training (about 60%), validation (about 20%), and test (about 20%) sets. Keep the seizure proportion similar across the sets. `GroupShuffleSplit` or `StratifiedGroupKFold` from scikit-learn can help.
2. Check that no recording ID appears in more than one set. Report rows, recordings, and seizure percentage for each set.
3. Fit any scaler on the training set only.

**Questions**

- Explain how a random row split could leak information. Why might it make test performance look better than it should?
- Use the validation set to compare models and choose thresholds. Do not use the test set until Part E.

### Part C: Baseline and comparison models (30 min)

Train these models on the raw 178 values:

1. **Baseline:** logistic regression with standardised inputs.
2. **At least two comparison models**, for example random forest, gradient boosting, support vector machine, or k-nearest neighbours.

Then create a small set of **engineered features** for each segment, such as mean, standard deviation, minimum, maximum, peak-to-peak range, energy, zero-crossing count, and line length (the sum of absolute differences between neighbouring values). Train at least one model on these features and compare it with the raw-signal models.

**Questions**

- Which input worked better, raw values or engineered features? Give a possible reason.
- Which engineered features are easiest to explain to a clinician?

### Part D: Evaluate on the validation set (25 min)

For each model, report on the validation set:

- Confusion matrix (true positives, false positives, true negatives, false negatives).
- Sensitivity (recall for seizure), specificity, PPV (precision), and NPV.
- ROC-AUC and precision–recall AUC.
- A calibration plot or Brier score for at least the baseline and your best model.

Then **choose a decision threshold** for your best model. Do not just use 0.5. State your rule, for example "the highest threshold that gives at least 95% sensitivity", and report the resulting false positives.

**Questions**

- In this workflow, is a missed seizure segment or a false alarm more costly? How did that shape your threshold?
- If the tool flagged 100 segments for review, about how many would be true seizure segments at your threshold? Which metric answers this?
- Does a model with a higher ROC-AUC always give better results at your chosen threshold?

### Part E: Final test and error analysis (20 min)

1. Evaluate your **one selected model and threshold** on the test set once. Report the same metrics as in Part D. If feasible, add a bootstrap 95% confidence interval for sensitivity and specificity, resampling recordings rather than rows.
2. Look at false negatives and false positives. Count them by **original label** (`y = 2, 3, 4, 5`) and plot a few examples.
3. Aggregate to the recording level: flag a recording if any one (or at least *k*) of its segments is predicted positive. How do sensitivity and specificity change?

**Questions**

- Which non-seizure label produced the most false positives? Suggest why.
- Do mistakes cluster in particular recordings? What does that suggest?

### Part F: Reflection (10 min)

Write a short reflection (150–250 words) covering:

- Why strong results on this dataset do not show clinical usefulness. Consider the small number of people, single-channel and pre-selected segments, balanced classes, and missing clinical context.
- What external validation would be needed before anyone considered this tool in a hospital.
- How the tool should fit the workflow: who reviews its output, and how alert fatigue could be avoided.

## Deliverables

Submit one Jupyter notebook that runs from top to bottom in a fresh kernel. It should:

1. Load data from a relative path and state where the file came from.
2. Use a fixed random seed for all splits and models.
3. Include code, outputs, and plots for Parts A–E.
4. Include a final results table that compares all models on the validation set and the selected model on the test set.
5. Answer every question in Markdown cells next to the relevant code.
6. Include the reflection from Part F.

## Grading rubric (100 points)

| Criterion | Points | What earns full credit |
|---|---:|---|
| Data understanding and target definition | 15 | Correct checks; clear distinction between rows, recordings, and people; target justified |
| Leakage-safe splitting | 15 | Recording-level split verified; preprocessing fitted on training data only; test set used once |
| Modelling | 20 | Baseline plus at least two comparison models; engineered features compared with raw signal |
| Evaluation and threshold choice | 20 | All required metrics; calibration checked; threshold rule stated and justified |
| Error analysis | 15 | Errors examined by original label and recording; plausible explanations |
| Interpretation and limitations | 10 | Thoughtful answers and reflection; no claims of clinical readiness |
| Reproducibility and communication | 5 | Notebook runs cleanly; readable code, tables, and plots |

## Optional extensions

- Train a small 1D convolutional neural network (CNN) on the raw signal and compare it with your best classical model.
- Add frequency-band power features (for example, using Welch's method). The reported sampling rate of the source recordings is about 173.61 Hz; check this against the original study before using it.
- Use leave-recordings-out cross-validation on the training set instead of a single validation split.

## References

- [Kaggle: Epileptic Seizure Recognition](https://www.kaggle.com/datasets/harunshimanto/epileptic-seizure-recognition)
- [UCI metadata for Epileptic Seizure Recognition](https://github.com/uci-ml-repo/ucimlrepo-data/blob/master/datasets/388.json)
- Andrzejak RG, et al. (2001). Indications of nonlinear deterministic and finite-dimensional structures in time series of brain electrical activity. *Physical Review E*, 64, 061907. [doi:10.1103/PhysRevE.64.061907](https://doi.org/10.1103/PhysRevE.64.061907)
- [scikit-learn: cross-validation iterators for grouped data](https://scikit-learn.org/stable/modules/cross_validation.html#group-k-fold)
