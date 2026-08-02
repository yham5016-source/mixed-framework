# Phase Implementation Checklist

Date: 2026-08-02

Status: derived from [README.md](../README.md)'s Implementation Plan, cross-checked against [Analyzer-bot Main Agent Design](2026-08-01-analyzer-bot-main-agent-design.md) and [Adaptive Strategy Controller](2026-08-02-adaptive-strategy-controller.md)

This document takes the six phases from the README's Implementation Plan and expands each with two additional sub-items pulled from the two design docs that superseded parts of the original plan (Team Lead Planner → `analyzer-bot` Head Orchestrator; 7 workers → 4 LangChain bots). Checklists are the primary content; outcome and exit criteria are kept to one line each for orientation only.

---

## Phase 0: Inventory

Outcome: know what is currently running and where the boundaries are.

- [ ] List active services: OpenClaw gateway, Hermes gateway, Discord bot bridge, `outta-mcp`.
- [ ] Capture existing environment variables without printing secrets.
- [ ] Identify where Discord events enter OpenClaw.
- [ ] Identify where iPhone node requests are routed.
- [ ] Identify whether Hermes has a usable worker profile with NVIDIA credentials.
- [ ] Confirm the canonical checkpoint DB path (`~/.openclaw/agents/analyzer/agent/analyzer-state.sqlite`) does not collide with the existing `openclaw-agent.sqlite` for `main` at the same directory level.
- [ ] Confirm the `analyzer` agentId already exists (session keys of the form `agent:analyzer:discord:chan-1`, autonomous allowlist `["main", "analyzer"]`) so Phase 2 promotes an existing agent rather than creating a parallel one.

Exit criteria: service inventory document exists; secrets referenced by name only; a rollback note exists per service touched.

---

## Phase 1: Discord UX Stabilization

Outcome: OpenClaw feels responsive even before worker orchestration is rebuilt.

- [ ] Add fast acknowledgement for inbound messages.
- [ ] Add owner/status display.
- [ ] Add 30-second progress heartbeat for long tasks.
- [ ] Add button controls for plan approval/rejection/revision.
- [ ] Add text fallback commands for the same controls (`/plan approve`, `/plan reject`, `/plan revise`, `/status`, `/handoff`).
- [ ] Add idempotency key per request.
- [ ] Map the owner/status display to the nine dispatch states (`intent` → `planning` → `waiting_approval` → `approved` → `dispatched` → `worker_running`/`worker_blocked` → `reviewing` → `needs_revision` → `complete`) instead of an ad hoc status vocabulary.
- [ ] Ensure a stalled OAuth/provider failure renders as a `blocked`/`failed` state rather than silently re-triggering the OAuth flow while the user is waiting on another task.

Exit criteria: user can see who owns the task; user can approve/reject without re-mentioning the bot; a stalled provider becomes a clear blocked/failed state.

---

## Phase 2: `analyzer-bot` Plan-Only Mode

> Superseded naming: was "Team Lead Plan-Only Mode." Planning now lives inside `analyzer-bot`'s Head Orchestrator.

Outcome: `analyzer-bot`'s Head Orchestrator cannot accidentally execute.

- [ ] Create the `analyzer-bot` OpenClaw agent profile.
- [ ] Remove shell/Git/OAuth/device tools from the `analyzer-bot` profile — enforced by tool profile + PolicyGate, not prompt wording.
- [ ] Allow only planning, review, dispatch, and status tools.
- [ ] Implement `waiting_approval` as a LangGraph interrupt; approval resumes the graph, rejection routes to `needs_revision`.
- [ ] Add rejection path back to plan revision.
- [ ] Wire the `analyzer-state.sqlite` StateStore (`ledger_events`, `graph_checkpoints`, `artifacts`, `approval_tokens`, `core_assumptions`, `core_assumption_aliases`, `core_assumption_registry_meta`) and confirm a restart replays `ledger_events` into the same canonical state.
- [ ] Register `analyzer-bot` as an allowed proposer under `skills.workshop.autonomous` (proposals land as `pending` only) and confirm subagent/worker sessions stay excluded from proposing skills.

