# OpenClaw Runtime Skill Evolution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make OpenClaw itself learn from repeated agent work by creating, revising, curating, and applying skill proposals from runtime traces.

**Architecture:** OpenClaw already has the right foundation: `agent_end` side effects, `runSkillResearchAutoCapture`, Skill Workshop proposals, proposal origin tracking, and a curator. This plan turns that foundation into a general runtime skill-evolution loop: capture durable run signals, generate pending Skill Workshop proposals, let humans approve workspace/global promotion, and keep agent-created skills curated.

**Tech Stack:** OpenClaw TypeScript runtime, Vitest, Skill Workshop, OpenClaw state DB, `agent_end` side effects, `openclaw config`, `openclaw doctor`.

## Status (2026-07-10, post-implementation)

Upstream `main` already contained most of Tasks 2–6 (signal extraction, dedup,
caps, revision, suppression, curator, workshop CLI), so implementation landed as
a delta on upstream instead of the module layout planned below:

- **Done, in PR https://github.com/openclaw/openclaw/pull/103817** (branch
  `codex/skill-evolution-loop` on fork `yham5016-source/openclaw-upstream`):
  - `skills.workshop.autonomous.agents.allow/deny` per-agent policy
  - One-shot pending-proposal review notice on the next interactive turn
  - Doctor guardrails: unknown `allow` entry warning, all-agents-excluded warning
    (adapted Task 7; the planned `agentAllowlist` key shipped as `agents.allow/deny`)
  - End-to-end loop test: `agent_end` capture → pending proposal → apply →
    next-run skill loading (`src/skills/research/skill-evolution-loop.test.ts`)
  - Fix: `pendingSkillProposalNotice` registered in reserved session slot keys
    (caught by `tsgo:core`, which the first commit had not run)
- **Done — Task 10 (Controlled Local Activation), 2026-07-10:** the 2026.6.10
  pin was lifted. #98416 (reply-session reentrancy dist bug) is verified fixed
  in `2026.7.1-beta.2`, which also ships the autocapture/workshop skeleton, so
  the gateway was upgraded to that version (config backed up, `doctor --fix`
  migrations applied, unit metadata refreshed). `skills.workshop.autonomous.enabled=true`
  is live. Full loop proven end to end on the running gateway: a durable
  correction in a real `openclaw agent` run queued pending proposal
  `github-pr-workflow`, `openclaw skills workshop apply` promoted it, and the
  next run loaded the skill and quoted its content.
  - **Operational gotcha:** enabling `skills.workshop.autonomous.enabled` via
    `openclaw config set` claims "No gateway restart needed", but agent runs
    kept the old value until the gateway was restarted. Restart after toggling.
  - Per-agent `agents.allow/deny` scoping and proposal review notices are not
    in beta.2; they arrive when PR #103817 is released.

---

## Current Finding

The feature is not a blank rewrite. OpenClaw source already contains these relevant pieces:

- `src/agents/harness/agent-end-side-effects.ts`
- `src/skills/research/autocapture.ts`
- `src/skills/research/signals.ts`
- `src/skills/workshop/service.ts`
- `src/skills/workshop/config.ts`
- `src/skills/workshop/curator.ts`
- `src/skills/workshop/curator.test.ts`
- `src/skills/research/autocapture.test.ts`
- `src/flows/doctor-core-checks.ts`
- `packages/gateway-protocol/src/schema/agents-models-skills.ts`

The installed local config currently has `tools.profile` set to `coding`, which includes `skill_workshop`. `skills.workshop.autonomous.enabled` is unset, so the default is `false`.

## Target Behavior

```text
OpenClaw agent run completes
  -> agent_end side effect receives final messages and runtime context
  -> skill evolution extracts durable signals:
       - "from now on"
       - "next time"
       - "remember to"
       - user corrections after failures
       - repeated task patterns
  -> Skill Workshop creates or revises a pending proposal
  -> proposal stays pending/quarantined until approved
  -> approved skill becomes workspace skill
  -> curator records usage and archives stale agent-created skills
```

## Safety Policy

- Never write `SKILL.md` directly from the learning loop.
- Always use Skill Workshop proposal services.
- Default apply policy remains `pending`.
- Global/shared skill promotion requires explicit operator approval.
- Core, bundled, ClawHub, managed, and protected skills are not auto-mutated.
- Subagent, cron, heartbeat, memory, and overflow sessions remain excluded unless explicitly enabled later.
- All captures must preserve `origin` with `agentId`, `sessionKey`, and `runId` when available.

