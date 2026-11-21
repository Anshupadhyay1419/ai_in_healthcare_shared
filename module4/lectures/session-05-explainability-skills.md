# Session 5 — Interpretability and Explainability in Clinical Practice, and the Skills Healthcare AI Requires

**AI in Healthcare · Unit IV · B.Tech AI, Third Year**  
**Student lecture notes and primary study material**  
**Previous session:** [Session 4 — Deploying AI in clinical care](session-04-deployment-settings-workflows.md)  
**Companion practical:** Lab 14 — Explainability (SHAP and permutation importance; saliency maps on the ECG 1D CNN)  
**Worked numbers:** computed with [`session_05_explanation_numbers.py`](session_05_explanation_numbers.py) from the UCI Heart Disease data

## Learning objectives

After studying this chapter, you should be able to:

1. Distinguish interpretable models from post-hoc explanations, and global from local explanations.
2. Explain a logistic regression prediction by decomposing it into input contributions, and interpret permutation importance.
3. Describe gradient, occlusion, and Grad-CAM saliency maps for signal and image models, and how to read them.
4. Explain why an explanation describes the model, not the patient, and how sanity checks expose misleading maps.
5. Identify the skills different roles need across the healthcare AI lifecycle.

You need Sessions 1–4 of this unit, the Unit II ECG beat-classification lecture (the 1D CNN), and Unit III Sessions 4–5 (shortcut learning; the Jabbour et al. study of explanations). Medical knowledge is not assumed.

## Contents

