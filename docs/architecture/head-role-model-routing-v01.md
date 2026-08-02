# Head Orchestrator v0.1 — Role and Model Routing

Date: 2026-08-02

Status: Accepted for v0.1 design; model route strings are configuration, not commitments

Decision: six fixed logical roles — one leader, five bounded workers; assignment is not activation

## One-Line Decision

Exactly one agent — `leader` (alias `analyzer-bot`) — converses with the user and issues final judgment; five model-bound workers (`gpt-analyzer`, `gemini-investigator`, `deepseek-drafter`, `glm-coder`, `claude-reviewer`) execute bounded contracts, never call each other, and never hold authority.

## 1. Structure

```text
User
  ↕
leader / analyzer-bot
  ├── gpt-analyzer
  ├── gemini-investigator
  ├── deepseek-drafter
  ├── glm-coder
  └── claude-reviewer
```

`leader / analyzer-bot` is the only Head Orchestrator. Every other position is a bounded worker the Head invokes. Workers never invoke each other:

```text
Forbidden:
gemini → glm
glm → claude
deepseek → gemini
claude → gpt
worker → worker (any pair)
```

Every request, result, retry, budget change, and strategy change passes through the Head:

```text
worker
  → Caveman result
  → Ledger/Broker
  → Head
  → next worker, or final response
```

