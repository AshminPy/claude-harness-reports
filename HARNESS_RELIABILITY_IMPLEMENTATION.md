# Harness Reliability Fixes — Implementation Report

Authoritative requirements source: `HARNESS_RELIABILITY_AUDIT.md`.
Branch: `feature/harness-reliability-fixes` (off `init`, clean working tree confirmed
before starting). Task contract: `.claude/task-contract.md`.
All work done in the source repo (`~/projects/ai-projects/claude-toolkit`) first.
**Nothing has been synced to the live `~/.claude` mirror yet** — that is a separate,
explicit step (see §7) gated on your confirmation, since it changes hook behavior for
every Claude Code session on this machine, including the one running right now.

---

## 1. Summary

All 6 fixes the audit flagged Critical/High/Medium were implemented and verified, in
the required order, each phase's acceptance tests passing before the next began.

| Phase | Fix | Status |
|---|---|---|
| 1 | C1 — trusted evidence provenance | ✅ Implemented, 7/7 tests pass |
| 2 | H1 — retry tracking / repeated-failure detection | ✅ Implemented, 9/9 tests pass |
| 3 | H3 — enforced retry limits in orchestrated skills | ✅ Implemented, 5/5 tests pass |
| 4 | H2 — stale-state protection | ✅ Implemented, 6/6 tests pass |
| 5 | M1 — scope-fence bookkeeping exclusion | ✅ Implemented, exact acceptance test matches |
| 6 | M2 — command outcome persistence | ✅ Implemented, 6/6 tests pass |
| 7 | Regression suite | ✅ 25 deterministic tests, wired into `harness_check.sh` |

**Deferred / not implemented** (out of the audit's Critical/High/Medium scope, or
explicitly out of scope for this task):
- Fix L1 (commit `~/.claude/rules/troubleshooting.md` into the repo) — Low priority,
  not requested in this implementation task's phase list.
- Fix L2 (cap/rotate `commands_run`) — Low priority; a related but distinct cap
  (`MAX_COMMAND_OUTCOMES`) was added for the *new* `command_outcomes` list in Phase 6,
  since Phase 6 explicitly required bounding, but the older `commands_run` list itself
  was left untouched (touching it wasn't in any phase's file list, and the task said
  "do not redesign unrelated parts").
- Karpathy checks 1-15 from the audit beyond what Phases 1-7 directly address (e.g.
  Karpathy #2 "Think→Act→Observe, no batched uncontrolled edits" has no code path in
  this harness at all) — none of these were in the phase list either.
- `worktree_isolate.py`'s existing broad `.claude/` exclusion (used for
  "is the tree clean enough to create a worktree") was **not** narrowed to match the
  new, tighter `changed_files_fence.py` exclusion — Phase 5's file list named only
  `scripts/changed_files_fence.py`, and changing `worktree_isolate.py`'s working,
  tested (mistake M010) behavior was judged out of scope. Noted as a residual
  inconsistency in §6.

**Key design decisions**
- **Trust is always recomputed, never stored-and-trusted.** `evidence_ledger.is_trusted()`
  ignores the `"trusted"` boolean persisted in each ledger entry and recomputes it from
  `recorded_by`/`exit_code`/`evidence_type`/`task_id`/`observed_at` every time. Hand-editing
  the JSON file to say `"trusted": true` has zero effect on any gate.
- **Success/failure for Bash is derived from *which hook event fired*, not from a
  structured exit-code field.** Official Claude Code hooks docs (`code.claude.com/docs/en/
  hooks.md`, checked 2026-07-21, cross-verified via two independent fetches — one direct,
  one via a `claude-code-guide` subagent) show `PostToolUse` delivers a single combined
  output string with no separate stdout/stderr/exit_code fields, and a successful tool
  call is signaled by `PostToolUse` firing at all (a failure instead fires
  `PostToolUseFailure`, whose `error_message` string sometimes embeds a textual exit code).
  This is why `bash_output_filter.py` (PostToolUse) always records `exit_code=0` and
  `post_tool_failure.py` (PostToolUseFailure) parses whatever numeric code it can find in
  the error text, defaulting to `None` otherwise. **Field-name ambiguity**: the docs fetch
  showed `tool_output`/`error_message`, while this repo's pre-existing code read
  `tool_response`/`error`. Both fetches were partially truncated and could not be fully
  reconciled with the working code, so both hooks now read **both** key names
  defensively (whichever is present, non-empty) rather than assuming one — documented
  as an explicit, honest limitation rather than a confirmed fact.