## File Structure

Implementation target is the OpenClaw source repository, not this planning repository.

- Modify: `src/skills/research/autocapture.ts`
  - Rename concepts from "research autocapture" to general "runtime skill evolution" without breaking imports immediately.
  - Extend extraction inputs beyond explicit durable instructions to repeated runtime patterns.
- Modify: `src/skills/research/signals.ts`
  - Add pattern classification for repeated workflows, failed verification, and user rejection.
- Create: `src/skills/evolution/types.ts`
  - Shared types for signal, policy, proposal decision, and trace summary.
- Create: `src/skills/evolution/trace.ts`
  - Convert raw agent messages into bounded, redacted trace summaries.
- Create: `src/skills/evolution/policy.ts`
  - Decide whether a trace can create, revise, suggest, ignore, or quarantine a skill proposal.
- Create: `src/skills/evolution/proposal.ts`
  - Build Skill Workshop proposal content and evidence strings.
- Create: `src/skills/evolution/index.ts`
  - Public runtime entrypoint called by `agent-end-side-effects.ts`.
- Modify: `src/agents/harness/agent-end-side-effects.ts`
  - Call the new evolution entrypoint, while keeping old autocapture compatibility during migration.
- Modify: `src/skills/workshop/config.ts`
  - Add narrow controls for capture thresholds, max proposals per run, and agent allowlist.
- Modify: `src/flows/doctor-core-checks.ts`
  - Warn when runtime evolution is enabled but `skill_workshop` is unavailable.
- Modify: `docs/tools/skill-workshop.md`
  - Document runtime skill evolution as a native OpenClaw loop.
- Modify: `docs/tools/skills-config.md`
  - Document new config keys.
- Test: `src/skills/evolution/*.test.ts`
- Test: `src/skills/research/autocapture.test.ts`
- Test: `src/agents/harness/agent-end-side-effects.test.ts`
- Test: `src/flows/doctor-core-checks.test.ts`

## Task 1: Baseline Verification

**Files:**
- Read: `src/skills/research/autocapture.ts`
- Read: `src/skills/research/autocapture.test.ts`
- Read: `src/skills/workshop/curator.ts`
- Read: `src/agents/harness/agent-end-side-effects.ts`

- [ ] **Step 1: Confirm source revision**

Run:

```bash
git -C /path/to/openclaw rev-parse --short HEAD
git -C /path/to/openclaw status -sb
```

Expected:

```text
<commit>
## <branch>...origin/<branch>
```

- [ ] **Step 2: Run existing targeted tests**

Run:

```bash
pnpm test \
  src/skills/research/autocapture.test.ts \
  src/skills/workshop/curator.test.ts \
  src/agents/harness/agent-end-side-effects.test.ts \
  src/flows/doctor-core-checks.test.ts
```

Expected:

```text
PASS src/skills/research/autocapture.test.ts
PASS src/skills/workshop/curator.test.ts
PASS src/agents/harness/agent-end-side-effects.test.ts
PASS src/flows/doctor-core-checks.test.ts
```

- [ ] **Step 3: Record baseline behavior**

Run:

```bash
openclaw config get tools.profile
openclaw config get skills.workshop.autonomous.enabled || true
openclaw doctor --lint --only core/doctor/skill-workshop-tool-policy
```

Expected:

```text
coding
```

The second command may print nothing when the value is unset. That means default `false`.

## Task 2: Create Evolution Trace Module

**Files:**
- Create: `src/skills/evolution/types.ts`
- Create: `src/skills/evolution/trace.ts`
- Test: `src/skills/evolution/trace.test.ts`

- [ ] **Step 1: Write failing trace tests**

