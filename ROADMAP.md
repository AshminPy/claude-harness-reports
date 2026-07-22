# Harness Roadmap

Status: **FROZEN** as of this evaluation — see `FINAL_HARNESS_EVALUATION.md` for the
readiness verdict (READY FOR DAILY PROJECT WORK). Future work happens from
`HARNESS_BACKLOG.md`, on demand, not by continuing this roadmap indefinitely.

## Completed

### Reliability audit and implementation
- [x] `HARNESS_RELIABILITY_AUDIT.md` — 25-control + 16-Karpathy-check audit, 10 live
  behavioral tests, evidence-backed findings.
- [x] `HARNESS_RELIABILITY_IMPLEMENTATION.md` — 7 phases implementing the
  Critical/High/Medium fixes (trusted evidence provenance, retry tracking, enforced
  skill retry limits, stale-state protection, scope-fence bookkeeping exclusion,
  bounded command-outcome persistence, deterministic regression suite). PR #44,
  merged.
- [x] Installed into `~/.claude` and verified end-to-end (live 4x-repeat escalation
  test against the real installed hook).

### Autonomous continuation (this run)
- [x] **Phase 1 — Close the reliability work**: reviewed the retry/authorization
  interaction (confirmed safe, fixed a misleading comment), reviewed evidence-forgery
  vectors (all theoretical, none confirmed defects), fixed the one confirmed low-cost
  gap (`commands_run` unbounded growth), added 6 new regression tests (25 → 31).
  Verdict: PASS — READY TO INSTALL. PR #45.
- [x] **Phase 2 — Install + verify**: backed up, installed, byte-for-byte compared,
  5-category smoke test against the real installed hooks. SUCCESS.
- [x] **Phase 3 — Context engineering**: protected `rules/troubleshooting.md` (was
  live-only, at risk of loss — now committed), audited the full context hierarchy
  (confirmed already well-designed), documented (not force-cut) the always-loaded
  context total vs. the harness's own historical target.
- [x] **Phase 4 — Skills audit**: reviewed all 164 candidates in
  `ComposioHQ/awesome-claude-skills`, found zero genuinely relevant to this
  environment's AWS/GCP/Kubernetes/Terraform priorities, imported nothing, wrote one
  new local agent (`automation-reviewer`) for the one confirmed gap (general
  automation/pipeline-design review). Routing verified against 10 representative
  prompts.
- [x] **Phase 5 — Practical awareness**: added `assumptions`/`open_questions` state,
  `awareness_snapshot()` (read-time aggregator, no new persisted file),
  `completion_checklist()` (informational, any risk mode, never a new blocking gate),
  and the `ASSUMED` confidence label in `rules/troubleshooting.md`.
- [x] **Phase 6 — End-to-end evaluation**: 12 representative scenarios evaluated; 2
  real live subagent dispatches against deliberately flawed fixtures (both caught the
  planted defects correctly); found and documented one real limitation (new agents
  need a fresh session to become dispatchable by name). Verdict: READY FOR DAILY
  PROJECT WORK.
- [x] **Phase 7 — Final documentation and freeze** (this phase): `ROADMAP.md`,
  `HARNESS_OPERATIONS.md`, `HARNESS_BACKLOG.md`.

Final regression state: 36/36 deterministic tests pass, 143/143 destructive-command
classification tests pass, `harness_check.sh` PASS — all re-verified at the end of
every phase in this run, not just at the start.

## Not on this roadmap (see `HARNESS_BACKLOG.md` instead)

Nothing further is planned by default. The explicit goal was to reach a stable,
trustworthy, daily-usable harness — not to keep expanding it. Backlog items are
optional, non-blocking, and only actioned on request.