- **Phase-level retries reuse the same fingerprint/attempts machinery as command-level
  retries** (Phase 3 built no separate data structure) — a `phase_id` (optionally combined
  with a `hypothesis_id`) is just a different fingerprint seed than a raw shell command,
  so `consecutive_failures()`/`attempts_exhausted()`/`reset_attempts()` are shared code,
  not parallel implementations that could drift apart.
- **A fresh, explicit privileged grant always bypasses the retry ladder.** The ladder
  governs unattended repetition by the agent; a human who just deliberately typed/tapped
  approval for this exact command is real, in-the-moment authorization, not something the
  circuit breaker should second-guess.
- **`.claude/` bookkeeping exclusion is narrow, not blanket** (Phase 5) — only
  `.claude/validation-state.json` and `.claude/state/**` are excluded from scope-fence
  violations. `.claude/commands/*.md`, `.claude/task-contract.md`, `.claude/CLAUDE.md`,
  etc. remain normally scope-checked, since a task can legitimately target those.

---

## 2. Files changed

| File | Purpose | Key implementation details |
|---|---|---|
| `scripts/evidence_ledger.py` | Evidence storage/trust model | Added `evidence_id`, `observed_at`, `exit_code`, `recorded_by`, cached-but-never-trusted `trusted` field. New `RECORDED_BY_HOOK`/`RECORDED_BY_MANUAL`, `TRUSTABLE_EVIDENCE_TYPES`. New `is_trusted()` (pure, recomputed), `record_hook_evidence()` (the only sanctioned way to write `recorded_by="hook"`), `has_trusted_evidence()`. CLI `add-claim` unchanged positionally — never exposes `recorded_by`/`exit_code`, so a manual claim can never masquerade as trusted. |
| `scripts/final_gate.py` | STRICT-mode completion gate | `claims.validation_passed` now requires `has_trusted_evidence(..., evidence_type=TRUSTABLE_EVIDENCE_TYPES, task_started_at=...)` instead of mere `has_evidence()` presence. |
| `scripts/validation_state.py` | Shared state manager | Added `task_started_at` + `start_task()` (Phase 1 freshness). Added the full attempts/retry model: `action_fingerprint()`, `record_attempt()`, `consecutive_failures()`, `attempts_exhausted()`, `record_change_note()`/`has_recent_change_note()`, `reset_attempts()` (reason-required), `escalation_summary()` (Phase 2). Added phase-level wrappers `start_attempt()`, `record_attempt_result()`, `get_attempt_count()` with hypothesis-scoped fingerprints (Phase 3). Added `redact_secrets()` (moved here as single source of truth), `_bound_output()`, `record_command_outcome()`, `MAX_OUTPUT_BYTES`/`MAX_COMMAND_OUTCOMES` caps (Phase 6). New CLI subcommands for all of the above. |
| `hooks/command_policy_hook.py` | PreToolUse Bash gate | Computes `action_fingerprint`/`consecutive_failures` for blocked/ask classifications before deciding a response. Attempt 1 unchanged; attempt 2 appends a repeat notice; attempt 3 requires a recorded change-note (else asks for one instead of the normal message); attempt 4+ overrides the response to a hard `deny` with a full escalation summary, regardless of mode. A consumed privileged grant still bypasses the ladder. Evidence auto-logging switched to `record_hook_evidence(..., exit_code=None)` — honestly hook-recorded but never trusted, since PreToolUse fires before execution. |
| `hooks/post_tool_failure.py` | PostToolUseFailure | Now reads `error_message` **and** `error` defensively (found a latent field-name bug matching the `tool_response`/`tool_output` pattern). Records a `"failed"` attempt into the shared attempts log with a normalized error fingerprint and (Phase 6) a bounded/redacted `command_outcome` with whatever exit code could be parsed from the error text. |
| `hooks/bash_output_filter.py` | PostToolUse Bash | Now defensively reads `tool_response` **or** `tool_output`. On every real Bash success, records a `command_outcome` (`exit_code=0`), closes any open failure sequence for that action (`record_attempt(outcome="success")`), and logs real hook-recorded evidence (`exit_code=0`) — this is what lets an ordinary allowed command (not just a blocked one) genuinely satisfy `claims.validation_passed`. Local `redact_secrets`/pattern list removed in favor of importing `validation_state.redact_secrets`. |
| `scripts/session_reset_check.py` | SessionStart archive/reset | Added `check_stale_state()` — runs independent of whether `task-queue.md` exists. If `validation-state.json` is older than `STALE_STATE_HOURS` (default 24) and no unfinished task-queue exists, archives `validation-state.json`, `task-contract.md`, `evidence-ledger.json`, `source-freshness.json` (timestamped copies, never deleted) then resets/removes the live copies. Existing task-queue-based cases 2/3 unchanged. |
| `scripts/changed_files_fence.py` | Scope-violation check | New `is_harness_bookkeeping()` — narrow prefix match on `.claude/validation-state.json` and `.claude/state/`. `git_changed_paths()` now filters these out before returning, so they never appear in the changed-files list or the violations list. |
| `scripts/harness_check.sh` | Harness acceptance test | Added a check + execution of the new `scripts/test_reliability_fixes.py` suite — a harness install is not "passing" unless all 25 new tests pass, same bar as the existing `command_policy.py --test`. |
| `scripts/test_reliability_fixes.py` | **New** — Phase 7 regression suite | 25 deterministic tests, each in its own throwaway git repo (`tempfile.mkdtemp`), covering every acceptance criterion from Phases 1–6, plus regression coverage of the existing destructive-command/quoted-text classification suite and worktree rollback. No live session required. |
| `skills/ship/SKILL.md` | Ship pipeline | "3 attempts" guardrail replaced with explicit `start-attempt`/`record-attempt-result` CLI calls and an instruction to stop on `exhausted=true`. |
| `skills/verified-ship/SKILL.md` | Verified-ship pipeline | Same change as `ship/SKILL.md`. |
| `skills/forge-agent/SKILL.md` | Multi-agent orchestration | Circuit-breaker section and escalation format now call the same CLI helpers instead of self-tracking a `Loop: n/3` count in the shared-memory JSON by hand. |
| `.claude/task-contract.md` | **New** — this task's own contract | Goal/scope/allowed-files/prohibited-files/acceptance-criteria/rollback, per Phase 0. |