Create `src/skills/evolution/trace.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { buildSkillEvolutionTrace } from "./trace.js";

describe("buildSkillEvolutionTrace", () => {
  it("extracts bounded user-visible trace without raw secrets", () => {
    const trace = buildSkillEvolutionTrace({
      messages: [
        { role: "user", content: "From now on, always verify CI before final PR replies." },
        { role: "assistant", content: "I will check CI." },
        { role: "toolResult", content: "token=abc123 SECRET_KEY=hidden" },
      ],
      ctx: {
        agentId: "analyzer",
        runId: "run-1",
        sessionKey: "agent:analyzer:discord:chan-1",
        workspaceDir: "/tmp/workspace",
      },
    });

    expect(trace.agentId).toBe("analyzer");
    expect(trace.sessionKey).toBe("agent:analyzer:discord:chan-1");
    expect(trace.turnText).toContain("always verify CI before final PR replies");
    expect(trace.turnText).not.toContain("abc123");
    expect(trace.turnText).not.toContain("SECRET_KEY");
    expect(trace.messagesExamined).toBe(3);
  });

  it("keeps trace size bounded", () => {
    const trace = buildSkillEvolutionTrace({
      messages: [{ role: "user", content: "Next time, always verify screenshots. ".repeat(200) }],
      ctx: { sessionKey: "agent:main:discord:chan-1" },
      maxChars: 240,
    });

    expect(trace.turnText.length).toBeLessThanOrEqual(240);
    expect(trace.truncated).toBe(true);
    expect(trace.turnText).toContain("Next time");
  });
});
```

- [ ] **Step 2: Run test to verify RED**

Run:

```bash
pnpm test src/skills/evolution/trace.test.ts
```

Expected:

```text
FAIL src/skills/evolution/trace.test.ts
Error: Failed to resolve import "./trace.js"
```

- [ ] **Step 3: Implement trace types**

Create `src/skills/evolution/types.ts`:

```ts
export type SkillEvolutionContext = {
  agentId?: string;
  runId?: string;
  sessionKey?: string;
  trigger?: string;
  workspaceDir?: string;
};

export type SkillEvolutionTrace = SkillEvolutionContext & {
  messagesExamined: number;
  turnText: string;
  truncated: boolean;
};
```

- [ ] **Step 4: Implement trace builder**

Create `src/skills/evolution/trace.ts`:

```ts
import type { SkillEvolutionContext, SkillEvolutionTrace } from "./types.js";

const SECRET_PATTERNS = [
  /\b(token|api[_-]?key|secret|password)\s*=\s*[^ \n\r\t]+/gi,
  /\bsk-[A-Za-z0-9_-]{16,}\b/g,
];

function blockToText(value: unknown): string {
  if (typeof value === "string") return value;
  if (Array.isArray(value)) return value.map(blockToText).filter(Boolean).join("\n");
  if (!value || typeof value !== "object") return "";
  const record = value as { text?: unknown; content?: unknown };
  if (typeof record.text === "string") return record.text;
  return blockToText(record.content);
}

function redact(text: string): string {
  return SECRET_PATTERNS.reduce(
    (current, pattern) => current.replace(pattern, "$1=[REDACTED]"),
    text,
  );
}

export function buildSkillEvolutionTrace(params: {
  messages: readonly unknown[];
  ctx: SkillEvolutionContext;
  maxChars?: number;
}): SkillEvolutionTrace {
  const maxChars = params.maxChars ?? 4000;
  const text = params.messages
    .map((message) => {
      if (!message || typeof message !== "object" || Array.isArray(message)) return "";
      const record = message as { role?: unknown; content?: unknown };
      const role = typeof record.role === "string" ? record.role : "unknown";
      return `${role}: ${blockToText(record.content)}`;
    })
    .filter(Boolean)
    .join("\n");
  const redacted = redact(text);
  const truncated = redacted.length > maxChars;
  return {
    ...params.ctx,
    messagesExamined: params.messages.length,
    turnText: truncated ? redacted.slice(0, maxChars) : redacted,
    truncated,
  };
}
```

- [ ] **Step 5: Run test to verify GREEN**

Run:

```bash
pnpm test src/skills/evolution/trace.test.ts
```

Expected:

```text
PASS src/skills/evolution/trace.test.ts
```

## Task 3: Add Evolution Policy

**Files:**
- Create: `src/skills/evolution/policy.ts`
- Test: `src/skills/evolution/policy.test.ts`

- [ ] **Step 1: Write failing policy tests**

