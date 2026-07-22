# Phase 1 Final Check — Reliability Work Closure

Follow-up review of the merged reliability implementation (PR #44), per the autonomous
roadmap. Scope: `HARNESS_RELIABILITY_AUDIT.md`, `HARNESS_RELIABILITY_IMPLEMENTATION.md`,
every hook/script the implementation touched, retry logic, evidence logic, task-state
logic, installation scripts, and the regression suite.

---

## Reviewed scope

- `hooks/command_policy_hook.py`, `hooks/bash_output_filter.py`, `hooks/post_tool_failure.py`
- `scripts/validation_state.py`, `scripts/evidence_ledger.py`, `scripts/final_gate.py`
- `scripts/session_reset_check.py`, `scripts/changed_files_fence.py`
- `scripts/test_reliability_fixes.py` (existing 25-test suite)
- `scripts/harness_check.sh`, `install.sh`
- `skills/ship/SKILL.md`, `skills/verified-ship/SKILL.md`, `skills/forge-agent/SKILL.md`
- All "Remaining risks" items in `HARNESS_RELIABILITY_IMPLEMENTATION.md` §6

Method: re-read every file directly (not from memory), traced the actual control flow
for the two specific concerns below line-by-line, then verified each conclusion with a
real, executable test — not asserted from reading alone.

---

## Concern A — Retry circuit and privileged approval

**Question:** does privileged/destructive-command approval accidentally erase retry
history, bypass retry exhaustion, or confuse authorization with reliability controls?

**Traced behavior** (`hooks/command_policy_hook.py`):
1. The hard circuit-breaker check (`_prior_failures >= vs.DEFAULT_MAX_RETRIES`, line ~97)
   runs immediately after classification, for every `blocked_*` classification, and
   calls `_decide("deny", ...)` — which exits the process — BEFORE the code ever reaches
   the `blocked_needs_privileged` branch further down (line ~255) where
   `vs.consume_privileged_grant()` is checked.
2. **Consequence:** once a command has failed/been blocked 3+ times in a task, a valid,
   unconsumed privileged grant for that exact command CANNOT be consumed — the circuit
   breaker denies unconditionally. The grant is not erased or lost; it simply is never
   reached, and remains available (confirmed by inspecting `state["privileged_grants"]`
   after a circuit-breaker deny — unchanged, `consumed: false`).
3. For attempts 1-2 (`_prior_failures` 0 or 1) and attempt 3's change-note requirement
   (`_prior_failures == 2`), the grant-check IS reached and a valid grant bypasses both
   the plain "ask" and the change-note requirement — a human's real-time explicit
   approval outranks a self-authored "what changed" note.
4. Consuming a grant never itself records an attempt outcome (no call to
   `record_attempt`) — the real success/failure is recorded later, post-execution, by
   `bash_output_filter.py`/`post_tool_failure.py`. Authorization and retry-outcome
   tracking are structurally separate: a grant only ever affects whether the command is
   *allowed to run*, never whether an "attempt" is logged or a sequence is closed.

**Defect found:** the inline code comment at the grant-consumption site claimed the
grant "proceeds regardless of the retry ladder above" — true for attempts 1-3, **false**
for attempt 4+ (the circuit-breaker deny fires first). This is a documentation-accuracy
defect: a future reader (human or a future Claude session) could reasonably conclude
grants always override retry exhaustion, which is not what the code does.

**Verdict on the actual (not documented) behavior:** SAFE, and now confirmed
INTENTIONAL. Authorization and retry-exhaustion are independent by construction —
retry-exhaustion is checked strictly earlier in the control flow and always wins, so a
stale or blanket grant can never silently reactivate a tripped breaker. This is the
more conservative of the two possible designs and was left unchanged.

**Fix applied:**
- Corrected both comments (the hard-gate block and the grant-consumption block) to
  accurately describe the real interaction.
- The circuit-breaker escalation message now checks for and mentions a pending,
  unconsumed matching grant, so the user isn't confused about why an earlier approval
  "didn't work" (it's still there, waiting on an explicit `reset-attempts`).
- Added 3 new regression tests (see Tests run) that had no prior coverage: grant
  bypasses ask (attempts 1-2), grant bypasses change-note requirement (attempt 3), grant
  does NOT bypass a tripped breaker and is NOT erased by the deny (attempt 4+).

---

## Concern B — Trusted evidence forgery

**Question:** can trusted evidence realistically be forged via manual file creation,
editing ordinary state, copying evidence across tasks, or directly invoking
`record_hook_evidence()`/equivalent internal functions?

| Vector | Feasible? | Classification |
|---|---|---|
| Manually creating/hand-editing `evidence-ledger.json` with `recorded_by:"hook"`, `exit_code:0` | Yes — `is_trusted()` reads whatever fields are present in the JSON; there is no cryptographic signature binding an entry to a real execution | **Theoretical trust-boundary limitation** |
| Editing ordinary state (e.g. `validation-state.json` `checks`) directly | Yes, same reason | **Theoretical trust-boundary limitation** |
| Copying a trusted entry from another task and editing its `task_id` | Yes — `task_id` is a self-reported string field with no external binding; this is mechanically the same act as the two rows above (a direct file edit) | **Theoretical trust-boundary limitation** |
| Directly invoking `evidence_ledger.record_hook_evidence()` (or `evidence_ledger.py`'s internals) from a Python one-liner run via the Bash tool, with a fabricated `exit_code=0` and no real command behind it | Yes — nothing distinguishes "called from the real PostToolUse hook" from "called from an arbitrary Bash-invoked Python process"; both use the identical import + function call | **Theoretical trust-boundary limitation**, but flagged as the practically most significant of the four — it requires no unusual-looking file surgery, only a normal Bash/Python invocation indistinguishable in form from many legitimate diagnostic commands |

**Why none of these are classified as a confirmed reliability defect:** the CLI path
(`evidence_ledger.py add-claim`) — the only evidence-writing surface reachable via an
ordinary, expected command — correctly and verifiably can never produce a trusted
entry (no `recorded_by`/`exit_code` argument exists on that CLI path; regression-tested).
All four vectors above require the agent to deliberately step outside the intended,
designed usage pattern (either raw file editing or directly importing an internal
module function instead of letting the real hook call it) — this is the same class of
limitation every fail-safe hook in this codebase already accepts (documented
repeatedly elsewhere in this harness, e.g. self-unlock command blocks, guard_secrets'
filename-heuristic-not-content-scan approach). The implementation performs exactly as
designed for its stated threat model: preventing *casual or accidental* fabrication
through normal usage, not defending against a fully adversarial agent with unrestricted
code execution — the latter is a materially larger engineering problem (out-of-process
attestation, signed entries, a privilege-separated evidence writer) that would be
disproportionate "infinite complexity" relative to this environment's actual daily use.

**Future hardening opportunity identified (not implemented, logged to
`HARNESS_BACKLOG.md`):** cross-reference `evidence-ledger.json` entries against
`validation-state.json`'s `command_outcomes` (same task, matching command/timestamp)
before treating an entry as trusted. This would not stop a maximally deliberate direct-
function-call attacker (who could populate both structures), but would meaningfully
raise the bar against copy-paste forgery, partial hand-edits, and sloppy/non-deliberate
tampering, at moderate implementation cost. Deferred per the instruction to fix
confirmed defects, not build theoretical hardening, during this phase.

---

## Concern C — Remaining implementation limitations

Reviewed every item in `HARNESS_RELIABILITY_IMPLEMENTATION.md` §6:

| Item | Disposition |
|---|---|
| Skills remain prompt-driven (model must choose to call the retry helpers) | Inherent to a prompt-based skill system — not a defect, no code fix possible without a different architecture. Left as documented limitation. |
| `record_hook_evidence()` defeatable by raw Python (concern B) | Reviewed above — theoretical, deferred to backlog. |
| `worktree_isolate.py`'s `.claude/` exclusion remains broad (unlike `changed_files_fence.py`'s narrow one) | Reviewed: worst case is a slightly conservative isolation decision (worktree created when in-place-fallback might have been marginally more cautious) — does not cause data loss or incorrect command execution. Not a defect affecting normal operation; left as a backlog optimization. |
| Max-edits/wall-clock/tool-call ceilings do not exist | Known, scoped-out gap, not a regression. Left for backlog (would need real usage data before designing meaningful defaults). |
| `commands_run` still unbounded | **Fixed** — see below. Cheap, safe, additive; reuses the exact bounding pattern already proven for `command_outcomes` in Phase 6. |
| `tool_response`/`tool_output` and `error`/`error_message` field-name ambiguity | Already mitigated as well as reasonably possible (both keys read defensively). No further action — re-confirming official docs would require access this environment doesn't reliably have; the defensive dual-read is the correct mitigation either way. |

**Fix applied:** `validation_state.log_command()` now caps `commands_run` at
`MAX_COMMANDS_RUN = 200` (oldest entries dropped first), identical in spirit to
`MAX_COMMAND_OUTCOMES`. Verified: after 250 simulated log_command calls, the list holds
exactly 200 entries and the most recent one is preserved (not dropped).

---

## Additional check — corrupted/partial state fails safe

Not previously covered by an explicit test (only an "old-schema-but-valid-JSON" case
was tested). Verified directly and now permanently regression-tested:
- A truncated/invalid `validation-state.json` → `load_state()` catches the parse error
  and returns `default_state()` (fresh, safe defaults) rather than raising or returning
  partial/garbage data.
- A non-JSON `evidence-ledger.json` → `load_ledger()` returns `[]` rather than raising;
  `has_trusted_evidence()` correctly returns `False` against it.

No defect found — both were already fail-safe by design (`try/except: return default`),
simply not previously asserted by name in the test suite.

---

## Fixes made (summary)

1. `hooks/command_policy_hook.py` — corrected two misleading comments about the
   authorization/retry-exhaustion interaction; added pending-grant visibility to the
   circuit-breaker escalation message.
2. `scripts/validation_state.py` — added `MAX_COMMANDS_RUN` cap to `log_command()`.
3. `scripts/test_reliability_fixes.py` — added 6 new tests: 3 for concern A
   (grant-bypasses-ask, grant-bypasses-change-note, grant-does-not-bypass-tripped-breaker),
   1 for the `commands_run` cap, 2 for corrupted-state fail-safe behavior.
4. `HARNESS_RELIABILITY_IMPLEMENTATION.md` — addendum documenting this review's findings.

No changes to: `command_policy.py`'s destructive-command classification logic,
`block_push.py`, `guard_secrets.py`, or any approval-tier threshold.

---

## Tests run

```
$ python3 scripts/test_reliability_fixes.py
[... 31 tests ...]
31 tests — ALL PASS
$ echo $?
0

$ bash scripts/harness_check.sh "$PWD"
PASS — harness check clean (/Users/ashmin/projects/ai-projects/claude-toolkit)
$ echo $?
0

$ python3 scripts/command_policy.py --test
[...]
ALL 143 PASS
```

Full, real command output (not summarized) — every one of the following ran and passed
on this branch, re-verified immediately before writing this report:

- All 25 pre-existing regression tests (Phases 1-6 of the original implementation).
- 6 new tests from this review (listed above).
- `command_policy.py --test`: 143/143, unchanged.
- `harness_check.sh`: PASS (which itself runs the full `test_reliability_fixes.py` suite
  as part of its own gate, per the Phase 7 wiring).
- `python3 -m py_compile` on every modified file: clean.

---

## Retry and approval review

See Concern A above. Verdict: **behavior is safe and correctly independent**;
documentation was the only defect, now fixed. No change to actual authorization or
retry-exhaustion logic was needed or made.

## Evidence trust review

See Concern B above. Verdict: **no confirmed defect** in the intended usage path (CLI,
hooks). Four theoretical trust-boundary limitations identified, classified, and
explicitly not "fixed" here since none constitute a defect in normal (non-adversarial)
operation — doing so would require disproportionate complexity for this environment's
threat model. One future hardening idea logged to the backlog.

## Deferred hardening items

- Cross-reference evidence-ledger entries against `command_outcomes` before trusting
  them (raises the bar against copy/paste and partial-hand-edit forgery).
- Narrow `worktree_isolate.py`'s `.claude/` exclusion to match `changed_files_fence.py`'s
  tighter one (currently broad — safe, but inconsistent).
- Real max-edits / max-wall-clock / max-tool-call ceilings (no usage data yet to size
  sensible defaults).

## Deferred optimizations

- None beyond the `commands_run` cap fixed in this phase; the only outstanding
  "unbounded growth" item left was the `command_outcomes`-adjacent `commands_run`
  fix itself, now closed.

## Known limitations

- Skills remain prompt-driven; no hook forces a skill to call its retry-tracking helpers.
- Evidence trust and retry-tracking both assume non-adversarial execution of this
  harness's own Python by the agent — a deliberately adversarial agent with full Bash
  access could bypass either by writing custom code, same as it could bypass most
  local, non-sandboxed safety tooling.
- Max-edits/wall-clock/tool-call bounding does not exist.

## Repository commit hash

Committed on branch `feature/harness-phase1-closure`, based on `init` at merge commit
`261bcea` (PR #44). This report's own commit hash: see the commit that includes this
file (`git log -1 --oneline -- PHASE1_FINAL_CHECK.md` after commit).

## Final verdict

**PASS — READY TO INSTALL**