---

## 3. Data model changes

### `evidence-ledger.json` entry — before → after

```diff
 {
   "timestamp": "...",
   "task_id": "...",
   "claim": "...",
   "evidence_type": "command",
   "source": "...",
   "result": "...",
   "confidence": "high",
-  "notes": ""
+  "notes": "",
+  "evidence_id": "ev-000001",
+  "observed_at": "...",           // == timestamp today; kept distinct for future use
+  "exit_code": 0,                 // null for manual/pre-execution entries
+  "recorded_by": "hook" | "manual",
+  "trusted": false                // CACHED for humans only — never read by any gate
 }
```
`VALID_EVIDENCE_TYPES` grew from 7 to 11 (`command, file, diff, test, official-doc,
user-provided, assumption` + `build, lint, validation, observation`).
`TRUSTABLE_EVIDENCE_TYPES = {command, test, build, lint, validation}` is new — the only
types `is_trusted()` will ever accept toward `claims.validation_passed`.

### `validation-state.json` — new top-level keys

```diff
 {
   "task": null,
+  "task_started_at": null,
   "risk_mode": null,
   ...
   "commands_run": [],
+  "attempts": [],                  // [{attempt_id, task_id, phase_id, action_fingerprint,
+                                    //   command, classification, outcome, exit_code,
+                                    //   error_fingerprint, timestamp, hypothesis_id,
+                                    //   changed_since_previous_attempt}]
+  "change_notes": [],              // [{task_id, action_fingerprint, note, ts}]
+  "circuit_breaker_resets": [],     // [{task_id, action_fingerprint, reason, approved_by, ts}]
+  "command_outcomes": [],           // bounded (≤200 entries), redacted, real post-exec records
   ...
 }
```
All additions default to empty lists/`None` — an old `validation-state.json` written
before this change loads cleanly (`load_state()`'s `default_state().update(loaded)`
merge already handled this; verified by `test_command_outcome_records_...`/
`test_legacy_evidence_file_fails_safe` in the regression suite).

