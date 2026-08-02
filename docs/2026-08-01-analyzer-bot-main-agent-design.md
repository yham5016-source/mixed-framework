# Analyzer-bot Main Agent Design

Date: 2026-08-01

Status: v0.1 draft, awaiting review

Decision: single main agent (`analyzer-bot`) hosted on OpenClaw, orchestrating with LangGraph, calling LangChain-wrapped specialist bots

> **Supersedes:** this document replaces the `Team Lead Planner` sections of [README.md](../README.md) — the separate planner agent described there (Core Architecture, Framework Split, Agent Contracts) is folded into `analyzer-bot`. README remains accurate for Discord ingress, worker boundaries, and dispatch states.

## One-Line Decision

`analyzer-bot` is the single main agent attached to OpenClaw; its internal Head Orchestrator owns state and call order through LangGraph, and invokes coder / drafter / reviewer / searcher bots as LangChain-standardized components.

## Why This Document Exists

Two existing documents disagree about who owns the main agent.

- [README.md](../README.md) (2026-07-10) puts a **separate `Team Lead Planner` agent** between the OpenClaw conversation agent and seven workers.
- [LangGraph Head Reference Patterns](2026-07-31-langgraph-head-reference-patterns.md) (2026-07-31) reverses that: a single **Head** owns conversation, planning, judgment, and state, and explicitly lists `Team Lead Planner as a separate agent`, `LLM Dispatch Broker as an agent`, and `LangChain Supervisor / AgentExecutor` as things to keep out of v0.1.

Read the first and there are three agent layers. Read the second and there is one. An implementer cannot decide what to build from that. This document settles it.

The 7/31 note already contains the resolution in its own closing line — *"Head is not a model name. Head is the boundary that owns conversation, state, policy interpretation, ledger replay, and contract validation."* Head is a boundary, so it needs a host. In v0.1 that host is an OpenClaw agent profile named `analyzer-bot`.

## Structure

```text
사용자
  ↕
OpenClaw edge  (Discord ingress, session, route, delivery, iPhone node)
  └─ analyzer-bot            ← OpenClaw agent profile
      └─ Head Orchestrator
          └─ LangGraph
              ├─ 상태 관리 (checkpoint / ledger)
              ├─ 작업 분기
              ├─ 호출 순서
              ├─ 재시도 · 중단 · 재개 (interrupt / resume)
              └─ LangChain으로 규격화된 전문 봇
                  ├─ coder-bot
                  ├─ drafter-bot
                  ├─ reviewer-bot
                  └─ searcher-bot
```

`analyzer-bot` is not a new identity. It already exists as an OpenClaw agent id — the runtime skill evolution work uses `agentId: "analyzer"`, session keys of the form `agent:analyzer:discord:chan-1`, and an autonomous allowlist of `["main", "analyzer"]`. This design promotes that existing agent to main-agent status rather than introducing a parallel one.

## Layer Boundaries

This is the section to read first. The most likely misreading of the diagram is that LangChain conducts the bots. It does not.

| Layer | Owns | Does not own |
| --- | --- | --- |
| OpenClaw | Discord and device ingress, session, route health, delivery, agent hosting, tool profile enforcement | Planning, judgment, worker call order |
| `analyzer-bot` | Conversation with the user, intent interpretation, final answer accountability | Direct execution — shell, git, OAuth, device writes |
| LangGraph | State, branching, call order, retry / interrupt / resume | Model invocation shape, tool binding |
| LangChain | Wrapping each bot as `model + tools + prompt` into a connectable component | Conducting, ordering, Supervisor / AgentExecutor behavior |

In one sentence: **LangChain is the connector standard; LangGraph is the brain that decides what to plug in and when.**

This is consistent with the 7/31 note's `Do not add LangChain Supervisor, AgentExecutor, or a second planner` — that rule bans LangChain as an orchestration layer, and this design uses it only as a component-packaging layer. No conflict; this document narrows the rule rather than bending it.

## Conflict Resolution

Item-by-item ruling so no implementer has to ask which document wins.