1. [The problem: "why this risk?"](#1-the-problem-why-this-risk)
2. [What an explanation is for](#2-what-an-explanation-is-for)
3. [Explaining a tabular model](#3-explaining-a-tabular-model)
4. [Model-agnostic explanations](#4-model-agnostic-explanations)
5. [Saliency maps for signals and images](#5-saliency-maps-for-signals-and-images)
6. [The limits of explanations](#6-the-limits-of-explanations)
7. [Education and skill requirements](#7-education-and-skill-requirements)
8. [Unit IV synthesis](#8-unit-iv-synthesis)
9. [Revision summary](#9-revision-summary)
10. [Review questions](#10-review-questions)
11. [Further reading](#11-further-reading)

## 1. The problem: "why this risk?"

The heart-disease model from Sessions 3–4 gives the first patient in the Cleveland data a risk of **0.60**. With a threshold of 0.5, it would suggest referral. The physician asks: *why?*

The patient's record:

| Input | Value |
|---|---|
| Age | 63 |
| Sex | Male |
| Chest-pain type | 1 — typical angina (chest pain with the classic features of reduced blood supply to the heart) |
| Resting blood pressure | 145 mm Hg |
| Maximum heart rate on exercise | 150 beats per minute |
| Angina during exercise | No |
| ST depression on exercise (`oldpeak`) | 2.3 |
| Resting ECG | 2 — signs of an enlarged left ventricle |

The angiography result for this patient was **no disease**. So the model's suggestion is a false positive. An explanation should help us understand how the model reached it — and, as we will see, it cannot tell us whether the model was right.

## 2. What an explanation is for

### 2.1 Different people need different explanations

| Who | Question | Useful kind of explanation |
|---|---|---|
| Developer | Is the model using sensible information, or a shortcut? | Global importance; saliency maps across many cases |
| Clinician | Why this risk for this patient, and should I trust it here? | Local: the inputs that drove this prediction |
| Patient | What is this based on, and what can I change? | Plain-language summary of main factors |
| Auditor or regulator | How does the system work, and was it tested properly? | Model documentation, validation evidence, model card |

The most valuable use of explanations is often the **developer's**: finding problems before deployment, as in the shortcut examples of Unit III.

### 2.2 Two distinctions

- **Interpretable models** are simple enough to read directly: a logistic regression's coefficients, a short decision tree, a points-based score. **Post-hoc explanations** are produced afterwards for a model too complex to read, such as a neural network.
- **Global explanations** describe the model's behaviour over all patients. **Local explanations** describe one prediction.

## 3. Explaining a tabular model

### 3.1 A global view: coefficients

A logistic regression computes a **log-odds** score as a weighted sum of its inputs, then converts it to a risk with the sigmoid function:

$$\text{log-odds} = b_0 + \sum_j w_j z_j, \qquad \text{risk} = \frac{1}{1 + e^{-\text{log-odds}}}$$

Here $z_j$ is input $j$ after standardisation (its value minus the training mean, divided by the training standard deviation), $w_j$ is its coefficient, and $b_0$ is the intercept. Because the inputs are standardised, coefficients are on a comparable scale: the change in log-odds for a one-standard-deviation change in the input.

The coefficients of the Session 3 model, largest first:

| Input | Coefficient |
|---|---|
| Chest pain type 4 (asymptomatic) | +1.06 |
| Sex (male) | +0.79 |
| ST depression on exercise (`oldpeak`) | +0.74 |
| Maximum heart rate (`thalach`) | −0.49 |
| Angina during exercise | +0.38 |
| Resting blood pressure | +0.30 |
| Resting ECG type 2 | +0.25 |
| Chest pain type 2 | +0.24 |
| Age | +0.23 |
| Chest pain type 3 | +0.08 |
| Resting ECG type 1 | +0.05 |

Chest-pain type 1 and resting ECG type 0 are the **reference** categories, so their effect is built into the intercept.

**Interpretation.** Most signs agree with clinical expectations: more ST depression and angina during exercise raise the risk; a higher achievable heart rate lowers it. But the strongest input is chest-pain type **4, "asymptomatic"** — no chest pain at all. That looks wrong until we remember who is in the data (Session 1): every patient was sent for angiography. A patient with *no* chest pain who was still referred probably had another strong reason, such as an abnormal test. The coefficient reflects how patients were selected, not how chest pain relates to disease in the general population. **An explanation can reveal such problems, but a person with clinical and data knowledge has to recognise them.**

### 3.2 A local view: contributions for one patient

Because the model is a sum, each input's **contribution** to one patient's log-odds is simply $w_j z_j$. For our patient:

| Input | Patient's value | Contribution to log-odds |
|---|---|---|
| Not chest-pain type 4 | type 1 | −1.01 |
| ST depression | 2.3 (well above average) | +0.80 |
| Sex | male | +0.54 |
| No angina during exercise | no | −0.26 |
| Resting ECG type 2 | yes | +0.26 |
| Resting blood pressure | 145 | +0.23 |
| Age | 63 | +0.22 |
| Other inputs | — | −0.18 in total |
| Intercept | — | −0.19 |
| **Total log-odds** | | **+0.42** |

The risk is $1 / (1 + e^{-0.42}) = 0.60$. *(Values are rounded to two decimals, so the rows add up to 0.41 rather than the unrounded 0.42.)*

**Interpretation.** Having typical angina rather than "asymptomatic" chest pain pulled the risk *down*, for the selection reason above. Marked ST depression, male sex, and the resting ECG pushed it *up*. The explanation is exact for this model: these numbers are how the risk was computed. It still does not say whether the patient has disease — the angiogram showed none.

## 4. Model-agnostic explanations

### 4.1 Permutation importance

**Permutation importance** works for any model. Shuffle one input's values across patients, breaking its link with the outcome, and measure how much performance drops. A large drop means the model relies on that input.

For the Cleveland model evaluated on the 292 Budapest patients (baseline AUROC 0.875; 200 shuffles per input):

| Input | Mean AUROC drop | Middle 95% of shuffles |
|---|---|---|
| Chest-pain type | 0.065 | 0.036 to 0.092 |
| ST depression | 0.038 | 0.022 to 0.054 |
| Sex | 0.029 | 0.011 to 0.052 |
| Angina during exercise | 0.017 | 0.007 to 0.026 |
| Maximum heart rate | 0.005 | −0.009 to 0.019 |
| Resting blood pressure | −0.002 | −0.009 to 0.005 |
| Resting ECG | −0.002 | −0.007 to 0.002 |
| Age | −0.002 | −0.007 to 0.004 |

**Interpretation.**

- At Budapest, four inputs carry nearly all the model's ranking ability. Shuffling the others makes no reliable difference.
- Maximum heart rate has a large coefficient (−0.49) but little importance here. Two likely reasons: it is **correlated** with exercise angina and ST depression, so other inputs carry much of the same information, and its distribution differs between sites.
- Importance depends on the **data it is measured on**. These are importances at Budapest, not universal facts about heart disease.

### 4.2 SHAP and LIME

Two widely used local methods work with any model:

- **SHAP** (SHapley Additive exPlanations; Lundberg and Lee, 2017) divides a prediction among the inputs using a principle from game theory. For a linear model with independent inputs it gives contributions like those in Section 3.2.
- **LIME** (Local Interpretable Model-agnostic Explanations; Ribeiro et al., 2016) fits a simple model to the complex model's behaviour around one patient.

Both are **approximations** for complex models, and both can be unstable when inputs are correlated. Lab 14 uses SHAP and permutation importance.

## 5. Saliency maps for signals and images

### 5.1 The idea

For a model whose input is a signal or an image, there are too many inputs (187 samples per heartbeat; thousands of pixels) to list contributions. A **saliency map** instead shows, for each sample or pixel, how much it affected the output.

Recall the Unit II ECG model: a 1D CNN that classifies one heartbeat — 187 samples recorded 125 times per second (1.5 seconds, 8 ms per sample) — into five classes, including N (normal) and V (ventricular ectopic beat). A V beat starts in the heart's lower chambers and usually has a wide, abnormally shaped **QRS complex** (the sharp main spike of the beat).

### 5.2 Three methods

| Method | How it works | Strength | Weakness |
|---|---|---|---|
| **Gradient** (Simonyan et al., 2013) | Computes how much the class score changes for a tiny change in each sample: $\partial\,\text{score} / \partial x_t$ | One backward pass; fine detail | Noisy; measures sensitivity, not contribution |
| **Gradient × input** | Multiplies the gradient by the sample value | Ignores flat regions near zero | Inherits gradient noise |
| **Occlusion** (Zeiler and Fergus, 2014) | Replaces a short window (say 10 samples = 80 ms) with a neutral value and measures the drop in the class probability; slides the window along the beat | Easy to explain; model-agnostic | Slower; result depends on window size and replacement value |

**Grad-CAM** (Selvaraju et al., 2017) uses the last convolutional layer's feature maps, weighted by their gradients. It is widely used for images. For our ECG CNN it is coarse: three pooling layers shrink 187 samples to about 23 time steps.

### 5.3 How to read an ECG saliency map

The sketch below is **schematic**, drawn to show what to look for. It is not output from a trained model; Lab 14 produces real maps.

```text
 ECG beat   ____/\____         _                         (zero padding)
 (class V)            \       / \__________ . . . . . . . . . . . . . .
                       \_____/
 saliency          ######                 ?            ##########
                 (QRS region)         (T wave)      (padding: a warning sign)
                   0 ms ......................................... 1,500 ms
```

Questions to ask:

- **Is the saliency on the part of the beat that defines the class?** For V beats, we expect the QRS region.
- **Is it on something meaningless?** High saliency on the **zero padding** at the end of short beats would suggest the model uses beat *length* as a shortcut.
- **Is it consistent across many beats of the same class?** One map proves nothing; look at dozens, including errors.

### 5.4 Images

For images, saliency and Grad-CAM produce **heatmaps** over the picture. Unit III showed the most important use: DeGrave et al. (2021) found, using saliency maps, that COVID-19 chest X-ray models relied on text markers and patient positioning rather than the lungs. In Lab 12's chest X-ray models, a map concentrated outside the lungs would be a similar warning.

## 6. The limits of explanations

### 6.1 Explanations describe the model, not the patient

- An explanation shows what the **model** used, not what **causes** disease. The chest-pain coefficient in Section 3.1 reflects patient selection, not physiology.
- A plausible-looking explanation does not make a prediction correct. Our patient's explanation looks reasonable, and the prediction was wrong.
- People tend to accept explanations that look plausible. Unit III Session 5 showed that image explanations did not significantly protect clinicians from a biased model (Jabbour et al., 2023).

### 6.2 Why a sensible-looking heat map is not proof

A heat map answers one question: *which parts of this image most changed the model's output?* It does not answer a different question: *is the output correct?* There are four reasons why a map can look sensible while the prediction is wrong.

1. **The model explains itself.** The map is computed from the same model that made the prediction, and it has no access to the true diagnosis. A wrong prediction gets a map too. Grad-CAM, for example, highlights the regions that supported whichever class the model chose, so a wrong answer can come with a map that is just as focused as a right one.
2. **A map shows where, not what.** A highlighted region of a chest X-ray can contain lung tissue, ribs, a tube or a text marker. The map does not say which of these the model used. Jabbour et al.'s biased model (Unit III Session 5) relied on the low bone density that goes with old age; in a different image, the same shortcut could sit inside a region a reader expects to matter. DeGrave et al. (2021) needed careful analysis across many images, not one map, to show that COVID-19 models used text markers and patient positioning.
3. **The method may not reflect the model.** Some saliency methods produce similar maps even from a model that learned nothing (Section 6.3). A sensible-looking map from such a method only shows edges and shapes in the image.
4. **One case says little about accuracy.** A good map for one patient, or for a few demonstration cases, does not tell you how often the model is right. Accuracy, calibration and subgroup performance come from validation on many patients (Session 3).

**A worked example.** A model labels a chest X-ray "pneumonia", and its Grad-CAM map is centred on the right lower lung, where a radiologist would look. Reasons 1 and 2 say the map would look like this whether the model detected pneumonia there or reacted to something else in that region. Reason 3 says the map is worth less if the method has not passed a sanity check. Reason 4 says that even if this prediction were correct, it would not tell us how the model performs across patients. The map is a prompt to check, not a verdict.

### 6.3 Sanity checks for saliency maps

Adebayo et al. (2018) proposed a simple test: **randomise the model's weights** (or train it on shuffled labels) and recompute the maps. If the map barely changes, it is showing properties of the input — such as edges — rather than what the trained model learned. Some popular methods failed this test.

**Rule:** before trusting a saliency method, check that its maps change when the model is destroyed. Lab 14 includes this check.

### 6.4 What explanations are good for

Ghassemi et al. (2021) argue that current explanation methods cannot reliably justify individual decisions at the bedside, and that **rigorous validation** — the evidence of Session 3 — is a better basis for trust. A balanced position for this course:

| Use | Value |
|---|---|
| Finding shortcuts and data problems during development | High |
| Auditing a model across many cases and subgroups | High |
| Showing a clinician the main inputs behind one risk | Moderate: helps check data errors, but does not prove correctness |
| Proving to a patient or regulator that a prediction is right | None on its own |

When a decision must be fully explainable — for example, a simple screening score — an **interpretable model** may be the better choice, provided it performs adequately.

**When explanations are offered as the safety measure.** If a vendor says that explanations will let clinicians catch errors, a hospital can ask for evidence and safeguards that do not depend on the map:

1. **Local validation** of the model on the hospital's own patients before use (Session 3).
2. **Evidence about people, not pictures**: a study of clinicians using the tool, including cases where the model is wrong, as in Jabbour et al.
3. **A safer workflow**: clinicians record their own reading before they see the AI output, and the system shows uncertainty or abstains (Unit III Session 5, §7).
4. **Monitoring after go-live**: how often clinicians accept AI output that turns out to be wrong, and whether performance drifts (Session 4).
5. **A sanity-checked explanation method**, used by developers and auditors across many cases to look for shortcuts.

## 7. Education and skill requirements

### 7.1 Skills across the lifecycle

Every session in this unit needed people with particular skills. No single role covers all of them:

| Role | Skills needed | Where you saw them |
|---|---|---|
| Clinicians | Interpret outputs and their limits; recognise automation bias; know when to override; explain AI use to patients | Sessions 4–5; Unit III Session 5 |
| Data scientists and ML engineers | Clinical problem framing; healthcare data and standards; leakage-safe evaluation; calibration; documentation | Sessions 1–3 |
| Clinical informatics and IT staff | EHR integration (FHIR, CDS Hooks); input checks; monitoring pipelines | Session 4 |
| Managers and procurement teams | Evaluate evidence and fit; estimate workload and cost; plan ownership and rollback | Session 4 |
| Governance, legal, and ethics staff | Regulation, data protection, accountability | Unit III Session 6; Unit V |
| Patients and the public | Understand when AI is involved and how to raise concerns | Unit III Session 5 |

### 7.2 A competency framework for clinicians

Russell et al. (2023) interviewed experts and proposed six competency domains for health professionals using AI-based tools:

1. **Basic knowledge of AI** — what it is and where it is used in healthcare.
2. **Social and ethical implications** — how social and economic systems shape AI tools, and effects on equity.
3. **AI-enhanced clinical encounters** — combining AI output with other information in patient-centred care.
4. **Evidence-based evaluation** — judging the quality, accuracy, safety, fit, and biases of tools and their data.
5. **Workflow analysis** — adapting to changes in roles, responsibilities, and workflows.
6. **Practice-based learning and improvement** — continuing development as tools change.

Domain 4 corresponds closely to Session 3, and domain 5 to Session 4. Training requirements are also entering regulation: for example, Article 4 of the EU AI Act, which has applied since February 2025, requires providers and deployers of AI systems to take measures to ensure sufficient AI literacy among their staff. Unit V discusses regulation, including in India.

### 7.3 Keep the core skills

Unit III Session 6 discussed early evidence of **deskilling**: clinicians who rely on AI may lose skills they need when it is wrong or unavailable. Education for healthcare AI therefore has two parts: learning to use the tools well, and **continuing to train and assess the skills the tools support**.

## 8. Unit IV synthesis

Unit IV followed one model through the lifecycle:

| Session | Stage | Key lesson from the heart-disease example |
|---|---|---|
| 1 | Data | All Cleveland patients were angiography-referred; `ca` may not exist at decision time; Zurich cholesterol is all zeros |
| 2 | Development | Split by patient where possible; baselines first; validation data for every choice |
| 3 | Validation | Good ranking in Budapest; poor calibration in Zurich and Long Beach; tiny subgroups |
| 4 | Deployment | A checkup clinic would get about 8 referrals a day, 5 unnecessary; answer: not yet |
| 5 | Explanation | The strongest input reflects how patients were selected; a plausible explanation accompanied a wrong prediction |

The model never became a clinical tool. That is the realistic outcome for most models, and recognising it early is a skill in itself.

## 9. Revision summary

| Concept | Key idea |
|---|---|
| Interpretable vs post-hoc | Read the model directly, or explain it afterwards |
| Global vs local | All patients versus one prediction |
| Contributions | For logistic regression, coefficient × standardised value; exact |
| Permutation importance | Performance drop when an input is shuffled; depends on the evaluation data |
| SHAP, LIME | Model-agnostic local approximations |
| Saliency | Gradient, gradient × input, occlusion, Grad-CAM; look for meaningful regions and shortcuts |
| Sanity check | Maps must change when the model's weights are randomised |
| Limits | Explanations describe the model, not causes or correctness |
| Heat maps | Show where the model looked, not what it used there or whether it is right |
| Skills | Different roles need different skills; keep core clinical skills |

## 10. Review questions

1. Give one question each that a developer, a clinician, and a regulator might want an explanation to answer.
2. Using the contribution table in Section 3.2, what would the patient's log-odds and risk be if they had had no ST depression (contribution 0 instead of +0.80)?
3. Why is the largest coefficient in the model (asymptomatic chest pain) not evidence that having no chest pain increases heart-disease risk?
4. Maximum heart rate has a large coefficient but almost no permutation importance at Budapest. Give two possible reasons.
5. An occlusion map for V beats is highest over the zero padding at the end of the beat. What might the model be using, and how would you test it?
6. A saliency map looks the same before and after the model's weights are randomised. What does this tell you about the method?
7. A hospital plans to show SHAP explanations to clinicians "so that they can catch the model's errors". What does the evidence from Unit III suggest, and what would you recommend instead or in addition?
8. Which of Russell et al.'s six domains does each of these activities train: (a) reading a calibration plot; (b) redesigning a referral workflow; (c) explaining to a patient that a risk score was used?
9. A Grad-CAM map for a chest X-ray predicted as "pneumothorax" (collapsed lung) is centred on the upper right chest, where a radiologist would look. Give two reasons why the prediction may still be wrong, and one check that would give stronger evidence than the map.

## 11. Further reading

- Ghassemi M, Oakden-Rayner L, Beam AL. [The false hope of current approaches to explainable artificial intelligence in health care](https://doi.org/10.1016/S2589-7500(21)00208-9). *Lancet Digital Health* 2021;3(11):e745–e750.
- Adebayo J et al. [Sanity checks for saliency maps](https://arxiv.org/abs/1810.03292). *NeurIPS* 2018.
- Simonyan K, Vedaldi A, Zisserman A. [Deep inside convolutional networks: visualising image classification models and saliency maps](https://arxiv.org/abs/1312.6034). 2013.
- Zeiler MD, Fergus R. [Visualizing and understanding convolutional networks](https://arxiv.org/abs/1311.2901). *ECCV* 2014. — occlusion.
- Selvaraju RR et al. [Grad-CAM: visual explanations from deep networks via gradient-based localization](https://arxiv.org/abs/1610.02391). *ICCV* 2017.
- Lundberg SM, Lee S-I. [A unified approach to interpreting model predictions](https://arxiv.org/abs/1705.07874). *NeurIPS* 2017. — SHAP.
- Ribeiro MT, Singh S, Guestrin C. ["Why should I trust you?": explaining the predictions of any classifier](https://arxiv.org/abs/1602.04938). *KDD* 2016. — LIME.
- Russell RG et al. [Competencies for the use of artificial intelligence-based tools by health care professionals](https://doi.org/10.1097/ACM.0000000000004963). *Academic Medicine* 2023;98(3):348–356.
- UCI Machine Learning Repository. [Heart Disease dataset](https://archive.ics.uci.edu/dataset/45/heart+disease) (doi:10.24432/C52P4X), CC BY 4.0.