### `task-queue.md` / `evidence-ledger.json`/`source-freshness.json` archive dirs — new

```
.claude/validation-state-archive/<ts>.json     (pre-existing, unchanged)
.claude/task-queue-archive/<ts>.md             (pre-existing, unchanged)
.claude/task-contract-archive/<ts>.md          (NEW — Phase 4)
.claude/evidence-ledger-archive/<ts>.json      (NEW — Phase 4)
.claude/source-freshness-archive/<ts>.json     (NEW — Phase 4)
```

---

## 4. Security review

- **Destructive-command protection is unchanged or stronger, never weaker.**
  `test_no_existing_classification_becomes_weaker` re-asserts the exact 11-case matrix
  the audit verified live (Test 8) — `terraform apply/destroy`, `kubectl delete`,
  `git push origin main` → `blocked_needs_privileged`; `rm -rf /`, `rm -rf ~` →
  `blocked_hard`; safe commands and quoted-text edge cases → `allowed`. All still pass.
  Additionally verified live (not just via the deterministic suite): `rm -rf /` and
  `cat .env` each retried 5× through the real `command_policy_hook.py` — every single
  attempt still returned `deny`, never `allow`, even after the retry ladder's internal
  counters incremented. The retry ladder can only escalate a repeated blocked/ask
  response to a *stricter* `deny` — there is no code path where it turns a blocked
  classification into `allow`.
- **Manual evidence cannot become trusted execution evidence.** The CLI `add-claim`
  subcommand has no argument that sets `recorded_by` or `exit_code` — every entry it
  writes is `recorded_by="manual", exit_code=None`, which `is_trusted()` rejects
  unconditionally (`recorded_by != "hook"` short-circuits to `False` before any other
  check runs). `record_hook_evidence()` — the only function that can set
  `recorded_by="hook"` — is called exclusively from hook code that imports
  `evidence_ledger` as a Python module (`command_policy_hook.py`,
  `bash_output_filter.py`), never reachable via a shell command a model could type.
  Residual risk: a model could in principle write and run a Python one-liner that
  imports `evidence_ledger` directly and calls `record_hook_evidence()` itself,
  bypassing the intended call sites — this is the same class of "if it really tried"
  limit every other fail-safe hook in this harness already accepts (documented in §6,
  not hidden).
- **Secrets are not persisted.** `record_command_outcome()` always redacts (`vs.
  redact_secrets()`, the same patterns previously local to `bash_output_filter.py`,
  now centralized) *before* bounding/truncating, on every stdout/stderr field it stores.
  Verified: `AWS_API_KEY=AKIAABCDEFGHIJKLMNOP` → persisted as `AWS_[REDACTED]`, both in
  the deterministic suite and via a real hook invocation with intentionally malformed-
  then-corrected JSON input to rule out a false pass.
- **Retry-circuit reset requires an explicit, valid condition.** `reset_attempts()`
  raises `SystemExit` on an empty/whitespace-only reason — verified both in the
  deterministic suite and manually. There is no code path that clears
  `attempts_exhausted()` other than a real success (`outcome="success"` closing the
  sequence) or this explicit, reason-required reset. Nothing added in this
  implementation exposes a way for the agent to self-approve a reset — `approved_by`
  is a free-text field for audit-trail purposes, not an authorization check (the actual
  gate is the non-empty-reason requirement plus the fact that `reset-attempts` is
  already in `command_policy.py`'s existing self-unlock blocklist alongside
  `grant-privileged`/`set-explicit-approval`/`force-set-risk-mode`, unchanged by this work).

