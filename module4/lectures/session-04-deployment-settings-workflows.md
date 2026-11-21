# Session 4 — Deploying AI in Clinical Care: Settings, Selection, and Workflow Integration

**AI in Healthcare · Unit IV · B.Tech AI, Third Year**  
**Student lecture notes and primary study material**  
**Previous session:** [Session 3 — Model validation, performance assessment, and challenges in healthcare data](session-03-validation-performance.md)

## Learning objectives

After studying this chapter, you should be able to:

1. Compare clinical settings by decision pace, who acts on the output, available data, and the cost of errors.
2. Place an AI application on a scale from informing care to diagnosing or treating, and explain why its role matters.
3. Apply a structured set of questions to decide whether a clinic should adopt an AI tool.
4. Map a clinical workflow and identify where an AI output should appear, who receives it, and what happens next.
5. Describe how a tool connects to an electronic health record, and why deployment is staged and monitored.
6. Design a monitoring plan with measures for inputs, outputs, use and outcomes, state what change in each would be worrying, and say who acts on it.

You need Sessions 1–3 of this unit, Unit I Session 7 (stakeholders, "a model is one part of a service"), and Unit III Sessions 5–6 (automation bias, approval fatigue, accountability). Medical knowledge is not assumed.

## Contents

1. [The problem: a validated model is not yet a service](#1-the-problem-a-validated-model-is-not-yet-a-service)
2. [Clinical settings for deployment](#2-clinical-settings-for-deployment)
3. [Applications in clinical care delivery](#3-applications-in-clinical-care-delivery)
4. [Deciding whether to adopt a tool](#4-deciding-whether-to-adopt-a-tool)
5. [Integrating the tool into the workflow](#5-integrating-the-tool-into-the-workflow)
6. [Staged deployment and ownership](#6-staged-deployment-and-ownership)
7. [Case discussion](#7-case-discussion)
8. [Revision summary](#8-revision-summary)
9. [Review questions](#9-review-questions)
10. [Further reading](#10-further-reading)

## 1. The problem: a validated model is not yet a service

Session 3 left us with a heart-disease model developed in Cleveland that ranked patients well in Budapest but was miscalibrated at two other sites. Suppose a general physician's clinic now asks: **should we use it, and if so, how?**

Validation answers "does the model predict well on these patients?" Deployment raises different questions:

- Where in the clinic's day would the prediction appear, and who would see it?
- Is the input data available at that moment, in the same form the model was trained on?
- Can the clinic act on the output — are there enough cardiology appointments?
- Who is responsible when it is wrong, and how would anyone notice?

In Unit I you saw that a model is **one part of a service**: data checks, the risk estimate, an alert, a review, an action, and monitoring. This session is about building that service.

## 2. Clinical settings for deployment

### 2.1 Settings differ in pace, people, and data

A **clinical setting** is the place and context in which care is delivered. The same kind of model behaves very differently depending on where it is used.

| Setting | Decision pace | Who acts | Data typically available | Example from this course |
|---|---|---|---|---|
| Primary care / outpatient clinic | Minutes, during a visit | General physician | Vital signs, history, some laboratory results | Heart-disease referral model (Lab 4) |
| Emergency department | Minutes, under time pressure | Emergency doctors, triage nurses | Vital signs, rapid tests, imaging; history often incomplete | — |
| Inpatient ward | Hours | Ward nurses and doctors | Repeated observations, laboratory results, medications | Deterioration example (Unit I) |
| Intensive care unit (ICU) | Seconds to minutes | ICU team | Continuous monitor signals, frequent tests | — |
| Diagnostic department (radiology, cardiology, pathology) | Hours to days, queue-based | Radiologists, cardiologists, pathologists | Images, recorded signals, slides | Imaging (Unit III), ECG beats (Unit II) |
| Home and remote monitoring | Days, asynchronous | Patient, remote nursing team | Wearable and patient-reported data, often with gaps | Patient-portal inbox agent (Unit III) |
| Population and public health | Weeks | Public-health teams | Aggregated counts, registries | Disease surveillance (Unit II) |
| Operations and administration | Hours to weeks | Managers, schedulers | Bookings, bed and staff records | Resource optimisation (Unit II) |

### 2.2 The same model in a different setting is a different tool

Consider moving the heart-disease model from the checkup clinic to the **emergency department**:

- **A different question.** A patient arriving with chest pain needs to know whether a heart attack is happening *now*. The model predicts whether narrowed arteries are present — a slower, different question.
- **Different patients.** Emergency patients are more acutely ill, so prevalence and case mix change (Session 3).
- **Different timing.** The exercise-test inputs (`thalach`, `exang`, `oldpeak`) come from a stress test that would not be done before emergency treatment.

**Rule:** a model's intended setting is part of its definition. Using it elsewhere requires new validation, and often a new model.

## 3. Applications in clinical care delivery

### 3.1 What the AI does in the care process

Unit II covered applications by clinical area. For deployment, a more useful question is **what role the output plays** in care:

| Role | What the output does | Course example |
|---|---|---|
| Prioritise | Changes the order in which work is done | ECG beats flagged for review first |
| Detect or flag | Draws attention to a possible finding | Imaging finding flagged for a radiologist |
| Predict risk | Estimates the chance of a future event | Heart-disease referral risk |
| Recommend | Suggests an action | Referral suggested above a threshold |
| Draft or document | Produces text for a person to check | Inbox agent's draft reply |
| Act | Carries out an action | Agent proposing an appointment change, held for approval |

### 3.2 Inform, drive, diagnose, treat

Regulators grade software by how strongly its output affects care. The International Medical Device Regulators Forum (IMDRF) uses three levels of significance for Software as a Medical Device, which the UK's NICE framework (Section 4.3) also uses:

| Level | Meaning | Example |
|---|---|---|
| **Inform** clinical management | Provides information; the clinician decides | Showing the referral risk alongside other results |
| **Drive** clinical management | Guides the next step directly | "Refer to cardiology" recommendation |
| **Diagnose or treat** | Itself produces the diagnosis or treatment | Software that diagnoses disease without a clinician's interpretation |

The higher the level, and the more serious the condition, the stronger the evidence and oversight required. Unit V covers the regulatory consequences. For now, note that **the same model can be deployed at different levels**: displaying a risk informs, while an automatic referral drives.

## 4. Deciding whether to adopt a tool

### 4.1 Questions to ask, in order

A clinic choosing whether to adopt an AI tool can work through a fixed sequence of questions. A "no" at any step is a reason to stop or to fix that problem first.

| # | Question | For the heart-disease referral model |
|---|---|---|
| 1 | **What problem does it solve, and what happens now?** | Referral decisions vary between physicians. Current referral rate and missed-disease rate should be measured first, as the baseline |
| 2 | **What is the evidence, and how strong?** | Internal and external validation on historical UCI data (Session 3); no prospective or local evaluation |
| 3 | **Does it fit our patients and data?** | Trained on angiography-referred patients, not checkup patients; prevalence in a checkup clinic is likely much lower |
| 4 | **Are the inputs available at decision time?** | Exercise-test inputs need a stress test, which most checkups do not include |
| 5 | **Can we act on the output?** | Depends on cardiology appointment capacity |
| 6 | **What are the costs?** | Integration, staff time for alerts, extra referrals, monitoring |
| 7 | **Is it safe, permitted, and accountable?** | Regulatory status, data protection, a named owner (Unit V; Unit III Session 6) |
| 8 | **How will we monitor it, and when would we stop?** | Monitoring measures and a rollback plan (Section 6) |

For this model, questions 3 and 4 already raise serious problems. A responsible answer is **not yet**: a local evaluation with the clinic's own patients would be needed first.

### 4.2 A workload calculation

Question 5 can be estimated before deployment. Suppose (illustrative numbers) the clinic sees **40 checkups a day**, 10% of these patients have disease, and the model keeps the sensitivity (0.75) and specificity (0.85) it had in Cleveland:

| Quantity | Calculation | Per day |
|---|---|---|
| Patients with disease | 40 × 0.10 | 4 |
| Correctly flagged (true positives) | 4 × 0.75 | 3 |
| Missed (false negatives) | 4 − 3 | 1 |
| Patients without disease | 40 × 0.90 | 36 |
| Wrongly flagged (false positives) | 36 × 0.15 | 5.4 |
| **Total flagged for referral** | 3 + 5.4 | **8.4** |

The clinic would generate about 8 referrals a day, of which about 5 are unnecessary, and still miss about one patient with disease. Whether that is acceptable depends on cardiology capacity, the cost of unnecessary tests, and what the clinic does today. These numbers also assume Cleveland's sensitivity and specificity carry over, which Session 3 showed cannot be taken for granted.

### 4.3 Published frameworks

Several frameworks structure these questions. They overlap; each emphasises something different.

| Framework | From | What it helps with |
|---|---|---|
| **NICE Evidence Standards Framework (ESF)** for digital health technologies | NICE, UK; first published 2018, updated August 2022 to cover AI and adaptive algorithms | What evidence a buyer should expect, graded by the technology's function (tiers A, B, C; tier C split into inform, drive, diagnose, treat) |
| **NASSS** (Non-adoption, Abandonment, Scale-up, Spread, Sustainability) | Greenhalgh et al., 2017 | Why technologies fail to be adopted or are abandoned; seven domains: the condition, the technology, the value proposition, the adopters, the organisation, the wider system, and adaptation over time |
| **DECIDE-AI** | Vasey et al., 2022 | How to report the first small-scale clinical evaluations of AI decision support, including human factors |
| **FUTURE-AI** | Lekadir et al., 2025 | International consensus principles for trustworthy, deployable healthcare AI: fairness, universality, traceability, usability, robustness, explainability |

NASSS is a useful warning: most failures it describes are not about model accuracy. A technically good tool can still fail because the condition is complex, the value is unclear to the people paying, staff do not want it, or the organisation cannot support it.

## 5. Integrating the tool into the workflow

### 5.1 Map the workflow before and after

A **workflow** is the sequence of steps, people, and systems involved in a task. Before deciding where the AI fits, map the current workflow; then map the proposed one.

```text
CURRENT                                  PROPOSED
Patient checks in                        Patient checks in
      |                                        |
Nurse records vital signs                Nurse records vital signs
      |                                        |
Physician examines patient               Physician opens the record
      |                                        |  <-- risk computed from recorded inputs;
Physician decides on referral                  |      shown with the main inputs used
      |                                  Physician examines patient
Referral letter (if any)                       |
                                         Physician decides; records agreement or
                                         reason for overriding the suggestion
                                               |
                                         Referral letter (if any)
```

The map makes the open questions visible: Which inputs exist when the record is opened? What does the physician see if an input is missing? Is the override reason optional or required? How does the referral team cope with extra referrals?

### 5.2 The "five rights" of decision support

A widely used checklist for clinical decision support (CDS), set out by Osheroff and colleagues, asks whether the tool delivers:

| Right | Question | Heart-disease example |
|---|---|---|
| **Information** | Is the content relevant and evidence-based? | The risk, with its validation status and main inputs |
| **Person** | Does it reach the person who decides? | The physician, not the nurse recording vital signs |
| **Format** | Is the form suited to the task? | A short note in the record rather than an interrupting pop-up |
| **Channel** | Through which system? | Inside the electronic health record (EHR), not a separate website |
| **Time** | At the point of decision? | When the physician opens the record, before deciding |

A tool that fails one right — for example, an alert that arrives after the patient has left — can be accurate and still useless. Unit III Session 5 covered the human side: automation bias, alert and approval fatigue, and interaction design.

### 5.3 Connecting to the electronic health record

A deployed model needs a data path in and a result path out:

```text
EHR  --(patient data as FHIR resources)-->  model service  --(result)-->  EHR screen
      Patient, Observation, ...                 risk +                    card or app
                                                checks                    shown to physician
```

Two HL7 standards built on **FHIR** (Fast Healthcare Interoperability Resources, which you met in Unit I) are commonly used:

- **CDS Hooks** (version 2.0.1, based on FHIR R4): the EHR calls an external service at a defined moment — a **hook** such as `patient-view` (a record is opened) or `order-sign` (an order is signed) — and the service returns **cards** to display.
- **SMART App Launch** (version 2.2.0): a separate app opens inside the EHR with permission to read the patient's FHIR data.

A simplified, illustrative card that a `patient-view` hook might return:

```json
{
  "cards": [
    {
      "summary": "Heart-disease referral risk: 62% (educational model, not locally validated)",
      "indicator": "info",
      "detail": "Main inputs: chest-pain type, ST depression on exercise, maximum heart rate. Exercise-test inputs from 2026-10-12.",
      "source": { "label": "Course teaching model (UCI Cleveland data)" }
    }
  ]
}
```

*This is a partial, illustrative example. It has not been validated against the CDS Hooks specification, and the risk value is invented.*

`indicator` can be `info`, `warning`, or `critical`. Choosing `info` rather than `critical` is a workflow decision: a "critical" card for a routine referral risk would contribute to alert fatigue.

### 5.4 Keep live inputs identical to training inputs

A frequent deployment failure is that the live data differ from the training data in ways nobody notices:

- **Units:** cholesterol in mg/dL in training, mmol/L in the live record.
- **Codes:** a chest-pain category recorded with a different coding scheme.
- **Timing:** a test result used in training that arrives hours after the decision in practice (Session 1).
- **Missing values:** training data were imputed with the mode; live data with no value should trigger a defined response, such as showing "risk not available" rather than a silent default.
- **Images outside the validated scope:** a model validated on frontal chest X-rays of children is sent an adult image, a side (lateral) view, a blank or badly exposed image, or an image from a new machine. It still produces a score, but nobody has measured how good that score is.

The integration should **check each input** — presence, units, plausible range — before computing a risk, and record what it received.

For images, the check compares each image with the model's intended use. Many of the facts it needs are in the DICOM header you met in Unit III Session 2, for example `BodyPartExamined`, `ViewPosition`, `PatientAge` and `ManufacturerModelName`; simple pixel checks catch blank or very dark images. An image that fails gets **no AI output**: it is marked "not assessed by AI" and handled in the usual way, never given a low priority by default. The check has its own limits. Header fields are often empty or filled in inconsistently, and a new machine can pass every rule while producing images that look different to the model. That is why monitoring after go-live (Section 6) is still needed.

## 6. Staged deployment and ownership

### 6.1 Deploy in stages

Deployment is usually staged, with a decision point after each stage:

| Stage | What happens | What it shows |
|---|---|---|
| **Silent (shadow) mode** | The model runs on live data, but no one sees the output; predictions are compared with outcomes later | Whether live data and performance match validation, without affecting care (Kwong et al., 2022) |
| **Limited pilot** | A few clinicians see the output, with close monitoring | Usability, workload, overrides, unexpected effects |
| **Wider rollout** | Use across the service | Effects at scale; continued monitoring |

Each stage needs a **rollback plan**: who can switch the tool off, and what the workflow returns to.

### 6.2 Why a fixed model still needs monitoring

A deployed model's code usually does not change. Its performance can still change, because the world around it does: new patients, a new laboratory test, a new recording practice, an epidemic. Unit III Session 4 (§6) called this **dataset shift**. Validation and silent mode describe the model at one moment; monitoring is how you find out that the moment has passed.

Think of the deployed tool as a chain. Data go **in**, the model produces an **output**, a clinician **uses** it, and the patient has an **outcome**. A change anywhere in the world shows up somewhere along this chain, so monitoring places one measure at each link.

### 6.3 Four places to look

| Area | Question it answers | What to measure | A worrying change | How soon you can see it |
|---|---|---|---|---|
| **Inputs** | Are we feeding the model the kind of data it learned from? | Missing-value rate of each input; the distribution of each input (average, spread, share of each category), compared with the training data | A missing-value rate that jumps; values moving to a new range; a category that appears or disappears | Immediately: no outcome needed |
| **Outputs** | Is the model behaving as it did in validation? | The proportion of patients flagged (or the average risk), per day or week, compared with the rate expected from validation or silent mode | The flag rate moving well above or below the expected level, with no known reason | Immediately |
| **Use** | Are clinicians acting on the output, and why not? | How often clinicians follow or override the suggestion, and the reasons they give | Overrides rising (the tool is wrong more often, or staff have stopped trusting it: alert fatigue); overrides falling to almost zero (staff may be accepting it without checking: automation bias, Unit III Session 5) | Within days |
| **Outcomes** | Is the model still right? | Performance against confirmed outcomes: sensitivity, PPV, calibration (Session 3, §4–5), and the same measures for each patient group (Session 3, §8) | PPV or sensitivity falling; predicted risk drifting away from the observed rate; one group doing worse than the others | Late: only once confirmed outcomes arrive, often weeks or months later |

Three points explain why all four are needed:

1. **Fast measures warn; slow measures confirm.** Inputs and outputs can be checked the same day, but they only show that something has *changed*, not that the model is *wrong*. Outcomes show whether it is wrong, but only after patients have already been affected. A plan needs both.
2. **Each measure catches problems the others miss.** A unit change can shift the inputs while the flag rate barely moves. A model can keep a steady flag rate while flagging the wrong patients, which only outcomes reveal. Clinicians may stop reading alerts that are still correct, which only the use measure reveals.
3. **"A worrying change" needs a baseline and a direction.** "Watch the flag rate" is not a plan. Record the expected value from validation or silent mode, and decide in advance which change matters: for example, "a flag rate more than twice the silent-mode rate for a week".

**Example: one change, seen at each link.** In the heart-disease clinic, suppose the laboratory switches cholesterol reporting from mg/dL to mmol/L (Section 5.4). Typical values fall from about 200–250 to about 5–6.5.

- **Inputs** show it on the first day: the cholesterol distribution moves to a range never seen in training.
- **Outputs** may show it next: if the model relies on cholesterol, the proportion flagged changes.
- **Use** may show it as more overrides, as physicians see risks that do not match the patient in front of them.
- **Outcomes** would show it last, months later, as worse PPV or more missed disease.

The input check finds the problem before any patient is affected. The outcome measure, used alone, would find it only after many were.

### 6.4 From a signal to an action

A measure is only useful if someone acts on it. Before go-live, write down:

- **An owner.** A named person or group (for example, a clinical lead with an AI governance committee) who reviews the measures, responds to clinicians' reports, approves updates, and decides when to pause or retire the tool.
- **A pause rule.** Which change, of what size, for how long, pauses the tool. Use measures that are available quickly, so a pause does not wait for outcome data.
- **A fallback.** What staff do while the tool is paused: usually the workflow from before the tool, with staff told that it is off.
- **A condition for switching back on.** Evidence on recent local data, including the patients who caused the problem, that performance is acceptable again. A retrained or recalibrated model is a new model: validate it, ideally first in silent mode.

Unit V covers monitoring and post-market obligations in detail.

## 7. Case discussion

The Budapest site from Session 3 is the Hungarian Institute of Cardiology. Suppose (a fictional scenario) its team considers using the Cleveland model to decide which patients need angiography sooner.

1. Which of the questions in Section 4.1 does Session 3's external validation answer, and which remain open?
2. The model overpredicted risk in Budapest (41% predicted versus 36% observed). What would you do before showing risks to clinicians?
3. Where in the institute's workflow would the risk appear, and at which level — inform or drive?
4. Design a silent-mode evaluation: how long would it run, and what would you measure?
5. What would make you stop the deployment after the pilot?

## 8. Revision summary

| Concept | Key idea |
|---|---|
| Setting | Pace, people, data, and error costs differ; the setting is part of the model's definition |
| Role of output | Prioritise, detect, predict, recommend, draft, act; inform versus drive versus diagnose/treat |
| Adoption questions | Problem and baseline → evidence → fit → inputs → capacity → cost → safety → monitoring |
| Workload | Estimate flagged, true, and missed cases per day before deploying |
| Frameworks | NICE ESF (evidence), NASSS (adoption failure), DECIDE-AI (early evaluation), FUTURE-AI (principles) |
| Five rights | Right information, person, format, channel, time |
| Integration | FHIR data in, CDS Hooks card or SMART app out; check every live input |
| Staged deployment | Silent mode → pilot → rollout, with rollback and a named owner |
| Monitoring | Inputs, outputs, use, outcomes; each with a baseline and a worrying change; inputs and outputs warn early, outcomes confirm late |
| Response | Owner, pause rule, fallback workflow, evidence needed before switching back on |

## 9. Review questions

1. Give two reasons why the heart-disease model should not be used in an emergency department without new validation.
2. A tool displays a sepsis risk score on a ward dashboard. Another automatically orders blood cultures above a threshold. Which informs and which drives clinical management? Why does the difference matter?
3. Repeat the workload calculation in Section 4.2 for a clinic seeing 60 patients a day with 5% prevalence. How many referrals per day, and how many unnecessary?
4. Name the NASSS domain most relevant to each problem: (a) staff distrust the tool; (b) the hospital cannot pay for integration; (c) the tool's performance decays over three years.
5. An alert reaches the physician after the patient has left the clinic. Which of the five rights does it fail?
6. A live record stores cholesterol in mmol/L, but the model was trained in mg/dL. What would happen, and how should the integration prevent it?
7. What can silent-mode deployment show that retrospective validation cannot? What can it not show?
8. The heart-disease clinic starts a free screening camp, and many younger, healthier patients now attend. For each of the four monitoring areas in Section 6.3, say what you would expect to change and in which direction. Which measure would show the change first, and which would show whether the model is still useful?
9. A clinician says: "We check the model's PPV every quarter, so we don't need anything else." Give two problems the quarterly PPV check would miss or find too late.

## 10. Further reading

- Greenhalgh T et al. [Beyond adoption: a new framework for theorizing and evaluating nonadoption, abandonment, and challenges to the scale-up, spread, and sustainability of health and care technologies](https://doi.org/10.2196/jmir.8775). *Journal of Medical Internet Research* 2017;19(11):e367. — the NASSS framework.
- NICE. [Evidence standards framework for digital health technologies](https://www.nice.org.uk/corporate/ecd7) (updated August 2022).
- Vasey B et al. [Reporting guideline for the early-stage clinical evaluation of decision support systems driven by artificial intelligence: DECIDE-AI](https://doi.org/10.1038/s41591-022-01772-9). *Nature Medicine* 2022;28:924–933.
- Lekadir K et al. [FUTURE-AI: international consensus guideline for trustworthy and deployable artificial intelligence in healthcare](https://doi.org/10.1136/bmj-2024-081554). *BMJ* 2025;388:e081554.
- Kwong JCC et al. [The silent trial — the bridge between bench-to-bedside clinical AI applications](https://doi.org/10.3389/fdgth.2022.929508). *Frontiers in Digital Health* 2022.
- Osheroff JA et al. *Improving Outcomes with Clinical Decision Support: An Implementer's Guide*, 2nd edition. HIMSS, 2012. — source of the CDS "five rights".
- HL7. [CDS Hooks specification, version 2.0.1](https://cds-hooks.hl7.org/) and [SMART App Launch, version 2.2.0](https://hl7.org/fhir/smart-app-launch/) (checked October 2026).
