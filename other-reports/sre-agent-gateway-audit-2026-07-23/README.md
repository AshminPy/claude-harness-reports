# SRE Agent (sre-agent-gateway) — Evidence-Backed Design Audit

**Date:** 2026-07-23
**Repo audited:** [`AshminPy/sre-agent-gateway`](https://github.com/AshminPy/sre-agent-gateway) (local: `~/projects/testing2-gcp-sre-agent`)
**Method:** every claim below is backed by a direct code/Terraform read or a live query against the deployed production resource (`sreagent-t2-demo`), not memory or assumption. File:line citations point at the actual repo at commit `897e48b` (main, 2026-07-18).

---

## 1. End-to-end flow

Request lifecycle, in order:

1. **`SREAgent.query()`** ([agent/main.py:785](https://github.com/AshminPy/sre-agent-gateway/blob/main/agent/main.py#L785)) — entry point called by Vertex AI Agent Engine.
2. **Model Armor input sanitize** (`_sanitize()`, main.py:816) — see §6, currently a no-op in production.
3. **Memory Bank recall** (`_mb_recall()` / `_recall_memory()`, main.py:828) — past RCAs for this cluster/namespace injected as context, **before** the graph runs. Not a gate — investigation always proceeds regardless of what's recalled.
4. **`investigate()`** (main.py:323) builds the incident envelope and calls `graph.invoke()`.
5. **The LangGraph state machine** ([agent/graph.py](https://github.com/AshminPy/sre-agent-gateway/blob/main/agent/graph.py)) — 9 nodes, structurally sequential (cannot be skipped or reordered):

```python
# agent/graph.py:44-86
g.set_entry_point("input_normalizer")
g.add_conditional_edges("input_normalizer", _after_input,
    {"ok": "context_resolver", "failed": "rca_builder"})
g.add_edge("context_resolver", "task_planner")
g.add_edge("task_planner", "mcp_router")
g.add_edge("mcp_router", "tool_executor")
g.add_conditional_edges("tool_executor", _after_tool,
    {"extract": "evidence_extractor", "skip": "task_evaluator"})
g.add_edge("evidence_extractor", "task_evaluator")
g.add_edge("task_evaluator", "loop_controller")
g.add_conditional_edges("loop_controller", _after_loop,
    {"continue": "task_planner", "finish": "rca_builder"})
g.add_edge("rca_builder", END)
```

| Node | Job |
|---|---|
| `input_normalizer` | Parses alert/query, extracts incident context. CLI hints override LLM guesses. |
| `context_resolver` | Resolves target cluster + picks MCP source (GKE Remote MCP primary, custom fallback). |
| `task_planner` | Gemini decides what evidence is needed next. |
| `mcp_router` | Picks ONE MCP source + ONE tool this round, enforces allowlist. |
| `tool_executor` | Executes the tool call. Raw MCP output never enters graph state — only ok/error/duration. |
| `evidence_extractor` | Writes raw evidence to GCS, keeps compressed facts in state. |
| `task_evaluator` | Multi-signal confidence scoring (§4). |
| `loop_controller` | Continue-or-exit decision (§9). |
| `rca_builder` | Builds the final cited RCA + structured observability event. |

6. Output sanitized again (§6), persisted to GCS + Memory Bank, returned to caller.

**Live confirmation, not just code reading:** a real production run (`run_20260718_011402_aqgo`, pulled directly from Cloud Logging on `sreagent-t2-demo`) matches this exactly — `primary_mcp_source: gke_remote_mcp`, `tools_called: 2`, `loop_exit_reason: confidence_sufficient`.

---

## 2. Would this work smoothly and accurately in production?

**Mostly yes, with real gaps found during this audit — not a clean bill of health.**

**What's solid (evidence):**
- Structurally enforced node order (§1) — no way to skip evaluation or evidence extraction.
- Real, live-verified CI: `terraform-apply.yml` runs a genuine smoke test against the deployed agent on every merge to `main` (not a mock) — [scripts/smoke_test.sh](https://github.com/AshminPy/sre-agent-gateway/blob/main/scripts/smoke_test.sh) invokes the real deployed engine via `invoke_agent.py`.
- Deterministic confidence cap prevents hallucinated high-confidence RCAs on thin evidence (§4) — though its protection window is narrower than it looks (§4).
- Raw MCP tool output is deliberately kept out of graph state ([agent/nodes/tool_executor.py:1-7](https://github.com/AshminPy/sre-agent-gateway/blob/main/agent/nodes/tool_executor.py#L1-L7)) — reduces prompt-injection blast radius from k8s log content.

**Gaps found (new findings from this audit, not previously flagged):**

1. **Model Armor is not actually active in the current production deployment.** See §6 — confirmed via a live query of the deployed reasoning engine's env vars, not just code reading.
2. **The multi-cluster registry has no protection against being silently reverted.** See §5 — a routine CI-triggered `terraform apply` will overwrite manually-added clusters back to just the single default.
3. **The primary evidence source (GKE Remote MCP) is explicitly Preview/Pre-GA per its own registry entry:**
   ```python
   # agent/mcp_client.py:97-103
   "gke_remote_mcp": {
       ...
       "ga_status":   "Preview/Pre-GA",
   },
   ```
   Every production investigation depends on a Google API that is not yet GA. The custom Cloud Run fallback exists but defaults to a placeholder image until someone builds the real one (§5) — so if GKE Remote MCP has an outage, the fallback path may not actually be deployed.
4. **Zero automated per-node tests** (§10) — only whole-graph evaluation exists. A regression in one node's logic (e.g. `task_evaluator`'s cap formula) would only surface via a full run, not a fast targeted test.

None of these are fatal — the agent has demonstrably worked end-to-end in production (§1's live evidence) — but "smooth and accurate" should be qualified: accurate when it runs, with a safety layer (Model Armor) that is currently dormant and a primary data path that is not GA.

---

## 3. Observability — logs, metrics, tracing

**Structured logging** — every investigation emits one JSON event to stdout (Cloud Logging auto-parses as `jsonPayload`):

```python
# agent/main.py:450-476 (obs_event) — printed at main.py:479
{
  "event_type": "sre_agent_run", "run_id", "incident_type", "cluster",
  "selected_mcp", "tools_called", "evidence_count", "confidence",
  "confidence_band", "status", "loop_exit_reason", "human_review",
  "latency_ms", "tokens_input/output/total", "estimated_cost_usd", "error_count",
}
```
Plus a **per-node token event** on every node call ([agent/otel.py:223-238](https://github.com/AshminPy/sre-agent-gateway/blob/main/agent/otel.py#L223-L238), `log_node_tokens()`), and a dedicated **tool-failure log** (`sre-agent-tool-failures` logger, [agent/nodes/tool_executor.py:67](https://github.com/AshminPy/sre-agent-gateway/blob/main/agent/nodes/tool_executor.py#L67)).

**Distributed tracing** — Cloud Trace via OpenTelemetry, one span per node:
```python
# agent/otel.py:189-220
def trace_node(span_name: str):
    ...
    with tracer.start_as_current_span(span_name) as span:
        set_span_attributes(span, _state_attrs(state, span_name))  # cluster, step, confidence, cost...
```
Every node file uses `@trace_node("langgraph.<node>")` — confirmed on `input_normalizer`, `mcp_router`, `tool_executor`, `task_evaluator`, `loop_controller`, `rca_builder`.

**Metrics + alerts (Terraform, [iac/agent/monitoring.tf](https://github.com/AshminPy/sre-agent-gateway/blob/main/iac/agent/monitoring.tf)):**

| Log-based metric | Filter | Purpose |
|---|---|---|
| `sre_agent/invocations` | `jsonPayload.run_id:*` | volume |
| `sre_agent/errors` | `jsonPayload.status="error"` | error rate |
| `sre_agent/escalations` | `jsonPayload.confidence_band="escalate"` | low-confidence rate |
| `sre_agent/investigation_cost_usd` | distribution, buckets $0.01–$1.00 | cost distribution |
| `sre_agent/confidence_band` | labeled auto/review/escalate | band mix |
| `sre_agent/loop_exit_reason` | labeled by reason | why runs stop |
| `sre_agent/tool_failures` | by `sre-agent-tool-failures` log, labeled tool+cluster | which tool/cluster fails most |

**3 alert policies** (email notification channel), all in monitoring.tf:
- `high_error_rate` — >5 errors / 5 min
- `high_escalation_rate` — >3 escalations / 5 min
- `cost_spike` — single investigation >$0.10

**Audit trail:** full RCA + request persisted to GCS per run (`_save_to_gcs()`, main.py:627-655), never blocks the response if it fails.

**Assessment: this is genuinely comprehensive** — logs, metrics, traces, cost, and an audit trail all exist and are wired to alerting, not just "logging enabled." The one gap: none of these are dashboarded (no `google_monitoring_dashboard` resource found in `iac/agent/`) — metrics exist but you'd build charts ad hoc in the console today.

---

## 4. Confidence score — configuration and scale behavior

**Formula** ([agent/nodes/task_evaluator.py:90-110](https://github.com/AshminPy/sre-agent-gateway/blob/main/agent/nodes/task_evaluator.py#L90-L110)):
```python
coverage_score = min(1.0, evidence_count / 4.0)   # 4 items = full coverage
confidence_cap = min(1.0, coverage_score + 0.25)  # max 0.25 bonus above coverage
if confidence > confidence_cap:
    confidence = confidence_cap   # LLM's own score is capped, never trusted blindly

if confidence >= 0.85:   confidence_band = "auto"
elif confidence >= 0.65: confidence_band = "review"
else:                    confidence_band = "escalate"
```
Also: a **safety gate returns confidence=0.0/escalate immediately if zero evidence exists** (task_evaluator.py:35-50), and **min_steps=2 forces at least 2 rounds** regardless of apparent confidence (task_evaluator.py:53-64, state.py:96) before the real evaluation ever runs.

**Does it always work at scale? Qualified yes — one real limitation found:**

The deterministic cap only *constrains* when `evidence_count <= 2`:

| evidence_count | coverage_score | confidence_cap |
|---|---|---|
| 0 | 0.0 | — (forced to 0.0, escalate) |
| 1 | 0.25 | 0.50 |
| 2 | 0.50 | 0.75 |
| **3** | **0.75** | **1.00 — cap stops constraining** |
| 4+ | 1.00 | 1.00 |

**Once an investigation collects 3+ evidence items, the cap saturates at 1.0 and the LLM's self-reported confidence is trusted directly** — the guardrail's real protection window is only 0–2 evidence items. This isn't a bug (a 3rd item genuinely earns full trust by design), but it means the "prevent hallucinated high confidence" comment in the code (task_evaluator.py:90-92) protects thin-evidence cases only, not moderate-evidence overconfidence.

**Concurrency/scale:** confidence state is per-run (`AgentState`, no shared mutable state across concurrent investigations), so parallel investigations don't interfere with each other's scoring — verified by reading `state.py`'s `get_initial_state()` (fresh dict per call, no module-level mutable score state).

---

## 5. Scaling design — adding clusters and MCPs (Q5 + Q11)

**Adding a new cluster — config-driven, but with a real gotcha:**

The agent resolves clusters at runtime from a GCS-backed registry, not hardcoded config:
```python
# agent/mcp_client.py:115-134
# Source of truth: gs://<CLUSTER_CONFIG_BUCKET>/clusters.json
# To add a cluster: edit clusters.json in GCS — no Terraform or redeploy needed.
# Agent SA needs: roles/storage.objectViewer on the config bucket.
```
5-minute TTL cache (`_REGISTRY_TTL = 300.0`, mcp_client.py:176) — a GCS edit is picked up within 5 minutes, no restart needed.

**But:** `clusters.json` is *also* Terraform-managed as the initial bootstrap object, with **no lifecycle guard**:
```hcl
# iac/agent/buckets.tf:61-64
resource "google_storage_bucket_object" "clusters_json" {
  name    = "clusters.json"
  bucket  = google_storage_bucket.cluster_config.name
  content = local.clusters_json   # templated from a SINGLE gke_cluster_name var
}
```
Since `terraform-apply.yml` runs automatically on every merge to `main`, **the next merge after you manually add a cluster to GCS will silently revert `clusters.json` back to just the single default cluster** — there's no `lifecycle { ignore_changes = [content] }`. Cross-project read access is granted at the **project** level ([iac/gke-access/crossproject_iam.tf:19-24](https://github.com/AshminPy/sre-agent-gateway/blob/main/iac/gke-access/crossproject_iam.tf#L19-L24), `google_project_iam_member` on `var.project_b_id`) — so a new cluster *in an already-granted project* needs zero Terraform; a cluster in a *new* project needs one `terraform apply` of the `gke-access` stack for that project.

**Adding a new MCP server type (e.g. Grafana/Prometheus per NEXTSTEPS.md item 9) — NOT config-driven, requires code changes:**

```python
# agent/mcp_client.py:95-109 — hardcoded, exactly 2 entries
MCP_REGISTRY = {
    "gke_remote_mcp": {...},
    "k8s_mcp": {...},
}
```
To add a third source you must: (1) define a new tool allowlist set, (2) add an entry to `MCP_REGISTRY`, (3) extend `call_tool()`'s dispatch logic in mcp_client.py, (4) update `mcp_router.py`'s prompt/allowlist, (5) update `context_resolver.py`'s primary/fallback strategy, (6) grant IAM. This is real engineering work, not a config change — worth knowing before scoping "add Grafana" as a small task.

---

## 6. Model Armor — level, status, and where to change it

**Two possible layers, per the code's own comments:**
1. App/code level — `agent/main.py`'s `SREAgent._sanitize()`
2. Agent Gateway level — a `CONTENT_AUTHZ` extension (Model Armor inspecting gateway traffic)

**Finding, confirmed against the LIVE deployed engine (not just Terraform code) — layer 2 was removed, and layer 1 is currently OFF:**

The gateway's actual current authz extension is **IAP REQUEST_AUTHZ** (header/attribute-based, no content inspection), not Model Armor:
```hcl
# iac/agent/agent_gateway.tf:70-77
# Per the official codelab..., the gateway is governed by an IAP REQUEST_AUTHZ
# extension... the gateway does NOT TLS-terminate or inspect payload content,
# so there is no MITM...
resource "google_network_services_authz_extension" "iap" { ... }
```
And the app-level env var is deliberately **not set when the gateway is on** (which is the default, and the current production config):
```hcl
# iac/agent/agent_engine.tf:74-78
var.enable_agent_gateway ? {} : {
  MODEL_ARMOR_TEMPLATE = google_model_armor_template.sre_agent_request.name
},
```
**Verified directly against the live production reasoning engine** (`sreagent-t2-demo`, engine `6299408329517039616`, queried via the Vertex AI API — full env var list captured, `MODEL_ARMOR_TEMPLATE` is absent):
```
agentGatewayConfig.agentToAnywhereConfig.agentGateway = sre-agent-egress   ← gateway IS on
MODEL_ARMOR_TEMPLATE  ← NOT in the live env var list
```
In `agent/main.py`, `_sanitize()` fails safe-to-passthrough when the template is unset:
```python
# agent/main.py:38, :581-582
MODEL_ARMOR_TEMPLATE = os.environ.get("MODEL_ARMOR_TEMPLATE", "")
...
if not cls._armor_client or not MODEL_ARMOR_TEMPLATE:
    return text, False   # no blocking, ever
```
**Conclusion: Model Armor is currently inactive end-to-end in production** — not blocking prompt injection on input, not redacting PII on output, despite `google_model_armor_template.sre_agent_request` / `sre_agent_response` existing as deployed Terraform resources. This corrects something said informally earlier in this project's working sessions ("Model Armor sanitizes the input") — that was true of the code path read in isolation, not of the current deployed configuration.

**Where to change it, to re-enable:**
- To turn it on with the gateway ON: edit the ternary at `iac/agent/agent_engine.tf:74-78` to always set `MODEL_ARMOR_TEMPLATE` (accept the "redundant call" tradeoff noted in the comment, since gateway-level inspection is no longer active anyway).
- To turn it on at the gateway instead: `iac/agent/agent_gateway.tf` would need a `CONTENT_AUTHZ` extension re-added — but this was deliberately removed because it causes a TLS MITM that breaks the agent's mutual-TLS Vertex endpoint (documented root cause, `docs/ADR-002-agent-identity-and-gateway.md`). Re-adding it means re-solving that conflict, not a flag flip.
- Confidence level for prompt-injection/jailbreak detection is a single Terraform variable: `var.model_armor_pi_confidence` ([iac/agent/variables.tf:129-137](https://github.com/AshminPy/sre-agent-gateway/blob/main/iac/agent/variables.tf#L129-L137), validated enum `LOW_AND_ABOVE | MEDIUM_AND_ABOVE | HIGH`).

---

## 7. Overall manageability — enable/disable/test components

**What's genuinely well-managed (flag-driven, documented):**

| Flag | File | Default | Effect |
|---|---|---|---|
| `enable_agent_gateway` | iac/agent/variables.tf:66 | `true` | Gateway on/off |
| `enable_custom_mcp` | iac/agent/variables.tf:78 | `false` | Custom MCP fallback deployed or not |
| `iap_iam_enforcement_mode` | iac/agent/variables.tf:90 | `null` (enforce) | `"DRY_RUN"` to audit-only |
| `authz_fail_open` | iac/agent/variables.tf:101 | `true` | Fail-open vs fail-closed on gateway authz |
| `model_armor_pi_confidence` | iac/agent/variables.tf:129 | validated enum | PI/jailbreak sensitivity |
| `create_gke_cluster` | iac/gke-access/variables.tf:29 | `true` | Demo cluster vs bring-your-own |
| `MAX_TOKENS_PER_RUN` (env, not TF var) | agent/nodes/loop_controller.py:15 | `100000` | `0` disables the cost cap entirely |

Every flag above has an inline description explaining what it does and when to flip it — genuinely above-average for a project this size.

**What's not as manageable (gaps found in this audit):**
- Model Armor has **no dedicated on/off variable** — it's implicitly tied to `enable_agent_gateway` (§6), so you can't independently test "gateway on + Model Armor on" without editing the ternary yourself.
- No per-node test harness (§10) — "test independently" currently means running the whole graph.
- `clusters.json`'s Terraform-vs-GCS split (§5) is a real trap for someone who doesn't know to look for it.
- Multi-repo sprawl (§8) — four related repos exist; picking the wrong one to modify is a proven failure mode this session's own memory system had already flagged before this audit.

---

## 8. Which repos are clean and usable as reference

Checked all four locally-present, SRE-agent-shaped repos directly (git status, CI history, default branch):

| Repo | Local path | Status | Verdict |
|---|---|---|---|
| **`AshminPy/sre-agent-gateway`** | `~/projects/testing2-gcp-sre-agent` | Clean (0 dirty), CI green, live-verified in production | ✅ **The reference implementation. Use this one.** |
| `AshminPy/testing-adk-agent` | `~/projects/testing-adk-agent` | Clean, but an unmodified seed clone of sre-agent-gateway (`README.md` still says "# testing2-gcp-sre-agent") | Not yet its own reference — inherits sre-agent-gateway's state until ADK migration work starts |
| `AshminPy/testing-gcp-sre-agent` (original) | `~/Downloads/cloud-sreagent/gcp-sreagent/sre-agent-gcp` | **23 uncommitted dirty files** (iac/*.tf, agent/gemini_client.py, agent.tar.gz) | ❌ Deprecated predecessor — superseded by sre-agent-gateway, do not use as reference |
| same remote, 2nd clone | `~/projects/ai-projects/testing-gcp-sre-agent` | Clean, but a stale duplicate checkout of the same deprecated repo above | ❌ Same repo as above — redundant clone |
| `AshminPy/gcp-sre-agent` | `~/projects/gcp-sre-agent` | `main` branch dormant since 2026-06-26, **zero CI runs ever executed against the actual agent code** | ❌ Earliest prototype in the lineage, abandoned, not CI-validated |

**Important correction found during this audit:** `gcp-sre-agent`'s README describes a real SRE-agent architecture (same LangGraph node names confirmed on its `main` branch) — it is *not* an unrelated project, contrary to what a quick glance at its locally-checked-out branch (`claude/opencti-dependencytrack-deploy-jyux9n`, containing unrelated OpenCTI/DependencyTrack manifests) would suggest. It's a real but abandoned early iteration, one step further back in the same lineage as `testing-gcp-sre-agent` → `sre-agent-gateway`.

**Bottom line: `sre-agent-gateway` is the only repo to use as a reference or to build on.** The other three are historical/abandoned.

---

## 9. Loop exit — is it really just a hard cap?

**No — the hard cap (`max_steps=5`) is the exit of last resort, not the primary mechanism.** Full priority-ordered exit logic ([agent/nodes/loop_controller.py:107-149](https://github.com/AshminPy/sre-agent-gateway/blob/main/agent/nodes/loop_controller.py#L107-L149)):

```python
if enough:                          exit_reason = "confidence_sufficient"   # highest priority
elif timed_out:                     exit_reason = "timeout"
elif over_budget:                   exit_reason = "token_budget_exceeded"
elif step >= max_steps:             exit_reason = "max_iterations"          # the hard cap — last resort
elif tool_done and step >= min_steps: exit_reason = "tool_signaled_done"
elif consec_failures and step >= min_steps: exit_reason = "consecutive_tool_failures"
elif oscillating:                   exit_reason = "oscillation_detected"    # A→B→A→B pattern
elif stuck:                         exit_reason = "stuck_detected"          # same tool+args twice
elif zero_facts:                    exit_reason = "zero_new_facts"          # 2 consecutive empty results
```

Six independent safety valves exist besides the step-count cap:
- **Token budget** — `MAX_TOKENS_PER_RUN` env var, default 100,000, **this is your direct cost control lever** (loop_controller.py:15, `0` disables it)
- **Wall-clock timeout** — `max_duration_seconds`, default **540s / 9 min** ([agent/state.py:109](https://github.com/AshminPy/sre-agent-gateway/blob/main/agent/state.py#L109)) — note: `loop_controller.py:49`'s `inv.get("max_duration_seconds", 300)` shows `300` only as a defensive fallback if the key were ever missing; the real effective default from `get_initial_state()` is 540s.
- **Oscillation detection** (A→B→A→B in last 4 tool calls)
- **Stuck detection** (identical tool+args called twice in a row, both successful)
- **Zero-new-facts detection** (2 consecutive successful-but-empty tool results — "dry zone")
- **Consecutive-failure detection** (last 2 tool calls both failed)

**Answering the actual worry — "for complex issues, will the hard cap limit the agent too early, and if we remove it, will cost run away?"**

You don't need to choose between those two failure modes; the design already gives you three *independent* dials to widen the *time/step* budget for genuinely complex investigations **without** removing cost protection:

1. `max_steps` (currently 5, `state.py:95`) — raise this if 5 rounds is genuinely too few for complex cases. This doesn't touch cost.
2. `MAX_TOKENS_PER_RUN` (currently 100k) — this is the **actual cost cap**, independent of step count. A complex investigation that needs 15 tool calls but uses efficient prompts still exits at 100k tokens regardless of `max_steps`.
3. `max_duration_seconds` (540s) — wall-clock safety net independent of both.

**Recommendation:** don't remove the hard cap — raise `max_steps` moderately (e.g. 5→8) for complex-investigation headroom, and treat `MAX_TOKENS_PER_RUN` as the real cost control (it already directly bounds spend regardless of how many steps that takes). The existing `investigation_cost_usd` alert policy (§3, fires >$0.10/run) is your live tripwire if a change to either value causes cost creep — this is a config change (2 numbers), not a design change.

---

## 10. Evaluation — testing end-to-end and per-component

**Three independent evaluation mechanisms exist, all end-to-end (none test a single node in isolation):**

1. **`agent/eval/run_eval.py`** — trajectory-based: scores tool-call precision/recall/in-order-match against `agent/eval/golden_cases.py`'s expected trajectories (e.g. `crashloop-001` expects `["list_pods", "get_pod_logs", "get_events"]`, `expected_confidence_min: 0.65`). Runs LOCAL (direct graph invoke, no Agent Engine needed) or REMOTE (deployed engine), can submit to Vertex AI Gen AI Evaluation Service.
2. **`eval.py`** — simple keyword-matching scorer (`root_cause_hit`, `has_fix`, `latency_ok`), outputs to console + local JSON + optional GCS upload.
3. **`eval_native.py`** — Agent Platform's native AutoRater metrics (`question_answering_quality`, `groundedness`), visible in the Vertex AI Experiments UI.

**Gap found: zero per-node unit tests.**
```
$ find . -iname "test_*.py" -o -iname "*_test.py"
(no results)
```
Every node is a plain function `(state: AgentState) -> dict` — architecturally trivial to unit test in isolation (e.g. `task_evaluator({"evidence_ids": [...], ...})` directly, no graph needed) — but no such harness currently exists. Today, "test independently" means running one of the three eval mechanisms above end-to-end; there's no fast, cheap way to verify e.g. `task_evaluator`'s confidence-cap math changed correctly without a full (LLM-calling, cost-incurring) investigation run.

**Recommendation if you want this:** a `tests/test_nodes.py` using plain `pytest` + hand-built fake `AgentState` dicts would be low-effort given the existing node signatures — no new architecture needed, just test files that don't exist yet.

---

## 11. How to add clusters and MCPs in future

Covered in detail in §5. Summary:

- **New cluster, same GCP project already granted access:** edit `clusters.json` in the `CLUSTER_CONFIG_BUCKET` GCS bucket directly. No Terraform, no redeploy. Picked up within 5 minutes (TTL cache). **Caveat:** the next CI-triggered `terraform apply` on `main` will silently revert this — either stop Terraform from managing that GCS object's content (`lifecycle { ignore_changes = [content] }` on `iac/agent/buckets.tf`'s `google_storage_bucket_object.clusters_json`), or extend `gke_cluster_name` into a list variable so Terraform manages the full set correctly. This is the single highest-value fix to make multi-cluster actually safe to operate.
- **New cluster, new GCP project:** one `terraform apply` of `iac/gke-access` for that project (grants the 4 read-only cross-project roles), then the `clusters.json` step above.
- **New MCP server type** (Grafana, Prometheus, Elasticsearch, etc. — per `NEXTSTEPS.md` item 9): real code change, not config. Touch points: `agent/mcp_client.py` (`MCP_REGISTRY` + `call_tool()` dispatch + a new tool allowlist), `agent/nodes/mcp_router.py` (routing prompt/logic), `agent/nodes/context_resolver.py` (primary/fallback strategy), plus IAM for the new endpoint.

---

## Summary of new findings from this audit (not previously known/flagged)

1. **Model Armor is currently inactive in production** — confirmed via live deployed-engine query, not just code reading (§6).
2. **`clusters.json` will be silently reverted by the next CI-triggered `terraform apply`** — no lifecycle guard exists (§5, §11).
3. **`gcp-sre-agent` is a real (if abandoned) SRE agent, not an unrelated project** — corrects an earlier assumption; its `main` branch has never had CI run against it (§8).
4. **The confidence cap only constrains at 0–2 evidence items** — at 3+, the LLM's own score is trusted directly (§4).
5. **GKE Remote MCP, the primary evidence source, is Preview/Pre-GA per its own registry entry** — an external dependency risk (§2).
6. **Zero per-node unit tests exist** — only three end-to-end eval mechanisms (§10).
