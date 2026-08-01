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

Control semantics:

- `waiting_approval` is a LangGraph **interrupt**. Approval resumes the graph; rejection routes to `needs_revision`. This is exactly what the 7/31 note recommends LangGraph for — *"Head state machine, interrupt, resume, and checkpoint flow."*
- Approval tokens carry explicit scope, expiry, one-time use, cancel semantics, and an audit trail.
- Retry / abort / resume is per-node: retry policy, worker lease with heartbeat, timeout, and cancel. A result arriving after its lease expired is rejected, not merged.
- Worker results are schema- and version-validated. Malformed output is rejected rather than summarized.
- The checkpoint store *is* the append-only ledger. Restarting `analyzer-bot` replays the ledger into the same canonical state.
- Duplicate inbound events, worker commands, worker results, and delivery receipts are harmless — idempotency keys per request.

## What The Head Never Does Directly

Inherited verbatim from README's Team Lead forbidden list, now applied to `analyzer-bot`:

- No direct code edits.
- No shell execution.
- No git push.
- No OAuth repair loops.
- No device mutation.

Merging conversation and planning into one agent revives the risk README avoided by splitting them — *"Team Lead executes instead of plans."* The mitigation is that these restrictions are enforced by **OpenClaw tool profile plus PolicyGate**, not by prompt wording. A tool the Head must not use is not exposed to the Head's profile at all.

## Relationship To Runtime Skill Evolution

The [runtime skill evolution loop](superpowers/plans/2026-07-10-openclaw-runtime-skill-evolution.md) is already live on the gateway, so the interaction needs to be explicit:

- `analyzer-bot` is an allowed proposer under `skills.workshop.autonomous`. Proposals are always created as `pending`; nothing is written to a live `SKILL.md` without an explicit apply.
- Worker sessions spawned by the graph fall into the subagent session class, which the loop excludes. Bots therefore cannot propose skills. That exclusion is intentional — skill evolution should learn from the conversation the user actually had, not from internal dispatch traffic.
- Operational note carried forward: toggling `skills.workshop.autonomous.enabled` reports "no gateway restart needed", but agent runs keep the old value until the gateway is restarted. Restart after toggling.

## v0.1 Scope

Build:

- `analyzer-bot` OpenClaw agent profile with a restricted tool set.
- LangGraph graph: nodes, branching, interrupt/resume, checkpointing.
- Append-only ledger with replay.
- Approval tokens with scope, expiry, one-time use, and cancel.
- LangChain wrappers for the four bots.
- `WorkerCommand` / `WorkerResult` typed schemas with version validation.
- PolicyGate for external side effects.

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
| Merging conversation and planning raises accidental-execution risk | Tool exposure gating at the OpenClaw profile and PolicyGate, not prompt instructions |
| LangChain is mistaken for the conducting layer | Layer boundary table plus the one-sentence rule; LangChain never selects the next step |
| The graph grows into a general workflow engine | v0.1 nodes limited to four bots, approval, and review |
| `analyzer-bot` context grows unbounded | Ledger owns state; bots receive bounded contracts only |
| Head boundary gets welded to OpenClaw internals | Contracts defined without OpenClaw-specific types, so the Head can move hosts later |
| Workers self-report success falsely | Head requires evidence before accepting a result; schema validation rejects malformed output |

## Open Questions

- Head model: keep GPT-5.5 as the primary reasoning candidate from the 7/31 note?
- Per-bot model assignment: is GLM-5.2 the default worker model for all four, or per-bot?
- Checkpoint store: reuse the OpenClaw state DB, or a separate ledger store?
- Does Tester split back out of `reviewer-bot` in v0.2, or stay merged?
- When do Device Agent and Operator return as first-class roles?

## References

- [Mixed Framework Orchestration Plan](../README.md)
- [LangGraph Head Reference Patterns](2026-07-31-langgraph-head-reference-patterns.md)
- [OpenClaw Runtime Skill Evolution Implementation Plan](superpowers/plans/2026-07-10-openclaw-runtime-skill-evolution.md)
- LangGraph — state machine, interrupt, resume, checkpoint flow
- OpenAI Agents SDK — handoff boundaries, guardrails, human-in-the-loop, tracing
- 12-factor-agents — Head owns context, workers receive bounded contracts
- Hatchet and durable workflow systems — lease, heartbeat, timeout, cancel, idempotency
