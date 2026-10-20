# High-Level Design — Patient-Message Inbox Agent

**Capstone project · AI in Healthcare · Unit III**  
**Project brief:** [README.md](README.md) · **Lecture:** [Session 3](../../lectures/session-03-agentic-ai-in-healthcare.md)

This document describes one design that meets the [requirements](README.md#5-functional-requirements). You may change internal details if your report explains why. The [interface contract](README.md#7-interface-contract) and the [safety requirements](README.md#6-safety-requirements) are fixed.

## Contents

1. [Design principles](#1-design-principles)
2. [Components](#2-components)
3. [Suggested project layout](#3-suggested-project-layout)
4. [The agent graph](#4-the-agent-graph)
5. [Data: FHIR record and SQLite](#5-data-fhir-record-and-sqlite)
6. [The life of one message](#6-the-life-of-one-message)
7. [Pausing and resuming with SqliteSaver](#7-pausing-and-resuming-with-sqlitesaver)
8. [Testing without a model](#8-testing-without-a-model)
9. [Practical notes](#9-practical-notes)
10. [What not to build](#10-what-not-to-build)

## 1. Design principles

The whole design follows from one idea: **the model chooses steps; code and people control consequences.**

| Principle | Where it appears |
|---|---|
| The patient comes from the login, not the message | Tools are created for one `patient_id` and take no patient argument |
| Propose, don't act | Proposal tools only validate and record; only `execute` changes the clinic's state |
| A fixed layer before the model | The safety screen can route to `urgent_nurse` without any model call |
| A person decides | `interrupt()` pauses the graph; a named reviewer approves or rejects each proposal |
| Conditions can change while waiting | `execute` re-checks every appointment change before applying it |
| Everything can be inspected later | Checkpoints keep the trace; `audit_log` keeps decisions and what the reviewer saw |

## 2. Components

| Component | Responsibility | Reads | Writes |
|---|---|---|---|
| **FHIR reader** (`fhir.py`) | Load the bundle once; answer questions about patients, doctors, lab results, medicines | `data/clinic_bundle.json` | nothing |
| **Clinic database** (`db.py`) | Open connections; queries and the few writes on the calendar, proposals, audit log, outbox and queues | `clinic.db` | `clinic.db` |
| **Tools** (`tools.py`) | `make_tools(patient_id, ...)` returns read tools and proposal tools bound to one patient | FHIR reader, database | `proposals` (via the record step) |
| **Safety screen** (`screen.py`) | `screen(text) -> bool` from documented warning-sign patterns | message text | nothing |
| **Graph** (`graph.py`) | Build the `StateGraph`: screen → agent ⇄ tools → record → review → execute | all of the above | via `db.py` |
| **Review flags** (`flags.py`) | Compute flags for each proposal (may reuse `reply_flags` from `score.py`) | proposals, FHIR reader | nothing |
| **Command line** (`__main__.py`) | `run` and `review` commands from the contract | database, graph | results JSONL |

## 3. Suggested project layout

```text
your-repo/
  clinic_inbox/
    __init__.py
    __main__.py        # python -m clinic_inbox run | review
    fhir.py
    db.py
    tools.py
    screen.py
    flags.py
    graph.py
    prompts.py         # the system prompt, kept in one place
  tests/
    fakes.py           # a scripted fake chat model for offline tests
    test_tools.py
    test_graph.py
    test_screen.py
  data/                # from the starter kit
  schema.sql  reset_db.py  score.py  requirements.txt  .env.example
  redteam_messages.json
  results/             # your JSONL runs (small; commit them)
  README.md
```

## 4. The agent graph

```text
                   ┌────────────── flagged ──────────────┐
                   │                                     ▼
 START ──► screen ─┤                                   urgent ──┐
                   │                                  (fixed    │
                   └─ not flagged ─► agent ◄────┐      proposals)│
                                       │        │                │
                          tool calls?  ├─ yes ─► tools           │
                                       │                         │
                                       └─ no ──► record ◄────────┘
                                                   │
                                                   ▼
                                     review  (interrupt: waits for a person)
                                                   │  resumed with a decision
                                                   ▼
                                                execute ──► END
```

| Node | What it does |
|---|---|
| `screen` | Runs `screen(message_text)`; sets `flagged` |
| `urgent` | Adds two fixed proposals: route to `urgent_nurse` (priority `urgent`) and the urgent reply template. **No model call** |
| `agent` | Calls the chat model with the system prompt, the message and the tool results so far |
| `tools` | Runs the tool calls the model asked for (LangGraph's `ToolNode` is fine) |
| `record` | Collects proposals from tool results (and from `urgent`), computes flags, and inserts them into `proposals`; sets the message to `awaiting_review` |
| `review` | Calls `interrupt({...proposal ids...})` and nothing else |
| `execute` | For each proposal: if approved, re-check and apply; write `audit_log`; update the message to `done` |

**Why `record` is separate from `review`.** When a paused graph is resumed, the node that called `interrupt()` **runs again from its first line**. If `review` also inserted the proposals, they would be inserted twice. Nodes that have already finished, such as `record`, do not run again.

**State** (a `MessagesState` subclass is a good start):

| Field | Meaning |
|---|---|
| `messages` | The conversation: the patient's message, model replies, tool results |
| `message_id`, `patient_id`, `today` | Fixed for the run; `today` comes from `sent` |
| `flagged` | Whether the safety screen fired |
| `proposals` | Fixed proposals from `urgent` (the agent's come from tool results) |
| `proposal_ids` | Rows written by `record` |
| `decision` | `{"reviewer": ..., "approve": [ids], "reject": [ids]}` from the resume |

**Routing rule:** the model must call `route_to_staff` exactly once. Decide what `record` does if it calls it zero or two times. One safe choice is to add a `front_desk` routing with reason "agent did not route; needs a person", and a review flag.

## 5. Data: FHIR record and SQLite

### The FHIR bundle (read-only)

A FHIR `Bundle` is a JSON object whose `entry` list holds **resources**. Each resource has a `resourceType` and an `id`. Resources point to each other with references such as `"Patient/P07"`.

| Resource | Used for | Fields you need |
|---|---|---|
| `Patient` | Patient summary | `name[0].given[0]`, `name[0].family`, `communication[0].language.text`, `generalPractitioner[0].reference` |
| `Practitioner` | Doctor names | `name[0]` (`PR1` Dr Meera Iyer, `PR2` Dr Arjun Mehta) |
| `Observation` | Lab-test status | `subject.reference`, `code.coding[0]` (LOINC), `status`, `effectiveDateTime`, `valueQuantity` |
| `MedicationRequest` | Active medicines | `subject.reference`, `status`, `medicationCodeableConcept.text`, `authoredOn` |

A **pending** result looks like this (abridged from the bundle):

```json
{"resourceType": "Observation", "id": "O03", "status": "registered",
 "code": {"coding": [{"system": "http://loinc.org", "code": "718-7",
                      "display": "Hemoglobin [Mass/volume] in Blood"}]},
 "subject": {"reference": "Patient/P05"}, "effectiveDateTime": "2026-11-05"}
```

There is **no `valueQuantity`**: the result does not exist yet. It is not zero and not normal. Your tool should return `"pending"`, not a number. `O03` is a **record identifier** for this one test; `718-7` is a **terminology code** (LOINC) for the kind of test.

`Slot`, `Schedule` and `Appointment` resources are in the bundle too, but `reset_db.py` copies them into SQLite. Use the database for the calendar.

### The SQLite database

`schema.sql` defines seven tables. `reset_db.py` fills the first three.

| Table | Rows | Changed by |
|---|---|---|
| `slots` | 120 half-hour slots, `free` or `busy` | `execute` only |
| `appointments` | 12 at the start (11 booked, 1 cancelled) | `execute` only |
| `messages` | One per patient message, **without labels** | `record` (`awaiting_review`), `execute` (`done`), errors (`error`) |
| `proposals` | Every proposal, its JSON payload, source and flags | `record` (insert), `execute` (status) |
| `audit_log` | One row per human decision; **append-only** (triggers block update and delete) | `execute` |
| `outbox` | Approved replies (never actually sent) | `execute` |
| `staff_queue` | Approved routings | `execute` |

Status changes:

```text
messages:   new ──► awaiting_review ──► done            (or ──► error)
proposals:  pending ──► rejected
            pending ──► approved ──► applied             (re-check passed)
                                 └─► failed              (re-check failed: slot taken, not this patient's...)
```

When `execute` approves and applies a booking, it must in **one transaction**: set the slot to `busy`, insert or update the appointment, set the old slot back to `free` for a reschedule, update the proposal, and write the audit row. A half-applied change is worse than none.

## 6. The life of one message

Patient P07 writes: *"Can I move my diabetes review on 12 Nov? Mornings are better for me."*

1. `run` reads `M001` (status `new`) and starts a graph with `thread_id = "M001"`, `patient_id = "P07"` and `today = 2026-11-06 (Friday)`.
2. `screen` finds no warning signs.
3. `agent` asks for `get_appointments()`. The tool returns A05, a diabetes review on 12 Nov at 15:00.
4. `agent` asks for `search_slots(date_from="2026-11-09", date_to="2026-11-20", time_of_day="morning")`.
5. `agent` calls `propose_appointment_change(action="reschedule", appointment_id="A05", slot_id=...)`. Code checks that A05 belongs to P07 and the slot is free, then returns the proposal.
6. `agent` calls `draft_reply(...)` and `route_to_staff(queue="front_desk", ...)`, then stops calling tools.
7. `record` writes three proposals with their flags; M001 becomes `awaiting_review`. `review` calls `interrupt()`. `run` writes the JSON line and exits. **Nothing in the calendar has changed.**
8. Later, `review` (a new process) lists M001's three proposals. `staff-01` approves all three.
9. The graph resumes. `execute` re-checks the slot, moves A05 in one transaction, puts the reply in `outbox` and the routing in `staff_queue`, writes three `audit_log` rows, and marks M001 `done`.

## 7. Pausing and resuming with SqliteSaver

`SqliteSaver` (package `langgraph-checkpoint-sqlite`) saves the graph's state after every step, keyed by `thread_id`. A paused graph can be resumed from another process.

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.types import Command

con = sqlite3.connect("clinic.db", check_same_thread=False)   # SqliteSaver uses a lock internally
graph = builder.compile(checkpointer=SqliteSaver(con))         # creates its own tables on first use

config = {"configurable": {"thread_id": "M001"}, "recursion_limit": 20}
result = graph.invoke(initial_state, config)       # stops at interrupt(); result["__interrupt__"] holds the payload

# ... later, in another process, after compiling the graph the same way:
graph.get_state({"configurable": {"thread_id": "M001"}}).next     # ('review',) while waiting
graph.invoke(Command(resume={"reviewer": "staff-01", "approve": [1, 2, 3], "reject": []}),
             {"configurable": {"thread_id": "M001"}})
```

Notes:

- The value passed to `Command(resume=...)` becomes the **return value of `interrupt()`** inside `review`.
- The review tool can find pending work from the `proposals` table (status `pending`) instead of scanning checkpoints.
- Keep `SqliteSaver`'s tables in the same `clinic.db`. One file means one reset and one thing to inspect.
- `GraphRecursionError` is raised when the step limit is reached. Catch it in `run`, mark the message `error`, and write the JSON line (SR9).

## 8. Testing without a model

Your tests must pass offline. Write a **scripted fake chat model**: a `BaseChatModel` subclass whose `_generate` returns hand-written `AIMessage` objects (with `tool_calls`) in order, and whose `bind_tools` returns itself. With it you can test that:

- no tool has a `patient_id` argument, and tools only ever return the session patient's data
- a proposal for another patient's appointment, or a busy slot, is refused in code
- the graph pauses before anything changes; approving applies, rejecting leaves the calendar alone
- a slot taken by someone else while a proposal waits makes `execute` record `failed`
- a red-flag message never reaches the model
- a looping model stops at the step limit and leaves the message as `error`

A fake model shows the control flow, not how a real model behaves. Model behaviour is what the evaluation in the brief (E1–E4) measures.

## 9. Practical notes

- **Temperature 0** reduces, but does not remove, run-to-run variation.
- **Dates and times:** keep the ISO strings with their `+05:30` offset; compare dates as `YYYY-MM-DD` strings. Give the model "today" with the weekday, for example `2026-11-06 (Friday)`.
- **Slow disks:** each SQLite commit forces a disk sync, which can take over a second on some machines. Group related writes in one transaction. `SqliteSaver` switches the file to WAL mode, which helps.
- **Rate limits:** a message needs about 3–8 model calls. Use the 15-message `subset` for repeated runs on a free API tier, and pause between messages if needed.
- **Local models** differ a lot in tool-calling reliability. Try your model on a few messages in the first days, and change model early rather than late.
- **Keep prompts in one file** and record which prompt version produced each results file.

## 10. What not to build

These cost time and earn no marks: a login system, real messaging, a web interface (unless as the optional extension), a general medical chatbot, retrieval over medical literature, fine-tuning, or any clinical advice. Keep the agent narrow; the marks are for safety engineering and honest evaluation.
