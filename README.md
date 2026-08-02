# Mixed Framework Orchestration Plan

Status: v0.1 planning repository

Date: 2026-07-10

Decision: mixed framework, single ingress

## Current Notes

- [LangGraph Head Reference Patterns](docs/2026-07-31-langgraph-head-reference-patterns.md) - reference systems and patterns to absorb into the OpenClaw Edge + LangGraph Head v0.1 plan.
- [Analyzer-bot Main Agent Design](docs/2026-08-01-analyzer-bot-main-agent-design.md) - settles the main agent question; supersedes the Team Lead Planner sections below.
- [Adaptive Strategy Controller](docs/2026-08-02-adaptive-strategy-controller.md) - how the Head counts attempts, grades progress, and stops; the strategy-control skeleton inside analyzer-bot.
- [Head Role and Model Routing v0.1](docs/architecture/head-role-model-routing-v01.md) - the six fixed roles (leader + five workers), their model routes, authority matrix, fallback rules, and final gate.
- [Head Orchestrator v0.1 Master Plan](docs/architecture/head-orchestrator-v01-master-plan.md) - the consolidated implementation-governing plan: full architecture, phased roadmap (P0-P7), rollout stages, and test strategy.

## One-Line Decision

Use OpenClaw as the only Discord and device-facing entry point, then route work to Codex, Hermes, Apify MCP, and other MCP-backed workers behind it.

## Problem

The current Discord agent experience feels unstable because planning, execution, long-running work, device access, and status feedback are all perceived as one blurred agent. Users cannot reliably tell who is working, whether a message was received, whether a plan is waiting for approval, or whether an external worker stalled.

The fix is not to replace everything at once. The fix is to make the boundary strong:

- OpenClaw owns conversation ingress, Discord UX, and iPhone node access.
- Planning, review, approval, rejection, and dispatch live inside `analyzer-bot`'s Head Orchestrator — not a separate Team Lead Planner agent. *(Updated 2026-08; see [Analyzer-bot Main Agent Design](docs/2026-08-01-analyzer-bot-main-agent-design.md).)*
- Execution workers do coding, research, testing, review, device actions, and operations under explicit contracts.

## Core Architecture

*Updated 2026-08 to match [Analyzer-bot Main Agent Design](docs/2026-08-01-analyzer-bot-main-agent-design.md) — the prior diagram had a separate `Team Lead Planner` and `Dispatch Broker / Queue`; both are folded into `analyzer-bot`'s Head Orchestrator, and LangGraph owns branching directly with no separate broker.*

```text
Discord / device
  -> OpenClaw edge (ingress, session, route, delivery, iPhone node)
     -> analyzer-bot (OpenClaw agent profile)
        -> Head Orchestrator
           -> LangGraph
              - state (checkpoint / ledger)
              - branching
              - call order
              - retry / interrupt / resume
              -> LangChain-wrapped specialist workers
                 -> gpt-analyzer
                 -> gemini-investigator
                 -> deepseek-drafter
                 -> glm-coder
                 -> claude-reviewer
     -> OpenClaw edge posts progress/final output
```

