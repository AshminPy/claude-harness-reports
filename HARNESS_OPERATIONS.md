# Harness Operations Guide

Short, practical reference for running and maintaining this harness day-to-day. For
the full design rationale, see the individual `HARNESS_*`/`PHASE*`/`*_REPORT.md` files
this repo already has — this doc is the quick-reference layer on top of those.

## How the harness works

- **Source of truth:** this repo (`~/projects/ai-projects/claude-toolkit`). Everything
  live at `~/.claude` is a *mirror*, synced by `install.sh` — never edit `~/.claude`
  directly; edit the repo, then sync.
- **Enforcement lives in code** (`hooks/`, `scripts/`), not just prose. The
  destructive-command gate (`command_policy_hook.py`), retry/circuit-breaker
  (Phases 1-2 of the reliability work), evidence-trust model (Phase 1), and
  stale-state protection (Phase 4) are all real Python, tested by
  `scripts/test_reliability_fixes.py` and `scripts/command_policy.py --test`.
- **Knowledge lives in skills/agents**, loaded on demand, not always-on. Always-loaded
  context is deliberately kept small (`~/.claude/CLAUDE.md` + `rules/*.md`) — see
  `CONTEXT_ENGINEERING_REPORT.md`.
- **Task state** lives in `<project>/.claude/validation-state.json` (machine-readable)
  and `<project>/.claude/state/task-queue.md` (human-readable), per-project, not
  global.
- **Evidence** lives in `<project>/.claude/state/evidence-ledger.json` — only entries
  with `recorded_by: "hook"` and a real `exit_code: 0` can satisfy a STRICT-mode
  completion claim; anything entered manually (`add-claim`) never can.

## How to install or update

```bash
cd ~/projects/ai-projects/claude-toolkit
bash scripts/harness_check.sh "$PWD"   # must PASS before installing
bash install.sh                         # idempotent, safe to re-run
```
`install.sh` only ever copies from `skills/`, `hooks/*.py`, `hooks/*.sh`, `agents/*.md`,
`rules/*.md`, `mistakes/*.yaml`, `templates/*.md`, `policies/*.md`, `scripts/*.py`,
`scripts/*.sh`, `.claude/commands/*.md` — it never touches `settings.json`, secrets,
runtime state, or any per-project `.claude/` directory. It never deletes anything.

**After adding a brand-new skill or agent**, a fresh Claude Code session is required
before it's dispatchable by name (confirmed in `FINAL_HARNESS_EVALUATION.md` — the
available subagent-type list is fixed for a session's lifetime).

## How to roll back

Every install this roadmap performed took a timestamped backup first:
```bash
ls ~/.claude/backups/
```
To restore a specific file:
```bash
cp ~/.claude/backups/<backup-dir>/<path>/<file> ~/.claude/<path>/<file>
```
Nothing is ever deleted by this process — a rollback is always a plain file copy back
over the live version. After rolling back, re-run `harness_check.sh` to confirm.

## How skills are selected

- Routing is description-based: `.claude/commands/review.md` (reviewers),
  `policies/subagent-routing.md` (all subagents), `policies/skill-routing.md` (skills)
  are the three tables that determine which skill/agent handles a given request.
- Before adding a new skill/agent: check these three tables first — if an existing
  entry already covers the request, don't add a duplicate (see
  `SKILLS_IMPLEMENTATION_REPORT.md` for a worked example: 164 external candidates
  reviewed, 0 imported, because existing coverage was already stronger).
- External skills (e.g. from a catalog/awesome-list) need individual review before
  import: read the actual `SKILL.md`, scripts, dependencies, permissions, network
  behavior, and maintenance status — don't install a package sight-unseen.

## Where reports and state live

| What | Where |
|---|---|
| Harness-level audit/implementation/evaluation reports | Repo root (`HARNESS_*.md`, `PHASE*.md`, `*_REPORT.md`, this file, `ROADMAP.md`, `HARNESS_BACKLOG.md`) |
| Per-project deploy state | `~/projects/ai-projects/claude-toolkit/deploy-states/<project>.md` |
| Per-project task/retry/evidence state | `<project>/.claude/validation-state.json`, `.claude/state/*.json`, `.claude/state/task-queue.md` |
| Investigation state (troubleshooting workflow) | `<project>/CURRENT_STATE.md`, `TROUBLESHOOTING_LOG.md`, `FINAL_RCA.md` (created on demand, per `rules/troubleshooting.md`) |
| Mistake memory (auto-injected on matching prompts) | `mistakes/mistakes.yaml` (active) / `mistakes/candidates.yaml` (pending human promotion) |
| Personal cross-session memory | `~/.claude/projects/-Users-ashmin-projects/memory/` (index: `MEMORY.md`) |

## How to diagnose common problems

- **"A hook isn't firing / behaving as expected"** — check
  `~/.claude/settings.json`'s `hooks` block first (is it actually wired to the right
  event/matcher?), then check `~/.claude/logs/prompt-audit.log` /
  `~/.claude/logs/tool-failures.log` for what actually ran.
- **"final_gate keeps blocking / won't clear"** — run
  `python3 scripts/validation_state.py show <project-root>` and
  `python3 scripts/final_gate.py <project-root>` directly to see the exact reasons;
  it always prints every blocking reason, never a bare fail.
- **"A command I expected to be blocked wasn't, or vice versa"** — run
  `python3 scripts/command_policy.py --test` first (does the whole suite still pass?);
  then `python3 -c "import command_policy as cp; print(cp.classify('<the command>',
  'project-autopilot'))"` to see the exact classification.
- **"I don't know where things stand on a long task"** — run
  `python3 scripts/validation_state.py awareness <project-root>` for one aggregated
  view (task, phase, open failure sequences, assumptions, evidence count, whether
  human input is required), or `completion-checklist <project-root>` before claiming
  something is done.
- **"A newly added skill/agent isn't showing up"** — start a new session (see "How to
  install or update" above).

## How to add a new skill safely

1. Check `policies/skill-routing.md` / `policies/subagent-routing.md` — does an
   existing skill/agent already cover this? If yes, don't add a duplicate.
2. If genuinely new: write it locally, following an existing skill/agent's exact
   structure as a template (frontmatter shape, output format, tool restrictions).
   Prefer read-only tools (`Read, Grep, Glob, Bash`) unless write access is the
   explicit point.
3. Add it to the relevant routing table (`review.md`, `subagent-routing.md`, or
   `skill-routing.md`).
4. Run `bash scripts/harness_check.sh "$PWD"` — must still PASS.
5. `bash install.sh` to sync.
6. Start a new session, then verify the new skill/agent is actually dispatchable and
   produces a sensible result against a small, deliberately-flawed test fixture before
   trusting it on real work (see `FINAL_HARNESS_EVALUATION.md`'s methodology for an
   example).
7. If it's an *external* skill: individually inspect it first (see "How skills are
   selected" above) — never install a whole package/repo wholesale.