Exit criteria: `analyzer-bot` can produce and revise plans; its profile cannot run shell commands; no worker dispatch happens before approval.

---

## Phase 3: Worker Mapping

Outcome: each worker has a narrow role and tool set.

- [ ] Attach `outta-mcp` and Ponytail to Codex Coding Agent.
- [ ] Attach Apify MCP only to Researcher with an Actor allowlist.
- [ ] Attach Ponytail Review to Critic.
- [ ] Attach test verification responsibilities to Tester.
- [ ] Attach iPhone node only to Device Agent.
- [ ] Attach systemd/log/OAuth runbook tools only to Operator.
- [ ] Reconcile the table above into the four v0.1 LangChain bots (Codex → `coder-bot`; Apify + web search → `searcher-bot`; Critic + Tester → `reviewer-bot`; new `drafter-bot`) and update the role/tool matrix so it matches the 8/1 worker mapping, not the original seven-role split.
- [ ] Implement the Caveman envelope, per-task-kind payload schemas (`code_result.v1`, `research_result.v1`, `review_result.v1`, `device_result.v1`), and the five exception channels, with `generic_result.v1` gated as a non-completion path from the start.

Exit criteria: tool exposure is role-specific; no single worker has all tools; research/coding/testing/device/operations paths are separable.

---

## Phase 4: Hermes Backend Worker Pool

Outcome: Hermes can be used without competing with OpenClaw for Discord ingress.

- [ ] Start Hermes only as a backend worker pool.
- [ ] Connect the NVIDIA API key to the Hermes worker profile without logging the key.
- [ ] Add a dispatch route from `analyzer-bot`'s Head Orchestrator to Hermes worker tasks.
- [ ] Use Hermes for background research, critique, synthesis, and planner-review.
- [ ] Enforce the data-plane/control-plane split for Hermes tasks: artifact exchange goes through the Ledger; any goal, budget, method, or scope change routes back through the Head — no direct Hermes-to-Hermes or Hermes-to-Head shortcut.
- [ ] Apply per-node retry/lease/heartbeat/timeout/cancel semantics to Hermes dispatch so a result arriving after its lease expired is rejected, not merged.

Exit criteria: Hermes receives tasks from the LangGraph graph directly (no separate Dispatch Broker); Hermes does not listen to Discord in v0.1; results return to the Head Orchestrator for review before Discord output.

---

## Phase 5: Verification and Runbook

Outcome: the system is operable and debuggable.

- [ ] Add smoke test: Discord message → OpenClaw acknowledgement.
- [ ] Add smoke test: plan draft → approval wait state.
- [ ] Add smoke test: approval → worker dispatch.
- [ ] Add smoke test: worker blocked report.
- [ ] Add smoke test: worker result → Head Orchestrator review.
- [ ] Add smoke test: final answer → Discord.
- [ ] Add manual runbook entries: restart OpenClaw; restart Hermes worker pool; inspect active tasks; cancel active task; recover from OAuth/provider failure.
- [ ] Add the ASC-derived verification set to CI/smoke coverage: `make test-v01-replay`, `make test-outbox-dedupe`, `make test-approval-resume`, `make test-route-reconnect`, `make test-worker-timeout-cancel`.
- [ ] Add a runbook entry for inspecting and retiring `core_assumption` registry records (`proposed`/`active`/`challenged`/`retired`/`superseded`) without rewriting history — retirement only, never in-place edits.

Exit criteria: a dry-run request passes through the whole route without mutating external state; failures report as blocked/failed with owner and next action; a human can restart or inspect the system without guessing.

---

## References

- [Mixed Framework Orchestration Plan](../README.md)
- [Analyzer-bot Main Agent Design](2026-08-01-analyzer-bot-main-agent-design.md)
- [Adaptive Strategy Controller](2026-08-02-adaptive-strategy-controller.md)