See the [worker mapping](docs/2026-08-01-analyzer-bot-main-agent-design.md#worker-mapping) for how the seven roles below fold into these five workers, and [Head Role and Model Routing v0.1](docs/architecture/head-role-model-routing-v01.md) for their model routes and authority matrix.

## Framework Split

| Layer | Primary Framework | Responsibility | Must Not Do |
| --- | --- | --- | --- |
| Discord ingress | OpenClaw | Receive Discord messages, show typing/status, render approval controls, maintain active session context | Deep planning, shell execution, Git writes, OAuth loops, worker orchestration |
| Device bridge | OpenClaw | Keep iPhone node available for contacts, calendar, and native device actions | General research, coding, planning, autonomous writes without confirmation |
| Planning | `analyzer-bot` Head Orchestrator *(was a separate Team Lead Planner; superseded)* | Convert conversation into plans, reject weak plans, approve dispatch, review worker outputs | Direct code execution, direct API key use, direct Git push |
| Coding | Codex | Code changes, repo edits, TDD, verification, commits when authorized | Social scraping, broad research, iPhone mutation |
| Worker pool | Hermes Agent | Durable background workers, summaries, planner-review, critic/research support using NVIDIA-backed profiles | Second Discord frontend, competing lead planner |
| Research | Apify MCP + normal web search | Recover original sources, social/video evidence, GitHub/forum usage patterns | Expose all Actors, mutate repos, execute code |
| Review | Critic / Ponytail Review | Correctness review and over-engineering review | Implement feature code |
| Test | Tester | FAIL_TO_PASS, PASS_TO_PASS, mutation proof, log checks | Product decisions or scope changes |
| Operations | Operator | systemd, logs, OAuth runbook, restart/rollback notes | Planning authority |

## Agent Contracts

### OpenClaw Conversation Agent

Purpose: make Discord feel responsive and predictable.

Required behavior:

- Acknowledge messages quickly.
- Show typing or working status while a downstream agent is active.
- Keep a channel/session active so users do not need to mention the bot every time.
- Render plan controls as buttons when possible.
- Support text fallback commands:
  - `/plan approve`
  - `/plan reject`
  - `/plan revise`
  - `/status`
  - `/handoff`
- Produce a compact handoff envelope for `analyzer-bot`'s Head Orchestrator.

Forbidden behavior:

- No deep planning.
- No direct worker dispatch.
- No shell, Git, OAuth, or broad API execution.
- No iPhone data mutation without routing through the Device Agent contract.

### Team Lead Planner

> **Superseded:** this is not a separate agent. The required/forbidden behavior below is inherited verbatim by `analyzer-bot`'s Head Orchestrator — see [Analyzer-bot Main Agent Design](docs/2026-08-01-analyzer-bot-main-agent-design.md#what-the-head-never-does-directly). Kept here as the source list; only "separate agent" is wrong.

Purpose: act as a strict plan-mode agent.

Required behavior:

- Clarify requirements when the task is ambiguous.
- Draft a plan before execution.
- Check the plan for scope shrinkage, missing verification, and unclear ownership.
- Send the plan to the user for approval or rejection.
- Dispatch only after approval.
- Review returned worker results before the final answer.
- Reject worker output that lacks evidence.

Recommended skills:

- `brainstorming`
- `writing-plans`
- `dispatching-parallel-agents`
- `requesting-code-review`
- `receiving-code-review`
- `verification-before-completion`

Forbidden behavior:

- No direct code edits.
- No direct shell execution.
- No direct Git push.
- No OAuth repair loops.
- No device mutation.

### Codex Coding Agent

Purpose: execute code and repository work.

Required stack:

- Codex runtime
- `outta-mcp` runtime contract
- Ponytail for YAGNI and anti-overengineering pressure
- TDD for behavior changes
- Independent verification before completion

Required evidence:

- RED before GREEN for new tests.
- Mutation proof for meaningful test suites.
- PASS_TO_PASS regression checks.
- Exact command output for completion claims.
- Clean Git status before final claim.

Forbidden behavior:

- No broad research scraping.
- No plan negotiation with the user.
- No privacy-sensitive device actions.

### Hermes Worker Pool

Purpose: durable background collaboration without making Hermes the Discord frontend.

Recommended use:

- Planner-review worker
- Critic worker
- Research summarizer
- Long-running synthesis
- NVIDIA-backed model profiles

Boundary:

- Hermes may run as a backend worker framework.
- Hermes must not become a second Discord ingress unless OpenClaw is retired later.
- Hermes tasks should be created from `analyzer-bot`'s Head Orchestrator (LangGraph), not from casual Discord messages.

### Apify Researcher

Purpose: source recovery and social/video evidence collection.

Default flow:

```text
normal search first
  -> if original source is missing, social URL is detected, or evidence is weak
  -> Apify MCP
  -> allowlisted Actor only
  -> normalize output into an evidence object
```

Initial Actor allowlist:

- `apify/google-search-scraper`
- `apify/instagram-scraper`
- `streamers/youtube-scraper`

Expansion candidates:

- Reddit scraper
- TikTok scraper
- General Web Scraper

Rules:

- Do not expose every Actor to every agent.
- Do not let coding agents call Apify directly.
- Do not report "searched but no result" when the provider was unavailable.
- Separate `preview`, `blocked`, `failed`, `no_results`, and `executed`.

### Critic / Reviewer

Purpose: catch correctness risks and unnecessary complexity.

Required behavior:

- Lead with bugs, risks, regressions, and missing tests.
- Use file and line references when reviewing code.
- Run Ponytail review separately for over-engineering pressure.
- Recommend deletion or simplification when abstractions are speculative.

Forbidden behavior:

- No implementation unless explicitly promoted to coding worker.

### Tester

Purpose: verify user-visible behavior.

Required behavior:

- Write tests against the desired behavior, not current implementation.
- Show FAIL_TO_PASS for new or changed behavior.
- Show PASS_TO_PASS for existing regression coverage.
- Inject a deliberate mutant and confirm the suite fails.
- Prefer end-to-end entry points over isolated unit-only proof.

### Device Agent

Purpose: use OpenClaw iPhone node safely.

Required behavior:

- Read-only actions may summarize what will be accessed.
- Write actions require explicit confirmation.
- Calendar/contact writes should support dry-run previews.
- Never leak contact/calendar data into worker prompts unless required and approved.

## Discord UX Contract

Minimum v0.1 behavior:

- Acknowledge user intent within 2 seconds when possible.
- Show `typing` or equivalent activity while any worker is active.
- Post progress every 30 seconds during long tasks.
- Show current owner:
  - `Conversation`
  - `Planning`
  - `Waiting for approval`
  - `Researching`
  - `Coding`
  - `Reviewing`
  - `Testing`
  - `Blocked`
  - `Done`
- Never silently start a new OAuth flow while the user is waiting on another task.
- If a provider fails, report the provider failure as blocked/failed rather than asking the user to repeat the same OAuth step.

## Handoff Envelope

Every route from OpenClaw edge ingress to `analyzer-bot`'s Head Orchestrator should include:

```json
{
  "conversation_id": "discord-channel-or-thread-id",
  "request_id": "stable-idempotency-key",
  "user_intent": "short summary of requested outcome",
  "raw_user_message": "latest user message",
  "context_summary": "relevant prior context",
  "requested_mode": "plan_only | execute_after_approval | status | device_action",
  "constraints": ["must not execute before approval"],
  "known_tools": ["codex", "hermes", "apify", "openclaw-device"],
  "approval_required": true
}
```

## Dispatch States

| State | Meaning |
| --- | --- |
| `intent` | User request received, not yet planned |
| `planning` | `analyzer-bot`'s Head Orchestrator is drafting or revising a plan |
| `waiting_approval` | User must approve, reject, or request revision |
| `approved` | Plan accepted and ready to dispatch |
| `dispatched` | Worker task has been assigned |
| `worker_running` | Worker is active |
| `worker_blocked` | Worker cannot proceed without input or external state |
| `reviewing` | Head Orchestrator is checking worker output |
| `needs_revision` | Output failed review or user rejected plan |
| `complete` | Final answer posted with evidence |

## Implementation Plan

### Phase 0: Inventory

Outcome: know what is currently running and where the boundaries are.

- List active services:
  - OpenClaw gateway
  - Hermes gateway
  - Discord bot bridge
  - outta-mcp
- Capture existing environment variables without printing secrets.
- Identify where Discord events enter OpenClaw.
- Identify where iPhone node requests are routed.
- Identify whether Hermes has a usable worker profile with NVIDIA credentials.

Exit criteria:

- A service inventory document exists.
- Secrets are referenced by name only.
- A rollback note exists for every service touched.

### Phase 1: Discord UX Stabilization

Outcome: OpenClaw feels responsive even before worker orchestration is rebuilt.

- Add fast acknowledgement for inbound messages.
- Add owner/status display.
- Add 30-second progress heartbeat for long tasks.
- Add button controls for plan approval/rejection/revision.
- Add text fallback commands for the same controls.
- Add idempotency key per request.

Exit criteria:

- A user can see who owns the task.
- A user can approve or reject without mentioning the bot repeatedly.
- A stalled provider becomes a clear blocked/failed state.

### Phase 2: analyzer-bot Plan-Only Mode

> **Superseded:** was "Team Lead Plan-Only Mode", building a separate Team Lead Planner profile. Planning now lives inside `analyzer-bot`'s Head Orchestrator; steps below are updated to match.

Outcome: `analyzer-bot`'s Head Orchestrator cannot accidentally execute.

- Create the `analyzer-bot` OpenClaw agent profile.
- Remove shell/Git/OAuth/device tools from the `analyzer-bot` profile — enforced by the OpenClaw tool profile and PolicyGate, not by prompt wording.
- Allow only planning, review, dispatch, and status tools.
- Implement `waiting_approval` as a LangGraph interrupt; approval resumes the graph, rejection routes to `needs_revision`.
- Add rejection path back to plan revision.

Exit criteria:

- `analyzer-bot` can produce and revise plans.
- `analyzer-bot`'s profile cannot run shell commands.
- No worker dispatch happens before approval.

### Phase 3: Worker Mapping

Outcome: each worker has a narrow role and tool set.

- Attach `outta-mcp` and Ponytail to Codex Coding Agent.
- Attach Apify MCP only to Researcher with an Actor allowlist.
- Attach Ponytail Review to Critic.
- Attach test verification responsibilities to Tester.
- Attach iPhone node only to Device Agent.
- Attach systemd/log/OAuth runbook tools only to Operator.

Exit criteria:

- Tool exposure is role-specific.
- No single worker has all tools.
- Research, coding, testing, device, and operations paths are separable.

### Phase 4: Hermes Backend Worker Pool

Outcome: Hermes can be used without competing with OpenClaw for Discord ingress.

- Start Hermes only as a backend worker pool.
- Connect NVIDIA API key to Hermes worker profile without logging the key.
- Add dispatch route from `analyzer-bot`'s Head Orchestrator to Hermes worker tasks.
- Use Hermes for background research, critique, synthesis, and planner-review.

Exit criteria:

- Hermes can receive a task from the LangGraph graph (no separate Dispatch Broker).
- Hermes does not listen directly to Discord in v0.1.
- Results return to the Head Orchestrator for review before Discord output.

### Phase 5: Verification and Runbook

Outcome: the system is operable and debuggable.

- Add smoke tests for:
  - Discord message to OpenClaw acknowledgement
  - Plan draft to approval wait state
  - Approval to worker dispatch
  - Worker blocked report
  - Worker result to Head Orchestrator review
  - Final answer to Discord
- Add manual runbook:
  - restart OpenClaw
  - restart Hermes worker pool
  - inspect active tasks
  - cancel active task
  - recover from OAuth/provider failure

Exit criteria:

- A dry-run request can pass through the whole route without mutating external state.
- Failures are reported as blocked/failed with owner and next action.
- A human can restart or inspect the system without guessing.

## First Build Scope

v0.1 should build only these:

- OpenClaw Discord UX status layer.
- `analyzer-bot` plan-only profile (Head Orchestrator).
- Approval/rejection controls.
- Dispatch envelope schema.
- Static worker role/tool matrix.
- Hermes worker-pool connection as backend only.

v0.1 should not build:

- Full framework migration.
- Public multi-user SaaS behavior.
- Autonomous device writes.
- All Apify Actors.
- A second Discord frontend.
- A general-purpose agent marketplace.

## Active Plans

- [OpenClaw Runtime Skill Evolution Implementation Plan](docs/superpowers/plans/2026-07-10-openclaw-runtime-skill-evolution.md)

## Risks

| Risk | Mitigation |
| --- | --- |
| Two frameworks compete for the same Discord channel | Keep OpenClaw as the only ingress in v0.1 |
| `analyzer-bot`'s Head Orchestrator executes instead of plans | Tool exposure gating at the OpenClaw profile and PolicyGate, not prompt wording |
| Tool selection explodes | Role-specific tool allowlists |
| OAuth loops waste user time | Provider failures become blocked/failed state, not repeated prompts |
| Research creates unverifiable claims | Normalize evidence and preserve provider execution status |
| Workers self-report success falsely | Head Orchestrator requires verification evidence before final |
| Device data leaks into prompts | Device Agent requires explicit confirmation and minimal context sharing |

## References

- OpenClaw docs: https://docs.openclaw.ai/
- OpenClaw GitHub: https://github.com/openclaw/openclaw
- OpenClaw Discord channel docs: https://docs.openclaw.ai/channels/discord
- Hermes Agent GitHub: https://github.com/nousresearch/hermes-agent
- Hermes Kanban docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
- Apify MCP docs: https://docs.apify.com/platform/integrations/mcp
- Apify MCP GitHub: https://github.com/apify/apify-mcp-server
- Ponytail GitHub: https://github.com/DietrichGebert/ponytail
- Ponytail review skill: https://github.com/DietrichGebert/ponytail/blob/main/skills/ponytail-review/SKILL.md
