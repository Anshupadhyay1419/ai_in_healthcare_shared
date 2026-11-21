# Session 3 — Model Validation, Performance Assessment, and Challenges in Healthcare Data

**AI in Healthcare · Unit IV · B.Tech AI, Third Year**  
**Student lecture notes and primary study material**  
**Previous session:** [Session 2 — Model development and training](session-02-model-development-training.md)  
**Worked numbers:** computed with [`session_03_validation_numbers.py`](session_03_validation_numbers.py) from the UCI Heart Disease data

## Learning objectives

After studying this chapter, you should be able to:

1. Distinguish apparent, internal, temporal, external, and prospective validation, and rank the evidence each provides.
2. Assess a model's discrimination, calibration, and performance at a threshold, and explain why each is needed.
3. Explain how prevalence changes PPV and NPV even when the model does not change.
4. Report uncertainty with confidence intervals and judge whether a difference is meaningful.
5. Recognise how missing inputs, implausible values, case mix, and subgroup size appear during validation.

You need Sessions 1 and 2 of this unit, Lab 4 (sensitivity, specificity, PPV, NPV, AUROC), and Unit III Session 4 (dataset shift, shortcut learning, proxy labels). Medical knowledge is not assumed.

## Contents

1. [The problem: one model, four hospitals](#1-the-problem-one-model-four-hospitals)
2. [Kinds of validation](#2-kinds-of-validation)
3. [Discrimination](#3-discrimination)
4. [Calibration](#4-calibration)
5. [Performance at a threshold](#5-performance-at-a-threshold)
6. [Uncertainty](#6-uncertainty)
7. [Challenges in healthcare data, seen through validation](#7-challenges-in-healthcare-data-seen-through-validation)
8. [Subgroup performance](#8-subgroup-performance)
9. [Error analysis and reporting](#9-error-analysis-and-reporting)
10. [Revision summary](#10-revision-summary)
11. [Review questions](#11-review-questions)
12. [Further reading](#12-further-reading)

## 1. The problem: one model, four hospitals

In Session 1 you saw that the UCI Heart Disease collection contains patients from four sites. Suppose a team develops a heart-disease model on the Cleveland patients and three other hospitals want to use it:

| Site | Patients | With disease |
|---|---|---|
| Cleveland Clinic Foundation, USA (development) | 303 | 46% |
| Hungarian Institute of Cardiology, Budapest | 294 | 36% |
| University Hospital, Zurich, Switzerland | 123 | 94% |
| V.A. Medical Center, Long Beach, USA | 200 | 75% |

The question for this session is: **does the model work at the other hospitals, and how would we know?**

### 1.1 The first finding: the Lab 4 model cannot be run

The Lab 4 model used all 13 inputs, including `ca` (major vessels seen on fluoroscopy) and `thal` (thallium stress-test result). At the other sites these are mostly missing:

| Site | `ca` missing | `thal` missing |
|---|---|---|
| Cleveland | 4 of 303 | 2 of 303 |
| Budapest | 291 of 294 | 266 of 294 |
| Zurich | 118 of 123 | 52 of 123 |
| Long Beach | 198 of 200 | 166 of 200 |

Filling in 99% of a column with Cleveland's most common value would not be validating the model; it would be validating a guess. **The first result of an external validation can be that the model cannot be applied as built.**

### 1.2 The model used in this session

For this session we use a logistic regression trained on Cleveland with eight inputs that all four sites recorded: age, sex, chest-pain type, resting blood pressure, maximum heart rate, exercise-induced angina, ST depression on exercise (`oldpeak`), and resting ECG result. Cholesterol is left out because Zurich recorded it as 0 for every patient (Session 1). After one-hot encoding the model has 11 inputs.

All numbers below were computed by the companion script. Patients missing any of the eight inputs were excluded at each external site; Section 7 discusses what that does.

## 2. Kinds of validation

### 2.1 From weakest to strongest evidence

| Kind | Evaluated on | What it shows | Example |
|---|---|---|---|
| **Apparent** | The training data itself | Almost nothing: the model has seen these patients | Cleveland model scored on Cleveland |
| **Internal** | Held-out data from the same source (split, cross-validation, bootstrap) | Performance on *similar* new patients | 10-fold cross-validation in Cleveland |
| **Temporal** | Later patients from the same source | Whether performance survives changes over time | Train on 2019–2021, test on 2022 |
| **External** | Patients from other sites | Whether performance transfers to other populations and practices | Cleveland model tested in Budapest |
| **Prospective** | New patients as they arrive, in real use | Performance and effects in the actual workflow | A trial in the clinic |

**Cross-validation** splits the data into *k* parts (folds), trains on *k* − 1 of them, scores the remaining part, and repeats so that every patient is scored once by a model that did not see them.

### 2.2 Apparent performance is optimistic

| Cleveland | AUROC (95% CI) |
|---|---|
| Apparent | 0.90 (0.86–0.93) |
| Internal, 10-fold cross-validation | 0.87 (0.84–0.91) |

*CI = confidence interval: the range in brackets shows how precise each number is. [Section 6.1](#61-the-same-ratio-different-evidence) explains it.*

The gap is **optimism**: the model fitted some noise in its training data. You saw the same pattern in Lab 12, where the validation set — cut randomly from the training pool — gave better results than the separately assembled test set.

Internal validation answers "how well does this work on more patients like these?" It cannot answer "how well does it work somewhere else?" For that we need external data.

## 3. Discrimination

**Discrimination** is the model's ability to give higher risks to patients with the outcome than to those without. It is usually measured by **AUROC** (area under the receiver operating characteristic curve).

AUROC has a direct meaning: pick one patient with disease and one without at random; AUROC is the probability that the model gives the patient with disease the higher risk. An AUROC of 0.5 is a coin toss; 1.0 is perfect ranking.

| Site | Patients evaluated | AUROC (95% CI) |
|---|---|---|
| Cleveland (internal) | 303 | 0.87 (0.84–0.91) |
| Budapest | 292 | 0.88 (0.83–0.92) |
| Zurich | 116 | 0.75 (0.59–0.90) |
| Long Beach | 141 | 0.72 (0.61–0.81) |

**Interpretation.** In Budapest the model ranks patients about as well as in Cleveland. In Zurich and Long Beach it ranks them noticeably worse, and the confidence intervals are wide.

AUROC ignores the actual risk values: it would not change if every predicted risk were halved. A model can rank well and still give the wrong risks. That is what calibration measures.

## 4. Calibration

### 4.1 What calibration means

A model is **well calibrated** if its predicted risks match what actually happens. Among patients given a 30% risk, about 30% should have the disease.

Calibration matters whenever the risk number itself is used: telling a patient their risk, comparing it with a guideline threshold, or planning how many referrals to expect.

### 4.2 Two simple measures

- **Calibration-in-the-large** compares the **mean predicted risk** with the **observed outcome rate**. If the model predicts 41% on average but 36% have disease, it **overpredicts**.
- The **calibration slope** describes the spread of the predictions. It is estimated by regressing the outcome on the model's log-odds:
  - slope ≈ 1: the spread is right;
  - slope < 1: predictions are **too extreme** — high risks too high, low risks too low;
  - slope > 1: predictions are too timid.

A **reliability diagram** shows the same information as a plot: patients are grouped by predicted risk, and each group's observed rate is plotted against its mean prediction. A well-calibrated model lies on the diagonal.

```text
 observed
 rate
 1.0 |                    .  *
     |               .  *
     |          .   *          * = slope < 1: the line is flatter than the
     |      .    *                  diagonal, so high predictions are too high
     |  .     *                     and low predictions too low
     |.    *
 0.0 +-------------------------> predicted risk
     0.0                    1.0
```

### 4.3 Calibration at the four sites

| Site | Mean predicted risk | Observed rate | Calibration slope |
|---|---|---|---|
| Cleveland (internal) | 46% | 46% | 0.88 |
| Budapest | 41% | 36% | 0.89 |
| Zurich | 68% | 93% | 0.54 |
| Long Beach | 80% | 79% | 0.48 |

**Interpretation.**

- **Budapest:** the model slightly overpredicts on average, because fewer Budapest patients have disease than Cleveland patients.
- **Zurich:** the model badly underpredicts. It expects about two in three patients to have disease; in fact almost all do.
- **Long Beach:** the average is right, but the slope of 0.48 means individual predictions are far too spread out. Calibration-in-the-large alone would have hidden this.

### 4.4 Recalibration

When a model ranks well but is miscalibrated at a new site, the site can **recalibrate** it: refit the intercept (to fix the average) or the intercept and slope, using local data. This is cheaper than building a new model, but the recalibrated model must itself be validated, and it does not fix poor discrimination.

**Worked example: recalibrating at Long Beach.** At Long Beach the average was right (80% predicted, 79% observed) but the slope was 0.48, so the individual predictions are too extreme. The 141 Long Beach patients with complete inputs are used to refit.

1. **Convert each predicted risk to log-odds:** log-odds = ln(p ÷ (1 − p)).
2. **Train a second, very small logistic regression** with the log-odds as its only input and the real outcome as its target. Its two fitted numbers are **a = 0.53** and **b = 0.48**.
3. **Replace each prediction** with the new equation, then convert back to a probability: p_new = 1 ÷ (1 + e^−(a + b × log-odds)).

**Where the formula in step 3 comes from.** The second regression is a straight line on the log-odds scale. Log-odds can take any value, positive or negative, so a straight line suits it. A probability cannot leave the range 0 to 1, so a straight line fitted directly to the probability would fail at the ends. Write z for the right-hand side of the line, z = a + b × (model log-odds), so that

ln( p ÷ (1 − p) ) = z

Solving this for p gives the formula:

1. Undo the logarithm by raising e to both sides: p ÷ (1 − p) = e^z.
2. Multiply both sides by (1 − p): p = e^z × (1 − p) = e^z − e^z × p.
3. Collect the p terms on one side: p + e^z × p = e^z, so p × (1 + e^z) = e^z.
4. Divide by (1 + e^z): p = e^z ÷ (1 + e^z).
5. Divide the top and bottom by e^z: **p = 1 ÷ (1 + e^−z)**.

This is the **logistic** (sigmoid) function. Check with the 90% patient in the table below: log-odds = ln(9) = 2.20, so z = 0.53 + 0.48 × 2.20 = 1.59, and p = 1 ÷ (1 + e^−1.59) = 1 ÷ 1.204 = 0.83. Going backwards, ln(0.83 ÷ 0.17) ≈ 1.59, which is z again. The function always returns a value between 0 and 1, and z = 0 gives p = 0.5. The first model converts its own weighted sum into a risk with the same function; recalibration changes only the value of z that goes in.

**How step 2 is done in code.** Two models are involved, and it helps to keep them apart:

| | First model | Second model |
|---|---|---|
| Purpose | Predict heart disease from patient data | Correct the first model's risks |
| Trained on | Cleveland patients | The 141 Long Beach patients |
| Inputs | 11 columns (age, chest-pain type, and so on) | 1 column: the first model's log-odds |
| Target | Disease: 0 or 1 | Disease: 0 or 1 |
| Output | Predicted risk | Recalibrated risk |

The first model is not retrained. Only the second model is fitted:

```python
fit = LogisticRegression(penalty=None).fit(logit_lb, y_lb)
a, b = fit.intercept_[0], fit.coef_[0, 0]
```

Read it from left to right:

- **`logit_lb`** holds the first model's log-odds for the 141 patients. It is arranged as 141 rows and 1 column, because scikit-learn expects one row per patient and one column per input.
- **`y_lb`** holds the real outcomes for the same 141 patients: 1 if the patient has disease, 0 if not.
- **`.fit(logit_lb, y_lb)`** trains the model. It searches for the intercept and slope that make the real outcomes most likely, given the log-odds. The search is repeated until the numbers stop changing, and you don't do it by hand.
- **`fit.intercept_[0]`** is the intercept **a**, and **`fit.coef_[0, 0]`** is the slope **b**. 

The fit returns **a = 0.53** and **b = 0.48**. The recalibrated risks are then `fit.predict_proba(logit_lb)[:, 1]`, which applies the formula in step 3 to every patient.

The values of a and b come from all 141 patients. Two patients could not determine them, because each patient contributes only a yes or no outcome, and the fit needs many outcomes to average out the noise. Once a and b are known, the equation can be applied to any patient. For two patients the model scored at 90% and at 10%:

| | Model risk p | Log-odds | a + b × log-odds | Recalibrated risk |
|---|---|---|---|---|
| Patient 1 | 0.90 | ln(9) = 2.20 | 0.53 + 0.48 × 2.20 = 1.59 | 1 ÷ (1 + e^−1.59) = **0.83** |
| Patient 2 | 0.10 | ln(1/9) = −2.20 | 0.53 + 0.48 × (−2.20) = −0.52 | 1 ÷ (1 + e^0.52) = **0.37** |

The slope pulls the extremes toward the middle: 90% becomes 83%, and 10% becomes 37%. The intercept keeps the average from falling. Checking the result in four groups of patients, ordered from lowest to highest predicted risk:

| Predicted-risk group (patients) | Mean predicted | Mean recalibrated | Observed rate |
|---|---|---|---|
| Lowest (36) | 47% | 60% | 61% |
| Second (35) | 81% | 78% | 80% |
| Third (35) | 93% | 86% | 83% |
| Highest (35) | 98% | 91% | 91% |

After recalibration, each group's prediction is close to what was observed. Before, the lowest group was predicted at 47% when 61% had disease, and the highest at 98% when 91% did.

**What did not change.** The patients are in exactly the same order, so the AUROC is still 0.72. Recalibration changed the numbers, not how well they separate patients with disease from those without.

**A limit of this example.** The intercept and slope were fitted and checked on the *same* 141 patients, so the agreement in the table is optimistic. A real recalibration would fit `a` and `b` on one part of the local data and check them on another (or use cross-validation), and would then be monitored over time as in Session 4.

## 5. Performance at a threshold

### 5.1 Threshold metrics at the four sites

A referral decision needs a threshold. At a threshold of 0.5:

| Site | Sensitivity | Specificity | PPV | NPV |
|---|---|---|---|---|
| Cleveland (internal) | 104/139 = 0.75 | 140/164 = 0.85 | 104/128 = 0.81 | 140/175 = 0.80 |
| Budapest | 81/105 = 0.77 | 156/187 = 0.83 | 81/112 = 0.72 | 156/180 = 0.87 |
| Zurich | 83/108 = 0.77 | 4/8 = 0.50 | 83/87 = 0.95 | 4/29 = 0.14 |
| Long Beach | 104/111 = 0.94 | 8/30 = 0.27 | 104/126 = 0.83 | 8/15 = 0.53 |

Showing the counts, not only the ratios, lets a reader see how much evidence lies behind each number. Zurich's specificity is based on **eight** patients.

### 5.2 PPV depends on prevalence

Sensitivity and specificity in Cleveland and Budapest are similar, yet PPV falls from 0.81 to 0.72. The model has not changed; the **prevalence** has.

A worked calculation shows how large the effect can be. Use Cleveland's internal sensitivity (0.75) and specificity (0.85), and imagine 1,000 patients:

| | Prevalence 46% (Cleveland) | Prevalence 10% (a checkup clinic, illustrative) |
|---|---|---|
| Patients with disease | 460 | 100 |
| True positives (0.75 × with disease) | 345 | 75 |
| Patients without disease | 540 | 900 |
| False positives (0.15 × without disease) | 81 | 135 |
| **PPV** = TP ÷ (TP + FP) | 345 ÷ 426 = **0.81** | 75 ÷ 210 = **0.36** |

In the lower-prevalence setting, almost two of every three referrals would be unnecessary. This is why Session 1 insisted on matching the training population to the intended setting.

## 6. Uncertainty

### 6.1 The same ratio, different evidence

**The problem.** Two studies test the same kind of model on patients known to have the disease, and count how many it detects:

| Study | Patients with disease | Detected | Sensitivity |
|---|---|---|---|
| A | 20 | 18 | 18 ÷ 20 = 90% |
| B | 200 | 180 | 180 ÷ 200 = 90% |

Both report 90%. Is the evidence equally strong?

**Step 1: a measured sensitivity is only an estimate.** The quantity we care about is the model's *true* sensitivity: the proportion it would detect among all future patients with the disease. We cannot test all of them, so we test a sample and use the sample's proportion as an estimate.

A sample can be lucky or unlucky. Suppose the true sensitivity were only 80%. In a study of 20 patients, the model would still detect 18 or more about **1 time in 5**, just by chance. In a study of 200 patients, detecting 180 or more would happen about **1 time in 10,000**. Small samples bounce around; large samples settle close to the true value. (Tossing a coin is the same: 9 heads in 10 tosses hardly proves the coin is biased; 900 heads in 1,000 tosses does.)

**Step 2: a confidence interval states how precise the estimate is.**

> A **95% confidence interval** is a range calculated from the sample by a method that, if the study were repeated many times, would contain the true value in about 95% of the repetitions. In practice, read it as *the range of true values that are reasonably consistent with the data*.

Write a result as the count, the estimate, and the interval: **18/20 = 0.90 (95% CI 0.70–0.97)**. The single number 0.90 is the *point estimate*; the interval shows how far the truth could plausibly be from it.

**A common misreading.** The 95% describes the *method* over many repeated studies, not one interval. It does not mean "there is a 95% chance that the true value lies in this particular interval": the true value is a fixed number, so for one interval it is either inside or it is not. A narrower interval means a more precise estimate.

**Step 3: apply it to the two studies.**

| Study | Result | 95% confidence interval | Width | What we can say |
|---|---|---|---|---|
| A | 18 of 20 | 0.70 to 0.97 | 27 points | The true sensitivity could be as low as 70% (missing 3 in 10 patients) or as high as 97% |
| B | 180 of 200 | 0.85 to 0.93 | 8 points | The true sensitivity is close to 90% |

*(Wilson intervals, a standard method for proportions that behaves well with small samples or proportions near 0 or 1. You need to interpret intervals, not compute them by hand.)*

Study A is consistent with a sensitivity as low as 70%. The ratio alone hides this. With ten times as many patients, Study B's interval is about a third as wide: **more patients give a narrower interval, which means stronger evidence**.

**Step 4: a very small denominator tells us almost nothing.** In the site results above, Zurich had only 8 patients without disease, and the model correctly cleared 4: a specificity of 4/8 = 0.50 with an interval of 0.22 to 0.78. The true specificity could be poor or reasonably good; these data cannot tell which.

**Limitation.** The interval assumes the patients are independent and represent the intended population. It covers only the uncertainty from chance in a small sample. It does not correct for a biased sample, a poor reference standard, or leakage, so a narrow interval from a flawed study is still flawed evidence.

### 6.2 Comparing models

The same caution applies to comparisons. Cleveland's internal AUROC interval (0.84–0.91) and Budapest's (0.83–0.92) overlap almost entirely, so we cannot claim the model is better or worse in Budapest. A claim like "model A beats model B by 0.01 AUROC" needs a large test set and a proper paired comparison before it means anything.

**Rule:** report every performance number with its count or confidence interval.

## 7. Challenges in healthcare data, seen through validation

Unit III Session 4 explained *why* models fail in new settings: dataset shift, shortcut learning, proxy labels. Validation is where these problems *become visible*. In our four-site example:

| Challenge | How it appeared here |
|---|---|
| **Missing inputs** | `ca` and `thal` are almost absent outside Cleveland, so the full Lab 4 model cannot be run (Section 1.1) |
| **Implausible or placeholder values** | Zurich records cholesterol as 0 for everyone; a model using cholesterol would receive nonsense |
| **Missingness that selects patients** | Long Beach: 59 of 200 patients lacked a core input and were excluded. Prevalence among those kept was 79% versus 75% overall, so the evaluated group is not the full population |
| **Different case mix** | Prevalence ranges from 36% to 94%: each site referred different kinds of patients for angiography |
| **Small external samples** | Zurich has 8 patients without disease; its specificity cannot be estimated usefully |
| **Changes over time** | Not testable here: the data have no dates. A temporal validation needs them |

The same pattern occurs at a much larger scale. Unit III Session 1 described the independent evaluation of a widely used commercial sepsis model at Michigan Medicine, which found an AUROC of 0.63 and missed 67% of sepsis cases (Wong et al., 2021).

**Lesson:** an external validation report should describe *what data were available and who was excluded* before it reports any metric.

## 8. Subgroup performance

### 8.1 Why look at subgroups?

A model can perform well overall and poorly for a group of patients — for example women, older patients, or a particular ethnic group. Checking subgroups is part of assessing whether a model is fair and safe for everyone it will be used on.

### 8.2 Men and women in Cleveland

Internal (cross-validated) results by sex, at a threshold of 0.5:

| Group | Patients | With disease | AUROC (95% CI) | Sensitivity (95% CI) | Specificity (95% CI) |
|---|---|---|---|---|---|
| Men | 206 | 114 (55%) | 0.84 (0.79–0.89) | 86/114 = 0.75 (0.67–0.82) | 72/92 = 0.78 (0.69–0.85) |
| Women | 97 | 25 (26%) | 0.89 (0.81–0.96) | 18/25 = 0.72 (0.52–0.86) | 68/72 = 0.94 (0.87–0.98) |

**Interpretation.**

- Sensitivity looks similar, but the women's interval (0.52–0.86) is wide because only **25** women had disease. A real difference of 10–20 percentage points could be hidden.
- Specificity is higher for women. A likely reason is that women here had a lower disease rate, and sex is one of the model's inputs, so women received lower risks overall.
- The external sites cannot answer this question: Budapest, Zurich, and Long Beach include only 81, 10, and 6 women, of whom 12, 10, and 3 had disease.

### 8.3 Which notion of fairness?

Different fairness criteria compare different metrics:

| Criterion | Asks for equal | Suits a decision where |
|---|---|---|
| Equal sensitivity | Detection rate among those with disease | Missing disease is the main harm |
| Equal specificity | Correct clearance among those without disease | Unnecessary referral is the main harm |
| Equal PPV | Proportion of flagged patients who truly have disease | Trust in each alert matters |

When disease prevalence differs between groups, a model generally cannot make all of these equal at once. The team must choose the criterion that matches the clinical decision, state it, and report the others.

**Rule:** report subgroup results with their counts and intervals, and do not conclude "no difference" from a small subgroup.

## 9. Error analysis and reporting

### 9.1 Look at the errors themselves

Summary metrics say how many errors the model makes, not which ones. Error analysis looks at the individual false negatives and false positives:

1. List the misclassified patients with their inputs and predicted risks.
2. Look for patterns: a chest-pain type, an age range, a site, values near the edge of what the training data contained.
3. Ask whether a pattern suggests missing information, a labelling problem, or a population the model was not trained on.
4. Decide whether the pattern changes how the model may be used — for example, excluding a patient group from its intended use.

Lab 13 practises this.

### 9.2 What a validation report contains

A validation should be reported in enough detail for a reader to judge it. TRIPOD+AI (Collins et al., 2024) is the reporting checklist for prediction-model studies. At minimum, report:

- the kind of validation (internal, temporal, external) and the data source, dates, and setting;
- the population, how patients were selected, and who was excluded and why;
- the outcome definition and its prevalence;
- discrimination, calibration, and threshold metrics, each with uncertainty;
- subgroup results with their sizes;
- what was changed (for example recalibration) and what was not tested.

## 10. Revision summary

| Concept | Key idea |
|---|---|
| Kinds of validation | Apparent < internal < temporal < external < prospective |
| Optimism | Apparent performance overstates performance on new patients |
| Discrimination | AUROC: does the model rank patients with disease higher? |
| Calibration | Do predicted risks match observed rates? Check the average and the slope |
| Threshold metrics | Sensitivity, specificity, PPV, NPV, with counts |
| Prevalence | Changes PPV and NPV even when the model is unchanged |
| Uncertainty | Report confidence intervals; small denominators mean wide intervals |
| Data challenges | Missing inputs, placeholders, exclusions, and case mix appear during validation |
| Subgroups | Report with sizes; choose and state a fairness criterion |

## 11. Review questions

1. Why is apparent performance not evidence that a model works? What does internal validation add, and what can it still not show?
2. In one or two sentences, explain what an AUROC of 0.88 means.
3. A model's mean predicted risk equals the observed rate, but its calibration slope is 0.5. What is wrong with its predictions?
4. Using a sensitivity of 0.75 and a specificity of 0.85, compute the PPV at a prevalence of 20% in 1,000 patients.
5. Long Beach excluded 59 of 200 patients for missing inputs. Why could this make the external results misleading?
6. A subgroup of 30 patients contains 6 with disease, and the model detects 5. Why should you not conclude that sensitivity in this group is 83%?
7. For a referral model where missing disease is the main harm, which fairness criterion would you prioritise, and why?
8. List four items a reader needs in an external-validation report before trusting its AUROC.

## 12. Further reading

- Collins GS et al. [Evaluation of clinical prediction models (part 1): from development to external validation](https://doi.org/10.1136/bmj-2023-074819). *BMJ* 2024;384:e074819.
- Steyerberg EW, Vergouwe Y. [Towards better clinical prediction models: seven steps for development and an ABCD for validation](https://doi.org/10.1093/eurheartj/ehu207). *European Heart Journal* 2014;35:1925–1931.
- Van Calster B et al. [Calibration: the Achilles heel of predictive analytics](https://doi.org/10.1186/s12916-019-1466-7). *BMC Medicine* 2019;17:230.
- Collins GS et al. [TRIPOD+AI statement](https://doi.org/10.1136/bmj-2023-078378). *BMJ* 2024;385:e078378. — reporting checklist.
- Wong A et al. [External validation of a widely implemented proprietary sepsis prediction model in hospitalized patients](https://doi.org/10.1001/jamainternmed.2021.2626). *JAMA Internal Medicine* 2021;181(8):1065.
- Obermeyer Z et al. [Dissecting racial bias in an algorithm used to manage the health of populations](https://doi.org/10.1126/science.aax2342). *Science* 2019;366:447–453. — a subgroup failure caused by a proxy label.
- *Optional:* Vickers AJ, Elkin EB. [Decision curve analysis: a novel method for evaluating prediction models](https://doi.org/10.1177/0272989X06295361). *Medical Decision Making* 2006;26:565–574. — weighs the benefit of correct referrals against the harm of unnecessary ones across thresholds.
- UCI Machine Learning Repository. [Heart Disease dataset](https://archive.ics.uci.edu/dataset/45/heart+disease) (doi:10.24432/C52P4X), CC BY 4.0.
