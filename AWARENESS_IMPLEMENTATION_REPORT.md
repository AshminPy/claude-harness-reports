# Awareness Implementation Report

Goal: lightweight, day-to-day-useful awareness — explicitly not a complex
self-awareness framework. Most of what the roadmap asked for already existed from the
Phase 1-7 reliability work; this phase's real contribution is a small, additive
aggregation layer over that existing state, plus one genuinely new distinction
(the ASSUMED confidence label) and one generalization (completion checklist usable
outside STRICT mode).

---

## Awareness features implemented

| Requested feature | Source of truth | Status |
|---|---|---|
| Current objective | `validation_state.json`'s `task` field | Already existed |
| Current phase | `attempts[].phase_id` | Already existed (Phase 2/3) |
| Acceptance criteria | `.claude/task-contract.md` "Definition Of Done" | Already existed |
| Completed / remaining work | `task-queue.md` status rows | Already existed |
| Last meaningful result | `command_outcomes[-1]` | Already existed (Phase 6) |
| Current blocker | `task-queue.md` blocker field + tripped circuit breakers | Already existed, now surfaced by `completion_checklist()` |
| Failed hypotheses | `attempts[]` with `outcome="failed"` + `hypothesis_id` | Already existed (Phase 2/3) |
| Retry count | `get_attempt_count()` / `consecutive_failures()` | Already existed (Phase 2/3) |
| Important assumptions | **New**: `validation_state.assumptions[]` | **Added this phase** |
| Evidence gathered | `evidence-ledger.json` | Already existed (Phase 1/6) |
| Next action | `task-queue.md` next pending row / task-contract | Already existed |
| Whether human input is genuinely required | `escalation_summary()`'s implicit signal | **Generalized this phase** into `awareness_snapshot()`'s explicit `human_input_required` boolean |
| Context-budget awareness (detail in files, summarize older work) | `rules/behavior.md`'s reply-format rules, `troubleshooting.md`'s 3-file model | Already existed |
| Confidence awareness (VERIFIED/INFERRED/ASSUMED/UNKNOWN) | `rules/troubleshooting.md`'s FACT/HYPOTHESIS/UNKNOWN labels | **Extended this phase** — added ASSUMED as a 4th, distinct label |
| Completion awareness | `final_gate.py` (STRICT only) | **Generalized this phase** — `completion_checklist()` gives the same signal at any risk mode, informationally, never as a new blocking gate |
| Dependency awareness | `task-queue.py`'s existing `blocker` field | Already existed — confirmed sufficient, not rebuilt |
| Failure awareness (remember failed approaches, change strategy) | `attempts[]`, `consecutive_failures()`, circuit breaker (Phase 2) | Already existed |

## What was actually built this phase

1. **`validation_state.py`: `assumptions[]` / `open_questions[]`** — two small new
   state arrays plus `add_assumption()`/`add_open_question()`. The one gap the audit's
   own Karpathy review had already flagged ("no explicit assumptions/open-questions
   field") and recommended exactly this fix for.
2. **`validation_state.py`: `awareness_snapshot(root)`** — a read-time aggregator, NOT
   a new persisted file. Pulls `task`/`task_started_at`/`risk_mode`/
   `task_contract_exists` from `validation-state.json`, task-queue summary from
   `task-queue.md` (if present), open failure sequences + `human_input_required` from
   the attempts log, last result from `command_outcomes`, evidence count from
   `evidence-ledger.json`, and the new assumptions/open-questions. One CLI call
   (`validation_state.py awareness <root>`) instead of manually re-reading 3-4 files.
3. **`validation_state.py`: `completion_checklist(root)`** — generalizes
   `final_gate.py`'s STRICT-only pass/fail signal into an on-demand, **informational**
   checklist usable at any risk mode: task contract exists?, evidence recorded?, diff
   reviewed?, mistakes checked?, any known blocker (task-queue blocker or a tripped
   circuit breaker)? Deliberately NOT wired into any hook as a new blocking gate —
   doing so for FAST/STANDARD work would be exactly the scope creep this roadmap
   explicitly warned against. STRICT tasks keep their real enforced gate unchanged.
4. **`rules/troubleshooting.md`**: added **ASSUMED** as a 4th label alongside the
   existing FACT/HYPOTHESIS/UNKNOWN, with an explicit definition distinguishing it from
   HYPOTHESIS (a hypothesis is actively being tested; an assumption is a parked default
   taken to keep moving, flagged and revisited only if it turns out to matter) and a
   pointer to `add-assumption`/`add-open-question` so assumptions survive as structured
   state, not just prose that gets lost.

## State model (additions only — full schema in `HARNESS_RELIABILITY_IMPLEMENTATION.md` §3)

```diff
 {
   ...
+  "assumptions": [],       # [{text, ts}]
+  "open_questions": [],    # [{text, ts}]
   ...
 }
```

## Context-budget behavior

Unchanged from what already existed — `awareness_snapshot()` is itself an example of
the principle: one aggregated call instead of re-reading `validation-state.json` +
`task-queue.md` + `evidence-ledger.json` separately, which is a small but real
context-usage improvement for any future long session that wants a "where do things
stand" check.

## Confidence classifications

`FACT` (VERIFIED) / `HYPOTHESIS` (INFERRED, actively being tested) / `ASSUMED` (parked
default, not being tested) / `UNKNOWN` (not yet established) — `rules/
troubleshooting.md`. Assumptions and open questions can now be recorded as structured
state (`add-assumption`/`add-open-question`), not just mentioned in passing prose that
doesn't survive a context compaction.

## Completion checks

`completion_checklist(root)` — see above. Informational at all risk modes;
enforcement remains exactly as strict as it was before (STRICT-only, via
`final_gate.py`), per the explicit instruction not to weaken or duplicate existing
gates.

## Failure handling

Unchanged — the full retry/circuit-breaker model from Phases 2/3 of the reliability
work already implements "remember failed approaches, avoid repeating an equivalent
failed action indefinitely, change strategy after repeated failure, stop only when the
retry policy or a genuine blocker requires it" exactly as specified. `awareness_snapshot()`
now surfaces this (`open_failure_sequences`, `human_input_required`) in one place
instead of requiring a separate query per fingerprint.

## Tests

5 new deterministic tests added to `scripts/test_reliability_fixes.py` (36 total, was
31): assumptions/open-questions persist; `awareness_snapshot()` correctly flags
`human_input_required=True` when a circuit breaker has tripped and `False` when
nothing is wrong; `completion_checklist()` correctly reflects real task-contract/
evidence/blocker state, including reporting a tripped circuit breaker as a known
blocker. All 36 pass; `command_policy.py --test` 143/143 unchanged; `harness_check.sh`
PASS.

## Known limitations

- Dependency awareness (missing credentials, unavailable services) has no dedicated
  new state — confirmed `task-queue.py`'s existing free-text `blocker` field is
  sufficient for this and was not duplicated with a more structured (and more complex)
  mechanism, per the explicit "lightweight" instruction.
- `awareness_snapshot()`/`completion_checklist()` are new and have not yet been
  exercised in a long real multi-day project session — their practical value will be
  best judged by actual use, not further speculative feature addition now.
- Confidence labeling (FACT/HYPOTHESIS/ASSUMED/UNKNOWN) remains a prose convention
  documented in `troubleshooting.md`, not a code-enforced requirement — same
  prompt-driven limitation already disclosed for every other prose-based rule in this
  harness.
