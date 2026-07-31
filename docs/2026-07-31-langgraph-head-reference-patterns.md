# LangGraph Head Reference Patterns

Date: 2026-07-31
Status: reference note
Decision: absorb only the smallest useful patterns into the mixed framework plan

## One-Line Takeaway

Keep LangGraph Head as the only conversation and judgment owner, keep OpenClaw as the edge adapter, and borrow runtime-control patterns from other systems only when they strengthen ledger replay, policy gates, worker contracts, or observability.

## Reference List

### High Priority

- LangGraph
  - Use for Head state machine, interrupt, resume, and checkpoint flow.
  - Do not add LangChain Supervisor, AgentExecutor, or a second planner.

- LangSmith LLM Gateway
  - Useful pattern: runtime controls for production agents, including rate limits, fallback, cost control, sensitive-data handling, and route-level observability.
  - Absorb as: `RouteRegistry`, `PolicyGate`, provider route health, and audit events.
  - Reference: <https://www.langchain.com/blog/langsmith-llm-gateway-runtime-controls-for-production-agents>

- Anthropic cybersecurity eval incident
  - Useful pattern: agent sandbox, network isolation, external side-effect boundaries, and eval environment separation.
  - Absorb as: deterministic policy gate, approval scope, route ownership checks, and no external execution without explicit permission.
  - Reference: <https://www.anthropic.com/news/investigating-incidents-cybersecurity-evals>

- OpenAI Agents SDK
  - Useful pattern: handoff boundaries, guardrails, human-in-the-loop, and tracing.
  - Absorb as: typed `WorkerCommand` / `WorkerResult`, approval interrupt/resume, and traceable worker handoff.
  - Do not absorb as: a competing orchestration runtime.

- promptfoo
  - Useful pattern: eval matrix, red-team checks, regression gates, and CI-friendly verification.
  - Absorb as: `make test-v01-replay`, `test-outbox-dedupe`, `test-approval-resume`, route reconnect tests, and worker timeout/cancel tests.
  - Reference: <https://github.com/promptfoo/promptfoo>

### Medium Priority

- 12-factor-agents
  - Useful pattern: small focused agents, explicit context ownership, stateless reducers, and durable external state.
  - Absorb as: Head owns context; workers receive bounded contracts; ledger is source of truth.

- Hatchet and durable workflow systems
  - Useful pattern: retry policy, leases, priority, rate limits, idempotency, and durable state.
  - Absorb as: `WorkerLease`, heartbeat, timeout, cancel, and idempotency keys.
  - Do not absorb as: Kafka/RabbitMQ or a second workflow platform in v0.1.

- Opik / RagaAI Catalyst style observability
  - Useful pattern: trace/span feedback, eval evidence, and operator-visible replay.
  - Absorb as: ledger event inspection, delivery receipts, worker result validation, and audit reports.

- codebase-memory-mcp
  - Useful pattern: repository memory and codebase retrieval for coding workers.
  - Absorb later as: optional worker-side retrieval capability.
  - Do not put it in Head canonical memory for v0.1.
  - Reference: <https://github.com/DeusData/codebase-memory-mcp>

### Watchlist

- OpenAI Codex Security
  - Useful pattern: security-focused code agent that identifies, verifies, and patches vulnerabilities.
  - Absorb later as: conditional reviewer/security worker for auth, policy, sandbox, and dependency changes.
  - Reference: <https://openai.com/index/codex-security-now-in-research-preview/>

- Strix
  - Useful pattern: agentic penetration-testing flow and vulnerability validation.
  - Absorb later as: security eval inspiration, not as a default worker.
  - Reference: <https://github.com/usestrix/strix>

- OfficeCLI
  - Useful pattern: file-native office/document automation through an agent-accessible CLI.
  - Absorb later as: document worker route behind PolicyGate.
  - Reference: <https://github.com/iOfficeAI/OfficeCLI>

- Gemini Robotics ER 2 and edge VLM work
  - Useful pattern: multimodal model expansion into physical environment control.
  - Absorb later as: device/action safety requirements, not v0.1 functionality.
  - Reference: <https://blog.google/innovation-and-ai/models-and-research/google-deepmind/gemini-robotics-er-2/>

## How This Changes The Mixed Framework Plan

### Keep

- OpenClaw as Discord, session, device, route, and delivery edge.
- LangGraph Head as the only conversation, planning, judgment, and state owner.
- Append-only ledger as source of truth.
- Transactional outbox for delivery.
- Typed contracts for inbound events, Head decisions, worker commands, worker results, and delivery receipts.
- PolicyGate for external side effects.
- Route health tracked by provider, model, and route.

### Add To v0.1 Acceptance Criteria

- Head restart replays ledger into the same canonical state.
- Duplicate inbound event, worker command, worker result, and delivery receipt are harmless.
- Approval token has explicit scope, expiry, one-time use, cancel semantics, and audit trail.
- Worker heartbeat timeout cancels lease and rejects late results.
- OpenClaw reconnect recovers route ownership and pending delivery state.
- Worker result schema/version validation blocks malformed output.
- Operator can inspect replay/audit evidence without reading raw private context.

### Keep Out Of v0.1

- LangChain Supervisor.
- Team Lead Planner as a separate agent.
- LLM Dispatch Broker as an agent.
- Kafka or RabbitMQ.
- Vector DB.
- Always-on reviewer/tester chain.
- Independent research service.
- Dynamic agent creation.
- Claude or Copilot recovery work.

## Provider And Route Notes

- GPT-5.5 is the current Head / primary reasoning candidate.
- GLM-5.2 is the current v0.1 worker candidate.
- Gemini should be treated as a CLI route capability, not a general provider API route, until direct route health is proven.
- Nemotron is configured but not healthy until smoke-tested.
- DeepSeek remains candidate or quarantined depending on latest route evidence.
- MiniMax, Copilot, and Claude should not be active v0.1 workers without fresh successful smoke evidence.

## Practical v0.1 Test Set

```bash
make test-v01-replay
make test-outbox-dedupe
make test-approval-resume
make test-route-reconnect
make test-worker-timeout-cancel
```

Minimum equivalent proof:

```bash
uv run pytest -q
```

is acceptable only if the test suite explicitly covers replay, dedupe, approval resume/cancel, route reconnect, worker timeout/cancel, and malformed result rejection.

## Final Design Sentence

Head is not a model name. Head is the boundary that owns conversation, state, policy interpretation, ledger replay, and contract validation. OpenClaw, GLM, Codex, Gemini, Hermes, and device executors are replaceable adapters behind that boundary.
