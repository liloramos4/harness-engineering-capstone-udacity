# 🏛️ ARCHITECTURE — Harness Engineering Capstone

**Full-size diagrams, data contracts and decision records for Systems 1 and 2.**

> This document is the *engineering annex*. The [README](README.md) is the executive dossier
> (what was built, what it measured, what it proves). This file is where the wiring is drawn at
> full resolution: runtime topology, state machines, data contracts, failure taxonomies and the
> Architecture Decision Records (ADRs) that explain **why** each contract exists.
>
> Every number here is copied from an artifact in `evidence/`. Where an artifact does not record
> a quantity, this document says so instead of deriving it — an estimate presented as a measurement
> is the one thing a harness engineer is never allowed to ship.

---

## 0. How to read this document

| If you want… | Go to |
|---|---|
| The three-layer model behind both systems | [§1](#1-the-three-layer-stack) |
| How System 1 spends a turn, byte by byte | [§2.1](#21-runtime-topology)–[§2.4](#24-the-tool-lattice) |
| Why the loop can never hang or lie | [§2.2](#22-the-control-loop-full) |
| The claim lifecycle and its terminal guarantee | [§2.6](#26-claim-lifecycle) |
| How System 2 sheds 21,807 tokens without losing answers | [§3.1](#31-data-flow-with-token-annotations)–[§3.4](#34-token-waterfall) |
| The ablation that makes 6/6 mean something | [§3.5](#35-evaluation-and-ablation) |
| The decisions, stated as ADRs | [§4.3](#43-architecture-decision-records) |
| What could still be wrong | [§6](#6-threats-to-validity) |

---

## 1. The three-layer stack

Both systems are the same architecture with different payloads: a **model** that judges, a
**harness** that guarantees, and an **orchestration** layer that persists. The capstone's whole
argument is that reliability comes from the boundary between these layers being explicit.

```mermaid
flowchart TB
    subgraph L3["🧭 LAYER 3 — Orchestration and persistence"]
        direction LR
        O1["run.py — fixture / run lifecycle"]
        O2["runs/<id>/ — artifacts on disk"]
        O3["Budget — token and wall-clock ceiling"]
        O4["Tracer — one JSONL line per call"]
    end

    subgraph L2["🛡️ LAYER 2 — Harness (deterministic, testable without a model)"]
        direction LR
        H1["loop.py — stop_reason control"]
        H2["tools.py — schemas, dispatch, error envelope"]
        H3["session.py — terminal_called, case file"]
        H4["assemble.py — section order contract"]
        H5["pruner.py — 5 fixed fields, no LLM"]
        H6["case_facts.py — 12 fields, nulls rejected"]
    end

    subgraph L1["🧠 LAYER 1 — Model (non-deterministic by nature)"]
        direction LR
        M1["classify and assess severity"]
        M2["choose tool and arguments"]
        M3["summarise a resolved segment"]
        M4["extract 12 case facts as JSON"]
    end

    L1 -->|"tool_use blocks / JSON"| L2
    L2 -->|"tool_result / schema violation"| L1
    L2 --> L3
    L3 -->|"artifacts a reviewer can re-derive"| L2
```

| Boundary | Direction | Contract |
|---|---|---|
| Model → Harness | inbound | The model may only speak in `tool_use` blocks or a final message. There is no free-form control channel. |
| Harness → Model | outbound | Tool results are JSON, and failures are **typed** (`is_error`, `error_category`, `is_retryable`). |
| Harness → Orchestration | downward | Every decision leaves an artifact: a trace line, a queue record, an escalation, a budget. |
| Orchestration → Harness | upward | Budgets and run directories bound what the harness may do; the harness may not exceed them. |

---

# 2. SYSTEM 1 — Claims Intake Agent

## 2.1 Runtime topology

```mermaid
flowchart LR
    subgraph PROC["🖥️ One process — python -m claims_intake.run"]
        direction TB
        MAIN["main()<br/>argparse --all or --fixture"]
        LOOP["run_loop()<br/>while True"]
        EXEC["make_executor(session)<br/>7 handlers"]
        BUD["Budget.check()<br/>before every call"]
        TRC["Tracer<br/>append + close"]
    end

    subgraph NET["☁️ Network"]
        API["Claude Messages API<br/>claude-haiku-4-5-20251001"]
    end

    subgraph DISK["💾 Artifacts"]
        A1["runs/<timestamp>/summary.md"]
        A2["runs/<timestamp>/traces/<claim>.jsonl"]
        A3["runs/<timestamp>/queues/<type>.jsonl"]
        A4["runs/<timestamp>/escalations.jsonl"]
    end

    subgraph CFG["📄 Read-only inputs"]
        C1["data/policies.json"]
        C2["fixtures/claims/claim_*.json"]
        C3["system_prompt.py — SYSTEM_PROMPT"]
    end

    C1 --> MAIN
    C2 --> MAIN
    C3 --> LOOP
    MAIN --> LOOP
    LOOP --> BUD
    BUD -->|"raises BudgetExceeded"| MAIN
    LOOP -->|"messages.create"| API
    API -->|"stop_reason + content"| LOOP
    LOOP --> TRC
    TRC --> A2
    LOOP -->|"tool_use blocks"| EXEC
    EXEC -->|"queues / escalations"| A3
    EXEC --> A4
    MAIN --> A1
```

**Default envelopes** (from `run.py` argparse): `--max-input-tokens 500000`,
`--max-wall-clock-s 180.0`, loop `max_tokens=4096`, per-claim `Budget` instance.

## 2.2 The control loop, full

```mermaid
stateDiagram-v2
    [*] --> BudgetCheck
    BudgetCheck --> CallModel: under ceiling
    BudgetCheck --> [*]: BudgetExceeded

    CallModel --> Record
    Record --> Dispatch: stop_reason = tool_use
    Record --> Return: stop_reason = end_turn
    Record --> Abort: any other stop_reason

    state Dispatch {
        [*] --> ForEachBlock
        ForEachBlock --> ExecuteTool: next tool_use block
        ExecuteTool --> AppendResult
        AppendResult --> ForEachBlock: more blocks
        AppendResult --> [*]: all blocks answered
    }

    Dispatch --> CallModel: assistant block + tool_result blocks
    Return --> [*]: FinalState
    Abort --> [*]: UnexpectedStopReason
```

Three invariants the diagram encodes, each of which is a test:

1. **Exactly one exit per call.** Every `CallModel` produces one `Record`; `Record` has three and
   only three outgoing edges. `test_loop.py` asserts the raise path, not just the happy path.
2. **`BudgetCheck` precedes `CallModel`.** The ceiling is enforced *before* spending, so
   `BudgetExceeded` is never raised after an irreversible API call.
3. **The loop has no counter.** There is no `turn < N` edge anywhere in the graph — termination is
   semantic (`end_turn`) or exceptional (`BudgetExceeded`, `UnexpectedStopReason`).

`Record` is where observability happens: the tracer line is written **before** branching, so a run
that dies in `Dispatch` still has a trace line for the call that produced it.

## 2.3 One turn, in wire order

```mermaid
sequenceDiagram
    autonumber
    participant CLI as 🖥️ run.py
    participant L as 🛡️ loop.py
    participant B as 📏 Budget
    participant API as ☁️ Messages API
    participant D as 🧰 dispatcher
    participant S as 🗂️ ClaimSession
    participant F as 💾 disk

    CLI->>L: run_loop(client, tools, messages, budget, tracer)
    loop until end_turn or raise
        L->>B: check()
        alt over ceiling
            B-->>L: raise BudgetExceeded
        end
        L->>API: messages.create(model, system, tools, messages)
        API-->>L: response
        L->>F: tracer.write(turn, stop_reason, tool_calls, latency, tokens)
        L->>B: record_input_tokens(n)
        alt stop_reason = tool_use
            loop for each tool_use block
                L->>D: execute(name, input)
                D->>S: read or mutate session state
                S-->>D: state
                D->>F: append JSONL if terminal
                D-->>L: JSON string — payload or is_error envelope
            end
            L->>API: append assistant block + tool_result blocks
        else stop_reason = end_turn
            L-->>CLI: FinalState(turns, tokens, content)
        else anything else
            L-->>CLI: raise UnexpectedStopReason
        end
    end
```

Note the two details that matter in production: the tracer write happens **before** the branch (so
control-flow evidence survives a crash), and the dispatcher **never raises** into the loop — a
handler exception is converted to a `transient` envelope inside `make_executor`.

## 2.4 The tool lattice

```mermaid
flowchart TB
    subgraph PH1["1️⃣ DISCOVERY"]
        T1["lookup_policy(policy_id)"]
    end
    subgraph PH2["2️⃣ EVIDENCE"]
        T2["record_claim_fact(field, value)"]
    end
    subgraph PH3["3️⃣ JUDGEMENT"]
        T3["classify_claim(claim_type, confidence, rationale)"]
        T4["assess_severity(severity, rationale)"]
        T5["request_clarification(question, ambiguity_between)"]
    end
    subgraph PH4["4️⃣ TERMINAL — at most one, enforced by session.terminal_called"]
        T6["route_to_adjuster(queue, claim_summary)"]
        T7["escalate_to_human(reason, structured_summary[6])"]
    end

    T1 --> T2
    T2 --> T3
    T3 --> T4
    T3 --> T5
    T5 --> T3
    T4 --> T6
    T4 --> T7
    T3 --> T7
    T6 --> OUT1["queues/<claim_type>.jsonl"]
    T7 --> OUT2["escalations.jsonl"]
```

| Slot | Precondition enforced in code | Failure if violated |
|---|---|---|
| `route_to_adjuster` | `classification != None` **and** `severity != None` | `permanent` error: `classify_claim must be called before routing` |
| `route_to_adjuster` | `queue ∈ {property_damage, theft, liability, auto}` | `permanent` error listing the enum |
| `escalate_to_human` | `structured_summary` has **all 6** keys | `permanent` error naming the missing keys |
| either terminal | `terminal_called == False` | `permanent` error: `terminal tool already called this claim` |

The two terminal tools are deliberately **not** interchangeable:
`route_to_adjuster` asserts *"I know the answer"*, `escalate_to_human` asserts *"I know I don't"*.
Escalation carries a `confidence` **and** `candidate_claim_types`, so a human reviewer receives the
model's uncertainty as data rather than as prose.

## 2.5 Error decision tree

```mermaid
flowchart TD
    E["tool call arrives"] --> Q1{"handler exists?"}
    Q1 -->|no| P1["permanent / not retryable<br/>unknown tool"]
    Q1 -->|yes| Q2{"schema valid?"}
    Q2 -->|no| P2["permanent / not retryable<br/>invalid argument"]
    Q2 -->|yes| Q3{"business precondition?"}
    Q3 -->|no| P3["permanent / not retryable<br/>e.g. terminal already called"]
    Q3 -->|yes| Q4{"handler raised?"}
    Q4 -->|yes| TR["transient / retryable<br/>TypeError: ..."]
    Q4 -->|no| OK["success payload"]

    P1 --> ENV["envelope<br/>is_error true<br/>error_category permanent<br/>is_retryable false"]
    P2 --> ENV
    P3 --> ENV
    TR --> ENV2["envelope<br/>error_category transient<br/>is_retryable true"]
    ENV --> MODEL["model selects a NEW action"]
    ENV2 --> MODEL2["model may REPEAT the call"]
    OK --> MODEL3["model continues the plan"]
```

This tree is the reason the tool layer can be tested without a model: every leaf is reachable by
calling `make_executor(session)` directly. All 12 cases in `test_tools.py` are leaves of this graph.

## 2.6 Claim lifecycle

```mermaid
stateDiagram-v2
    [*] --> Intake
    Intake --> Evidence: record_claim_fact
    Evidence --> Evidence: more facts
    Evidence --> Classified: classify_claim
    Classified --> Clarified: request_clarification
    Clarified --> Classified: claimant reply or NO_RESPONSE
    Classified --> Severity: assess_severity
    Severity --> Routed: route_to_adjuster
    Severity --> Escalated: escalate_to_human
    Classified --> Escalated: confidence below 0.6
    Intake --> Escalated: BudgetExceeded or UnexpectedStopReason
    Routed --> [*]
    Escalated --> [*]

    note right of Routed
        terminal_called = True
        outcome = routed
    end note
    note right of Escalated
        terminal_called = True
        outcome = escalated
        written by the SAME tool
        the model would have used
        backstop path is uniform
    end note
```

The important structural detail: `Escalated` has a **backstop ingress** (`Intake → Escalated`) that
the model cannot reach. The harness can always terminate a claim; the model cannot always be
trusted to. That asymmetry is the design.

## 2.7 Data contracts

```mermaid
classDiagram
    class ClaimSession {
        +str claim_id
        +str policy_id
        +Path run_dir
        +dict policies
        +dict clarification_responses
        +dict case_facts
        +dict classification
        +dict severity
        +bool terminal_called
        +dict routing
        +dict escalation
        +list clarifications_asked
        +str outcome
    }

    class Budget {
        +int max_input_tokens
        +float max_wall_clock_s
        +float _start
        +int input_tokens_used
        +record_input_tokens(n)
        +check()
    }

    class FinalState {
        +list messages
        +int total_input_tokens
        +int total_output_tokens
        +int turn_count
        +list final_content
    }

    class ToolExecutor {
        <<interface>>
        +execute(name, input) str
    }

    ClaimSession <-- ToolExecutor : bound per claim
    ClaimSession --> Budget : one per fixture
    ClaimSession ..> FinalState : one per fixture
```

`FinalState` is the **only** thing the loop returns, and it deliberately contains no outcome
verdict. Whether the claim was decided is a property of `ClaimSession`, evaluated by the harness
after the loop — because the loop must not be allowed to grade its own homework.

## 2.8 Artifact DAG

```mermaid
flowchart LR
    LOOP["loop.py"] --> TRACE["traces/<claim>.jsonl<br/>1 line per turn"]
    TOOLS["tools.py"] --> QUEUES["queues/<type>.jsonl"]
    TOOLS --> ESC["escalations.jsonl"]
    RUN["run.py"] --> SUM["summary.md<br/>one row per claim"]
    RUN -->|"backstop"| TOOLS
    LOOP -->|"FinalState tokens"| RUN
    TRACE -.->|"reconciles with"| SUM
    ESC -.->|"reconciles with"| SUM
    NOTE["summary.md is derived,<br/>the trace is primary.<br/>2,809 + 3,567 = 6,376 in<br/>400 + 87 = 487 out"]
    TRACE -.-> NOTE
```

---

# 3. SYSTEM 2 — Retail Support Context Strategy

## 3.1 Data flow with token annotations

```mermaid
flowchart TB
    RAW["data/transcript_48turns.json<br/>48 turns · 3 issues · 38,708 tok baseline"]
    TOOL["data/lookup_order_response.json<br/>57-field tool payload"]

    subgraph SEG["Partition — pure Python, no model"]
        P["transcript.load()<br/>group by issue_id<br/>first-appearance order"]
        S1["Segment refund — resolved"]
        S2["Segment subscription — resolved"]
        S3["Segment payment_update — active"]
    end

    subgraph EXTRACT["Extraction — model, schema-constrained"]
        CF["case_facts.extract()<br/>1 call · max_tokens 2048<br/>strict JSON · 12 fields"]
    end

    subgraph PRUNE["Pruning — zero tokens"]
        PR["prune_lookup_order()<br/>57 fields to 5"]
    end

    subgraph COMPRESS["Compression — model, paid"]
        C1["summarize_segment refund<br/>12,334 in / 409 out"]
        C2["summarize_segment subscription<br/>11,475 in / 491 out"]
    end

    subgraph ASSEMBLE["Assembly — deterministic order contract"]
        A["assemble.build()"]
    end

    subgraph PROVE["Proof"]
        B["budget.json<br/>38,708 to 16,901 · 56.34%"]
        EV["eval.jsonl<br/>6 of 6 pass"]
        CT["eval_control.jsonl<br/>Case Facts stripped · Q6 fails"]
    end

    RAW --> P
    P --> S1
    P --> S2
    P --> S3
    RAW --> CF
    TOOL --> PR
    PR --> CF
    CF --> A
    S1 --> C1
    S2 --> C2
    S3 -->|"refused by design"| A
    C1 --> A
    C2 --> A
    A --> B
    A --> EV
    A --> CT
```

Two edges carry the design intent: `S3 → A` is labelled **refused by design** (the compressor raises
if asked to compress the active segment), and `PR → CF` shows pruning happening *before* extraction,
so the extractor never sees 57 fields of noise.

## 3.2 Segmentation of 48 turns

```mermaid
flowchart LR
    subgraph T["48 turns — chronological"]
        direction LR
        B1["turns 1..n<br/>refund"]
        B2["turns n+1..m<br/>subscription"]
        B3["turns m+1..48<br/>payment_update"]
    end
    B1 --> R1["status = resolved"]
    B2 --> R2["status = resolved"]
    B3 --> R3["status = active"]
    R1 --> C1["compressible"]
    R2 --> C2["compressible"]
    R3 --> K1["verbatim, byte-exact"]
    ACT["transcript.json active_issue_id"] -.->|"exactly one"| R3
```

Segmentation is by `issue_id`, **not** by turn index: `active_turns` searches for the segment whose
status is `active` and raises `RuntimeError` if there is none. Order of first appearance is preserved
by an explicit `order` list, so a dict-iteration change can never reorder the assembly.

## 3.3 Assembly anatomy with caps

```mermaid
flowchart TB
    subgraph CTX["Assembled context — 16,901 tokens, exclusive sections"]
        direction TB
        A1["⬆️ TOP BOUNDARY<br/>Case Facts · 204 tok · cap 600<br/>fixed key order · 12 fields"]
        SEP1["—"]
        A2["MIDDLE<br/>Resolved: Refund inquiry · 422 tok · cap 500"]
        SEP2["—"]
        A3["MIDDLE<br/>Resolved: Subscription cancellation · 504 tok · cap 500"]
        SEP3["—"]
        A4["⬇️ BOTTOM BOUNDARY<br/>Active issue: Payment-method update · 15,789 tok<br/>verbatim · no compression"]
    end
    A1 --> SEP1 --> A2 --> SEP2 --> A3 --> SEP3 --> A4
    WHY["Position rationale<br/>primacy and recency are the high-attention zones.<br/>Durable facts go top. The live turn-by-turn goes bottom,<br/>directly against the new user turn. Only closed narrative<br/>is allowed to be lossy."]
    A3 -.-> WHY
```

The caps are not advisory: `budget.json` reports per-section counts, and
`test_antipatterns.py::test_budget_json_section_counts_sum_consistently` audits the accounting when
run artifacts are present.

## 3.4 Token waterfall

| Stage | Tokens | Recorded where | Precision |
|---|---:|---|---|
| Baseline 48-turn transcript | **38,708** | `budget.json::baseline_tokens` | measured |
| Compression call — refund | 12,334 in → 409 out | `budget.json::compression_api.refund` | measured (includes compressor system prompt) |
| Compression call — subscription | 11,475 in → 491 out | `budget.json::compression_api.subscription` | measured (includes compressor system prompt) |
| Assembled — `case_facts` | **204** | `budget.json::per_section_tokens` | measured |
| Assembled — `resolved_refund` | **422** | idem | measured |
| Assembled — `resolved_subscription` | **504** | idem | measured |
| Assembled — `active` | **15,789** | idem | measured |
| Assembled total | **16,901** | `budget.json::assembled_tokens` | measured on final markdown |
| Reduction | **−56.34 %** | `budget.json::reduction_pct` | derived from two measured values |

> **Deliberately absent: pre-compression segment sizes.** The artifact records what the compressor
> *consumed*, which includes its system prompt — not the raw segment length. Deriving segment size by
> subtracting the prompt would be an estimate dressed as a measurement, so this document reports the
> recorded number and names its composition. The 18-token delta between the sum of sections (16,919)
> and the assembled total (16,901) is likewise reported, not smoothed: each section count includes
> its own heading.

## 3.5 Evaluation and ablation

```mermaid
flowchart TB
    ASM["Assembled context<br/>16,901 tok"]
    ASM --> FULL["eval.jsonl<br/>6 questions · max_tokens 256<br/>pass = expected fragment present<br/>threshold 5 of 6"]
    ASM --> STRIP["evaluate.strip_case_facts()<br/>removes the Case Facts block only"]
    STRIP --> CTRL["eval_control.jsonl<br/>held questions Q1 and Q6"]
    FULL --> R1["6 / 6 pass"]
    CTRL --> R2["Q1 pass · Q6 fail<br/>1 of 2 regression"]
    R1 --> CLAIM1["The reduced context is sufficient"]
    R2 --> CLAIM2["The facts block is necessary<br/>for at least one requirement"]
    CLAIM1 --> V["Both claims<br/>required for the argument"]
    CLAIM2 --> V
```

The control is a **single-component ablation** with held questions. That is what converts *"the model
answered 6/6"* (weak: it might have answered from the verbatim active block) into *"removing this one
block destroys Q6"* (causal). The recorded control answer is the evidence:

> *"[…] there is no structured status token recorded for the payment-method update issue. […] The
> payment-method update issue is labeled as 'Active issue' but no specific status token or case code
> is present in the structured record."*

Q1 survives because the refund amount also lives in the resolved narrative. Q6 dies because
`in_progress` **as a status token** existed only in the facts block. The asymmetry is the finding.

## 3.6 Data contracts

```mermaid
classDiagram
    class Turn {
        +int turn
        +str role
        +str text
        +str issue_id
        +render() str
    }
    class Segment {
        +str issue_id
        +list~Turn~ turns
        +str status
        +text() str
        +turn_range() tuple
    }
    class Transcript {
        +list~Turn~ turns
        +list~Segment~ segments
        +int token_count
        +full_text() str
        +active_turns() list~Turn~
        +segment(issue_id) Segment
    }
    class CaseFacts {
        +str customer_id
        +str refund_order_id
        +float refund_amount_usd
        +str refund_status
        +str subscription_id
        +str subscription_plan
        +str subscription_cancel_reason
        +str subscription_status
        +str active_payment_method_last4
        +str new_payment_method_last4
        +str payment_update_failure_code
        +str payment_update_status
        +to_markdown() str
    }
    class AssembledContext {
        +str markdown
        +str case_facts_block
        +dict resolved_blocks
        +str active_block
        +str active_raw_text
        +section_tokens() dict
        +total_tokens() int
    }
    Transcript "1" *-- "many" Turn
    Transcript "1" *-- "many" Segment
    Segment "1" *-- "many" Turn
    CaseFacts --> AssembledContext : rendered into
    Transcript --> CaseFacts : source for extraction
```

`Transcript.token_count` is lazily cached (`_token_count`), and `count()` itself is `lru_cache`d at
4096 entries — both because the same strings are measured repeatedly during assembly and a second
counting path is how budgets start disagreeing with themselves.

---

# 4. Cross-cutting concerns

## 4.1 The determinism boundary

```mermaid
quadrantChart
    title Determinism required vs semantic judgement required
    x-axis Deterministic --> Probabilistic
    y-axis Low blast radius --> High blast radius
    quadrant-1 Judge here, bound it hard
    quadrant-2 Must never be prompt-only
    quadrant-3 Either way
    quadrant-4 Fine to be fuzzy
    loop control: [0.05, 0.95]
    terminal uniqueness: [0.05, 0.9]
    claim routing decision: [0.75, 0.85]
    severity bucket: [0.7, 0.7]
    field pruning: [0.02, 0.55]
    section order: [0.03, 0.5]
    segment compression prose: [0.9, 0.35]
    clarifying question wording: [0.95, 0.25]
```

| Contract | Implementation | Enforcement |
|---|---|---|
| Loop continues only on `tool_use` | code | `test_stop_reason_is_loop_control` |
| No string-matching on model prose | code | `test_no_string_membership_against_text_in_loop` (AST audit) |
| One terminal action per claim | code | `test_tools.py` double-terminal case |
| A claim is never left undecided | code | `test_run_backstop.py` |
| Tool failures are typed | code | `test_tools.py` envelope cases |
| Pruning is not probabilistic | code | `test_pruner_has_no_anthropic_import` (AST audit) |
| One token counter only | code | `test_only_tokens_module_uses_count_tokens_or_heuristic` |
| Section order and exclusivity | code | `test_assemble_module_section_order_contract` |
| *Which* claim type, *which* severity | prompt + enum schema | schema rejection at the tool boundary |
| *Which* facts are durable | prompt + schema, then structural validation | `CaseFactExtractionError` on any null |

## 4.2 Failure-mode catalogue

| Failure | Detected by | Behaviour | Artifact evidence |
|---|---|---|---|
| Model ends turn without deciding | backstop after loop | escalated via the real terminal tool | `summary.md` row with empty `claim_type` |
| Model calls a terminal tool twice | `terminal_called` guard | `permanent` error, no duplicate record | `test_tools.py` |
| Tool handler raises | `make_executor` catch | `transient / retryable` envelope | `test_tools.py` |
| Runaway spend | `Budget.check()` before every call | `BudgetExceeded`, exit path preserving traces | `test_loop.py` |
| `stop_reason` unexpected | explicit branch | `UnexpectedStopReason`, no partial success | `test_loop.py` |
| Extraction returns null field | `CaseFactExtractionError` | run aborts loudly, lists the gaps | `test_case_facts.py` |
| Pruner input incomplete | `PrunerMissingFieldError` | abort, never a silent partial dict | `test_pruner.py` |
| Asked to compress the active segment | explicit refusal | `ValueError`, active text stays verbatim | `test_compressor.py` |

## 4.3 Architecture Decision Records

### ADR-001 — `stop_reason` is the only control signal
**Context.** Agents routinely branch on the presence of keywords in model prose.
**Decision.** Control flow reads `response.stop_reason` only; unexpected values raise.
**Consequence.** Wording changes cannot alter control flow. Adding a new stop reason is a code change,
which is the point: it forces a human decision instead of a silent behavioural drift.

### ADR-002 — no iteration cap; a Budget is the ceiling
**Context.** `while turn < 10` is the most common way to "fix" non-termination.
**Decision.** No counter. A `Budget` with token and wall-clock limits raises before each call.
**Consequence.** Non-termination surfaces as a typed exception with a trace, not as a truncated run
that looks successful.

### ADR-003 — terminal tools are two, and neither implies the other
**Context.** One `finish(status=...)` tool makes the model guess the taxonomy.
**Decision.** `route_to_adjuster` and `escalate_to_human` are separate, with distinct preconditions.
**Consequence.** Routing cannot occur without classification and severity; escalation carries
`confidence` and `candidate_claim_types` so a human receives uncertainty as structured data.

### ADR-004 — the backstop lives in code and reuses the model's tool
**Context.** `end_turn` proves the turn ended, not that the claim was decided.
**Decision.** After the loop, if no terminal tool ran, call `escalate_to_human` through the dispatcher.
**Consequence.** One artifact schema for both paths; `exit 1` keeps a silent non-decision visible.

### ADR-005 — case facts are extracted, validated, then placed at the top
**Context.** Compressing a 48-turn transcript loses exactly the tokens the live issue needs.
**Decision.** Extract 12 fields in one call, reject any null, render them at the top boundary.
**Consequence.** Structured status survives compression; the cost is one extra call and a hard
failure mode when the extraction is incomplete (chosen over a silent `None`).

### ADR-006 — pruning is a dict comprehension, not a model call
**Context.** A 57-field tool payload could be "summarised" by the model.
**Decision.** Keep 5 named fields deterministically; the module has no `anthropic` import.
**Consequence.** Zero tokens, zero variance, and an AST audit that keeps it that way.

### ADR-007 — one token counter, methodology recorded
**Context.** Two counters produce two budgets and an unauditable claim.
**Decision.** All measurement through `tokens.count()`; the active backend is written into
`budget.json`.
**Consequence.** Numbers are reproducible or explicitly labelled heuristic; a second path fails an
AST audit.

---

# 5. Verification matrix

| Claim | Proof | Where |
|---|---|---|
| Loop obeys `stop_reason` | 5 loop tests + recorded 2-turn trace | `test_loop.py` · `traces/claim_01_kitchen_fire.jsonl` |
| Tool contract holds | 12 tool tests | `test_tools.py` |
| Backstop fires | 1 test + a real escalated claim | `test_run_backstop.py` · `summary.md` |
| Token accounting reconciles | trace sums equal summary totals | `2,809+3,567=6,376` · `400+87=487` |
| Context reduced without loss | 6/6 eval + 1/2 control regression | `eval.jsonl` · `eval_control.jsonl` |
| Assembly order is a contract | 3 assembly tests + AST audit | `test_assemble.py` · `test_antipatterns.py` |
| Extraction cannot silently fail | 5 case-fact tests | `test_case_facts.py` |
| Submission complete | audit gate | `submission/final-capstone-audit.txt` → **PASS** |

---

# 6. Threats to validity

| Threat | Reality | Mitigation in this pack |
|---|---|---|
| **Partial workspace checkout** | The environment used for the recorded runs held only part of the course tree (disk-size constraints). Artifact-dependent audits therefore **skip**, they do not fail: `test_assembled_context_active_segment_byte_exact` and `test_budget_json_section_counts_sum_consistently` need a `runs/` directory that was not present at test time. | Reported as `28 passed, 2 skipped` — never as "all green". The two audits did execute against the run documented here, whose artifacts are committed. |
| One fixture executed end-to-end | Only `claim_01_kitchen_fire` has a recorded trace; the other fixtures are covered by schema and loop tests. | Stated in the README and here; the routed path is exercised by `test_tools.py`, not by a recorded run. |
| Single model, single sample | All measurements are `claude-haiku-4-5-20251001`, one run each. | Numbers are presented as measurements of *this* run, not as benchmarks. |
| Token heuristic | `len(text)/3.8` when no API key is present. | The methodology string is written into `budget.json`; the recorded System 2 run used the model-authoritative endpoint. |
| Systems 3 and 4 | Analysed in the reflection brief, no artifacts here. | Declared in the README scope table; the audit reports no missing required evidence. |
| Mermaid rendering | Diagrams are linted structurally (fenced blocks balanced, `subgraph`/`end` paired, no reserved characters in labels) but were not rasterised in this environment. | If a diagram fails to render on GitHub, the surrounding tables carry the same information. |

---

<p align="center"><sub>Engineering annex · Harness Engineering capstone · Ivo Sánchez Molinero ·
numbers reproduced from <code>evidence/</code>, not from memory.</sub></p>
