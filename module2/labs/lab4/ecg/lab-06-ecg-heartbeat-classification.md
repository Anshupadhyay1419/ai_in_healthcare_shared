# Lab 6: Building ML models for ECG heartbeat classification

**AI in Healthcare · Unit II: Clinical applications of AI**

**Duration:** about 120 minutes

## Problem statement

An **electrocardiogram (ECG)** records the heart's electrical activity over time. A long recording, such as a 24-hour ambulatory (Holter) ECG, contains tens of thousands of beats. Clinicians look for unusual beats among them, which is slow. Researchers therefore study whether a model can label each beat so that unusual ones can be reviewed first.

In this lab, you will build and evaluate models that take **one already extracted heartbeat** and predict its beat group: **N, S, V, F or Q**.

| Item | Definition for this lab |
|---|---|
| Clinical motivation | Help organise review of recorded beats |
| Computational task | Five-class classification of one beat |
| Model input | Columns `0` to `186`: 187 ordered signal values from one beat |
| Target | Column `187`: 0 = N, 1 = S, 2 = V, 3 = F, 4 = Q |
| Unit of analysis | One beat row, **not** one recording or one person |
| Intended user | A person reviewing ECG recordings, who makes the final decision |

This is an educational exercise. Your model does not find beats in a continuous recording, diagnose a patient or recommend treatment, and it must not be described as ready for clinical use.

## Learning objectives

By the end of the lab, you should be able to:

1. Explain what one row means in this dataset, and why it is not a patient.
2. Show why accuracy is misleading when one class makes up most of the data.
3. Train a simple baseline, at least one classical model and at least one neural network on the same split.
4. Evaluate a multi-class model with a confusion matrix, per-class sensitivity and precision, and macro F1.
5. Explain why a beat-level split cannot show how a model performs on new patients.

## Prerequisites

- Python, pandas, NumPy, scikit-learn and matplotlib. PyTorch or Keras for the neural network.
- Train/validation/test splits, logistic regression, tree ensembles and confusion matrices.
- The ECG lecture notebooks: [concepts](../lectures/ecg_hb_classification/ecg_hb_clf.ipynb), [EDA](../lectures/ecg_hb_classification/ecg_hb_clf_eda.ipynb) and [models](../lectures/ecg_hb_classification/ecg_hb_clf_models.ipynb). They explain the ECG vocabulary, how beats were extracted and what each label means.

No medical training is assumed.

## Dataset

