# Session 5 — Humans Working with AI: Human–AI Interaction and the Patient–Provider–AI Relationship

**AI in Healthcare · Unit III · B.Tech AI, Third Year**  
**Student lecture notes and primary study material**  
**Previous session:** [Session 4 — When AI Causes Harm](session-04-risks-safety-unintended-consequences.md) · **Capstone project:** [inbox agent capstone project](../projects/patient-inbox-agent/README.md)

## Learning objectives

By the end of this lecture, you should be able to:

1. Describe common ways of combining a clinician and an AI system, and what each assumes about the human.
2. Define automation bias and explain, with evidence, how a wrong AI suggestion changes expert decisions.
3. Explain why adding explanations to AI output does not reliably prevent errors.
4. Estimate a human reviewer's workload and explain approval fatigue.
5. Discuss how AI changes the relationship between patients and clinicians, including disclosure.

**Prerequisites:** Session 4 (sources of harm, the Swiss cheese model) and Session 3 (the inbox agent's review pause). Sensitivity and accuracy.

## Contents

1. [The problem: a human in the loop is not automatically a safeguard](#1-the-problem-a-human-in-the-loop-is-not-automatically-a-safeguard)
2. [Ways to combine a human and an AI](#2-ways-to-combine-a-human-and-an-ai)
3. [When the combination helps](#3-when-the-combination-helps)
4. [Automation bias](#4-automation-bias)
5. [Do explanations help?](#5-do-explanations-help)
6. [Reviewer workload and approval fatigue](#6-reviewer-workload-and-approval-fatigue)
7. [Designing the interaction](#7-designing-the-interaction)
8. [The patient–provider–AI relationship](#8-the-patientproviderai-relationship)
9. [Summary](#9-summary)
10. [Review questions](#10-review-questions)
11. [References](#11-references)

## 1. The problem: a human in the loop is not automatically a safeguard

Almost every healthcare AI product says a clinician "remains in control". In Session 4 the human reviewer was one layer of the Swiss cheese model. This session asks **how good that layer really is**.

A reviewer only protects patients if they **notice** a wrong output and **act** on it. Both depend on how the AI is presented, how much work the reviewer has, and how much they trust the system.

## 2. Ways to combine a human and an AI

| Pattern | How it works | Example | What it assumes about the human |
|---|---|---|---|
| **Second reader** | The clinician decides first, then sees the AI output | AI reviews a mammogram after the radiologist | They form their own view before seeing the AI |
| **Concurrent aid** | The AI output is shown while the clinician decides | Probabilities shown next to a skin image | They can weigh it without simply copying it |
| **Triage** | The AI orders or filters the worklist | Urgent-looking scans read first | Items the AI ranks low still get looked at |
| **Draft and approve** | The AI prepares an action; a person approves it | The inbox agent's proposals | The approver reads each item carefully |
| **Autonomous** | The AI decides; humans check samples or exceptions | Autonomous diabetic-retinopathy screening (Session 1) | Monitoring catches drift and errors |

Each pattern moves risk to a different place. "Draft and approve" is only as safe as the approval step.

## 3. When the combination helps

**Skin cancer recognition (Tschandl et al., 2020).** Clinicians of different experience classified dermatoscopic images with and without AI support.

- Good-quality AI support made the clinician–AI combination **more accurate than either the AI or the clinicians alone**.
- The **least experienced** clinicians gained the most.
- **Faulty AI misled clinicians at every level, including experts.**
- How the AI was shown mattered: multiclass probabilities helped more than showing similar example images.

The same support that helps when the AI is right harms when it is wrong. Whether the combination helps overall depends on how often the AI is wrong and whether people can tell.

## 4. Automation bias

**Automation bias** is the tendency to rely too much on an automated suggestion: accepting it when it is wrong (an error of commission) or not looking for problems it did not flag (an error of omission).

**A systematic review (Goddard et al., 2012)** included 74 studies. It found that automation bias was influenced by:

- **the user:** task experience, confidence and trust in the system
- **the environment:** workload, task complexity and time pressure
- **the design:** where the advice appears on screen, whether confidence levels are shown, and whether the system gives **information** or a **recommendation**

Training and **emphasising the user's accountability** reduced it.

**Mammography (Dratsch et al., 2023).** 27 radiologists rated 40 mammograms with a purported AI system. Unknown to them, the AI's category was wrong for 12 of the 40.

| Radiologists | Correct when AI was right | Correct when AI was wrong |
|---|---:|---:|
| Inexperienced | 79.7% | 19.8% |
| Moderately experienced | 81.3% | 24.8% |
| Very experienced | 82.3% | 45.5% |

**Interpretation:** a wrong suggestion lowered accuracy at every experience level. Experience reduced the effect but did not remove it. This was an experiment with a fake AI, designed to measure the bias, not a real deployment.

## 5. Do explanations help?

A common proposal is to show **why** the model made its prediction, so that people can spot a wrong one.

**Jabbour et al. (2023)** tested this with 457 hospital clinicians, who diagnosed the cause of breathing failure in clinical vignettes. Some AI predictions were from a standard model; others were from a model deliberately biased in a systematic way. Half the clinicians also saw an image-based explanation.

| Condition | Change in accuracy vs. no AI (baseline 73.0%) |
|---|---:|
| Standard model, no explanation | +2.9 percentage points |
| Standard model, with explanation | +4.4 percentage points |
| Biased model, no explanation | −11.3 percentage points |
| Biased model, with explanation | −9.1 percentage points |

**Interpretation:** a biased model hurt more than an accurate model helped, and the explanations did **not** significantly reduce the harm. An explanation is something else to read, not a guarantee that a reader will catch the error.

## 6. Reviewer workload and approval fatigue

**Approval fatigue** is the reviewer's version of alert fatigue (Session 4). When almost every item is correct, approving becomes routine, and the rare wrong item gets approved too.

### Worked example (fictional clinic, illustrative numbers)

Suppose the inbox agent produces about **2.3 proposals per message** (a reply, a route, and sometimes an appointment change; an illustrative figure). Suppose also that a clinic receives 300 portal messages a day and a reviewer spends 20 seconds per proposal.

| Quantity | Calculation | Result |
|---|---|---:|
| Proposals per day | 300 × 2.3 | 690 |
| Reviewer time | 690 × 20 s | 13,800 s ≈ 3.8 hours |
| Wrong proposals if 1% are wrong | 690 × 0.01 | ≈ 7 |

**Interpretation:** about seven wrong items hidden among about 683 correct ones, every day. The reviewer has nearly four hours of repetitive work, which is exactly the setting in which Goddard et al. found more automation bias. The "human in the loop" is real, but it is a weak layer unless the design helps.

Ways to make the layer stronger:

1. **Reduce volume:** auto-approve nothing that matters, but group or skip items that carry no risk.
2. **Highlight risk:** flag claims of completion, result interpretation and cancellations, as the capstone project's review flags do. Remember that *no flag* is not proof of safety.
3. **Measure the reviewer:** if the approval rate is near 100%, ask whether reviews are really happening, for example by inserting known test items.

## 7. Designing the interaction

| Design choice | Safer default | Why |
|---|---|---|
| When the AI output appears | After the clinician's own first judgement, where time allows | Reduces anchoring on the AI |
| What it shows | Information and uncertainty, not only a verdict | Goddard et al.: recommendations increase over-reliance |
| When it is unsure | Says so, or abstains and hands over | A confident wrong answer is the most misleading |
| Disagreeing | One click, with a reason recorded | Makes rejection as easy as approval |
| Responsibility | Stated clearly in the interface and training | Emphasising accountability reduced automation bias |
| Monitoring | Track acceptance and override rates | Shows over-reliance and drift over time |

## 8. The patient–provider–AI relationship

AI is no longer only a back-office tool. It now enters the conversation between patient and clinician.

**AI in the consultation room.** **Ambient AI scribes** listen to a consultation, with consent, and draft the clinical note for the clinician to edit. At The Permanente Medical Group in the US, 3,442 physicians used such a tool in as many as 303,266 patient encounters in its first 10 weeks (Tierney et al., 2024). Early reports linked use to less time spent on documentation, and physicians said it helped them focus on the patient. These were early results from an uncontrolled roll-out, and draft notes still need checking for errors and omissions.

**AI speaking to patients.** [Unit II session 1](../../module2/lectures/session-01-patient-facing-ai-and-treatment-decisions.md) showed that chatbot answers to patient questions can be rated well. In the inbox agent, a drafted reply goes out under the clinic's name. The patient may not know a model wrote the first draft.

**Do patients want to know?** In a US survey of 926 adults (Khullar et al., 2022):

- **66%** said it was *very important* to be told when AI played a **big role** in their diagnosis or treatment
- **71.5%** were very or somewhat uncomfortable receiving a diagnosis from a program that was right 90% of the time but **could not explain** its decision

**Questions this raises:**

- **Disclosure:** when should patients be told that AI was involved, and how?
- **Consent:** recording a consultation for an AI scribe needs the patient's agreement.
- **Whose voice:** if a reply is AI-drafted and clinician-approved, the clinician and clinic are responsible for every word.
- **Equity:** patients with limited language skills or digital access may be helped most, or left out most.

### Thought experiment: two reviewers

Imagine one clinic day: 15 messages produce about 34 proposals (illustrative numbers). Among them are the three failures from Session 4: a reply claiming a booking that never happened, an injected cancellation, and stroke-like symptoms routed as routine. Only the first carries a specific warning flag (`claims_completion`). The injected cancellation is flagged only as a cancellation, exactly like a genuine one, and the stroke case carries no flag at all.

- A reviewer who **approves everything** lets all three through.
- A reviewer who **rejects only items with a warning flag** stops the false "booked" reply, but approves the other two, because they look like the routine items around them.

Flags help a tired reviewer, but the most dangerous errors are the ones that produce a normal-looking work item. In the capstone project you estimate your own agent's reviewer workload (evaluation E5).

## 9. Summary

| Idea | Key point |
|---|---|
| Human in the loop | A safeguard only if the human notices and acts |
| Combination | Good AI plus clinicians can beat either alone; faulty AI misleads even experts |
| Automation bias | A wrong AI suggestion cut inexperienced radiologists' accuracy from 79.7% to 19.8% |
| Explanations | Did not significantly reduce harm from a biased model |
| Approval fatigue | Hundreds of mostly correct items a day hide the few wrong ones |
| Design | Show uncertainty, make rejection easy, state accountability, monitor override rates |
| Patients | Most want to be told when AI plays a big role; consent and responsibility must be clear |

**Next session:** [Session 6](session-06-trust-liability-workforce-design.md): trust, liability, the workforce and human-centred design.

## 10. Review questions

1. For the "triage" pattern in §2, describe one way it could harm a patient whose case the AI ranks as low priority.
2. In Tschandl et al., why might the least experienced clinicians gain the most from good AI and also be the most at risk from faulty AI?
3. Using the Dratsch et al. table, calculate the drop in accuracy for very experienced radiologists when the AI was wrong. Why is experience not a complete defence?
4. A vendor says: "Our model shows a heat map, so clinicians will catch its mistakes." Use Jabbour et al. to respond.
5. Redo the worked example in §6 for 120 messages a day and 30 seconds per proposal. How much reviewer time is needed?
6. Propose two changes to the inbox agent's review screen that would reduce approval fatigue without removing the human.
7. Should a patient be told that a portal reply was drafted by AI and approved by a nurse? Give one argument for and one against.

## 11. References

- Tschandl P, Rinner C, Apalla Z, et al. Human–computer collaboration for skin cancer recognition. *Nature Medicine*. 2020;26(8):1229–1234. [doi:10.1038/s41591-020-0942-0](https://doi.org/10.1038/s41591-020-0942-0)
- Goddard K, Roudsari A, Wyatt JC. Automation bias: a systematic review of frequency, effect mediators, and mitigators. *Journal of the American Medical Informatics Association*. 2012;19(1):121–127. [doi:10.1136/amiajnl-2011-000089](https://doi.org/10.1136/amiajnl-2011-000089)
- Dratsch T, Chen X, Rezazade Mehrizi M, et al. Automation bias in mammography: the impact of artificial intelligence BI-RADS suggestions on reader performance. *Radiology*. 2023;307(4):e222176. [doi:10.1148/radiol.222176](https://doi.org/10.1148/radiol.222176)
- Jabbour S, Fouhey D, Shepard S, et al. Measuring the impact of AI in the diagnosis of hospitalized patients: a randomized clinical vignette survey study. *JAMA*. 2023;330(23):2275–2284. [doi:10.1001/jama.2023.22295](https://doi.org/10.1001/jama.2023.22295)
- Tierney AA, Gayre G, Hoberman B, et al. Ambient artificial intelligence scribes to alleviate the burden of clinical documentation. *NEJM Catalyst*. 2024;5(3). [doi:10.1056/CAT.23.0404](https://doi.org/10.1056/CAT.23.0404)
- Khullar D, Casalino LP, Qian Y, Lu Y, Krumholz HM, Aneja S. Perspectives of patients about artificial intelligence in health care. *JAMA Network Open*. 2022;5(5):e2210309. [doi:10.1001/jamanetworkopen.2022.10309](https://doi.org/10.1001/jamanetworkopen.2022.10309)