---

## 5. Test evidence

Full output of the automated regression suite (this is real, captured command output,
not summarized):

```
$ python3 scripts/test_reliability_fixes.py
test                                                              result
---------------------------------------------------------------------------
Fix C1: fabricated evidence rejected                              PASS
Fix C1: trusted evidence accepted                                 PASS
Fix C1: failed evidence rejected                                  PASS
Fix C1: cross-task evidence rejected                              PASS
Fix C1: assumption cannot satisfy validation_passed               PASS
Fix C1: missing exit_code not trusted                             PASS
Fix C1: legacy evidence file fails safe                           PASS
Fix H1: repeated identical failures increment same counter        PASS
Fix H1: 4th equivalent attempt is blocked                         PASS
Fix H1: materially different action gets new fingerprint          PASS
Fix H1: whitespace/quoting does not reset counter                 PASS
Fix H1: success closes the failure sequence                       PASS
Fix H1: different task does not inherit retry count                PASS
Fix H1: explicit reset clears circuit breaker                     PASS
Fix H1: destructive commands stay protected after circuit trips   PASS
Fix H1: no existing classification becomes weaker                 PASS
Fix H3: new hypothesis gets distinct attempt sequence             PASS
Fix H3: phase retry state persists across restart                 PASS
Fix H2: stale state without task-queue is archived                PASS
Fix H2: recent state is preserved                                 PASS
Fix H2: unfinished task-queue requires resume/archive             PASS
Fix M1: scope-fence excludes bookkeeping                          PASS
Fix M2: command outcome exit_code/bounding/redaction              PASS
Regression: destructive/quoted-text classification suite          PASS
Regression: worktree rollback                                     PASS
---------------------------------------------------------------------------
25 tests — ALL PASS
$ echo $?
0
```

Per-phase acceptance criteria, mapped to the test that proves them (all captured live
during development, exit codes shown where the criterion is itself an exit-code check):

| # | Acceptance criterion (from the task spec) | Test | Result |
|---|---|---|---|
| P1-1 | Fabricated manual evidence does not satisfy the final gate | `final_gate.py` on a fabricated claim → `FAIL — ... TRUSTED evidence ...` | exit 1, **PASS** |
| P1-2 | A real successful test command satisfies validation | `final_gate.py` after `record_hook_evidence(exit_code=0)` → `PASS` | exit 0, **PASS** |
| P1-3 | A failed command is recorded but does not satisfy validation | `final_gate.py` after `record_hook_evidence(exit_code=1)` → `FAIL` | exit 1, **PASS** |
| P1-4 | Evidence from a previous task does not satisfy the current task | New task after t1's trusted evidence → `FAIL` | exit 1, **PASS** |
| P1-5 | An assumption cannot satisfy validation_passed | assumption-type claim → `FAIL` | exit 1, **PASS** |
| P1-6 | Evidence missing an exit code is not trusted | `exit_code=None`, `recorded_by=hook` → `FAIL` | exit 1, **PASS** |
| P1-7 | Historical evidence files fail safely | pre-Fix-C1-schema entry (no `recorded_by`/`exit_code`) → `FAIL`, not silently trusted | exit 1, **PASS** |
| P2-1/2 | 3 equivalent failures increment the same counter; 4th is auto-blocked | Real `command_policy_hook.py` invocation ×4 on `terraform apply` → attempts 1–3 `ask` (escalating notices), attempt 4 `deny` (circuit breaker) | **PASS** (see full transcript in development log; reproduced deterministically in suite) |
| P2-3 | Materially different action gets new fingerprint | `kubectl delete pod x` → plain `ask`, no repeat language | **PASS** |
| P2-4 | Rewording/quoting does not reset the counter | `terraform   apply` (extra whitespace) → same fingerprint, still at circuit-breaker state | **PASS** |
| P2-5 | Success closes the failure sequence | `record_attempt(outcome="success")` → `consecutive_failures` back to 0 | **PASS** |
| P2-6 | Different task does not inherit retry count | fresh task, same fingerprint → count 0 | **PASS** |
| P2-7 | Explicit approved reset clears the breaker | empty reason refused (`SystemExit`); real reason → `attempts_exhausted` False | **PASS** |
| P2-8/9 | Destructive commands stay protected; no classification weaker | `rm -rf /` ×5 and `cat .env` ×5 through the real hook → `deny` every time | **PASS** |
| P3-1..5 | Phase-level retry ceiling, hypothesis-scoped, persists across "restart", success closes, cannot mark complete while open | `start_attempt`/`record_attempt_result`/`get_attempt_count` sequence, fresh-process re-check | **PASS** |
| P4-1..6 | Stale validation-state/task-contract detected without task-queue; archived + excluded; recent state preserved; unfinished task-queue still asks; new task gets new ID | `session_reset_check.py` against a manually-backdated `updated_at` (72h) with no `task-queue.md` → archived + reset; against a fresh state → untouched; against an unfinished task-queue → unchanged resume/archive prompt | **PASS** |
| P5-1 | Exact spec scenario: `scope_allowed=["*.md"]`, changed = `out_of_scope.js` + 2 bookkeeping files → exactly 1 violation | `changed_files_fence.py` → `FAIL — 1 violation(s): OUT_OF_SCOPE: out_of_scope.js` | exit 1, **PASS** (matches spec exactly) |
| P6-1..6 | Success records exit 0; failure records nonzero; secrets redacted; large output bounded; linkable to trusted evidence; existing files stay readable | Real `bash_output_filter.py`/`post_tool_failure.py` invocations + `final_gate.py` PASS driven by a genuine hook-observed success | **PASS** |