Create `src/skills/evolution/policy.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { decideSkillEvolutionAction } from "./policy.js";

describe("decideSkillEvolutionAction", () => {
  it("captures explicit future workflow instructions", () => {
    const action = decideSkillEvolutionAction({
      turnText: "user: From now on, always check CI before PR final replies.",
      messagesExamined: 1,
      truncated: false,
      sessionKey: "agent:main:discord:chan-1",
    });

    expect(action.kind).toBe("capture");
    expect(action.reason).toContain("durable instruction");
  });

  it("blocks cron and memory sessions", () => {
    const action = decideSkillEvolutionAction({
      turnText: "user: From now on, always check CI before PR final replies.",
      messagesExamined: 1,
      truncated: false,
      sessionKey: "agent:main:cron:daily",
      trigger: "cron",
    });

    expect(action.kind).toBe("ignore");
    expect(action.reason).toContain("blocked trigger");
  });
});
```

- [ ] **Step 2: Run test to verify RED**

Run:

```bash
pnpm test src/skills/evolution/policy.test.ts
```

Expected:

```text
FAIL src/skills/evolution/policy.test.ts
Error: Failed to resolve import "./policy.js"
```

- [ ] **Step 3: Implement policy**

Create `src/skills/evolution/policy.ts`:

```ts
import type { SkillEvolutionTrace } from "./types.js";

export type SkillEvolutionAction =
  | { kind: "capture"; reason: string }
  | { kind: "suggest"; reason: string }
  | { kind: "ignore"; reason: string };

const BLOCKED_TRIGGERS = new Set(["cron", "heartbeat", "memory", "overflow"]);
const BLOCKED_SESSION_SEGMENTS = new Set(["cron", "hook", "subagent"]);
const DURABLE_PATTERNS = [
  /\bnext time\b/i,
  /\bfrom now on\b/i,
  /\bgoing forward\b/i,
  /\bremember to\b/i,
  /\bmake sure to\b/i,
  /\balways\b.{0,80}\b(use|check|verify|record|save|prefer)\b/i,
  /\bthat(?:'s| is| was)? wrong\b/i,
  /\bnot what i (asked|meant|wanted)\b/i,
];

export function decideSkillEvolutionAction(trace: SkillEvolutionTrace): SkillEvolutionAction {
  const trigger = trace.trigger?.trim().toLowerCase();
  if (trigger && BLOCKED_TRIGGERS.has(trigger)) {
    return { kind: "ignore", reason: `blocked trigger: ${trigger}` };
  }
  const sessionKey = trace.sessionKey?.trim().toLowerCase();
  if (
    sessionKey?.split(":").some((segment) => BLOCKED_SESSION_SEGMENTS.has(segment)) ||
    sessionKey?.includes("active-memory")
  ) {
    return { kind: "ignore", reason: "blocked session class" };
  }
  if (DURABLE_PATTERNS.some((pattern) => pattern.test(trace.turnText))) {
    return { kind: "capture", reason: "durable instruction detected" };
  }
  return { kind: "ignore", reason: "no durable signal" };
}
```

- [ ] **Step 4: Run test to verify GREEN**

Run:

```bash
pnpm test src/skills/evolution/policy.test.ts
```

Expected:

```text
PASS src/skills/evolution/policy.test.ts
```

## Task 4: Wrap Existing Autocapture Behind Evolution Entry Point

**Files:**
- Create: `src/skills/evolution/index.ts`
- Modify: `src/agents/harness/agent-end-side-effects.ts`
- Test: `src/agents/harness/agent-end-side-effects.test.ts`

- [ ] **Step 1: Write failing agent-end test**

Add to `src/agents/harness/agent-end-side-effects.test.ts`:

```ts
it("runs runtime skill evolution from agent_end side effects", async () => {
  const runRuntimeSkillEvolution = vi.fn().mockResolvedValue(undefined);
  vi.doMock("../../skills/evolution/index.js", () => ({ runRuntimeSkillEvolution }));

  const { awaitAgentEndSideEffects } = await import("./agent-end-side-effects.js");
  await awaitAgentEndSideEffects({
    event: {
      type: "agent_end",
      messages: [{ role: "user", content: "From now on, always check CI." }],
    },
    ctx: {
      agentId: "main",
      sessionKey: "agent:main:discord:chan-1",
      workspaceDir: "/tmp/workspace",
      config: { skills: { workshop: { autonomous: { enabled: true } } } },
    },
  } as never);

  expect(runRuntimeSkillEvolution).toHaveBeenCalledWith(
    expect.objectContaining({
      event: expect.objectContaining({ messages: expect.any(Array) }),
      ctx: expect.objectContaining({ agentId: "main" }),
    }),
  );
});
```

