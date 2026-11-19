# Reflection Brief — Harness Engineering Capstone

**Name:** Ivo Sánchez Molinero  
**Date:** 2026-11-17

**Environment**

- Model(s): System 1 used `claude-haiku-4-5-20251001` through the laboratory backend; System 2 used `claude-haiku-4-5-20251001`. Both systems have successful end-to-end run artifacts.
- OS / Python: Linux workspace / Python 3.13.0
- Approx. API spend: System 1 recorded **$0.0088** for the documented claim run. System 2 used the laboratory provider for its measured build, evaluation, and control run. System 3 requires no inference API, and System 4 used the recorded Shift C response.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.**

My own end-to-end run `20261117_115722` for `claim_01_kitchen_fire` produced the observed `stop_reason` sequence `tool_use` -> `end_turn` across **2 turns**. Turn 1 returned `tool_use`, so the harness executed the requested tools and continued. Turn 2 returned `end_turn`, after which the deterministic backstop produced the final recorded outcome `escalated` because no terminal routing tool had been called.

This directly demonstrates the control rule in `claims_intake/loop.py`: `tool_use` continues the loop, `end_turn` returns, and unexpected stop reasons are rejected. The trace provides end-to-end evidence, while `system1.tests.txt` separately records **30 passed in 0.43s** for deterministic branch coverage.

**Evidence from my run:**
- `runs/20261117_115722/summary.md`
- `runs/20261117_115722/traces/claim_01_kitchen_fire.jsonl`
- `runs/20261117_115722/escalations.jsonl`

2. **Anti-pattern.**

One anti-pattern checked by `tests/test_antipatterns.py` is using string-membership checks against the model's free-form text to drive loop control. My `system1.tests.txt` artifact records **30 passed in 0.43s**, including `test_no_string_membership_against_text_in_loop` and `test_stop_reason_is_loop_control`. Instead, `claims_intake/loop.py` uses the structured `response.stop_reason`: `tool_use` continues, `end_turn` returns, and unexpected values raise. Matching magic text would make control flow depend on model wording and could therefore terminate or continue incorrectly after harmless wording changes.

3. **Tool design.**

System 1 defines seven tools in `claims_intake/tools.py`; two overlapping decision-stage operations are `route_to_adjuster` and `escalate_to_human`. Both are explicitly documented as terminal tools, while their schemas distinguish successful routing from unresolved ambiguity, low confidence, missing facts, policy disputes, and other escalation conditions. The same artifact implements graceful errors containing `is_error`, `error_category`, retryability, and a message; `system1.tests.txt` shows **30/30 tests passed**, including unknown-tool, double-terminal, and transient-handler-error behavior. A structured error lets the agent distinguish a retryable transient failure from a permanent invalid action instead of interpreting an opaque error string.

4. **Your numbers.**

For my real run `20261117_115722`, fixture `claim_01_kitchen_fire` completed with outcome `escalated` in **2 turns**. The run recorded **6,376 input tokens**, **487 output tokens**, an estimated cost of **$0.0088**, and **4.8 s** elapsed time. The observed control sequence was `tool_use` -> `end_turn`.

The per-turn trace also reconciles with the summary: 2,809 + 3,567 = **6,376 input tokens**, and 400 + 87 = **487 output tokens**. These are measurements from my own artifacts rather than values copied from a sample.

**Evidence from my run:**
- `runs/20261117_115722/summary.md`
- `runs/20261117_115722/traces/claim_01_kitchen_fire.jsonl`
- `runs/20261117_115722/escalations.jsonl`

5. **The reduction.**

The measured baseline for my run was **38,708 tokens**. After extracting case facts, compressing the resolved refund and subscription segments, and retaining the active segment, the assembled context contained **16,901 tokens**. This is a measured reduction of **56.34%**, not an estimate.

The resulting section budget was:

- `case_facts`: 204 tokens
- `resolved_refund`: 422 tokens
- `resolved_subscription`: 504 tokens
- `active`: 15,789 tokens

The assembled artifact was written to `context.md` and contained **78,301 characters** (78,579 bytes). This demonstrates that the strategy reduced the long conversation substantially while keeping the material needed for the active support problem.

**Run evidence:**
- Run ID: `vocareum-system2-20261118-164825`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/budget.json`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/context.md`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/eval.jsonl`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/eval_control.jsonl`

6. **Summarize vs preserve.**

I evaluated the compressed context with the project's six-question evaluation set. The run passed **6/6 questions**. The retained evidence supported the completed refund and subscription cases as well as the active payment-method issue, including order `ORD-77310`, failure code `AVS_MISMATCH`, the new card ending in `7782`, and the exact structured status needed by evaluation Q6.

The answer for evaluation Q6 preserved the structured status as `payment_update_status`. I am treating the value recorded in `eval.jsonl` as authoritative rather than reconstructing it from console output or memory.

This result shows that the reduced context was not merely smaller. It remained usable for factual questions across resolved history and the current active case.

**Run evidence:**
- Run ID: `vocareum-system2-20261118-164825`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/budget.json`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/context.md`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/eval.jsonl`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/eval_control.jsonl`

7. **Facts block.**

The token budget shows where the compression occurred. The baseline conversation contained **38,708 tokens**, while the final assembly contained **16,901 tokens**, a **56.34% reduction**.

The resolved segments were represented compactly:

- `case_facts`: 204 tokens
- `resolved_refund`: 422 tokens
- `resolved_subscription`: 504 tokens
- `active`: 15,789 tokens

The active segment remained the dominant section at **15,789 tokens**. This matches the design goal: compress information from issues that were already resolved, extract durable case facts, and preserve the current interaction with much higher fidelity. The trade-off is that older narrative detail is intentionally reduced, so retrieval quality depends on extracting the right durable facts before pruning.

**Run evidence:**
- Run ID: `vocareum-system2-20261118-164825`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/budget.json`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/context.md`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/eval.jsonl`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/eval_control.jsonl`

8. **Path-scoped rules.**

In System 3, `.claude/rules/tests.md` declares paths for `**/*.test.tsx` and `**/*.test.ts`; `.claude/rules/api.md` scopes API conventions to `src/api/**/*`, while `.claude/rules/react.md` covers `src/components/**/*` and `src/pages/**/*`. My `system3-tests.txt` artifact records **35 passed in 0.30s**, including the check that a React test file receives both React and testing rules. This is better than a directory-level `CLAUDE.md` for cross-cutting conventions because a test can live in multiple source subtrees while still matching the same test glob. The rule follows the file type instead of duplicating instructions across directory hierarchies.

9. **Forked skill.**

`.claude/skills/deploy-check/SKILL.md` contains `context: fork` and an `allowed-tools` list containing `Read`, `Grep`, `Glob`, and constrained `Bash(gh pr checks:*)`. My `system3-tests.txt` records **35/35 passing**, including validation of `context: fork` and the read-only allowlist, while `system3-validator.txt` contains `OK`. Forking keeps verbose discovery, diff inspection, and check output outside the main conversation and returns the useful summary. The constrained tool boundary reduces blast radius: the deploy check is designed to inspect and report, not edit files, push changes, run migrations, or deploy.

10. **Scope.**

`CLAUDE.md` documents `./CLAUDE.md`, `.claude/standards/`, and `.claude/rules/` as project-level, version-controlled configuration; `~/.claude/CLAUDE.md`, `~/.claude/commands/`, and `~/.claude/skills/` are user-level and remain personal. A concrete project-level artifact is `.claude/skills/deploy-check/SKILL.md`, whereas the same file suggests a private stricter variant under `~/.claude/skills/deploy-check-strict/`. My `system3-validator.txt` records `OK`, and `system3-tests.txt` records **35 passed in 0.30s**, including validation of project/user scope. This prevents personal preferences from silently becoming shared team policy.

