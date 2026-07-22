# Harness Backlog

Non-blocking future ideas surfaced during the reliability audit/implementation and the
Phase 1-6 roadmap continuation. **Nothing here is implemented or scheduled** — this is
a list to consider on request, not a plan. The harness is frozen at a
READY-FOR-DAILY-PROJECT-WORK state (see `FINAL_HARNESS_EVALUATION.md`); adding items
here is not a commitment to build them.

---

## Hardening

- **Cross-reference evidence-ledger entries against `command_outcomes`** before
  treating them as trusted (same task, matching command/timestamp). Would not stop a
  maximally deliberate direct-function-call forgery, but would raise the bar against
  copy-paste and partial hand-edit forgery. (`PHASE1_FINAL_CHECK.md`, concern B)
- **Narrow `worktree_isolate.py`'s `.claude/` exclusion** to match
  `changed_files_fence.py`'s tighter one (currently broad, safe but inconsistent —
  see `PHASE1_FINAL_CHECK.md`, concern C).
- **Real max-edits / max-wall-clock / max-tool-call ceilings.** No usage data yet to
  size sensible defaults — would need real long-session data before designing this,
  not a guess at numbers.

## Optimization

- **Reduce always-loaded context below the harness's own ~200-line historical
  target** without cutting `rules/troubleshooting.md`'s genuinely valuable content —
  a real content/editing exercise (condense, not delete), flagged but not attempted
  in `CONTEXT_ENGINEERING_REPORT.md` since it's a product judgment call, not a
  mechanical cleanup.
- **`commands_run`/`command_outcomes` archival** beyond the current 200-entry cap
  (e.g. roll dropped entries into a compressed archive file instead of discarding)
  if a real need for historical command audit beyond the cap ever comes up.

## Usability

- **Retire or wire up `rules/behavior.md`'s "Response counter" rule** ("every
  response ends with `[RN]`") — currently unenforced by any hook and wasn't followed
  during this session's long autonomous run with no ill effect, since `TaskCreate`/
  `TaskUpdate` already covers progress tracking. Flagged as a recommendation in
  `CONTEXT_ENGINEERING_REPORT.md`, not changed (this was the user's own recent,
  apparently deliberate edit to keep).
- **Live-dispatch routing tests for the AWS/Kubernetes/architecture scenarios** that
  Phase 6 evaluated only by routing-table trace, not independent live dispatch (see
  `FINAL_HARNESS_EVALUATION.md` limitations).
- **Exercise `automation-reviewer` against a real pipeline/workflow** from an actual
  project, not just the synthetic test fixture used in Phase 6 — first real use will
  tell if its review dimensions need adjustment.
- **Reconcile the outer platform's auto-injected "RESPONSE FORMAT" reminder text**
  with `rules/behavior.md`'s current section names — not fixable from this repo (the
  reminder isn't generated from any file here), but worth raising with the platform
  if this becomes confusing in practice (`CONTEXT_ENGINEERING_REPORT.md`).

## Future research

- **More structured dependency awareness** (missing credentials, unavailable
  services, blocked-by-another-task) if the existing free-text `blocker` field in
  `task-queue.md` proves insufficient in real use — not built now since it was
  judged adequate today (`AWARENESS_IMPLEMENTATION_REPORT.md`).
- **Re-check `github.com/ComposioHQ/awesome-claude-skills` (or similar catalogs)
  periodically** for a genuine GCP/Kubernetes/Terraform/cloud-architecture skill —
  today's snapshot (164 entries, reviewed in full) had none, but catalogs grow.
- **A confidence-labeling code check** (e.g. flag a subagent review response that
  omits any confidence marker) — noted as a possible enhancement in the original
  audit's Karpathy review, not built (prompt-level convention only, same as most of
  this harness's non-hook-enforced rules).
