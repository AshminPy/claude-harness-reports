# Phase 1 Install Report — Reliability Harness (Closure Round)

## Verified source commit

Branch `feature/harness-phase1-closure`, commit `7ec0d1b` ("Phase 1 closure: fix
retry/approval comment defect, cap commands_run, add tests"), based on `init` at
`261bcea` (PR #44 merge).

## Installation command

```bash
bash install.sh
```
(repository's supported, idempotent installer — copies skills/hooks/agents/rules/
mistakes/templates/policies/scripts/commands into `~/.claude`; never touches
`settings.json`, secrets, runtime state, caches, or per-project `.claude/` directories.)

## Pre-install inspection

`install.sh` was re-read before running. It only ever `cp -R`/`cp -f`s from:
`skills/`, `hooks/*.py`, `hooks/*.sh`, `agents/*.md`, `rules/*.md`, `mistakes/*.yaml`,
`templates/*.md`, `policies/*.md`, `scripts/*.py`, `scripts/*.sh`,
`.claude/commands/*.md` — none of these paths can contain runtime state, secrets,
caches, reports, or temp files (those live under a *project's* `.claude/` directory,
never under this repo's top-level tracked dirs). Confirmed by diffing the exact file
list this round would touch against what's already live: only 3 files actually
differ — `hooks/command_policy_hook.py`, `scripts/validation_state.py`,
`scripts/test_reliability_fixes.py`. Everything else in the repo is byte-identical to
what was already installed from PR #44's sync.

## Files installed (this round — only what actually changed)

- `hooks/command_policy_hook.py` (updated)
- `scripts/validation_state.py` (updated)
- `scripts/test_reliability_fixes.py` (updated)

All other files `install.sh` touches were re-copied but byte-identical (no functional
change) to what was already live.

## Files preserved

Everything under `~/.claude` not in the installer's copy list: `settings.json`,
`settings.local.json`, `logs/`, `state/`, per-project `.claude/` directories, and the
one known pre-existing local-only file `scripts/cost_report.sh` (unrelated to this
work, present before and after, not touched).

## Backup location

```
~/.claude/backups/phase1-closure-20260722-021403/
  hooks/command_policy_hook.py       (pre-change version)
  scripts/validation_state.py         (pre-change version)
  scripts/test_reliability_fixes.py   (pre-change version)
```

## Rollback procedure

```bash
BACKUP=~/.claude/backups/phase1-closure-20260722-021403
cp "$BACKUP/hooks/command_policy_hook.py" ~/.claude/hooks/command_policy_hook.py
cp "$BACKUP/scripts/validation_state.py" ~/.claude/scripts/validation_state.py
cp "$BACKUP/scripts/test_reliability_fixes.py" ~/.claude/scripts/test_reliability_fixes.py
```
Nothing was deleted; nothing else in `~/.claude` was touched by this round, so this
3-file copy fully reverses it. `bash scripts/harness_check.sh "$PWD"` re-run after
rollback would confirm the pre-change state is restored.

## Post-install tests

```
$ python3 -c "import json; json.load(open('~/.claude/settings.json'))"   # valid JSON, 10 hook event types (unchanged)
$ bash ~/.claude/scripts/harness_check.sh <repo path>
PASS — harness check clean
```

### Source-to-installed comparison

```
hooks/command_policy_hook.py:       IDENTICAL
scripts/validation_state.py:        IDENTICAL
scripts/test_reliability_fixes.py:  IDENTICAL
```
File-list diff of `hooks/` and `scripts/` directories (repo vs. live): only expected,
pre-existing difference is `scripts/cost_report.sh` (live-only, unrelated, predates
this work — noted in memory from an earlier session, not a regression here).

### Permissions

All three files installed executable (`-rwxr-xr-x`), matching `install.sh`'s
`chmod +x` step.

### Installed-path smoke test (real `~/.claude` paths, no simulation)

| Check | Result |
|---|---|
| Successful evidence creation → `final_gate` PASS | `reasons: []` ✅ |
| Fabricated evidence → `final_gate` still rejects it | non-empty reason returned, correctly blocked ✅ |
| Retry tracking, 4x repeat of `terraform apply` via the real installed hook | `ask, ask, ask, deny` ✅ |
| Stale-state rejection (backdated `updated_at`, no task-queue) | archived + reset message produced ✅ |
| Destructive-command approval protection (`classify()` via installed `command_policy.py`) | `terraform apply`→`blocked_needs_privileged`, `rm -rf /`→`blocked_hard`, `git push origin main`→`blocked_needs_privileged`, `ls -la`→`allowed` — all correct ✅ |

## Known limitations

Same as `PHASE1_FINAL_CHECK.md`'s "Known limitations" section — carried forward, not
re-litigated here.

## Final installation status

**SUCCESS.** All 3 changed files installed, verified byte-identical to source,
correct permissions, `harness_check.sh` PASS on the live copy, and all 5 required
smoke-test categories passed against the real installed hooks/scripts.
