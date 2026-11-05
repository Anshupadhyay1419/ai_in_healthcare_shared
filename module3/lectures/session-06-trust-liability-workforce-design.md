# Session 6 — Trust, Liability, Workforce and Human-Centred Design

**AI in Healthcare · Unit III · B.Tech AI, Third Year**  
**Student lecture notes and primary study material**  
**Previous session:** [Session 5 — Humans Working with AI](session-05-human-ai-interaction.md) · **Capstone project:** [inbox agent capstone project](../projects/patient-inbox-agent/README.md)

This session brings together four syllabus topics. Each is introduced through its main idea and one piece of evidence; Units IV and V return to the technical and legal detail.

## Learning objectives

By the end of this lecture, you should be able to:

1. Distinguish trust from trustworthiness and explain **calibrated** trust.
2. Explain how the standard of care shapes a clinician's liability when using AI, and who else is accountable.
3. Describe what an audit trail must record for an AI agent whose actions a human approved.
4. Discuss evidence on how AI changes healthcare work, including deskilling.
5. Apply human-centred design principles, including WHO's six principles, to an AI agent.

**Prerequisites:** Sessions 4 and 5 (sources of harm, automation bias, approval fatigue); the inbox agent from Session 3.

## Contents

1. [The problem: who is responsible?](#1-the-problem-who-is-responsible)
2. [Trust and trustworthiness](#2-trust-and-trustworthiness)
3. [Liability and accountability](#3-liability-and-accountability)
4. [Audit trails for AI actions](#4-audit-trails-for-ai-actions)
5. [AI and the healthcare workforce](#5-ai-and-the-healthcare-workforce)
6. [Human-centred design principles](#6-human-centred-design-principles)
7. [Design principles for AI agents](#7-design-principles-for-ai-agents)
8. [Summary](#8-summary)
9. [Review questions](#9-review-questions)
10. [References](#10-references)

## 1. The problem: who is responsible?

The inbox agent proposes cancelling a patient's appointment. A nurse approves it, among dozens of other items. The patient misses a review and comes to harm. Who is responsible: the nurse, the clinic, the company that built the agent, or the model provider?

There is no single answer, but there are **better and worse designs**. A good design makes it clear who decided what, on what information, and makes a harmful action hard to take by accident.

## 2. Trust and trustworthiness

- **Trust** is a person's willingness to rely on a system.
- **Trustworthiness** is whether the system actually deserves it, for this task, setting and patient group.

The goal is not maximum trust. It is **calibrated trust**: trust that matches the system's real capability (Lee and See, 2004).

| | System is reliable for this case | System is unreliable for this case |
|---|---|---|
| **High trust** | Appropriate reliance | **Misuse**: over-reliance, automation bias (Session 5) |
| **Low trust** | **Disuse**: benefits lost, e.g. alerts ignored | Appropriate scepticism |

Trust should be earned through the things this unit has covered:

1. **Evidence** for the specific task and setting (Session 1's evidence ladder)
2. **Transparency** about what the system does, its data and its known failure modes
3. **A visible track record**, from monitoring after deployment (Session 4)
4. **Honest uncertainty**: a system that says "I don't know" when it should

Patients' trust matters too. In Session 5, most surveyed patients wanted to be told when AI played a big role in their care.

## 3. Liability and accountability

**Liability** is legal responsibility for harm. **Accountability** is the wider duty to explain and answer for a decision, and to put things right.

### The clinician and the standard of care

In medical negligence, a clinician is usually judged against the **standard of care**: what a competent clinician would have done in the circumstances. Price, Gerke and Cohen (2019) analysed what this means under current US law for a physician using an AI recommendation:

- Following an AI recommendation that **matches standard care** is generally protected, even if the outcome is bad.
- The risky case is **following an AI recommendation that departs from standard care** when it turns out to be wrong.
- This pushes clinicians to use AI mainly to **confirm** what they would do anyway, which limits the value of an AI that is genuinely better than usual practice.
- The authors note that as AI becomes part of standard care, rejecting a correct AI recommendation could itself become a source of liability.

This is US analysis. Indian law, medical-device regulation and data-protection rules are covered in Unit V.

### Who else is accountable

| Party | Accountable for |
|---|---|
| **Developer / vendor** | The model's design, testing and documentation, including its known limits |
| **Deploying organisation** | Choosing the tool, validating it locally, training staff, monitoring, and being able to switch it off |
| **Clinician or reviewer** | Their own decision, including the decision to approve |
| **Regulator** | Rules for approval and post-market surveillance of medical devices |

Accountability that is shared by everyone can end up owned by no one. Each party's role should be written down before the tool goes live.

## 4. Audit trails for AI actions

An **audit trail** is a record of who did what, when, and based on what. For an AI agent whose actions a human approves, a useful record shows:

1. **The input:** the patient message and which patient's session it came from
2. **What the agent looked at:** each tool call and its result
3. **What it proposed:** every proposal, including ones the code rejected
4. **What the reviewer saw:** the proposals and any flags shown at review time
5. **The decision:** who approved or rejected each item, and when
6. **What happened:** the result of executing each approved item

In the capstone project, the `audit_log` table records items 4–6, and the saved graph checkpoints hold items 1–3. Item 4 matters for accountability: a reviewer can only be expected to catch what the screen showed them. That is why the project's audit log stores the flags shown at decision time, and why the table refuses updates and deletions.

## 5. AI and the healthcare workforce

### Replacement predictions

In 2016 a leading AI researcher said "we should stop training radiologists now" (quoted in Langlotz, 2019). Langlotz answered that AI would change radiology rather than replace radiologists: **"radiologists who use AI will replace radiologists who don't."**

AI tends to take over **tasks**, not whole **jobs**. A clinical job is a bundle of tasks, such as examining, interpreting, communicating, coordinating and documenting, and only some of them are suited to automation.

### Changes to work

| Change | Example |
|---|---|
| **Less clerical work** | Ambient AI scribes drafting clinical notes (Session 5) |
| **New work** | Reviewing AI proposals; checking AI-drafted notes; local validation and monitoring |
| **New roles** | Clinical informaticians; members of AI governance committees who choose, monitor and retire tools |
| **Task shifting** | Work moving between staff groups, e.g. AI-supported screening by non-specialists |

### Deskilling: early evidence

At four Polish endoscopy centres, AI tools for detecting polyps were introduced at the end of 2021 (Budzyń et al., 2025). The study compared colonoscopies done **without** AI by the same centres, 3 months before and 3 months after AI arrived:

| Period | Adenoma detection rate, colonoscopies without AI |
|---|---:|
| Before AI was introduced | 28.4% (226 of 795) |
| After AI was introduced | 22.4% (145 of 648) |

The **adenoma detection rate** is the share of colonoscopies that find at least one adenoma (a polyp that can become cancer); it is a standard quality measure. The fall suggests that regular AI use may have changed how endoscopists worked when AI was absent. This was an **observational** before–after study, so it cannot prove cause, but it is an early signal worth monitoring.

**Workforce implications:** keep training and assessing core skills; plan for the AI being unavailable; count the new review work in staffing.

## 6. Human-centred design principles

**Human-centred design** builds a system around the people who use it and are affected by it, involving them throughout rather than at the end.

WHO's 2021 guidance on the ethics and governance of AI for health sets out **six principles**:

| WHO principle | What it means for the inbox agent |
|---|---|
| Protecting human autonomy | Staff make every decision; patients know how their messages are handled |
| Promoting human well-being and safety and the public interest | Red-flag screening; evaluation before use; monitoring after |
| Ensuring transparency, explainability and intelligibility | The reviewer sees the trace and reasons; the clinic documents what the agent does |
| Fostering responsibility and accountability | Named reviewers; an audit trail; a way to question and fix decisions |
| Ensuring inclusiveness and equity | Works for Hindi–English messages and for patients with less digital access |
| Promoting AI that is responsive and sustainable | Reviewed and updated as the clinic changes; reasonable compute use |

Practical steps that follow:

1. **Involve users early:** reception staff, nurses and patients, not only clinicians and engineers.
2. **Study the real workflow** before building, as in the Thai retinopathy case (Session 1).
3. **Test with users**, including how people behave when the AI is wrong.
4. **Plan for failure:** what happens when the AI is down, slow or wrong.

## 7. Design principles for AI agents

These principles apply WHO's ideas to a system that acts. Each one is built into the inbox agent.

| Principle | Meaning | In the inbox agent |
|---|---|---|
| **Least privilege** | Give the agent only the access it needs | Tools are locked to the logged-in patient; no tool takes a patient ID |
| **Propose, don't act** | Changes happen only after human approval | Proposal tools change nothing; `execute` runs after the decision |
| **Reversible vs irreversible** | Actions that are hard to undo need more checks | A cancellation is flagged at review; a second approver is an optional project extension |
| **Escalation and handoff** | Hand over to a person when unsure or at risk | The safety screen routes red flags to the urgent queue with no model involved |
| **Transparent action traces** | Every step can be inspected later | The saved graph checkpoints plus the `audit_log` table |
| **Re-check at execution** | Conditions may change between proposal and approval | `apply` re-checks that the slot is still free |

## 8. Summary

| Idea | Key point |
|---|---|
| Calibrated trust | Trust should match real capability; over-trust → misuse, under-trust → disuse |
| Standard of care | Under current US analysis, following non-standard AI advice that is wrong is the riskiest case |
| Accountability | Developer, organisation, reviewer and regulator each own part; write it down |
| Audit trail | Record input, trace, proposals, what the reviewer saw, the decision and the result |
| Workforce | AI changes tasks more than it removes jobs; early evidence of deskilling |
| WHO principles | Autonomy, safety, transparency, accountability, equity, sustainability |
| Agent design | Least privilege, propose-don't-act, extra checks for irreversible actions, escalation, traces |

**Unit III wrap-up:** Session 1 asked whether the evidence supports a claim. Sessions 4–6 asked how a system that works can still harm, how humans succeed or fail at catching errors, and how to design so that responsibility is clear and failures are contained. Unit IV turns to how such systems are developed, validated and deployed.

## 9. Review questions

1. Give one healthcare example each of misuse and disuse of an AI system. What would calibrated trust look like in each?
2. Under the analysis of Price et al., why might a physician hesitate to follow an AI recommendation that is better than standard care?
3. In the opening scenario (§1), list what each party in the accountability table could have done to prevent the harm.
4. The audit trail records that a nurse approved a harmful cancellation. Why is it also important to record what flags the nurse was shown?
5. Why can the Budzyń et al. study not prove that AI caused deskilling? Suggest a study design that could test it better.
6. Pick two WHO principles and propose one concrete change to the inbox agent for each.
7. Classify these inbox actions as reversible or hard to reverse, and suggest how much checking each needs: sending a reply, routing to a queue, booking a slot, cancelling an appointment.

## 10. References

- Lee JD, See KA. Trust in automation: designing for appropriate reliance. *Human Factors*. 2004;46(1):50–80. [doi:10.1518/hfes.46.1.50_30392](https://doi.org/10.1518/hfes.46.1.50_30392)
- Price WN II, Gerke S, Cohen IG. Potential liability for physicians using artificial intelligence. *JAMA*. 2019;322(18):1765–1766. [doi:10.1001/jama.2019.15064](https://doi.org/10.1001/jama.2019.15064)
- Langlotz CP. Will artificial intelligence replace radiologists? *Radiology: Artificial Intelligence*. 2019;1(3):e190058. [doi:10.1148/ryai.2019190058](https://doi.org/10.1148/ryai.2019190058)
- Budzyń K, Romańczyk M, Kitala D, et al. Endoscopist deskilling risk after exposure to artificial intelligence in colonoscopy: a multicentre, observational study. *Lancet Gastroenterology & Hepatology*. 2025;10(10):896–903. [doi:10.1016/S2468-1253(25)00133-5](https://doi.org/10.1016/S2468-1253(25)00133-5)
- World Health Organization. *Ethics and governance of artificial intelligence for health: WHO guidance*. Geneva: WHO; 28 June 2021. [Link](https://www.who.int/publications/i/item/9789240029200)
- Tierney AA, Gayre G, Hoberman B, et al. Ambient artificial intelligence scribes to alleviate the burden of clinical documentation. *NEJM Catalyst*. 2024;5(3). [doi:10.1056/CAT.23.0404](https://doi.org/10.1056/CAT.23.0404)