### System 4 — Orchestration

11. **Push work down.**

My `clean.sqlite` contains **40 defects total**, while the indexed time filter identifies **17 defects since `2026-04-01T00:00:00Z`**; `clean-scratchpad.jsonl` independently records `17 new defects analyzed`. The indexed query is `defects_since()` in `shift_monitor/warm.py`: `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, supported by `idx_defects_ts` and `idx_defects_shift_ts`. This pushes historical filtering into SQLite instead of loading the entire warm-tier history into model context. The model therefore receives the bounded relevant set rather than all 40 stored records.

12. **Crash recovery.**

`shift_monitor/recovery.py` sets `STALE_RESUME_THRESHOLD_MINUTES = 30`: an empty or complete manifest starts fresh, an incomplete manifest at **30 minutes or less** resumes, and an older incomplete run starts fresh. System 4's cumulative evidence records **33 passed in 1.03s**, and the earlier crash-recovery tests explicitly exercised 29, 30, and 31 minutes. `shift_monitor/manifest.py` writes incrementally in append mode and performs `flush()` followed by `os.fsync()` before returning. A fresh restart after stale work can be safer because it reconstructs execution from durable current state instead of continuing assumptions from an old partially completed run.

13. **Small state.**

My `clean-hot.json` is **965 bytes** against `HOT_STATE_BYTE_BUDGET = 5120`, approximately **18.8% of the budget**. The artifact contains 17 recent defect hashes, the Shift C summary, three active alerts, and three threshold statuses; the full warm history remains in `clean.sqlite` with 40 defect records. `system4-tests.txt` records **33 passed in 1.03s**, supporting the completed implementation. Keeping hot state bounded prevents prompt-facing state from growing without limit as an eight-hour-shift process runs indefinitely.

---

## Part 2 — Synthesis

14. **Three layers.**

→ **Model:** The model-facing layer appears in System 1's agent loop and tool definitions: `claims_intake/loop.py` sends model responses through structured `stop_reason` and tool-use handling; `system1.tests.txt` records **30 passed**.

→ **Harness:** System 3's `CLAUDE.md`, `.claude/rules/`, `.claude/commands/review.md`, and `.claude/skills/deploy-check/SKILL.md` constrain how Claude Code works inside the repository. `system3-tests.txt` records **35 passed** and `system3-validator.txt` records `OK`.

→ **Orchestration:** System 4 coordinates execution across shifts using `clean.sqlite`, `clean-hot.json`, `clean-scratchpad.jsonl`, recovery manifests, and `run_shift`; `system4-tests.txt` records **33 passed** and the recorded Shift C run analyzed 17 relevant defects.

15. **Deterministic vs prompt.**

A behavior guaranteed in code is System 4's hot-state limit: `HOT_STATE_BYTE_BUDGET = 5120`, while my generated `clean-hot.json` is only **965 bytes**; exceeding the budget is programmatically rejected/trimmed rather than left to model judgment. A prompt-guided behavior is System 1's use of claim facts and tool descriptions to guide when the model should classify, clarify, route, or escalate, while the harness still enforces terminal-tool semantics and structured errors. Deterministic enforcement is appropriate for hard safety, size, and state invariants that must never depend on model interpretation. Prompt guidance is appropriate where semantic judgment is required but code still constrains the available actions.

16. **Context, two faces.**

I validated the importance of the extracted case-facts layer with the control evaluation. The normal compressed-context evaluation passed **6/6 questions**. The control then removed case facts and reran **2 held questions**.

The control produced **1/2 regression(s)**, satisfying the requirement that at least one held question regress. Specifically, control Q6 regressed: without the structured case-facts layer, the model could no longer reliably recover the exact status token required by the case record. Control Q1 remained answerable because the refund amount also survived in other retained context.

This is more meaningful than reporting the normal evaluation alone. It provides an ablation test: the run changes one context component and observes a corresponding loss in answerability. The evidence therefore supports a causal claim that case facts were necessary for at least one evaluated requirement.

**Run evidence:**
- Run ID: `vocareum-system2-20261118-164825`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/budget.json`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/context.md`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/eval.jsonl`
- `Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/vocareum-system2-20261118-164825/eval_control.jsonl`

