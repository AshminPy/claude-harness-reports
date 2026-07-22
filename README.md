# Claude Code Harness — Reliability Reports

A set of reports from a reliability audit, implementation, and hardening pass on a
personal Claude Code harness (hooks, retry/circuit-breaker logic, evidence-trust
model, skills routing, and lightweight task awareness). Published standalone so the
findings/design can be shared and discussed without exposing the private harness
repo itself.

## Reading order

0. **[HARNESS_GUIDE.md](HARNESS_GUIDE.md)** — **start here.** The canonical reference:
   architecture, how it works, update/rollback procedure, skill architecture, rule
   hierarchy, how to add a skill, how to debug problems, known limitations, and design
   principles. Everything below is historical record of how it got built and verified;
   this is the one doc to read for the current state.
1. **[HARNESS_RELIABILITY_AUDIT.md](HARNESS_RELIABILITY_AUDIT.md)** — the original
   audit: 25 reliability controls + 16 "Karpathy-inspired" checks, evidence-backed,
   with live behavioral tests (including a proof that fabricated evidence could pass
   the completion gate, and that a blocked command could be retried indefinitely with
   no memory of prior attempts).
2. **[HARNESS_RELIABILITY_IMPLEMENTATION.md](HARNESS_RELIABILITY_IMPLEMENTATION.md)** —
   the fix: trusted-evidence provenance, a retry/circuit-breaker model, stale-state
   protection, bounded command-outcome logging, and a 25-test regression suite.
3. **[PHASE1_FINAL_CHECK.md](PHASE1_FINAL_CHECK.md)** /
   **[PHASE1_INSTALL_REPORT.md](PHASE1_INSTALL_REPORT.md)** — a follow-up review
   (retry-vs-authorization interaction, evidence-forgery threat model) and the
   install/rollback record.
4. **[CONTEXT_ENGINEERING_REPORT.md](CONTEXT_ENGINEERING_REPORT.md)** — auditing the
   always-loaded context budget against a 7-level hierarchy (global → project → rules
   → skills → subagents → task state → evidence).
5. **[SKILLS_IMPLEMENTATION_REPORT.md](SKILLS_IMPLEMENTATION_REPORT.md)** — reviewing
   164 external skill candidates against the harness's actual priorities (result:
   0 imported, 1 small local agent written for a confirmed gap).
6. **[AWARENESS_IMPLEMENTATION_REPORT.md](AWARENESS_IMPLEMENTATION_REPORT.md)** —
   lightweight task/confidence/completion awareness added on top of the existing state.
7. **[FINAL_HARNESS_EVALUATION.md](FINAL_HARNESS_EVALUATION.md)** — 12 representative
   end-to-end scenarios, including two live subagent tests against deliberately flawed
   fixtures.
8. **[ROADMAP.md](ROADMAP.md)** / **[HARNESS_OPERATIONS.md](HARNESS_OPERATIONS.md)** /
   **[HARNESS_BACKLOG.md](HARNESS_BACKLOG.md)** — what's done, how to operate it
   day-to-day, and non-blocking future ideas.

## What this is / isn't

This repo is **reports only** — no hook/script source code, no personal notes, no
infrastructure identifiers. It exists to share findings and get feedback, not to be
installed as-is.