This is the same invocation rule the [analyzer-bot design](../2026-08-01-analyzer-bot-main-agent-design.md#graph-state-branching-resume) already sets — data-plane artifact exchange goes through the Ledger; control-plane changes go through the Head only; direct worker-to-worker invocation is forbidden in all cases.

**Naming.** `leader` is the registry `agent_id`; `analyzer-bot` is its alias, and the OpenClaw profile id stays `analyzer` (the id already live in the skill-evolution config). **`gpt-analyzer` is not the leader and not its peer** — despite the similar name, it is a subordinate worker with no conversation channel and no decision authority. The name collision is acknowledged and the hierarchy is fixed: one is the Head, the other is a tool the Head calls.

## 2. Agent Registry

### 2.1 `leader` / `analyzer-bot`

```yaml
agent_id: leader
alias: analyzer-bot
role: head_orchestrator
model_route: gpt-5.5
```

Responsibilities: sole conversation channel with the user; intent interpretation; canonical state ownership; task decomposition; task class and risk tier assignment; worker dispatch; budget allocation; strategy selection and switching; result synthesis; approval requests; final PolicyGate; final response; global stop.

Authority: change goals; change budgets (within [ASC](../2026-08-02-adaptive-strategy-controller.md) rules); open task lineages; change method families; invoke workers; request user approval; issue final success/failure verdicts.

Forbidden:

- Extending a hard cap on its own judgment (ASC §2).
- Declaring success without verifying worker results.
- Altering reviewer output.
- Bypassing verification or approval on a high-risk action.
- Counting a Grade 3 claim as confirmed progress (disabled in v0.1, ASC §6).

The leader consults workers; it never delegates judgment responsibility to them.

### 2.2 `gpt-analyzer`

```yaml
agent_id: gpt-analyzer
alias: gpt
role:
  - analyzer
  - structural_reasoner
  - scorer
  - synthesizer
model_route: gpt-5.5
```

Responsibilities: decomposing complex requests; extracting requirements and constraints; generating candidate solution paths; generating evaluation criteria; structuring results; coverage/consistency/risk scoring; structural comparison of multiple worker results; drafting synthesis for the Head.

No authority to: converse with the user; dispatch workers; extend budgets; change task class; issue final verdicts; execute externally; substitute for reviewer verdicts.

**Two GPT-5.5 instances are not two brains.** `leader` and `gpt-analyzer` share a model route; they run distinct logical roles in isolated contexts, but two results from the same model and the same source lineage never count as independent verification:

```text
leader judgment + gpt-analyzer judgment
≠ Grade 2 independent verification
```

This is the role-level corollary of ASC §6's axis enum (`model_provider` is a weak axis; a shared source lineage forfeits the strong one). **`gpt-analyzer` scores are advisory signals** — a score is not evidence, and can never ground a budget extension or a high-risk execution.

**Dispatch threshold — resolved for v0.1.** When the leader reasons inline versus dispatches `gpt-analyzer`:

```text
leader inline:
  single-step simple_query with no worker results to compare

dispatch gpt-analyzer:
  - investigation, code_change, high_risk_action — always
  - two or more worker results need comparison, scoring, or synthesis
  - evaluation criteria or acceptance criteria need generating
```

The rationale is contamination, not cost: anything that will later be scored or synthesized should be structured outside the leader's own reasoning stream, so the leader is not grading a structure it authored inline.

### 2.3 `gemini-investigator`

```yaml
agent_id: gemini-investigator
alias: gemini
role:
  - searcher
  - investigator
model_route:
  runtime: Antigravity agy
  model: Gemini 3.1 Pro
```

Responsibilities: external search; multi-source investigation; original-source recovery; fact verification; conflicting-evidence detection; cause-unknown investigation; per-hypothesis evidence collection; source lineage recording.

Tool profile:

```yaml
access:
  network_read: true
  repository_read: true
  external_write: false
  code_write: false
  deployment: false
```

Must return: query list, source list, key findings, per-source evidence, conflicting information, unverified items, investigation limits, confidence basis.

Forbidden: asserting facts without search results; synthesis without sources; deciding the final user response; editing code; changing budgets; invoking other workers.

Inherits `searcher-bot`'s rules from the analyzer doc verbatim: Apify escalation policy, Actor allowlist, and the `preview` / `blocked` / `failed` / `no_results` / `executed` status separation — never "searched, no result" when the provider was down. Gemini collects evidence; it does not judge.

### 2.4 `deepseek-drafter`

```yaml
agent_id: deepseek-drafter
alias: deepseek
role: drafter
model_route: deepseek-v4-flash
```

Responsibilities: document, message, and report drafts; user-response candidates; converting verified material into prose; tone and format application; readable rendering of structured results.

**Input is restricted to Head-approved facts and artifacts. DeepSeek is not a source of new facts:**

```text
verified evidence
  → deepseek-drafter
  → draft_result
```

Forbidden: inventing facts; fabricating sources; fact verification; sending the final response; direct delivery to the user; external side effects; budget or strategy judgment.

Every draft passes through reviewer or the Head's final gate before delivery.

### 2.5 `glm-coder`

```yaml
agent_id: glm-coder
alias: glm
role: coder
model_route: z-ai/glm-5.2
```

Responsibilities: writing and modifying code; writing and running tests; static analysis; migrations; patch and artifact generation; failure reproduction; implementation reports.

Tool profile:

```yaml
access:
  repository_read: true
  repository_write: sandboxed
  shell_exec: sandboxed
  tests: true
  network_write: false
  deploy: false
  merge: false
```

Must return: changed files, patch/commit artifact ref, commands run, raw test results, failing tests, diagnostics, remaining blockers, rollback information.

Forbidden: deploy or merge without user approval; hiding failed tests; weakening assertions; bypassing validators; unrelated refactors; self-declaring completion; invoking other workers.

Inherits `coder-bot`'s evidence rules: RED before GREEN, mutation proof, PASS_TO_PASS, exact command output, clean git status before any completion claim. GLM implements; fitness of the change is judged by `claude-reviewer` and the Head.

### 2.6 `claude-reviewer`

```yaml
agent_id: claude-reviewer
alias: claude
role:
  - reviewer
  - skeptic
  - adversarial_verifier
model_route:
  primary: claude-opus-4-8
  transport: Claude CLI
  availability: when_available
```

Responsibilities: design–implementation mismatch detection; hidden-assumption detection; counterexample generation; failure-mode analysis; diff review; missing-test detection; source-independence review; user-intent distortion detection; blocking review before high-risk execution; security, data-loss, and budget-laundering review.

Tool profile:

```yaml
access:
  repository_read: true
  artifact_read: true
  original_request_read: risk_tier_dependent
  repository_write: false
  external_write: false
  deploy: false
```

Verdicts: `pass` · `pass_with_warnings` · `request_changes` · `blocked`.

**A reviewer pass grants nothing:**

```text
Claude pass ≠ execution approval
```

Execution always passes the Head's final gate. The reviewer can block; it cannot: edit code; extend budgets; substitute for user approval; send the final response; change task goals; re-invoke workers; or acquire new authority from its own review results.

Reviewer context scales with risk tier per the analyzer doc's [risk-tiered input scope](../2026-08-01-analyzer-bot-main-agent-design.md#bot-contracts). **Tiers are cumulative — each tier adds to the one below, never replaces it:**

```text
low:   Caveman contract + worker result + evaluation criteria
mid:   low + relevant excerpt of the original request
high:  mid
       + full original user request
       + independently generated intent digest
       + Head contract, compared side by side
```

The worker result and evaluation criteria are therefore in scope at **every** tier. A high-tier review checks both directions: that the Head's contract matches the user's intent, *and* that the worker's result satisfies the contract — intent comparison adds to result review, it does not swap it out.

## 3. Authority Matrix

| Function | Leader | GPT | Gemini | DeepSeek | GLM | Claude |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| User conversation | O | X | X | X | X | X |
| Task decomposition | O | assist | X | X | X | review |
| Structural analysis | O | O | assist | X | assist | review |
| External investigation | directs | X | O | X | limited | verifies |
| Code writing | approves | X | X | X | O | X |
| Document drafting | approves | structure | material | O | assist | review |
| Result scoring | final | advisory | X | X | self-scoring forbidden | independent review |
| Budget change | O | X | X | X | X | X |
| High-risk approval | confirms user approval + final gate | X | X | X | X | blocking review |
| External execution | final gate | X | X | X | limited (sandboxed) | X |
| Final response | O | X | X | X | X | X |

On high-risk approval, the leader's role is to **verify that explicit user approval exists** and then run the final gate (§10, steps 9–10). The leader's own judgment never substitutes for user approval — "final gate" is a checkpoint the leader operates, not an approval the leader grants.

## 4. Control Plane and Data Plane

This restates and extends the [analyzer doc's data/control plane rule](../2026-08-01-analyzer-bot-main-agent-design.md#graph-state-branching-resume).

**Control plane — Head only:** goal, task class, risk tier, budget, method family, core assumptions, worker assignment, approval, suspend/resume, global stop, final response. Workers cannot modify control-plane values.

A worker that discovers a needed change raises a typed exception through the Caveman channel — the same five already fixed in the analyzer doc: `needs_clarification`, `schema_mismatch`, `out_of_scope_finding`, `assumption_violation`, `novel_observation`. **An exception is an alarm, not an authority-escalation path** — all Caveman hard limits apply (no auto-budget, report-only findings, repeated exceptions consume budget like attempts).

**Data plane — worker-produced, Ledger/Broker-mediated:** evidence, sources, artifacts, patches, drafts, test results, diagnostics, review findings, progress claims, resource usage. Everything moves by ID or immutable reference; workers never transmit artifacts directly to each other.

## 5. Default Routing Policies

Routing patterns are not budget classes. The [ASC classifier](../2026-08-02-adaptive-strategy-controller.md) (§9) assigns one of four budget classes per dispatched task unit; the patterns below describe which workers a class typically engages. Drafting has a routing pattern but no budget class of its own — a draft task inherits its class from the classifier (a repository/document write classifies as `code_change`; a chat-only reply as `simple_query`).

### `simple_query`

```text
leader
  → gpt-analyzer, only when the §2.2 dispatch threshold is met
  → final response
```

Gemini, Claude, GLM, and DeepSeek are invoked only when actually needed — a strategy switch inside `simple_query` is already suspicious (ASC §9).

### `investigation`

```text
leader
  → gpt-analyzer: problem structure and investigation plan
  → gemini-investigator: sources and evidence
  → gpt-analyzer: structural synthesis
  → claude-reviewer: counterexamples and source-independence, when needed
  → leader: final gate
```

### `code_change`

```text
leader
  → gpt-analyzer: requirements and acceptance criteria
  → glm-coder: implementation and tests
  → claude-reviewer: diff and test review
  → glm-coder: re-work, only on Head-approved change requests
  → leader: final gate
```

Claude never re-invokes GLM directly:

```text
claude → review_result → leader → new WorkerCommand → glm
```

### Writing / drafting

```text
leader
  → gpt-analyzer: structure and key messages
  → gemini-investigator: fact gathering, when needed
  → deepseek-drafter: draft
  → claude-reviewer: distortion, omission, and risk review
  → leader: final edit and response
```

### `high_risk_action`

All three conditions, always (matching ASC §9's resolved gate):

```text
Grade 2 independent verification
+ explicit user approval
+ PolicyGate pass
```

If `claude-reviewer` is unavailable and no independent verification path exists:

```text
review route unavailable
  → degraded
  → high-risk blocked
```

## 6. Model Availability and Fallback

Route failure is never disguised as success. Each worker carries a state: `available` · `degraded` · `unavailable`.

Automatic fallback is allowed only where it does not corrupt role semantics or independence accounting:

- A leader model change is recorded as an explicit configuration event.
- A `gpt-analyzer` outage is never recorded as the leader having performed independent verification.
- A Gemini outage never becomes "investigated, nothing found" — it is a blocked investigation.
- On a DeepSeek outage the Head may draft directly, recorded as a drafter fallback.
- On a GLM outage no other worker quietly assumes the coder role.
- On a Claude outage review may degrade to a non-independent review, recorded as such and **never counted as Grade 2**.
- High-risk actions are blocked outright while no independent reviewer exists.

Calling the same model twice under two names is not independence — ASC §6's axis enum makes this structural, not aspirational.

**Provider gates inherited from the [7/31 note](../2026-07-31-langgraph-head-reference-patterns.md):** assignment in this document is design-time; *activation* still requires fresh route smoke evidence. Gemini rides the Antigravity runtime until a direct route is proven; DeepSeek activates out of candidate/quarantine only on smoke evidence; Claude runs `when_available` per its CLI transport; MiniMax and Copilot hold no binding at all. Recommended activation order: `claude-reviewer` first among gated routes, because grade-2 independence and the high-risk gate both hinge on a distinct verification path existing.

## 7. Context Isolation

Each worker receives the minimum context its task needs — the analyzer doc's bounded-contract rule, made concrete per role:

| Role | Receives | Must not receive |
| --- | --- | --- |
| `leader` | Full canonical conversation state, ledger state, budgets, approvals, worker results | — |
| `gpt-analyzer` | Task contract, relevant originals, relevant evidence, evaluation criteria | The full conversation; other workers' hidden reasoning; any framing that fixes the Head's conclusion as the answer |
| `gemini-investigator` | Investigation questions, source criteria, time range, forbidden sources, required evidence schema | Unrelated conversation context |
| `deepseek-drafter` | Verified fact set, artifact refs, audience, tone, format, forbidden claims | Unverified material |
| `glm-coder` | Acceptance criteria, repository scope, permitted files, test commands, tool policy, budget | Device/calendar data; unrelated context |
| `claude-reviewer` | Risk-tier dependent and **cumulative** (§2.6): low = contract + worker result + evaluation criteria; mid = low + relevant original excerpt; high = mid + full original user request + independent intent digest + Head contract, side by side | — |

The high-tier reviewer must compare the original user request against the Head's compiled contract directly — that comparison is the point of the tier — while still verifying the worker result against the evaluation criteria, which every tier carries.

## 8. Caveman Result Contracts

Role-specific payload schemas extend the analyzer doc's [Caveman contract](../2026-08-01-analyzer-bot-main-agent-design.md#worker-message-contract-caveman). Full v0.1 payload set:

```text
analysis_result.v1
research_result.v1
draft_result.v1
code_result.v1
review_result.v1
device_result.v1
generic_result.v1
```

### `analysis_result.v1`

```yaml
task_decomposition: []
constraints: []
assumptions: []
evaluation_criteria: []
candidate_methods: []
scores: []
synthesis:
unresolved_questions: []
evidence_refs: []
```

`scores` are advisory and are never a progress grade (ASC §6 computes grades from evidence, and §2.2's rule applies).

### `research_result.v1`

```yaml
queries: []
sources: []
findings: []
conflicts: []
unverified_claims: []
limitations: []
evidence_refs: []
```

### `draft_result.v1`

```yaml
artifact_ref:
audience:
tone:
format:
source_fact_refs: []
unsupported_claims: []
open_placeholders: []
```

A non-empty `unsupported_claims` blocks automatic success at the final gate.

### `code_result.v1`

```yaml
artifact_ref:
files_changed: []
commands_run: []
tests: []
diagnostics: []
known_failures: []
rollback_plan:
```

### `review_result.v1`

```yaml
verdict:
blocking_issues: []
warnings: []
assumption_failures: []
missing_tests: []
independence_axes: []
intent_mismatches: []
requested_changes: []
evidence_refs: []
```

The reviewer never returns a modified artifact.

### `generic_result.v1`

Emergency envelope only, exactly as the analyzer doc already gates it: legal only for legacy compatibility or carrying a `schema_mismatch` exception; `status: ok` invalid; never completion, progress, budget-extension, or resume evidence; always treated as `blocked` or `partial`.

## 9. Independence Rules

Governed entirely by [ASC §6](../2026-08-02-adaptive-strategy-controller.md)'s fixed axis enum — strong: `source_lineage`, `data_partition`, `verification_method`, `toolchain`; weak: `model_provider`, `prompt_context`, `sampling_path`, `evaluator_instance`; grade 2 needs one strong or two weak. This document adds the role-level consequences:

- `leader` + `gpt-analyzer` share a model route → never mutually independent verifiers.
- Gemini and Claude both re-reading the same source does not create `source_lineage` independence — same upstream, no strong axis.
- A path labeled `claude-reviewer` with the same prompt, context, and source lineage is not independent verification. Independence is a property of the path, not of the name on it.

## 10. Final Gate

Before any final response or execution, the leader checks, in order:

```text
1.  task lineage and task class match
2.  Caveman schema and version valid
3.  no worker exceeded its role scope
4.  evidence and artifacts exist
5.  no generic_result or Grade 3 bypass
6.  budget and strategy-switch caps not exhausted
7.  reviewer blocking issues resolved
8.  required independence level satisfied
9.  user approval present, if high-risk
10. PolicyGate permits the tool
11. canonical ledger event recorded before delivery
```

Any single failure stops the success/execution path.

## 11. Stored State

The agent registry and route states live in canonical state:

```yaml
agent_registry:
  leader:
    model_route: gpt-5.5
  gpt-analyzer:
    model_route: gpt-5.5
  gemini-investigator:
    runtime: Antigravity agy
    model_route: Gemini 3.1 Pro
  deepseek-drafter:
    model_route: deepseek-v4-flash
  glm-coder:
    model_route: z-ai/glm-5.2
  claude-reviewer:
    runtime: Claude CLI
    model_route: claude-opus-4-8
    availability: when_available
```

Concrete model IDs and route strings are **configuration, not natural law hardcoded into design** — this doc fixes roles and authority, not vendor strings. Changes emit ledger events:

```text
model_route_changed
worker_availability_changed
review_independence_degraded
fallback_activated
```

## 12. v0.1 Scope

Included: the six fixed logical roles; Head-centric dispatch; per-role Caveman schemas; per-role tool policy; availability states; explicit fallback/degraded handling; reviewer independence evaluation; risk-tier routing; the final gate; ledger events; **per-role usage metering** — tokens, cost, and tool calls recorded per `agent_id`, measure-only.

Excluded: dynamic worker creation; direct worker-to-worker negotiation; worker self-routing; automatic role reassignment by model performance; automatic model tournaments; inheritance-based worker evolution; Grade 3 activation (ASC §6); always-on reviewer chains; reviewer direct code modification; worker budget self-extension; **per-role hard caps** — v0.1 enforces only the ASC task-class budgets, and per-role caps are a v0.2 decision made from the metering logs, not guessed now.

## 13. Implementation Order

For the implementation repository (`openclaw-upstream`) — this repo stays docs-only. Per the [master plan](head-orchestrator-v01-master-plan.md) §21, implementation starts from a clean clone of the latest upstream `main`; the verified spike is donor/reference only, its nine files ported selectively rather than built upon:

```text
1.  AgentRole / AgentRegistry types
2.  RoleCapability / ToolProfile
3.  ModelRoute / Availability
4.  per-role Caveman payloads
5.  routing policy
6.  context projection
7.  worker adapters
8.  review independence evaluator
9.  final gate
10. ledger events
11. E2E tests
```

## 14. Required Tests

- Workers cannot invoke each other.
- No agent but the leader can send a user response.
- A `gpt-analyzer` score is never used as a progress grade.
- Leader + `gpt-analyzer` results never count as independent verification.
- A Gemini result without source refs fails validation.
- A DeepSeek draft carrying unverified claims is blocked at the final gate.
- GLM modifying files outside its permitted scope is blocked.
- `claude-reviewer` cannot modify artifacts.
- Reviewer unavailability is recorded explicitly.
- High-risk execution is blocked while Claude is unavailable and no independent path exists.
- A `generic_result` success bypass is blocked.
- A Grade 3 budget extension is blocked.
- A role fallback never inflates the independence grade.
- A worker result with a schema mismatch is blocked.
- No external execution without the Head's final gate.
- Repeated calls to the same worker never count as a new independent verifier.

## 15. Core Principles

```text
The Head speaks like a person.
The organization moves like a protocol.
Models perform roles.
Models do not own authority.

GPT analyzer builds structure.
Gemini finds evidence.
DeepSeek produces prose.
GLM produces code.
Claude doubts and verifies.
Only the leader decides and answers the user.
```

## Risks

| Risk | Mitigation |
| --- | --- |
| `analyzer-bot` (leader) confused with `gpt-analyzer` (worker) | §1 fixes the hierarchy explicitly; distinct `agent_id`s; the worker has no conversation channel |
| Two GPT-5.5 instances treated as two independent brains | Advisory-only scores; the leader/analyzer pair never counts as grade 2 (§2.2, §9) |
| Fallback silently inflates independence | `review_independence_degraded` / `fallback_activated` ledger events; degraded review never grade 2 (§6) |
| A worker quietly assumes another's role during an outage | Forbidden; the outage becomes a blocked state, not a substitution (§6) |
| Drafter launders unverified claims into prose | Verified-facts-only input; non-empty `unsupported_claims` blocks the final gate (§2.4, §8) |
| Reviewer verdict mistaken for execution approval | `pass ≠ execution approval`; the final gate is always the leader's (§2.6, §10) |
| Route strings ossify into design commitments | §11: routes are configuration; changes are ledger events, not doc edits |

## Open Questions

- ~~The delegation threshold~~ — **resolved**: v0.1 defaults in §2.2 (inline only for single-step `simple_query` with nothing to compare; dispatch otherwise).
- ~~Per-role sub-budgets~~ — **resolved**: v0.1 enforces task-class caps only and meters per-role usage; per-role hard caps move to v0.2 on operating logs (§12).
- Antigravity runtime capability parity for `gemini-investigator` — which tool-profile fields the runtime can actually enforce.
- Whether `claude-reviewer`'s degraded-mode substitute should be pinned to a fixed model family or chosen per task.

## References

- [Mixed Framework Orchestration Plan](../../README.md)
- [Analyzer-bot Main Agent Design](../2026-08-01-analyzer-bot-main-agent-design.md) — host architecture, Caveman contract, PolicyGate, risk tiers
- [Adaptive Strategy Controller](../2026-08-02-adaptive-strategy-controller.md) — grading, independence enum, budgets, stop/resume policy
- [LangGraph Head Reference Patterns](../2026-07-31-langgraph-head-reference-patterns.md) — provider gates this document inherits