- [ ] **Step 2: Run test to verify RED**

Run:

```bash
pnpm test src/agents/harness/agent-end-side-effects.test.ts
```

Expected:

```text
FAIL src/agents/harness/agent-end-side-effects.test.ts
```

The failure should show `runRuntimeSkillEvolution` was not called or import missing.

- [ ] **Step 3: Implement evolution entry point**

Create `src/skills/evolution/index.ts`:

```ts
import { runSkillResearchAutoCapture } from "../research/autocapture.js";

type RuntimeSkillEvolutionParams = Parameters<typeof runSkillResearchAutoCapture>[0];

export async function runRuntimeSkillEvolution(
  params: RuntimeSkillEvolutionParams,
): Promise<void> {
  await runSkillResearchAutoCapture(params);
}
```

- [ ] **Step 4: Update agent-end side effects**

Modify `src/agents/harness/agent-end-side-effects.ts`:

```ts
async function runCoreAgentEndSideEffects(params: AgentEndSideEffectsParams): Promise<void> {
  try {
    const { runRuntimeSkillEvolution } = await import("../../skills/evolution/index.js");
    await runRuntimeSkillEvolution({
      event: params.event,
      ctx: params.ctx,
      ...(params.ctx.config ? { config: params.ctx.config } : {}),
    });
  } catch (error) {
    log.warn(`runtime skill evolution failed: ${String(error)}`);
  }
}
```

- [ ] **Step 5: Run test to verify GREEN**

Run:

```bash
pnpm test src/agents/harness/agent-end-side-effects.test.ts
```

Expected:

```text
PASS src/agents/harness/agent-end-side-effects.test.ts
```

## Task 5: Extend Config Without Changing Safe Defaults

**Files:**
- Modify: `src/skills/workshop/config.ts`
- Test: `src/skills/workshop/config.test.ts`
- Modify: `docs/tools/skills-config.md`

- [ ] **Step 1: Write failing config tests**

Create `src/skills/workshop/config.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { resolveSkillWorkshopConfig } from "./config.js";

describe("resolveSkillWorkshopConfig", () => {
  it("defaults runtime evolution to conservative limits", () => {
    expect(resolveSkillWorkshopConfig()).toMatchObject({
      autonomous: {
        enabled: false,
        maxProposalsPerRun: 3,
        minSignalChars: 24,
      },
      approvalPolicy: "pending",
    });
  });

  it("clamps runtime evolution limits", () => {
    expect(
      resolveSkillWorkshopConfig({
        skills: {
          workshop: {
            autonomous: {
              enabled: true,
              maxProposalsPerRun: 999,
              minSignalChars: 1,
            },
          },
        },
      } as never),
    ).toMatchObject({
      autonomous: {
        enabled: true,
        maxProposalsPerRun: 10,
        minSignalChars: 12,
      },
    });
  });
});
```

- [ ] **Step 2: Run test to verify RED**

Run:

```bash
pnpm test src/skills/workshop/config.test.ts
```

Expected:

```text
FAIL src/skills/workshop/config.test.ts
```

- [ ] **Step 3: Add config fields**

Modify `src/skills/workshop/config.ts`:

```ts
export type SkillWorkshopConfig = {
  autonomous: {
    enabled: boolean;
    maxProposalsPerRun: number;
    minSignalChars: number;
    agentAllowlist: string[];
  };
  allowSymlinkTargetWrites: boolean;
  approvalPolicy: "pending" | "auto";
  maxPending: number;
  maxSkillBytes: number;
};
```

Set defaults:

```ts
const DEFAULT_CONFIG: SkillWorkshopConfig = {
  autonomous: {
    enabled: false,
    maxProposalsPerRun: 3,
    minSignalChars: 24,
    agentAllowlist: [],
  },
  allowSymlinkTargetWrites: false,
  approvalPolicy: "pending",
  maxPending: 50,
  maxSkillBytes: 40_000,
};
```

Read config:

