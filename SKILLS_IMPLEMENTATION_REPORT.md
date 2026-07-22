# Skills Implementation Report

## Candidates reviewed

Fetched the complete catalog at `github.com/ComposioHQ/awesome-claude-skills`
(164 skills total, verified via direct fetch, not assumed from memory). Full breakdown
by category:

| Category | Count | Relevant to this environment's priorities? |
|---|---|---|
| App/SaaS automation (CRM, PM tools, email, social media, e-commerce, analytics, HR, calendars, design tools — all via Composio connectors) | ~120 | No — none touch AWS/GCP/Kubernetes/Terraform/cloud infra |
| Document processing (docx/pdf/pptx/xlsx) | 4 | No — already covered by this harness's existing `anthropic-skills:*` doc skills |
| Creative/media, business/marketing, communication/writing | ~25 | No |
| Development & code tools (generic) | ~15 | Partial — see below |
| Data & analysis / research | 4 | Partial — see below |
| Security & systems (forensics, Sigma rules) | 4 | Partial — different focus (incident forensics, not pre-deployment review) |
| Assistive technology | 1 | No |

**Finding, stated plainly:** this catalog has **zero** GCP-specific skills, **zero**
Kubernetes-specific skills, **zero** Terraform-specific skills, and **zero**
cloud-architecture-review skills. It is overwhelmingly a directory of third-party
SaaS/business-app integrations (CRM, marketing, e-commerce, social media), not
infrastructure/cloud engineering tooling. This is a real, evidence-based finding, not
an assumption — the full list was fetched and categorized above.

### Individually inspected candidates (the ones with any plausible relevance)

| Candidate | Inspected for | Verdict | Reason |
|---|---|---|---|
| `aws-skills` (@zxkane) | Relevance, stack match | **Rejected** | Focused on AWS CDK — this environment's actual IaC stack is Terraform/Spacelift (`rules/cloud.md`), not CDK. Importing a CDK-oriented skill would encourage a stack mismatch, not close a gap. |
| `deep-research` (@sanjay3290) | Dependencies, permissions | **Rejected** | Depends on the Gemini CLI/API — an external, separately-keyed dependency this environment doesn't have configured, and would need ongoing maintenance/cost exposure for a capability (`source-research` skill + `verify` skill) already covered locally without a new dependency. |
| `recursive-research` (@Anjos2) | Dependencies, maintenance | **Rejected** | Single-contributor repo, no clear evidence of test/maintenance rigor visible from the catalog entry, duplicates `source-research`/`verify` skill capability already present and better integrated (cited-evidence, doc-version-pinning rules already enforced here). |
| `root-cause-tracing`, `test-driven-development`, `using-git-worktrees`, `finishing-a-development-branch` (all @obra, "superpowers" collection) | Overlap with existing rules | **Rejected (duplication)** | This environment's own `rules/troubleshooting.md` (now protected in source control, see `CONTEXT_ENGINEERING_REPORT.md`) already mandates a more detailed root-cause workflow (reproduce → hypothesize → prove → smallest change → verify) with its own 3-file state-tracking model — more rigorous than a generic imported skill for the same purpose. `worktree_isolate.py` already provides tested, integrated git-worktree isolation wired into the retry/task-queue system. Importing parallel, less-integrated versions would fragment behavior rather than improve it. |
| `review-implementing` (@mhattingpete) | Overlap with existing agents | **Rejected (duplication)** | This environment's `verifier` agent (fresh-context adversarial review, requirements-coverage-with-evidence) and `verified-ship` skill (live-execution-gated completion) already do this more rigorously — `review-implementing`'s catalog description doesn't indicate an execution-verification gate. |
| `threat-hunting-with-sigma-rules`, `computer-forensics` | Category fit | **Rejected (different problem)** | These are incident-forensics/threat-hunting tools (post-compromise investigation), not the pre-deployment security/IAM review this environment's `security-reviewer`/`security-threat-modeler`/`iam-reviewer` agents already do. Different job; not a substitute or an addition to the stated priority list. |
| `software-architecture` (design patterns/SOLID) | Category fit | **Rejected (too generic)** | Generic OOP design-pattern guidance, not cloud/infrastructure architecture review — `cloud-architect-reviewer` already covers the actually-requested category (GCP/AWS/Azure design review). |
| The ~120 Composio SaaS-automation skills (CRM/PM/email/social/e-commerce/etc.) | Relevance | **Rejected (out of scope)** | None relate to AWS/GCP/Kubernetes/Terraform/cloud-architecture/troubleshooting/research/automation-design/security-review/verification — this environment's stated priorities. Importing any would add context and maintenance burden with zero connection to daily project work. |

**No skill from the external catalog was imported.** Every category the user
prioritized already has stronger, better-integrated, locally-built coverage than
anything found in the catalog, and the catalog's few generically-relevant entries
either mismatch this environment's actual stack (CDK vs. Terraform) or duplicate
existing, more rigorous local capability. This is the correct outcome of a careful
review, not a shortcut — importing skills for the sake of hitting a number would
violate the explicit instruction to reject "complexity without clear value."

## Existing coverage confirmed (the actual "8-10 primary skills" for this environment)

Mapping the 10 priority areas from the roadmap to what's *already* in this harness —
this is the real, current skill/agent set that matters for daily work:

| Priority area | Existing coverage |
|---|---|
| 1. AWS investigation/implementation | `cloud-audit` skill, `cloud-architect-reviewer` agent |
| 2. GCP investigation/implementation | `cloud-audit` skill, `cloud-architect-reviewer` agent |
| 3. Kubernetes troubleshooting | `sre-incident` agent, `kubernetes-reviewer` agent |
| 4. Terraform / infrastructure review | `terraform-reviewer` agent, `codegen` skill |
| 5. Cloud architecture review | `cloud-architect-reviewer` agent |
| 6. Systematic troubleshooting | `rules/troubleshooting.md`, `task-rigor` skill |
| 7. Evidence-first technical research | `source-research` skill, `verify` skill |
| 8. Automation design review | **Gap found and closed this phase** — new `automation-reviewer` agent |
| 9. Security review | `security-reviewer`, `security-threat-modeler`, `iam-reviewer` agents |
| 10. Verification before completion | `verified-ship` skill, `verifier` agent, code-enforced `final_gate.py` |

## Skills selected

**One new agent, written locally (not imported):**

- **`agents/automation-reviewer.md`** — reviews automation/pipeline designs meant to
  run unattended (GitHub Actions, cron, event-driven functions, n8n-style workflows,
  multi-agent orchestration) for idempotency, bounded retries/timeouts, blast radius,
  secret handling, observability, and trigger correctness. Modeled precisely on the
  existing `terraform-reviewer`/`kubernetes-reviewer` pattern (same frontmatter shape,
  same GO/NO-GO output convention, same "LOW CONFIDENCE, don't hide it" rule). Closes
  the one real gap found: `gitops-reviewer` only covers Kubernetes delivery/promotion
  specifically, not general automation.

## Skills rejected

See the candidate table above — full list with reasons. Summary: 164 candidates
reviewed, 0 imported, 1 new small local agent written for the one confirmed gap.

## Security findings

- No external skill was installed, so no new network behavior, new dependency, new
  script execution, or new permission surface was introduced by this phase.
- The new `automation-reviewer` agent is read-only by design (`tools: Read, Grep, Glob,
  Bash`, explicit rule: "never run/trigger the automation itself, never apply infra
  changes") — matches the existing reviewer-agent convention; does not weaken any
  existing destructive-command protection, evidence-trust requirement, retry limit, or
  approval requirement (it has no write/execute capability to weaken any of those with).

## Files installed or created

- `agents/automation-reviewer.md` (new)
- `.claude/commands/review.md` (updated — added automation-reviewer routing line)
- `policies/subagent-routing.md` (updated — added automation-reviewer row)

## Routing verification

This harness's routing is description-based (I, the orchestrating Claude, match a
request to the best-described skill/agent — there is no separate deterministic
dispatcher script to unit-test). Verified by tracing each representative prompt
against the actual routing tables (`policies/subagent-routing.md`,
`policies/skill-routing.md`, `.claude/commands/review.md`) for an unambiguous match:

| Representative prompt | Correct route | Ambiguous with anything else? |
|---|---|---|
| "Investigate why this Lambda function is throwing 500s" | `cloud-architect-reviewer` / `cloud-audit` (AWS) | No |
| "Review this Terraform module for a GCS bucket before I apply it" | `terraform-reviewer` | No |
| "This pod keeps crash-looping in prod, find out why" | `sre-incident` | No |
| "Review these Kubernetes manifests for resource limits and RBAC" | `kubernetes-reviewer` | No |
| "Is this a good architecture for a 3-tier app on GKE?" | `cloud-architect-reviewer` | No |
| "My deploy script fails intermittently, help me find the root cause" | `rules/troubleshooting.md` workflow (+ `task-rigor` if STANDARD/STRICT) | No |
| "Review this GitHub Actions workflow that auto-deploys on merge" | `automation-reviewer` (new) | Could plausibly overlap `gitops-reviewer` if the workflow deploys to Kubernetes specifically — `.claude/commands/review.md`'s rule "if content spans more than one category, run more than one reviewer" already handles this correctly, not a routing defect. |
| "What's the current recommended instance type for GKE Autopilot?" | `verify` skill (or `source-research`) | No |
| "Write me a haiku about the ocean" | **None of the above** — plain conversational reply | Confirmed: none of these skill descriptions match a creative-writing request; no skill would fire. |
| "What's 15% of 340?" | **None of the above** — plain direct answer | Confirmed: no skill fires for simple arithmetic. |

All 10 test prompts route correctly with no ambiguity requiring a fix. The one
noted overlap case (GitOps-flavored automation) is already correctly handled by the
existing "run more than one reviewer" rule, not a new problem introduced by adding
`automation-reviewer`.

## Context impact

- **Always-loaded context: zero change** — `automation-reviewer.md` is a subagent
  definition, loaded only when dispatched, exactly like the other 16 agents. It adds
  to the on-demand agent count (16 → 17) but nothing to the global always-loaded
  budget reviewed in `CONTEXT_ENGINEERING_REPORT.md`.
- No skills were added (0 imported), so the skill count (38) is unchanged.

## Known limitations

- `automation-reviewer` is newly written and has not yet been exercised against a real
  automation design in this environment (no live GitHub Actions/cron/n8n review has
  been run yet) — its dimensions are based on established SRE/reliability practice and
  mirror the existing reviewer-agent pattern, but its practical judgment quality on a
  real, messy pipeline is unverified until first real use.
- The routing verification above is prompt-matching analysis (how an orchestrating
  Claude would route each prompt against the current tables), not a live dispatch test
  of all 10 prompts — Phase 6's end-to-end evaluation exercises a subset of these
  scenarios with real tool use, to avoid duplicating the same verification twice.
- If a genuine GCP- or Kubernetes-specific external skill appears in this catalog (or
  a similar one) in the future, it would still need the same individual-inspection
  process before import — nothing here should be read as "the catalog is permanently
  irrelevant," only that today's snapshot has nothing worth importing for this
  environment's priorities.