---

## 6. Remaining risks

Still enforced only through prompts/documentation, not code — carried over honestly
from the audit, not resolved by this implementation (none of these were in the
Critical/High/Medium phase list):

- **Skills are still prompt-driven.** `ship`/`verified-ship`/`forge-agent` now call a
  real, stateful CLI instead of counting in natural language, but the model still has
  to choose to make that call and to honor `exhausted=true` when it sees it — there is
  no hook that forces a skill's Bash calls to include the `start-attempt` check. A model
  that skips the instruction entirely is not stopped by anything new here.
- **`is_trusted()` can theoretically be defeated by a model writing raw Python** that
  imports `evidence_ledger` directly and calls `record_hook_evidence()` itself, instead
  of going through the real hook path — same residual class of risk every fail-safe
  hook in this codebase already accepts (documented in §4, not new).
- **`worktree_isolate.py`'s `.claude/` exclusion remains broad**, unlike the new
  narrow one in `changed_files_fence.py` — left alone since it was out of Phase 5's
  file list and changing tested, working (M010) behavior risked scope creep.
- **Max-edits / max-wall-clock / max-tool-call ceilings** (Karpathy check 14 from the
  audit) still do not exist in any form — only retry-count and cost are bounded.
- **`commands_run` (the older, pre-existing log) is still unbounded** — Fix L2 was
  explicitly Low priority and not in this task's phase list; `command_outcomes` (the
  new Phase 6 list) is bounded, but the two are separate lists.
- **The `tool_response`/`tool_output` and `error`/`error_message` field-name questions
  were not conclusively resolved against official docs** (both fetches truncated before
  showing a complete PostToolUse/PostToolUseFailure JSON example) — both hooks read
  both possible key names defensively rather than picking one on faith, but this is a
  documented mitigation for an unresolved ambiguity, not a confirmed fact.

---

## 7. Final verdict