```ts
function readStringArray(value: unknown): string[] {
  return Array.isArray(value)
    ? value.filter((item): item is string => typeof item === "string" && item.trim() !== "")
    : [];
}

// inside resolveSkillWorkshopConfig()
autonomous: {
  enabled: readBoolean(autonomous.enabled, DEFAULT_CONFIG.autonomous.enabled),
  maxProposalsPerRun: readInteger(
    autonomous.maxProposalsPerRun,
    DEFAULT_CONFIG.autonomous.maxProposalsPerRun,
    1,
    10,
  ),
  minSignalChars: readInteger(
    autonomous.minSignalChars,
    DEFAULT_CONFIG.autonomous.minSignalChars,
    12,
    200,
  ),
  agentAllowlist: readStringArray(autonomous.agentAllowlist),
},
```

- [ ] **Step 4: Document new keys**

Add to `docs/tools/skills-config.md` under `skills.workshop.autonomous`:

```md
<ParamField path="skills.workshop.autonomous.maxProposalsPerRun" type="number" default="3">
  Maximum pending proposals a single completed agent run can create or revise.
</ParamField>

<ParamField path="skills.workshop.autonomous.minSignalChars" type="number" default="24">
  Minimum normalized user instruction length before runtime skill evolution considers a signal durable.
</ParamField>

<ParamField path="skills.workshop.autonomous.agentAllowlist" type="string[]" default="[]">
  Optional list of agent ids allowed to create autonomous skill proposals. Empty means every eligible foreground agent can propose.
</ParamField>
```

- [ ] **Step 5: Run config test to verify GREEN**

Run:

```bash
pnpm test src/skills/workshop/config.test.ts
```

Expected:

```text
PASS src/skills/workshop/config.test.ts
```

## Task 6: Apply Policy to Autocapture

**Files:**
- Modify: `src/skills/research/autocapture.ts`
- Test: `src/skills/research/autocapture.test.ts`

- [ ] **Step 1: Write failing tests for max proposals and allowlist**

Add to `src/skills/research/autocapture.test.ts`:

```ts
it("respects autonomous maxProposalsPerRun", async () => {
  const workspaceDir = await makeWorkspace();
  await runSkillResearchAutoCapture({
    event: {
      success: true,
      messages: [
        { role: "user", content: "Next time, always verify screenshots before replying." },
        { role: "user", content: "Next time, always verify animated GIF output before replying." },
        { role: "user", content: "Next time, always check GitHub CI before final response." },
      ],
    },
    ctx: { workspaceDir, agentId: "main", sessionKey: SESSION_KEY },
    config: {
      skills: {
        workshop: {
          autonomous: { enabled: true, maxProposalsPerRun: 1 },
        },
      },
    },
  });

  const proposals = await listSkillProposals({ workspaceDir });
  expect(proposals.proposals).toHaveLength(1);
});

it("ignores agents outside autonomous agentAllowlist", async () => {
  const workspaceDir = await makeWorkspace();
  await runSkillResearchAutoCapture({
    event: {
      success: true,
      messages: [{ role: "user", content: "From now on, always check CI." }],
    },
    ctx: { workspaceDir, agentId: "coder", sessionKey: SESSION_KEY },
    config: {
      skills: {
        workshop: {
          autonomous: { enabled: true, agentAllowlist: ["analyzer"] },
        },
      },
    },
  });

  expect((await listSkillProposals({ workspaceDir })).proposals).toHaveLength(0);
});
```

- [ ] **Step 2: Run test to verify RED**

Run:

```bash
pnpm test src/skills/research/autocapture.test.ts
```

Expected:

```text
FAIL src/skills/research/autocapture.test.ts
```

- [ ] **Step 3: Enforce config in autocapture**

Modify `src/skills/research/autocapture.ts` after resolving `workshopConfig`:

```ts
const allowlist = new Set(workshopConfig.autonomous.agentAllowlist);
if (allowlist.size > 0 && (!params.ctx.agentId || !allowlist.has(params.ctx.agentId))) {
  return;
}
```

Limit selected proposals:

```ts
const runnableProposals = proposals.slice(0, workshopConfig.autonomous.maxProposalsPerRun);
for (const proposal of runnableProposals) {
  // existing proposal loop body
}
```

- [ ] **Step 4: Run test to verify GREEN**

Run:

```bash
pnpm test src/skills/research/autocapture.test.ts
```

Expected:

```text
PASS src/skills/research/autocapture.test.ts
```