| Item | README (7/10) | LangGraph note (7/31) | Final |
| --- | --- | --- | --- |
| Discord ingress | OpenClaw only | OpenClaw edge | OpenClaw only — unchanged |
| Conversation ownership | OpenClaw Conversation Agent | Head | `analyzer-bot` |
| Planning ownership | Team Lead Planner (separate agent) | Head | Head Orchestrator inside `analyzer-bot` |
| Ordering and branching | Dispatch Broker (separate) | Queue | The LangGraph graph itself |
| LangGraph | Not mentioned | Head state machine | Required in v0.1, inside `analyzer-bot` |
| LangChain | Not mentioned | Supervisor forbidden | Component packaging only |
| State ownership | Implicit | Append-only ledger | LangGraph checkpoint *is* the ledger |
| Device access | Device Agent | Edge | OpenClaw edge capability behind PolicyGate |

## Worker Mapping

The seven workers in README fold into four bots. This is a mapping, not a deletion — every README role has a destination.

| README worker | v0.1 destination |
| --- | --- |
| Codex Coding Agent | `coder-bot` |
| Apify Researcher + normal web search | `searcher-bot` |
| Critic / Ponytail Review | `reviewer-bot` |
| Tester | absorbed into `reviewer-bot`'s counterexample checking (v0.1 only) |
| Hermes Worker Pool | not a bot role — an execution backend option behind a node |
| Device Agent | OpenClaw edge capability behind PolicyGate, not a LangGraph node |
| Operator | out of v0.1 scope; human runbook |
| — | `drafter-bot` is new; README had no equivalent |

## Bot Contracts

Each bot is `model + tools + rules`, wrapped by LangChain into a uniform callable.

### `coder-bot`

- Coding model + code tools + dedicated prompt.
- Evidence rules inherited from README: RED before GREEN, mutation proof, PASS_TO_PASS, exact command output for completion claims, clean git status before final claim.
- Must not: negotiate plans with the user, run broad scraping, touch device data.

### `searcher-bot`

- Search model + search tools + source-return rules.
- Normal search first; escalate to Apify MCP only when the original source is missing, a social URL is detected, or evidence is weak. Actor allowlist stays as in README.
- Must distinguish `preview`, `blocked`, `failed`, `no_results`, and `executed`. Never report "searched, no result" when the provider was unavailable.
- Must not: edit code or mutate repositories.

### `reviewer-bot`

- Review model + evaluation criteria + counterexample checking.
- Leads with bugs, risks, regressions, and missing tests, with file and line references. Runs over-engineering pressure separately from correctness review.
- For v0.1 it also owns test verification (FAIL_TO_PASS, PASS_TO_PASS, mutation proof).
- Must not: implement feature code.

**Risk-tiered input scope.** Same JSON contract everywhere is how a Head mis-compression survives review — the reviewer just re-checks the compiled artifact against itself. What the reviewer sees scales with how hard the decision is to undo:

| Tier | Example | Reviewer input |
| --- | --- | --- |
| Low | doc cleanup, rename, simple search | Caveman contract + worker result + evaluation criteria |
| Mid | code change, design call, ambiguous research | + relevant excerpt of the original request |
| High | delete, deploy, external send, payment, permission change | + original user request (full) + an independently generated intent digest + the Head's contract, compared side by side |

The independent intent digest (high tier only) is produced by the `llm-task` tool (`docs/tools/llm-task.md`) — a single JSON-only, no-tools LLM call, not a new agent or a standing reviewer chain, so this stays inside the 7/31 note's non-scope list. Its input is **only** the original user request plus a fixed intent-extraction schema and the minimum original context needed; it must **never** receive the Head's contract, the Head's own intent summary, or the Head's conclusion. Feeding it the Head's compiled output turns "independent re-derivation" into "restate the Head's summary more confidently" — a different model calling the same compressed input is not independence. Configure a distinct model for these calls (`llm-task`'s per-call `model`/`allowedModels`) so the digest doesn't share the Head's specific failure modes.

### `drafter-bot`

