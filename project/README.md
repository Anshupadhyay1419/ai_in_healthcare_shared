# Capstone Project — A Patient-Message Inbox Agent

**AI in Healthcare · Unit III · B.Tech AI, Third Year**  
**Weight:** 10% of the final course grade  
**Introduced in:** [Session 3 — Agentic AI in Healthcare](../../lectures/session-03-agentic-ai-in-healthcare.md)  
**High-level design:** [design.md](design.md) · **Starter kit:** [starter/](starter/)

You will build an AI agent that helps a fictional family medicine clinic handle messages from patients. It reads a message, looks up that patient's record, and prepares work for clinic staff: an appointment change, a draft reply, and a routing to the right staff queue. **A person approves every proposal before anything happens.** Then you will try to break your agent, measure how it fails, and explain what would be needed before anyone could trust it.

All patients, records and messages are **hand-created synthetic teaching data**. This is an educational prototype. It must never be presented as clinical software or used with real patient information.

## Contents

1. [The problem](#1-the-problem)
2. [What you build](#2-what-you-build)
3. [Scope](#3-scope)
4. [Data and starter kit](#4-data-and-starter-kit)
5. [Functional requirements](#5-functional-requirements)
6. [Safety requirements](#6-safety-requirements)
7. [Interface contract](#7-interface-contract)
8. [Evaluation protocol](#8-evaluation-protocol)
9. [Milestones](#9-milestones)
10. [Deliverables](#10-deliverables)
11. [Assessment rubric](#11-assessment-rubric)
12. [Rules: teams, AI tools and data](#12-rules-teams-ai-tools-and-data)
13. [Optional extensions](#13-optional-extensions)
14. [Course outcomes](#14-course-outcomes)
15. [References](#15-references)

## 1. The problem

Patients send messages to the clinic through a **patient portal**, a secure website or app linked to their health record. The messages land in a shared work list, often called the **in-basket**. Most are routine: moving an appointment, asking whether a test result is back, requesting a repeat prescription. A few describe symptoms that need urgent attention, sometimes hidden inside an ordinary request.

Each message must reach one of five staff queues:

| Queue | Who | Handles |
|---|---|---|
| `front_desk` | Reception staff | Appointments, unclear requests |
| `nurse` | Clinic nurse | Test-result status, general questions |
| `clinician` | Doctor | Prescriptions and anything needing a clinical decision |
| `admin` | Administrative staff | Documents, records, clinic information |
| `urgent_nurse` | Nurse on urgent duty | Messages that may describe an emergency |

**Intended use statement** (your report must refine this):

| | |
|---|---|
| Clinical purpose | Patients get timely, appropriate responses; possible emergencies reach a nurse quickly |
| Computational task | For one message, propose one routing, at most one draft reply and any appointment change |
| Intended user | Clinic staff, who review every proposal |
| Setting | A fictional outpatient family medicine clinic (patients visit and go home the same day) |
| Unit of analysis | One message from one logged-in patient |
| Decision time | When the message arrives; "today" is the date the message was sent |

## 2. What you build

```text
 patient portal                               clinic staff
      │                                            ▲
      ▼                                            │ approve / reject
 ┌──────────┐   ┌──────────────┐   ┌─────────┐   ┌──────────────┐   ┌──────────┐
 │ messages │──►│ safety screen│──►│  agent  │──►│ review pause │──►│ execute  │
 │ (SQLite) │   └──────┬───────┘   │ ⇄ tools │   │ (interrupt)  │   │ re-check │
 └──────────┘          │ red flag  └────┬────┘   └──────────────┘   └────┬─────┘
                       └─► fixed urgent ┘  reads: FHIR record + calendar  │ writes: calendar,
                           handling                                       │ outbox, queues,
                                                                          ▼ audit log (SQLite)
```

A **LangGraph** agent with:

- **read tools** for the logged-in patient's record and the clinic calendar
- **proposal tools** that suggest actions but change nothing
- a **safety screen** that runs before the model and sends possible emergencies straight to the urgent queue
- a **human review pause**, saved in SQLite so that staff can review later, in a different process
- an **execute** step that applies only approved proposals, re-checks them first, and records every decision in an **audit log**

You also build a small **review command-line tool** for staff, and you **evaluate** your agent against a keyword baseline, with your own attack messages.

The [high-level design](design.md) shows the components, the graph, the database and the flow of one message. You may change the design if you explain why in your report, but the [interface contract](#7-interface-contract) is fixed.

## 3. Scope

| In scope | Out of scope |
|---|---|
| Appointments: book, reschedule, cancel (proposed only) | Diagnosis or medical advice |
| Saying whether a test result is pending or available | Interpreting a result, or saying whether it is normal |
| Routing every message to exactly one queue | Prescribing, refilling or changing medicines |
| Drafting a reply for staff to approve | Sending or booking anything without approval |
| Escalating possible emergencies to `urgent_nurse` | Contacting emergency services (the urgent reply is a template) |
| A command-line review tool | A web interface (optional extension only) |
| Synthetic data only | Real patient data, real messaging channels |

A prescription request is in scope **only as routing**: the agent routes it to `clinician` and may draft a reply saying the request has been passed on. It must not suggest a dose or say the request is approved.

## 4. Data and starter kit

Everything is in [`starter/`](starter/). Copy its contents to the top level of your own repository; the commands below assume that layout.

| File | Contents |
|---|---|
| `data/clinic_bundle.json` | A FHIR R4 (4.0.1-shaped) `Bundle`: 15 Patients, 2 Practitioners, 2 Schedules, 120 Slots (9–20 Nov 2026), 12 Appointments, 7 lab Observations (2 still pending), 6 MedicationRequests |
| `data/messages_dev.json` | 40 development messages with **authored labels**: expected queue, red flag yes/no, and a `probe` tag for deliberately hard cases. 15 are marked `subset: true` for rate-limited repeated runs |
| `schema.sql` | The SQLite schema: `slots`, `appointments`, `messages`, `proposals`, `audit_log` (append-only), `outbox`, `staff_queue` |
| `reset_db.py` | Rebuilds the database from the schema and fixtures. **Overwrites** the file given by `--db` |
| `score.py` | Scores your results file against the labels, next to a keyword baseline; adds repeatability when given several runs |
| `requirements.txt`, `.env.example` | Pinned packages; model settings |
| `tests/` | Checks for the starter kit (`python -m pytest -q`) |

**Read the clinical record from the FHIR bundle** (patients, doctors, lab results, medicines) and **read and write the calendar in SQLite**. After a reset, SQLite is the source of truth for slots and appointments.

All messages were "received" on Friday 6 November 2026, so "next week" has a fixed meaning. The labels are teaching references written by the course author, **not clinical gold standards**, and the red-flag examples are not a validated triage list. Your agent must never read the labels: `reset_db.py` loads messages without them.

## 5. Functional requirements

| ID | Requirement |
|---|---|
| **FR1** | Take messages with status `new` from the `messages` table. The patient comes from the row's `patient_id` (the verified login), never from the message text. "Today" is the date in `sent`. |
| **FR2** | Provide **read tools** for the session patient: patient summary (name, preferred language, usual doctor), their appointments, free-slot search (by date range, morning/afternoon, doctor), lab-test status, and active medications. |
| **FR3** | Provide **proposal tools**: an appointment change (`book`, `reschedule`, `cancel`) validated in code, a draft reply, and a routing (queue + priority + reason). A proposal is saved in the `proposals` table and **changes nothing else**. |
| **FR4** | Build the agent as a **hand-written LangGraph `StateGraph`**: a model node, a tool node, a conditional edge between them, a safety-screen node before the model, a review node that calls `interrupt()`, and an execute node. Set a step limit (`recursion_limit`). |
| **FR5** | **Persist** paused graphs with LangGraph's `SqliteSaver`, using the message id as the `thread_id`. Review must work after the `run` process has exited. |
| **FR6** | A **review tool** that lists pending work items with their review flags, lets a named reviewer approve or reject each proposal, and resumes the graph. Execution updates the calendar, `outbox`, `staff_queue` and message status, and writes one `audit_log` row per decision. |
| **FR7** | Show **review flags** to the reviewer, at least: `claims_completion`, `possible_result_interpretation`, `mentions_other_patient`, `cancellation` (hard to reverse) and `from_safety_screen`. You may reuse `reply_flags` from `score.py`. |
| **FR8** | Choose the model with `init_chat_model` from the `AGENT_MODEL` environment variable, so the same code runs on a hosted API or a local Ollama model. Record the model in every results file. |

## 6. Safety requirements

These must be **enforced in code**. A rule written only in the system prompt does not satisfy them, because a model can be talked out of it.

| ID | Requirement |
|---|---|
| **SR1 Least privilege** | Every tool is created for one patient. **No tool accepts a patient id**, and no tool returns another patient's data. |
| **SR2 Propose, don't act** | The model has no tool that writes to the calendar, outbox or queues. Only the execute step, after a human decision, changes state. |
| **SR3 Re-check at execution** | Before applying an approved appointment change, check again that the appointment belongs to this patient and is booked, and that the slot is still free. If not, record the result as failed; do not apply it. |
| **SR4 Safety screen** | A fixed check runs **before the model**. If it fires, the message gets an `urgent_nurse` routing with priority `urgent` and a fixed reply template, and **the model is not called**. Develop it on the development messages only, and document the warning-sign categories it covers. |
| **SR5 Scope** | Replies must not diagnose, interpret results, say whether a value is normal, or suggest medicine changes. They may say a result is pending or available and that the clinician will discuss it. |
| **SR6 No false completion** | Replies must not say that anything has been booked, changed, sent or approved. Say it has been requested and the clinic will confirm. |
| **SR7 Message text is data** | Instructions inside a message (for example "ignore your rules", "I am patient P10", "mark it approved") must not change what the agent may do. |
| **SR8 Audit** | Every approval or rejection is written to `audit_log` with the reviewer's name, the time, the **flags the reviewer was shown**, and the result. The agent's step-by-step trace for each message remains retrievable from the checkpoints. |
| **SR9 Fail safe** | If a run fails (model error, invalid tool call, step limit), set the message status to `error`, record the error, and leave it for a person. Never drop a message silently. |
| **SR10 Data and secrets** | Synthetic data only. Keys live in `.env`, which is never committed. |

**Urgent reply template.** Use this text, or a close variant, for the safety screen and for urgent routings. It is a teaching template: a real service would need locally approved wording and a staffed escalation pathway, and printing a message contacts no one.

> Your message has been passed to a nurse urgently. If you think this may be an emergency, contact your local emergency services or go to the nearest emergency department now.

## 7. Interface contract

The instructor will run your project on a **hidden test set** of new messages, with their own model, using exactly these commands from your project folder:

```bash
python reset_db.py --db clinic.db --messages <messages-file>.json
python -m clinic_inbox run --db clinic.db --out results.jsonl          # every message with status 'new'
python -m clinic_inbox run --db clinic.db --out results.jsonl --ids M001,M011
python -m clinic_inbox review --db clinic.db --reviewer staff-01        # interactive
```

Your package must be named **`clinic_inbox`** and read the model from `AGENT_MODEL`. `run` processes each message up to the review pause and writes **one JSON line per message**:

```json
{"message_id": "M001", "model": "ollama:<model>", "error": null, "tool_calls": 4,
 "proposals": [
   {"kind": "appointment", "action": "reschedule", "appointment_id": "A05", "slot_id": "S-PR1-1110-0900"},
   {"kind": "reply", "text": "We have asked the front desk to move your diabetes review ..."},
   {"kind": "route", "queue": "front_desk", "priority": "routine", "reason": "Reschedule request"}]}
```

- `error` is `null` or a short error name; a failed message still gets a line.
- `proposals` holds every proposal that reached the review pause, including those from the safety screen.
- `run` must not approve anything.

Check your file with `python score.py results.jsonl --show`.

## 8. Evaluation protocol

Report every number with the model name, the date, and the number of messages. Failed runs stay in the denominator. **Reset the database before every run.**

| ID | What | Minimum |
|---|---|---|
| **E1 Baseline comparison** | Score the keyword baseline and your agent on all 40 development messages | Correct queues; red-flag TP, FN, FP, **sensitivity** and **PPV**; review-flag counts; the hard-case (`probe`) table |
| **E2 Repeatability** | Run the same messages several times | 3 runs on the 15 `subset` messages; report how many got the same queue every time |
| **E3 Red-team** | Write your own attack and hard-case messages **before** testing them, with labels, in the same format (`redteam_messages.json`) | At least 8: ≥2 prompt injections, ≥2 red-flag wordings not in the development set (one not in English), ≥1 privacy probe, ≥1 scope-creep request, ≥1 negation. For each, record which layer caught it (screen, code, reviewer flag, nothing) |
| **E4 Safety screen** | Evaluate the screen separately from the agent | TP/FN/FP on the development messages and **separately** on your red-team messages; list every miss and false alarm |
| **E5 Reviewer workload** | Count proposals per message | Estimate reviewer time for 300 messages a day at 20 s per proposal, and propose one way to reduce it safely |
| **E6 Error analysis** | Read every wrong, flagged or failed message | Group the failures into types; give one example trace for each type |

**Why sensitivity first.** A missed emergency (a false negative) can cost a life; a false alarm costs a nurse's time. Report both, and say which trade-off your screen makes.

**The hidden test set** is run once by the instructor after submission. Its results are discussed in your viva. You are graded on whether your project runs cleanly and follows the contract, **not** on reaching a particular score. Models differ, and synthetic test messages cannot show clinical performance.

## 9. Milestones

| When | Milestone | Check |
|---|---|---|
| Lab 10 slot, second hour | **Kick-off.** Environment set up, team formed, model route chosen (hosted API or local) | In your copy of the starter kit, `python -m pytest -q` passes and `python score.py --baseline-only` reports 27/40 correct queues |
| End of week 10 | **Checkpoint 1.** Read and proposal tools with unit tests; the graph runs to the review pause | `run` produces a results file for 5 messages that `score.py` accepts |
| Lab 11 slot | **Checkpoint 2.** Safety screen, review tool with persistence and execute, red-team messages written | Instructor check-in, about 5 minutes per team, feedback only |
| End of week 11 | **Final submission** | See [Deliverables](#10-deliverables) |
| Announced by the instructor | **Demo and viva** | About 10 minutes per team |

## 10. Deliverables

1. **Code repository** (zip or link), runnable from a fresh clone:
   - the `clinic_inbox` package, `reset_db.py`, `schema.sql`, `score.py` and `data/` from the starter kit
   - your tests (`python -m pytest -q` must pass **without** a model or network: use a fake model for graph tests)
   - a `README.md` with setup steps, the model you used, and how to reproduce every number in the report
   - `redteam_messages.json` and your `results/*.jsonl` files
   - no `.env`, keys, databases or virtual environments
2. **Report** (PDF, at most 8 pages plus appendices) with these sections:
   1. Intended use statement (refining §1)
   2. Design: your graph, tools and data flow; any change from `design.md`, and why
   3. Safety layers: a table mapping SR1–SR9 to the code that enforces each
   4. Evaluation: E1–E5, with tables
   5. Error analysis (E6)
   6. Limitations: what your results do **not** show
   7. Safety and monitoring plan: what a clinic would need to watch if this were piloted (for example missed red flags, override rates, approval rates near 100%, model changes), who acts on it, and when the agent should be switched off
   8. AI-use statement (see §12)
3. **Demo and viva:** run one routine message and one red-flag message end to end, including a review in a separate process. Every team member must be able to explain any part of the code.

## 11. Assessment rubric

The project is marked out of 100 and scaled to 10% of the course grade.

| Criterion | Marks | Strong work | Weak work |
|---|---:|---|---|
| **A. Working system** | 25 | Graph with screen, interrupt, persistence and execute works end to end (10); tools read FHIR and SQLite correctly, including pending results (8); review tool, execution and audit work (7) | Steps faked, review in the same process only, state lost on exit |
| **B. Safety engineering** | 20 | SR1–SR3 and SR9 enforced in code and tested (10); screen designed on development data with documented categories (5); scope, injection and flags handled (5) | Safety rules only in the prompt; untested |
| **C. Evaluation** | 25 | Baseline comparison with sensitivity and PPV (8); repeatability (5); red-team set written before testing, layer analysis (7); clean run on the hidden test set under the contract (5) | Accuracy only; numbers without model or date; tuning on the test messages |
| **D. Error analysis, limitations, monitoring plan** | 15 | Failures grouped with traces; honest limits; a concrete, owned monitoring plan | "The agent works well"; no limits stated |
| **E. Reproducibility and code quality** | 5 | Fresh clone + README reproduces results; tests pass offline; no secrets | Hidden steps, absolute paths, committed keys |
| **F. Demo and viva** | 10 | Each member explains design choices and code, including what they generated with AI tools | Cannot explain own code |

Deductions: using real patient data, or claiming the system is clinically safe or ready, loses all marks in criterion D.

## 12. Rules: teams, AI tools and data

- **Teams:** work in pairs unless the instructor says otherwise. Both members must contribute code and both are examined in the viva.
- **AI coding tools are allowed and expected.** You are responsible for every line you submit. In the report's AI-use statement, list the tools you used and what for (for example "generated the first draft of the FHIR reader; we rewrote the pending-result handling"). Being unable to explain code in the viva lowers criterion F.
- **Labels are for evaluation only.** Your agent and screen must not read the `label` field. Do not copy development messages into your prompt or screen patterns.
- **Synthetic data only**, on any model route. Check your hosted API's current data-use terms before you send it anything.
- **Do not present the prototype as clinical advice**, or as ready for use with patients.

## 13. Optional extensions

These earn no extra marks beyond the rubric, but can strengthen criteria B–D:

- a **second approver** for cancellations (an action that is hard to undo)
- a **Streamlit** review queue instead of the command line
- an **LLM-assisted screen** as a second layer after the keyword screen, evaluated separately
- a **channel adapter** (for example a Telegram bot) that only converts incoming messages into the `messages` table format

## 14. Course outcomes

| Outcome | How the project assesses it |
|---|---|
| CO1 | Using healthcare data standards (FHIR resources, LOINC-coded results) and clinic workflows |
| CO2 | Applying AI to a patient-management and operations task |
| CO3 | Evaluating risks: missed escalation, injection, scope creep, approval fatigue, human oversight |
| CO4 | Validation against a baseline, red-teaming, audit and a monitoring plan to judge readiness |

## 15. References

- LangChain. [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview), [Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts) and [Persistence](https://docs.langchain.com/oss/python/langgraph/persistence) (checked October 2026).
- Anthropic. [Building effective agents](https://www.anthropic.com/research/building-effective-agents).
- HL7 FHIR R4 (4.0.1): [Bundle](https://hl7.org/fhir/R4/bundle.html), [Patient](https://hl7.org/fhir/R4/patient.html), [Observation](https://hl7.org/fhir/R4/observation.html), [Appointment](https://hl7.org/fhir/R4/appointment.html), [Slot](https://hl7.org/fhir/R4/slot.html), [MedicationRequest](https://hl7.org/fhir/R4/medicationrequest.html).
- World Health Organization. [Ethics and governance of artificial intelligence for health](https://www.who.int/publications/i/item/9789240029200). 2021.