- **Can fabricated evidence pass the final gate?** No — `test_fabricated_evidence_rejected`
  and the live audit-style re-run (Test 10b's exact scenario) both now return `FAIL`.
  A manually entered claim can never carry `recorded_by="hook"`.
- **Can an identical failed action continue indefinitely?** No — verified live through
  the real `command_policy_hook.py`: attempts 1–3 escalate in tone/requirements, attempt
  4 is an automatic `deny` with a full escalation summary, and this holds regardless of
  approval mode (Away Mode, Project Autopilot Mode) or whether the command is
  `blocked_hard`/`blocked_secret`/`blocked_needs_privileged`.
- **Is retry state preserved across session restarts?** Yes — state lives in
  `.claude/validation-state.json` on disk; every test that simulates "a fresh process"
  (a brand-new `python3 -c` invocation reading the same file) sees the same exhausted/
  not-exhausted state, because nothing here depends on in-memory state.
- **Can stale state affect a new task?** Only within the residual risks noted above
  (raw-Python bypass; skills remaining prompt-driven). Within the harness's own
  mechanisms: no — `check_stale_state()` now catches and archives old
  `validation-state.json`/`task-contract.md`/`evidence-ledger.json`/
  `source-freshness.json` even with no `task-queue.md` ever created, and
  `is_trusted()`'s task_id + `task_started_at` freshness check independently prevents
  an old task's evidence from satisfying a new one even if archiving were somehow
  skipped.
- **Are harness bookkeeping files excluded correctly?** Yes, narrowly —
  `.claude/validation-state.json` and `.claude/state/**` are excluded from scope-fence
  violations; everything else under `.claude/` (including `task-contract.md` and
  `commands/*.md`) is still normally scope-checked, matching the explicit "do not
  broadly exclude" instruction.
- **Did all existing destructive-action tests continue to pass?** Yes —
  `command_policy.py --test` is still 143/143 after every phase, re-verified after the
  final phase, and the regression suite additionally re-asserts the exact 11-case
  matrix and 5×-repetition behavior the audit tested live, all unchanged.

**Every critical and high-priority acceptance test passes.** No critical test failed
during this implementation, so there was no need to stop, preserve a broken branch, or
avoid syncing on safety grounds — the remaining question is purely whether you want to
proceed with syncing to `~/.claude` now (see below), not whether the work is sound.

---

## Addendum — Phase 1 closure review (post-merge, see PHASE1_FINAL_CHECK.md)

After PR #44 merged and this was synced to `~/.claude`, a follow-up review (concerns A/B/C
from the reliability roadmap) found and fixed two confirmed issues and closed one deferred
item. Full detail in `PHASE1_FINAL_CHECK.md`; summary:

- **Fixed:** `command_policy_hook.py`'s inline comment claimed a privileged grant
  "proceeds regardless of the retry ladder" — true for attempts 1-3, but NOT attempt 4+
  (the circuit-breaker deny fires first and exits before the grant-check code is ever
  reached). Behavior was already safe; the comment was corrected to describe it
  accurately, and 3 new tests now cover all three cases (grant bypasses ask, grant
  bypasses the change-note requirement, grant does NOT bypass a tripped breaker and is
  not erased by the deny).
- **Fixed (Fix L2, previously deferred Low):** `commands_run` was still unbounded — now
  capped at `MAX_COMMANDS_RUN=200` using the same pattern already built for
  `command_outcomes`.
- **Reviewed, not changed:** evidence-forgery vectors (manual file edits, cross-task
  copying, direct internal-function calls) — all classified as theoretical
  trust-boundary limitations inherent to a JSON-state-file design with no cryptographic
  signing, same class of limitation every hook in this harness already accepts. A
  moderate-value hardening idea (cross-referencing evidence-ledger entries against
  `command_outcomes`) was identified and deferred to the backlog rather than built now.
- 31/31 regression tests pass (29 → 31 with the new coverage); `command_policy.py --test`
  still 143/143; `harness_check.sh` PASS.

---

## Sync history

- **2026-07-22, first sync:** all Phase 1-7 fixes from this report installed into
  `~/.claude` via `install.sh` after explicit confirmation. Verified file-identical,
  live end-to-end smoke test passed (4x-repeat escalation via the real installed hook).
- **Pending:** the Phase 1 closure fixes above (addendum) are on a new branch,
  not yet merged/synced — see `PHASE1_FINAL_CHECK.md` and `PHASE1_INSTALL_REPORT.md`
  for that round's verdict and installation record.