## Task 7: Doctor and Operational Guardrails

**Files:**
- Modify: `src/flows/doctor-core-checks.ts`
- Test: `src/flows/doctor-core-checks.test.ts`
- Modify: `docs/tools/skill-workshop.md`

- [ ] **Step 1: Write failing doctor test**

Add to `src/flows/doctor-core-checks.test.ts`:

```ts
it("warns when autonomous skill evolution is enabled for agents outside allowlist", async () => {
  const result = await runDoctorCoreCheckForTest("core/doctor/skill-workshop-tool-policy", {
    cfg: {
      tools: { profile: "coding" },
      skills: {
        workshop: {
          autonomous: {
            enabled: true,
            agentAllowlist: ["analyzer"],
          },
        },
      },
      agents: {
        list: [{ id: "coder" }],
      },
    },
  } as never);

  expect(result.findings.map((finding) => finding.message).join("\n")).toContain(
    "autonomous Skill Workshop is restricted to 1 agent",
  );
});
```

- [ ] **Step 2: Run test to verify RED**

Run:

```bash
pnpm test src/flows/doctor-core-checks.test.ts
```

Expected:

```text
FAIL src/flows/doctor-core-checks.test.ts
```

- [ ] **Step 3: Add doctor note**

Modify `src/flows/doctor-core-checks.ts` so the existing `core/doctor/skill-workshop-tool-policy` check also reports allowlist scope when enabled:

```ts
const autonomous = resolveSkillWorkshopConfig(ctx.cfg).autonomous;
if (autonomous.enabled && autonomous.agentAllowlist.length > 0) {
  notes.push(
    `autonomous Skill Workshop is restricted to ${autonomous.agentAllowlist.length} agent(s): ${autonomous.agentAllowlist.join(", ")}`,
  );
}
```

- [ ] **Step 4: Document activation command**

Add to `docs/tools/skill-workshop.md`:

````md
## Runtime skill evolution

Runtime skill evolution is disabled by default. To allow foreground agent runs
to create pending Skill Workshop proposals from durable user corrections:

```bash
openclaw config patch --stdin <<'JSON'
{
  skills: {
    workshop: {
      autonomous: {
        enabled: true,
        maxProposalsPerRun: 3,
        agentAllowlist: ["main", "analyzer"]
      },
      approvalPolicy: "pending"
    }
  }
}
JSON
```

This creates pending proposals only. It does not apply live `SKILL.md` files
without Skill Workshop approval.
````

- [ ] **Step 5: Run doctor tests to verify GREEN**

Run:

```bash
pnpm test src/flows/doctor-core-checks.test.ts
```

Expected:

```text
PASS src/flows/doctor-core-checks.test.ts
```

## Task 8: Mutation Proof

**Files:**
- Temporarily modify: `src/skills/research/autocapture.ts`
- Temporarily modify: `src/skills/evolution/policy.ts`

- [ ] **Step 1: Inject proposal-limit mutant**

Temporarily change:

```ts
const runnableProposals = proposals.slice(0, workshopConfig.autonomous.maxProposalsPerRun);
```

to:

```ts
const runnableProposals = proposals;
```

- [ ] **Step 2: Run test and verify mutant is killed**

Run:

```bash
pnpm test src/skills/research/autocapture.test.ts
```

Expected:

```text
FAIL src/skills/research/autocapture.test.ts
```

The failing test must be:

```text
respects autonomous maxProposalsPerRun
```

- [ ] **Step 3: Revert mutant**

Restore:

```ts
const runnableProposals = proposals.slice(0, workshopConfig.autonomous.maxProposalsPerRun);
```

- [ ] **Step 4: Inject blocked-session mutant**

Temporarily change policy so cron sessions are not blocked:

```ts
const BLOCKED_TRIGGERS = new Set(["heartbeat", "memory", "overflow"]);
```

- [ ] **Step 5: Run test and verify mutant is killed**

Run:

```bash
pnpm test src/skills/evolution/policy.test.ts
```

Expected:

```text
FAIL src/skills/evolution/policy.test.ts
```

The failing test must be:

```text
blocks cron and memory sessions
```

- [ ] **Step 6: Revert mutant**

Restore:

```ts
const BLOCKED_TRIGGERS = new Set(["cron", "heartbeat", "memory", "overflow"]);
```