- Writing model + document templates + result consolidation.
- Turns worker output into the shape the user actually receives.
- Must not: execute anything, or hold judgment authority.

Common rule: every bot receives a **bounded contract**, not the full conversation. The Head owns context; workers get only what their contract carries. Device and calendar data never enter a bot prompt unless required and explicitly approved.

## Graph: State, Branching, Resume

The dispatch states in README are reinterpreted as LangGraph nodes and kept as-is:

`intent` → `planning` → `waiting_approval` → `approved` → `dispatched` → `worker_running` / `worker_blocked` → `reviewing` → `needs_revision` → `complete`

Branching from the Head Orchestrator, reading graph state:

- 검색 필요 → `searcher-bot`
- 코드 필요 → `coder-bot`
- 초안 필요 → `drafter-bot`
- 검증 필요 → `reviewer-bot`

Each result returns to the Head, which decides the next node. Bots never call each other directly.

This rule stays absolute for *invocation* — no bot calls another bot's model or triggers its run. It does not ban data exchange. A bot that needs another bot's artifact (e.g. `tester` needing `coder`'s build command) requests it through the Ledger/artifact store, not by calling `coder-bot`:

```text
coder-bot   -> registers artifact in the Ledger
tester-bot  -> artifact_request(artifact_id) -> Head/broker -> Ledger lookup -> artifact_ref returned
```

The boundary is data plane vs. control plane, not "bots may never exchange information":

- **Data plane** (Ledger-mediated artifact reference): allowed without Head deliberation.
- **Control plane** (goal, budget, method, hypothesis, or authority changes): Head only, no exceptions.
- **Direct bot-to-bot invocation**: still forbidden.

Control semantics:

- `waiting_approval` is a LangGraph **interrupt**. Approval resumes the graph; rejection routes to `needs_revision`. This is exactly what the 7/31 note recommends LangGraph for — *"Head state machine, interrupt, resume, and checkpoint flow."*
- Approval tokens carry explicit scope, expiry, one-time use, cancel semantics, and an audit trail.
- Retry / abort / resume is per-node: retry policy, worker lease with heartbeat, timeout, and cancel. A result arriving after its lease expired is rejected, not merged.
- Worker results are schema- and version-validated. Malformed output is rejected rather than summarized.
- The checkpoint store *is* the append-only ledger. Restarting `analyzer-bot` replays the ledger into the same canonical state.
- Duplicate inbound events, worker commands, worker results, and delivery receipts are harmless — idempotency keys per request.

## Worker Message Contract ("Caveman")

Bare typed envelopes (`{ status, body: string }`) fix the wrapper but not the problem this document opened with: Head compiles the user's intent into a JSON contract, and workers execute that contract faithfully even when the compilation lost something. A worker that can only fill in the fields Head defined has no way to say "this contract looks wrong." Caveman is the message contract that gives it one, without reopening free-form back-and-forth.

**Caveman and the Adaptive Strategy Controller are one design, not two layers.** Caveman defines what a message can say — the grammar and typed payload. ASC defines who adjudicates what gets said — grading, independence, stop/resume policy. An exception channel with nothing grading it is just an unpoliced bypass of the whole contract system; ASC's rules in [Adaptive Strategy Controller](2026-08-02-adaptive-strategy-controller.md) are what keep the channels below from becoming that.

### Envelope

```ts
interface CavemanEnvelope<TPayload> {
  schema: string;
  schemaVersion: string;
  taskId: string;
  producer: string;
  status: "ok" | "partial" | "failed" | "blocked" | "cancelled";
  payload: TPayload;
  evidenceRefs: string[];
  exceptions: ContractException[];
  resourceUsage: ResourceUsage;
}
```

### Per-task payload, not one free-text field

`body: string` in the current spike (`openclaw-upstream`'s `HeadTransitionWorkerCommand`/`WorkerResult`) is a schema in name only — the payload itself is unconstrained prose. v0.1 replaces it with typed payloads per task kind: `code_result.v1`, `research_result.v1`, `review_result.v1`, `device_result.v1`, plus `generic_result.v1` for what does not fit yet (see restriction below). A schema is a cognitive frame as much as a wire format — one universal payload silently limits what a worker considers reporting; per-kind schemas keep that frame matched to the work.

### Exception channels

Workers cannot renegotiate their own contract, but they can raise one of five typed exceptions instead of silently working around a bad one:

```ts
type ContractExceptionType =
  | "needs_clarification"
  | "schema_mismatch"
  | "out_of_scope_finding"
  | "assumption_violation"
  | "novel_observation";
```

**A worker may raise an exception. A worker may not modify its own contract, budget, or scope by raising one.** Adjudication splits two ways:

- **Deterministic, validator-checked:** missing required field, enum mismatch, schema version mismatch, explicit contract contradiction. No judgment call.
- **Semantic, Head-adjudicated:** ambiguous intent, contract possibly diverging from the original request, a scope-external finding, a broken core assumption. High-risk tasks route these to `reviewer-bot`'s high-tier path (original request + independent digest), not to Head re-reading its own contract.

Hard limits, so the exception channel doesn't reopen the long-meeting problem it replaces:

- Raising an exception never auto-increases budget or scope.
- `out_of_scope_finding` is report-only by default — it does not authorize the worker to act on it.
- A repeated exception on the same claim/signature consumes budget exactly like a normal attempt (ASC §4-5); it is not a free retry.
- Unfounded repeated exceptions are grounds for `suspended`, same as unfounded repeated attempts.
- A worker never self-assigns `contract_status: verified` or a progress grade by raising or resolving an exception — ASC computes grades from evidence, never from a worker's own claim (ASC §6).

Think of these as an alarm, not a door: pulling it gets someone's attention, it doesn't open anything by itself.

### `generic_result.v1` is not a success path

`generic` payload exists only as an emergency envelope for "this contract cannot express my result" — it is **not** a normal completion type:

- Legal uses: legacy compatibility during the `body: string` migration, or carrying a `schema_mismatch` exception.
- `status: ok` (or any completion claim) is not valid with a `generic` payload.
- Never counts as progress evidence, never satisfies Grade 1–3, never justifies a budget or credit extension (ASC §4-6).
- ASC treats every `generic` result as `blocked` or `partial`, nothing else.
- To proceed, the Head must reissue the task on the correct typed schema — or, if no existing schema fits, escalate to a human and stop, rather than let `generic` quietly become the default path. An unmetered generic path is exactly how per-task schemas erode back into free text.

## What The Head Never Does Directly

Inherited verbatim from README's Team Lead forbidden list, now applied to `analyzer-bot`:

- No direct code edits.
- No shell execution.
- No git push.
- No OAuth repair loops.
- No device mutation.

Merging conversation and planning into one agent revives the risk README avoided by splitting them — *"Team Lead executes instead of plans."* The mitigation is that these restrictions are enforced by **PolicyGate** (which is itself built from `analyzer-bot`'s tool profile, exec-approval flow, and sandbox tool policy — see [What PolicyGate Actually Is](#what-policygate-actually-is)), not by prompt wording. A tool the Head must not use is not exposed to the Head's profile at all.

## What PolicyGate Actually Is

`PolicyGate`, used above and in v0.1 Scope, is not a class to build. It is a name for the composition of three primitives that already exist in OpenClaw — v0.1 wires them together, it does not add a fourth:

| Component | What it gates | Where it lives |
| --- | --- | --- |
| `tools.profile` | Which tools an agent can call at all (`minimal` / `messaging` / `coding`, per-agent override) | `agents.list[].tools.profile`, `docs/cli/policy.md` |
| Exec-approval flow | Shell/exec actions that need a human yes/no before running | `src/agents/bash-tools.exec-approval-request.ts`, `src/agents/bash-tools.exec-approval-followup.ts`, `extensions/discord/src/approval-runtime.ts` |
| Sandbox tool policy | What a sandboxed session may touch, independent of tool profile | `docs/gateway/sandbox-vs-tool-policy-vs-elevated.md` |

"PolicyGate" (§What The Head Never Does Directly) means: restrict `analyzer-bot`'s tool profile so the forbidden tools are absent, and route anything that reaches exec-approval or sandbox policy through those existing gates. No new gate object, no new config surface — this matches the repo's own bar for adding config (`AGENTS.md`: prove existing behavior can't solve it first).

## Relationship To Runtime Skill Evolution

The [runtime skill evolution loop](superpowers/plans/2026-07-10-openclaw-runtime-skill-evolution.md) is already live on the gateway, so the interaction needs to be explicit:

- `analyzer-bot` is an allowed proposer under `skills.workshop.autonomous`. Proposals are always created as `pending`; nothing is written to a live `SKILL.md` without an explicit apply.
- Worker sessions spawned by the graph fall into the subagent session class, which the loop excludes. Bots therefore cannot propose skills. That exclusion is intentional — skill evolution should learn from the conversation the user actually had, not from internal dispatch traffic.
- Operational note carried forward: toggling `skills.workshop.autonomous.enabled` reports "no gateway restart needed", but agent runs keep the old value until the gateway is restarted. Restart after toggling.

## Verified Spike

A contract-and-ledger spike exists against real OpenClaw internals, parked on its own branch — not merged, not part of this repo's history:

**[`openclaw-upstream@bf46d01`](https://github.com/yham5016-source/openclaw-upstream/commit/bf46d017ce0c999890e40ba50983c0b59fae4672)** (branch `head-transition-v01`)

- `HeadTransitionInboundEvent` / `Decision` / `WorkerCommand` / `WorkerResult` / `DeliveryReceipt` contracts, typed and validated (`src/head-transition/contracts.ts`).
- An append-only in-memory ledger with replay: invalid payloads fail before append, duplicate idempotency keys are harmless, replay rebuilds separate canonical maps per event kind (`src/head-transition/ledger.ts`).
- A durable `SQLiteLedger` next to it (`src/head-transition/sqlite-ledger.ts`) — same append/replay contract, backed by a dedicated SQLite file via `node:sqlite` + Kysely, per `openclaw-upstream`'s no-JSON/JSONL-sidecar storage policy. The in-memory ledger is unchanged and still the default; this is a second option, not a replacement.
- A Discord preflight-to-contract adapter that type-checks against the real `DiscordMessagePreflightContext` (`src/head-transition/discord-inbound-adapter.ts`) — not a hypothetical interface.
- 31/31 tests passing as of this commit. The `SQLiteLedger` addition followed RED-first TDD (module-not-found failure confirmed before the implementation existed) plus a mutation-proof pass (flipped the duplicate-detection condition, confirmed 2 tests failed, reverted).

This code does not implement the code in this repo — this repo stays docs-only. The link is the evidence trail from design to a fixed, runnable commit.

**Still open:** which store is canonical — the dedicated `SQLiteLedger` file, the shared OpenClaw state DB, or a per-agent DB — is unresolved (see Open Questions below). The dedicated-file version exists so this decision doesn't block having a durable ledger at all.

## v0.1 Scope

Build:

- `analyzer-bot` OpenClaw agent profile with a restricted tool set.
- LangGraph graph: nodes, branching, interrupt/resume, checkpointing.
- Append-only ledger with replay.
- Approval tokens with scope, expiry, one-time use, and cancel.
- LangChain wrappers for the four bots.
- `WorkerCommand` / `WorkerResult` typed schemas with version validation — the `body: string` payload field is superseded by the Caveman envelope below, not built as free text and migrated later.
- Caveman envelope, per-task-kind payload schemas, and the five exception channels — see [Worker Message Contract ("Caveman")](#worker-message-contract-caveman). `generic_result.v1` ships gated (never a completion path) from the start, not added as a restriction later.
- Reviewer risk tiers (low / mid / high input scope) for `reviewer-bot`, including the `llm-task`-based independent intent digest for high-tier review.
- PolicyGate wiring for external side effects — composing the three existing primitives (see [What PolicyGate Actually Is](#what-policygate-actually-is)), not a new class.
- Strategy control skeleton — pre-registered attempt specs, three counters (`attempt_credit`, `same_failure_confirmations`, `strategy_switch_count`), multi-dimensional budgets, local and global stop, and raw `progress_claims` as the structured worker return with grades derived by rule. Specified in [Adaptive Strategy Controller](2026-08-02-adaptive-strategy-controller.md). This is Head-internal policy and state; it adds no agent, no standing reviewer chain, and no dynamic agent creation, so the non-scope list below still holds.

Do not build (inherited from the 7/31 note):

- LangChain Supervisor or AgentExecutor.
- A separate Team Lead Planner agent.
- A Dispatch Broker as an agent.
- Kafka or RabbitMQ.
- Vector DB.
- Always-on reviewer/tester chain.
- Dynamic agent creation.
- A second Discord frontend.

## Acceptance Criteria

- `analyzer-bot` restart replays the checkpoint/ledger into the same canonical state.
- Duplicate inbound event, worker command, worker result, and delivery receipt are harmless.
- Approval interrupt suspends the graph; resume continues from the same node; cancel is recorded.
- Worker heartbeat timeout cancels the lease and rejects late results.
- OpenClaw reconnect recovers route ownership and pending delivery state.
- Worker result schema/version validation blocks malformed output.
- An operator can inspect replay and audit evidence without reading raw private context.

Verification set carried over from the 7/31 note:

```bash
make test-v01-replay
make test-outbox-dedupe
make test-approval-resume
make test-route-reconnect
make test-worker-timeout-cancel
```

## Risks

| Risk | Mitigation |
| --- | --- |
| Merging conversation and planning raises accidental-execution risk | Tool exposure gating at PolicyGate (tool profile is one of its three components — see [What PolicyGate Actually Is](#what-policygate-actually-is)), not prompt instructions |
| LangChain is mistaken for the conducting layer | Layer boundary table plus the one-sentence rule; LangChain never selects the next step |
| The graph grows into a general workflow engine | v0.1 nodes limited to four bots, approval, and review |
| `analyzer-bot` context grows unbounded | Ledger owns state; bots receive bounded contracts only |
| Head boundary gets welded to OpenClaw internals | Contracts defined without OpenClaw-specific types, so the Head can move hosts later |
| Workers self-report success falsely | Head requires evidence before accepting a result; schema validation rejects malformed output |

## Open Questions

- Head model: keep GPT-5.5 as the primary reasoning candidate from the 7/31 note?
- Per-bot model assignment: is GLM-5.2 the default worker model for all four, or per-bot?
- Checkpoint store: the spike's dedicated `SQLiteLedger` file, the shared OpenClaw state DB, or a per-agent DB?
- Does Tester split back out of `reviewer-bot` in v0.2, or stay merged?
- When do Device Agent and Operator return as first-class roles?

## References

- [Mixed Framework Orchestration Plan](../README.md)
- [Verified spike commit](https://github.com/yham5016-source/openclaw-upstream/commit/bf46d017ce0c999890e40ba50983c0b59fae4672) — `openclaw-upstream@bf46d01`, branch `head-transition-v01`, contracts + in-memory ledger + SQLiteLedger + Discord adapter, 31/31 tests
- [Adaptive Strategy Controller](2026-08-02-adaptive-strategy-controller.md) — how the Head decides when to stop, switch, suspend, and destroy a method
- [LangGraph Head Reference Patterns](2026-07-31-langgraph-head-reference-patterns.md)
- [OpenClaw Runtime Skill Evolution Implementation Plan](superpowers/plans/2026-07-10-openclaw-runtime-skill-evolution.md)
- LangGraph — state machine, interrupt, resume, checkpoint flow
- OpenAI Agents SDK — handoff boundaries, guardrails, human-in-the-loop, tracing
- 12-factor-agents — Head owns context, workers receive bounded contracts
- Hatchet and durable workflow systems — lease, heartbeat, timeout, cancel, idempotency