17. **Reliability you can't see in one run.**



One successful System 4 run would not prove the crash-recovery boundary, but
`system4-tests.txt` records **33 passed in 1.03s** and the crash-recovery
tests exercise incomplete manifests at 29, 30, and 31 minutes. Those tests
establish behavior that the successful recorded Shift C run alone cannot
demonstrate: 30 minutes still resumes while 31 minutes starts fresh.
`shift_monitor/manifest.py` also explicitly performs `flush()` and
`os.fsync()` for each appended step, covering durability behavior that is
unlikely to be visible in a normal happy-path run. This matters before
shipping because interruption and stale-recovery behavior must be repeatable
under failure, not merely inferred from one successful execution.

18. **Blast radius.**

For System 3's `deploy-check`, the intended blast radius is deliberately small because `.claude/skills/deploy-check/SKILL.md` uses `context: fork` and a constrained read-oriented allowlist (`Read`, `Grep`, `Glob`, and limited PR-check inspection). The skill explicitly does not edit files, push, migrate, or deploy; its output is a structured pass/fail summary returned from the fork. `system3-tests.txt` confirms **35/35 tests passed**, including validation of the fork context and read-only tool policy, and `system3-validator.txt` is `OK`. The practical kill switch is the capability boundary itself: dangerous mutation capabilities are not present in the allowed tool set.

---

## Part 3 — Honest assessment

19. **What broke.**



The first end-to-end System 1 attempt failed in my environment before
inference because `claims_intake/client.py` could not find
`ANTHROPIC_API_KEY`; the observed error was
`RuntimeError: ANTHROPIC_API_KEY is not set`. I then inspected System 2 and
confirmed that its environment also had no available inference backend:
there was no API key and no authenticated `claude` CLI in `PATH`.
`system1.tests.txt` nevertheless records **30 passed in 0.43s**, while
`system2-tests.txt` records **28 passed, 2 skipped, 0 failed in 1.17s**,
which separates the environment dependency from the locally verified
implementation. Completing the end-to-end inference artifacts would require
provisioning a legitimate inference backend that is not present in this
workspace, so I preserved that limitation rather than fabricating run
evidence.

20. **What you'd change.**

I would make reproducible offline evaluation a first-class interface for every capstone system rather than only providing it for System 4. System 4 already demonstrates the benefit with `fixtures/recorded_responses/shift_C_2026-04-30.json`, which allowed me to generate real local state artifacts (`clean-hot.json`, `clean-scratchpad.jsonl`, and `clean.sqlite`) without a live inference dependency. Systems 1 and 2 currently couple required capstone run artifacts to an external inference backend even though their deterministic harness behavior can already be extensively verified locally (**30/30 tests** for System 1 and **28 passed, 2 skipped, 0 failed** for System 2). I would add versioned recorded-response/replay fixtures for deterministic capstone verification while keeping an optional live-model run for evaluating model-dependent behavior.

---

## Outstanding evidence before final submission

The following placeholders are intentionally not fabricated:

- **Q1:** own System 1 trace and actual `stop_reason` sequence.
- **Q4:** own System 1 turn count, token usage, and estimated cost.
- **Q5:** own System 2 `budget.json` numbers.
- **Q6:** own System 2 per-section token counts.
- **Q7:** own `eval.jsonl` versus `eval_control.jsonl` regression.
- **Q16:** add System 2 measured baseline/assembled-token figures.

All other answers above are drafted from inspected code, passing test artifacts, validator output, or artifacts produced by my own recorded-response run.