**Source:** [ECG Heartbeat Categorization Dataset (Kaggle)](https://www.kaggle.com/datasets/shayanfazeli/heartbeat). It contains beats extracted from the [MIT-BIH Arrhythmia Database (PhysioNet)](https://physionet.org/content/mitdb/1.0.0/) and preprocessed by [Kachuee et al. (2018)](https://arxiv.org/abs/1805.00794). Check the licence and terms on the Kaggle page before redistributing any file.

**Files used:** `mitbih_train.csv` (87,554 beats, about 393 MiB) and `mitbih_test.csv` (21,892 beats, about 98 MiB). The PTB files in the same download belong to a different task; do not use them. Loading both files needs roughly 1–2 GB of RAM. A CPU is enough for the classical models; a small neural network trains in minutes on a CPU and faster on a GPU.

**Access:** download from Kaggle (a free account is required), or use the course copy in `module2/lectures/data/ecg_hb_clf/`. Do not commit the CSV files to your submission.

| Column | Meaning |
|---|---|
| `0` … `186` | One beat, sampled 125 times per second and scaled to the range 0–1. Shorter beats are padded with zeros at the end |
| `187` | Beat group: 0 = N (normal or non-ectopic), 1 = S (supraventricular ectopic), 2 = V (ventricular ectopic), 3 = F (fusion), 4 = Q (paced, or unclassifiable) |

**Important limit:** the original database has recordings from 47 people, but the CSV files contain **no patient or recording identifiers**. You cannot check whether beats from one person appear in both the training and test files.

## Tasks

### Part A: Load and check the data (15 min)

1. Load both files. Report their shapes, any missing values, and the number and percentage of beats in each class.
2. Plot at least three example beats from each class on the same axes.
3. For each class, plot the mean beat with a band of ±1 standard deviation.

**Questions**

- What does one row represent? Why is 109,446 beats not the same as 109,446 patients?
- What accuracy would a model get if it always predicted N? What would its sensitivity be for F?
- Which classes look easiest and hardest to tell apart by eye?

### Part B: Split and baseline (20 min)

1. Keep `mitbih_test.csv` aside for **one** final evaluation in Part E.
2. Split `mitbih_train.csv` into a fit set (about 80%) and a validation set (about 20%), **stratified** by class so that each set keeps the same class proportions. Use a fixed random seed.
3. Fit a **majority-class baseline** (scikit-learn's `DummyClassifier`) and a **multinomial logistic regression**. Train the logistic regression once without and once with `class_weight="balanced"`.

**Questions**

- Use the validation set for all comparisons and choices from now on. Why must the test set not be used to choose a model?
- What did `class_weight="balanced"` change in the confusion matrix? What did it cost?

### Part C: Stronger models (40 min)

Train these on the **same fit set**:

1. **At least one classical model** on the raw 187 values, for example random forest, gradient boosting or k-nearest neighbours.
2. **Engineered features:** compute a small set of features per beat, for example the maximum and its position, minimum, standard deviation, energy, an estimate of the non-padded length (the position after which all values are zero), and the area under the curve. Train one classical model on these features and compare it with the raw-signal version.
3. **At least one neural network** on the raw signal. A small **1D convolutional neural network (CNN)** is recommended: two or three convolution and pooling layers, then a dense output layer with five classes. Use the validation set for early stopping.

Record the training time of each model.

**Questions**

- Which worked better, raw values or engineered features? Give a possible reason.
- What can a 1D CNN learn from the beat that a model on summary features cannot?
- The zero padding at the end of short beats carries information about beat length. Is the model learning heart physiology, or a property of how the data were prepared? How could you check?

### Part D: Evaluate on the validation set (25 min)

For every model, report on the validation set:

- the 5 × 5 **confusion matrix**, also normalised by row (true class)
- **sensitivity** (recall) and **precision** (PPV) for each class
- **macro F1**, the unweighted mean of the five per-class F1 scores, alongside accuracy

Collect the results in one comparison table. Then choose your best model and justify the choice.

**Questions**

- Why is macro F1 a better headline metric than accuracy here? What does it still hide?
- Which pair of classes is confused most often? Look at a few misclassified beats and suggest why.
- Missing a V beat and wrongly flagging an N beat as V have different costs. Which would you rather reduce, and how would you shift your model towards that?

### Part E: Final test and error analysis (10 min)

1. Evaluate your **one selected model** on `mitbih_test.csv` once. Report the same metrics as in Part D.
2. Compare the test results with the validation results. Plot a few beats of the class with the lowest test sensitivity.

**Questions**

- Did test performance match validation performance? Why might it, even though neither shows performance on new patients?

### Part F: Reflection (10 min)

Write a short reflection (150–250 words) covering:

- **Why beat-level results overstate real performance.** Beats from the same person are very similar, and these files may place beats from one person in both training and test sets. [de Chazal et al. (2004)](https://doi.org/10.1109/TBME.2004.827359) instead split MIT-BIH **by record**, so that training and test beats come from different people. What would you expect to happen to your scores under such a split?
- What else would be needed before the tool could be used in a clinic: beat detection on raw recordings, other devices and leads, more and different patients, external validation.
- How the tool should fit the workflow: who reviews its labels, and what happens when it is wrong.

## Deliverables

Submit one Jupyter notebook that runs from top to bottom in a fresh kernel. It should:

1. Load data from a relative path and state where the files came from.
2. Use a fixed random seed for the split and for every model.
3. Include code, outputs and plots for Parts A–E.
4. Include one table comparing all models on the validation set, and the selected model on the test set.
5. Answer every question in Markdown cells next to the relevant code.
6. Include the reflection from Part F.

## Grading rubric (100 points)

| Criterion | Points | What earns full credit |
|---|---:|---|
| Data understanding | 15 | Correct checks and class counts; clear distinction between beats and patients |
| Split and baseline | 15 | Stratified validation split; test set kept aside and used once; baseline correctly interpreted |
| Modelling | 25 | Classical model on raw values and on engineered features; at least one working neural network; same split for all |
| Evaluation | 20 | Confusion matrices, per-class sensitivity and precision, macro F1 in one comparison table; justified model choice |
| Error analysis and interpretation | 15 | Misclassified beats examined; answers tied to evidence; padding question addressed |
| Reflection and limitations | 5 | Beat-level versus patient-level evaluation explained; no clinical-readiness claims |
| Reproducibility and communication | 5 | Notebook runs cleanly; readable code, tables and plots |

## Optional extensions

- Train an LSTM or a CNN–Transformer and compare it with your CNN on the same split.
- Try oversampling the rare classes in the fit set only, and compare with class weighting.
- Download the original MIT-BIH recordings from PhysioNet, extract beats yourself, and split **by record** following de Chazal et al. (2004). Compare your scores with the beat-level split.
- Track your experiments (configuration and validation metrics) with a tool such as Weights & Biases.

## References

- [Kaggle: ECG Heartbeat Categorization Dataset](https://www.kaggle.com/datasets/shayanfazeli/heartbeat)
- [PhysioNet: MIT-BIH Arrhythmia Database](https://physionet.org/content/mitdb/1.0.0/)
- Kachuee M, Fazeli S, Sarrafzadeh M. ECG heartbeat classification: a deep transferable representation. arXiv:1805.00794; 2018. [Link](https://arxiv.org/abs/1805.00794)
- de Chazal P, O'Dwyer M, Reilly RB. Automatic classification of heartbeats using ECG morphology and heartbeat interval features. *IEEE Transactions on Biomedical Engineering*. 2004;51(7):1196–1206. [doi:10.1109/TBME.2004.827359](https://doi.org/10.1109/TBME.2004.827359)
- [scikit-learn: model evaluation](https://scikit-learn.org/stable/modules/model_evaluation.html)