## Task 9: Full Verification

**Files:**
- All modified source and docs.

- [ ] **Step 1: Run targeted suite**

Run:

```bash
pnpm test \
  src/skills/evolution/trace.test.ts \
  src/skills/evolution/policy.test.ts \
  src/skills/research/autocapture.test.ts \
  src/skills/workshop/config.test.ts \
  src/skills/workshop/curator.test.ts \
  src/agents/harness/agent-end-side-effects.test.ts \
  src/flows/doctor-core-checks.test.ts
```

Expected:

```text
PASS
```

- [ ] **Step 2: Run agent runtime gate**

Run:

```bash
pnpm test \
  "src/agents/agent-*.test.ts" \
  "src/agents/embedded-agent-*.test.ts" \
  "src/agents/agent-tools*.test.ts" \
  "src/agents/agent-settings.test.ts" \
  "src/agents/agent-tool-definition-adapter*.test.ts" \
  "src/agents/agent-hooks/**/*.test.ts"
```

Expected:

```text
PASS
```

- [ ] **Step 3: Run repo checks**

Run:

```bash
pnpm check
pnpm build
```

Expected:

```text
exit code 0
```

- [ ] **Step 4: Validate local OpenClaw activation command in dry-run**

Run:

```bash
openclaw config patch --stdin --dry-run <<'JSON'
{
  skills: {
    workshop: {
      autonomous: {
        enabled: true,
        maxProposalsPerRun: 3,
        agentAllowlist: ["main", "analyzer"]
      },
      approvalPolicy: "pending"
    }
  }
}
JSON
```

Expected:

```text
```

The command should exit `0` and show a valid config preview without writing.

## Task 10: Controlled Local Activation

**Files:**
- Runtime config: `~/.openclaw/openclaw.json`

- [ ] **Step 1: Snapshot current config**

Run:

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.pre-skill-evolution-$(date -u +%Y%m%dT%H%M%SZ)
```

Expected:

```text
exit code 0
```

- [ ] **Step 2: Enable pending-proposal-only evolution**

Run:

```bash
openclaw config patch --stdin <<'JSON'
{
  skills: {
    workshop: {
      autonomous: {
        enabled: true,
        maxProposalsPerRun: 3,
        agentAllowlist: ["main", "analyzer"]
      },
      approvalPolicy: "pending"
    }
  }
}
JSON
```

Expected:

```text
exit code 0
```

- [ ] **Step 3: Validate config**

Run:

```bash
openclaw config validate
openclaw doctor --lint --only core/doctor/skill-workshop-tool-policy
```

Expected:

```text
exit code 0
```

- [ ] **Step 4: Restart gateway during a quiet window**

Run:

```bash
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

Expected:

```text
Active: active (running)
```

- [ ] **Step 5: Smoke test proposal creation**

Run:

```bash
openclaw agent --session-key agent:main:skill-evolution-smoke \
  --message "From now on, when working on GitHub PRs, always check CI before final response."
openclaw skills workshop list --agent main
```

Expected:

```text
github-pr-workflow
pending
```

- [ ] **Step 6: Do not auto-apply**

Run:

```bash
openclaw skills workshop inspect <proposal-id> --agent main
```

Expected:

```text
status: pending
```

No active `SKILL.md` is written until an explicit `apply`.

## Rollback

If the loop creates bad proposals or noisy suggestions:

```bash
openclaw config patch --stdin <<'JSON'
{
  skills: {
    workshop: {
      autonomous: {
        enabled: false
      }
    }
  }
}
JSON
systemctl --user restart openclaw-gateway.service
```

Pending proposals remain inspectable and rejectable:

```bash
openclaw skills workshop list --agent main
openclaw skills workshop reject <proposal-id> --agent main --reason "Runtime evolution rollback"
```

## Final Acceptance

The work is complete only when:

- Runtime traces create pending Skill Workshop proposals from durable user corrections.
- No proposal is applied without approval under default config.
- Agent allowlist and max-proposal limits are enforced.
- Curator still records usage and archives only eligible agent-created skills.
- RED, GREEN, mutation proof, targeted tests, agent runtime tests, `pnpm check`, and `pnpm build` have raw output recorded.
- Local activation is either completed with a gateway health check or explicitly left unactivated with a clear reason.
