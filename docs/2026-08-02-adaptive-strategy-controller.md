# Adaptive Strategy Controller

Date: 2026-08-02

Status: Accepted for v0.1 design; parameter and operating-policy values remain experimental

Decision: the Head controls reasoning in v0.1; it does not evolve it

> **Naming discipline:** this is a *controller*, not an *engine* or an *evolution framework*. v0.1 has variation and selection but no inheritance, so `inheritance_record` is accumulated and not consumed. The name goes up to `Reasoning Evolution` when it is consumed, not before. See §8.
>
> **File dating convention:** documents are dated by authoring date. The discussion behind this one began 2026-08-01; it was written on 2026-08-02.

## One-Line Decision

In v0.1 the Head does not evolve reasoning — it **controls** it, through pre-registered attempt specifications, three separate counters, graded evidence, multi-dimensional budgets, and separate local and global stops.

## Why This Document Exists

This records the outcome of the review-and-rebuttal rounds on the Head's cognitive engine — two rounds on the core design, followed by rulings on the open parameters. Leaving that in chat would reproduce this repository's existing disease: decisions scattered across documents that later contradict each other.

**What the first review legitimately broke:** using the Archimedes anecdote as a record of cognition; accepting the Head's own narration as progress; conflating local and global stop; having no state path for hypotheses; drawing a capability ladder as if it were runtime architecture; calling the scheme "evolution" without specifying inheritance.

**What the rebuttal legitimately recovered:** extracting candidate operators from extreme success cases (candidate generation is a search step, not a causal claim, and search needs no control group — studying birds to build a wing does not require a control group of flightless animals); the value of the 2/3 heuristic (expected-information-gain stopping is computed by the same Head whose self-assessment is distrusted, making it *more* gameable, not less); the implementability of approximate same-method judgment; structural progress as a real category; Cognitive Economics without a single numeraire; Wiles and Kepler as a unit-of-application error rather than counterexamples; and that the control skeleton does not conflict with the analyzer-bot v0.1 scope.

**Where the first review contradicted itself (retracted):** it attacked the circularity of the counting rule, then proposed replacing it with an expected-information-gain estimate that is more circular. It also flagged survivorship bias while asserting without evidence that problem selection and domain depth were "much more likely" the deciding factors.

**What the second round fixed — four blockers that would have re-collided at implementation:**

1. The cost basis for judgment asymmetry was wrong, and the method state machine was missing `suspended` (§2, §7).
2. `failure_signature` cannot be frozen before execution, because failures happen after it (§3).
3. Attempt weights and the 2/3 rule were counting different things through one counter, and a representation change reset the counter at no global cost — an escape hatch (§4, §5).
4. Having workers return their own progress grade relocates self-narration rather than removing it (§6).

Plus: `suspended` resurrection needed splitting, global stop needed more reasons than budget exhaustion, representation switching was over-prioritized, and the operator research needed a third track (§7, §10, §12, §13).

## 1. Two Levels

Architecture has exactly two levels:

- **Object-level reasoning** — reasoning that solves the problem.
- **Strategy-level control** — generation, selection, evaluation, termination.

The four-rung ladder (`Prompt → Reasoning → Meta Reasoning → Reasoning Generator`) is **not** runtime layering. Generating a strategy and evaluating a strategy are both metacognitive acts at the same level. That ladder is a `capability progression` describing how a user's use of AI matures, and it does not appear in the architecture.

## 2. Judgment Asymmetry — The Governing Principle

Every judgment rule below inherits this.

The asymmetry is **not** that stopping early is cheap. Discarding the only viable method is expensive. The asymmetry is about **boundedness and recoverability**:

- An early stop can be made **recoverable** — the method is suspended, its record is kept, and it can be resumed.
- An extension granted by self-approval has **no upper bound** — each extension can justify the next.

Therefore:

> Self-assessment may **reduce remaining budget or suspend the current method**. It may not increase budget or exceed a hard cap.
>
> A stop taken on self-assessment is by default a reversible `suspend`. Only a direct refutation or a reproduced hard failure permits an irreversible `destroy`.

The Head gets the brake, not the authority to scrap a working engine on a hunch.

This dissolves the circularity objection without pretending the judge is reliable. The counting rule is a regularizer bounding the damage of a roughly 70%-accurate judge, and the judge only ever moves the cap downward. An imperfect overcurrent sensor is why you also install a breaker.

**Consequence for naming:** what the original "self-destruction principle" mostly produces is `suspended`, not `destroyed`. It is a stop rule with a rare destroy path, and §5 is written that way.

## 3. Attempt Specification — Frozen Before, Recorded After

The signature splits by time. What must be committed before execution and what can only exist after it are different objects.

**Frozen before the attempt.** Canonically serialized, hashed, immutable:

```yaml
attempt_spec:
  signature_schema_version:
  attempt_id:
  lineage_id:
  parent_method_id:
  hypothesis_id:

  method_family:
    representation:
    core_assumption_ids:
    operator_family_ids:
    evidence_policy_id:

  execution_variant:
    prompt_template_id:
    tool_profile_id:
    sampling_profile_id:
    data_subset_id:

  success_criteria:
  stop_criteria:
  progress_claims:
  local_budget:
```

**Recorded after the attempt.** Append-only; never edits the spec:

```yaml
attempt_outcome:
  attempt_spec_hash:
  tests_passed:
  tests_failed:
  counterexamples:
  evidence_added:
  resource_usage:

  failure_signature:
    violated_constraint:
    failed_test:
    failure_stage:
    counterexample:
```

**What the hash does and does not buy.** It prevents post-hoc renaming — a method cannot be re-registered under a new name after seeing how it failed. It does **not** prevent false registration at the start; a model that wants to evade the counter can declare a misleading `core_assumption` from the beginning. Two mitigations:

- Core fields are **controlled IDs and enums**, not free strings. `core_assumption_ids` references a registry; new entries are created deliberately and are visible.
- `lineage_id` and `parent_method_id` are mandatory, so derivation stays on record even when labels drift.

`evidence_policy_id` resolves to a policy record that must define, at minimum:

```yaml
evidence_policy:
  allowed_sources:        # which evidence origins count at all
  independence_condition: # what makes two evidence paths independent (§4, §6)
  verification_method:    # how a claim is checked, not who checks it
  freshness_requirement:  # how old evidence may be before it is stale
  failure_determination:  # what counts as this method having failed
```

Without `failure_determination` fixed in advance, `failure_signature` in the outcome has nothing to be compared against, and the early-stop rule loses its anchor.

Fully mechanical identity judgment remains impossible and unnecessary. Approximate classification is enough — an imprecise scale is still a scale.

## 4. Three Counters, Not One

Attempt weight and the 2/3 rule count different things. Collapsing them into one counter is what made the earlier draft ambiguous about what "two" meant.

```text
attempt_credit
  How much local compute the method has consumed.
  Accumulates in fractions. Default hard cap 3.0.

same_failure_confirmations
  How many times the same failure_signature was reproduced
  along an independent evidence path. 2 permits early stop.

strategy_switch_count
  Global count of switches to a new method_family.
  A representation change also consumes this budget.
```

**What makes an evidence path independent.** A confirmation counts toward `same_failure_confirmations` only if it ran along a different evidence path, not merely through a different agent. Calling a second component does not create independence. This uses the same fixed axis enum as §6's grade-2 independence rule — at least one strong axis (`source_lineage`, `data_partition`, `verification_method`, `toolchain`) or two weak axes (`model_provider`, `prompt_context`, `sampling_path`, `evaluator_instance`) must differ, and the same `failure_signature` must still appear. Two runs of the same worker over the same data with a different temperature are one confirmation, not two — that's a single weak axis (`sampling_path`).

**Credit per attempt**, derived from which `attempt_spec` fields changed — never assigned by the Head:

| Changed field | attempt_credit |
| --- | --- |
| `execution_variant` only | 0.2 |
| `evidence_policy` | 0.5 |
| `operator_family`, core assumption unchanged | 0.8 |
| `representation` | new local method — local credit resets |
| `core_assumption` | new method family |

**These numbers are config defaults, not rules. Only their ordering carries meaning** — a variation that changes nothing but the prompt must cost less than one that changes the operator family. The specific values 0.2 / 0.5 / 0.8 are placeholders to be tuned against logs, and nothing in the design may depend on their exact magnitude.

**When several fields change at once, apply only the highest single weight.** Summing over-counts easily; max is simple and predictable.

**Closing the reset escape hatch.** A `representation` change resets the *local* credit counter. It does **not** reset `lineage_id` or refund `strategy_switch_count`. Without that rule, a model can rewrite the representation indefinitely and never reach any cap.

**Time is not a reset button.** `attempt_credit` does not decay with elapsed time inside a lineage. If it did, an overnight cron would wash off its record and return in the morning as a new person. A fresh lineage or a re-evaluation of accumulated credit requires an **explicit event** — new external evidence, an environment change, or user approval — the same event class that governs resumption in §10.

**Nor is suspension.** A suspended method (§7) resuming with its credit zeroed would be the same laundering through a different door: spend to 2.9, suspend, resume, spend another 3.0. But carrying the full spend over unconditionally makes `suspended` a slower `destroyed` — a method resumed on genuinely new evidence with 0.1 credit left cannot be tried. So restoration is **scaled to what actually changed**:

| Resume trigger | Credit effect |
| --- | --- |
| New external evidence at grade 1 or 2 bearing on the failure | Full restore, new lineage |
| Environment change that removes the blocker — tool restored, permission granted, data arrived | Full restore, new lineage |
| Explicit budget approval | No restore; adds the approved delta, same lineage |
| Instruction to retry with nothing else changed | No restore; the remaining credit carries over |

The evidence in the first row is graded by the §6 rules, not by the Head's assertion that it found something new. Grade 3 or 4 does not qualify — otherwise "I understand the problem better now" becomes a budget refill, which is the self-narration problem wearing yet another hat.

The principle underneath: **restoration is proportional to actual state change, not to judgment.**

| What changed | Restoration |
| --- | --- |
| Grade 1 or 2 evidence — information about the world | Full |
| Environment or tools — what is executable | Full |
| User approval — permission and budget only | The approved delta |
| Grade 3 or 4 — possibly only the model's interpretation | None |

### Manual Extension

An approval **adds a delta; it never resets the counter.** A method at 2.4 of 3.0 granted +0.5 does not return to 2.9 remaining — its total allowance becomes 3.5, it continues from 2.4, and only the 0.5 is newly spendable.

```yaml
manual_extension:
  max_grants_per_suspension_episode: 1
  max_cumulative_credit_per_method_lineage: configurable
```

One grant per suspension episode is not enough on its own, because a method could suspend, re-enter the same state, and claim a fresh episode. Hence the second limit, on the whole lineage. A **new suspension episode requires new grade 1 or 2 evidence or a real environment change** — cycling the state without anything changing does not open one.

When the granted delta is spent and no new grade 1 or 2 evidence or environment change has appeared, the method returns to `suspended`, and **the system does not ask for another grant.** Head soliciting "just one more, please" on a loop is the slow version of the infinite retry button.

**Exhausting an approval is not grounds for `destroyed`.** That a method failed even under extra budget the user authorized means it was unproductive, not refuted. Absent a hard failure, §2 still applies and the correct terminal state is `suspended`.

### `force_resume`

The user retains final authority, and that authority must be distinguishable from a routine grant:

```text
approval
  Bounded additional credit, operating inside existing policy.

force_resume
  An explicit user policy exception. Separately audited,
  consumes global budget, and never solicited by the system.
```

"Try once more" is an approval. `force_resume` requires something closer to *continue this method even without new evidence; override the stop rule for this one case.* Keeping them separate preserves the user's override while denying the Head a way to beg its way into an unbounded loop.

## 5. The Stop Rule

- Default hard cap: **`attempt_credit` 3.0**.
- **Early stop at `same_failure_confirmations` = 2** — the same `method_family` produced the same `failure_signature` under two independent confirmations.
- **Immediate stop at 1** when a structural contradiction is fully demonstrated in a single attempt.
- **Exceeding the cap is not permitted on self-assessment alone** — it requires external evidence or explicit approval.

A stop produces `suspended` by default. `destroyed` requires a hard failure (§7).

2 and 3.0 are not claims of optimality. They are a conservative compute limit for an environment where self-assessment is distrusted, and they are an **initial prior to be tuned against logs** — an empirical heuristic not yet quantitatively validated, which is more accurate than saying they have "no basis."

High-variance problems are not fixed by allowing another attempt. Three samples do not resolve what two could not; change the evaluation — batch it, aggregate it, reduce the variance.

## 6. Progress Grades — Computed, Never Self-Awarded

Not reaching the answer is not the same as making no progress. But neither the Head nor a worker may grade its own output.

**Workers return raw evidence, not grades:**

```yaml
progress_claims:
  - claim_id:
    claim_type:
    preregistered_claim_id:
    before_value:
    after_value:
    evidence_refs:
    verifier_refs:
```

**The grade is then derived by rule:**

| Grade | Derivation rule |
| --- | --- |
| 1 Hard | Test results, counterexamples, or a change in the candidate registry are present |
| 2 Cross-validated | A verifier result satisfying the independence condition is present |
| 3 Structural | A pre-registered structural claim has its condition satisfied — **disabled in v0.1, see below** |
| 4 Narrative | Everything else |

Grades 1 and 2 stand alone. Grade 3 counts **only against a `preregistered_claim_id`** — the claim must have been written into `attempt_spec.progress_claims` before the attempt, in a form that could have failed. Dimension reduction needs `before_value` and `after_value` on the candidate count; contradiction resolution needs the contradiction named earlier; a predictive model needs the prediction recorded first. Anything described only afterward lands in grade 4. Grade 4 never counts on its own.

Requiring only externally measurable progress would be too strict — it produces a system that chases what is measurable and manufactures checkboxes. Product concepts, organizational design, problem definition, and research hypotheses without a test yet are real work. Grade 3 exists so that work is not amputated; the pre-registration requirement is what stops grade 3 from becoming a loophole.

**Grade 3 evaluation is disabled in v0.1.** `preregistered_claim_id` only resolves against a real registry — deduplicated, retired, immutable claim IDs — and that registry does not exist yet (§3's `core_assumption_ids` registry has the identical gap: "Concrete schema for the `core_assumption_ids` registry" is still an Open Question). Shipping the field without the registry behind it would let a claim look pre-registered while nothing verifies it actually was. Until the registry, immutable claim IDs, and a before/after verifier land (v0.2, §14):

- Structural claims are still recorded, tagged `provisional`, and kept for later re-evaluation — this is not "grade 3 doesn't exist," it's "grade 3 doesn't count yet."
- A `provisional` claim is never treated as grade 3 anywhere in this document. It does not satisfy a stop condition (§5), does not justify a budget or credit extension (§4), and does not count toward `same_failure_confirmations` or a resume trigger (§4, §10).
- A `provisional` claim can still be promoted later, but only by separately clearing grade 1 or 2 (test evidence or a cross-validated verifier) — never by v0.2 registry backfill relabeling old claims wholesale.

**Independence is a property, not a job title.** Calling a component `reviewer-bot` does not make it independent; the same base model over a similar context shares the same blind spots — self-review in a wig. The criterion is **low error correlation**.

Rather than fixing a count of differing axes, grade 2 requires **one strong difference or at least two weak ones**, drawn from a fixed axis list rather than free description — a Head that can describe its own difference as "strong" can talk its way past the rule:

| Strength | Axes (fixed enum) |
| --- | --- |
| Strong | `source_lineage`, `data_partition`, `verification_method`, `toolchain` |
| Weak | `model_provider`, `prompt_context`, `sampling_path`, `evaluator_instance` |

**`source_lineage` counts only for genuinely separate upstream sources.** Two evidence paths that both trace back to the same upstream artifact do not get credited as a `source_lineage` difference just because something downstream of it differs — otherwise a single flawed input could pass as two independent confirmations.

Worked examples:

| Situation | Axes | Verdict |
| --- | --- | --- |
| Same document, different model | 1 weak (`model_provider`) | Insufficient |
| Same source material, different model + different prompt context | 2 weak (`model_provider`, `prompt_context`) | Conditionally sufficient |
| Genuinely separate upstream sources | 1 strong (`source_lineage`) | Sufficient |
| Same code, different test methods | 1 strong (`verification_method`) | Sufficient |

The point of the enum is not the axis count — it's whether the difference actually lowers the odds that both paths share the same error. The fixed list is a v0.1 approximation of that; it doesn't replace the underlying question, it makes the underlying question checkable without trusting the Head's own description of "how different" something is.

Changing temperature twice is not independence — it produces two weak differences at best (both `sampling_path`), and often none. Counting axes without weighting and without a fixed enum would let the second kind masquerade as the first.

## 7. Two State Machines

```text
Method:      active → weakened → suspended
                                   ↘ destroyed

Hypothesis:  candidate → supported → weakened → suspended → active
                                              ↘ falsified  ↘ reopened_for_exploration
```

**Methods.** Lack of progress sends a method to `suspended`, which is reversible and retains its record. Only a **hard failure** — a reproduced structural contradiction, or a demonstrated violation of the method's own premise — sends it to `destroyed`. This follows directly from §2: self-assessment gets the reversible action, evidence gets the irreversible one.

**Hypotheses.** Failing to solve is not refuting. A hypothesis reaches `falsified` only on direct evidence: a reproducible counterexample, a core prediction failing, an incompatible observation, or failing a discriminating test defined in advance. When several methods fail under one hypothesis, it becomes `suspended`, not `falsified`.

**Resurrection splits in two**, because the two paths carry different epistemic weight:

| Trigger | Transition | Confidence effect |
| --- | --- | --- |
| New positive evidence | `suspended → active` | May be re-evaluated upward |
| Alternative hypotheses exhausted | `suspended → reopened_for_exploration` | **No increase** |

Alternatives failing does not support the survivor. `reopened_for_exploration` means it may be investigated again, not that it gained support.

## 8. `inheritance_record` — Recorded In v0.1, Consumed In v0.2

Variation and selection without inheritance is not evolution; it is random restart with pruning. On method suspension or destruction, preserve: the assumption that failed, constraints discovered, partial results that survive, paths not to walk again, and counterexamples the next method must explain.

**v0.1 only accumulates this record.** It does not feed it into generating the next method. The day it does is the day the name goes up to `Reasoning Evolution`.

## 9. Cognitive Cost — Multi-Dimensional Hard Limits

No single utility score. Hard limits per dimension: tokens, wall-clock time, API cost, tool calls, user-facing latency, strategy switches.

Real optimization routinely treats cost, time, risk, and quality as separate objectives without converting to one currency. Forcing a single score early has a specific failure mode here: the Head starts manufacturing the score.

**Budgets are per task class, not global constants.** One set of numbers across every request is simultaneously extravagant for one class and starving for another:

| Task class | Shape of the budget |
| --- | --- |
| Simple query | Tight on every dimension; a strategy switch is already suspicious |
| Code change | Moderate compute, low strategy-switch allowance, high evidence requirement |
| Investigation | High compute and strategy-switch allowance, long horizon |
| High-risk action | Compute is not the binding constraint; evidence grade and approval are |

The classifier that assigns a task class runs before the budget is allocated, and the class is recorded in the lineage so a run cannot quietly re-classify itself upward mid-flight.

Accurate statement of where this stands: **resources and opportunity cost are defined; the utility function and allocation policy are not yet.**

## 10. Local Stop vs Global Stop

- **Local stop** — end this method. §5.
- **Global stop** — end this problem.

These are different questions. Local stopping alone reduces time inside one frame while *increasing* the number of frames tried; without a global stop, "thinking forever in one frame" becomes "cycling frames forever."

**Global stop reasons** — budget exhaustion is only one of six:

```text
success stop      pre-defined acceptance criteria met
budget stop       tokens, time, cost, tool calls, or strategy switches exhausted
exhaustion stop   no executable method family remains
blocked stop      permissions, tools, or external data unavailable
safety stop       safety or policy constraint
user stop         cancellation, or a request for the current best
```

**When no human is available** (cron, background, overnight), the default is stop and report, never wait:

```yaml
stop_report:
  best_current_result:
  confidence:
  stop_reason:
  exhausted_budget:
  unresolved_questions:
  suspended_methods:
  evidence_needed_to_resume:
  allowed_resume_triggers:
  checkpoint_id:
```

**Elapsed time is not a resume trigger.** Resumption requires an explicit event — new evidence, a new instruction, budget approval, or an environment change. Preventing an infinite retry button must not produce an infinite wait button, and a timer-based resume is just a slower retry button.

## 11. Unit Of Application

The rule applies to **tactical units**: a specific prompt strategy, tool combination, proof path, analytical representation, or method for testing one causal hypothesis.

It does not apply to research goals, the problem itself, or long-lived hypotheses. Sustaining a goal and destroying local methods are compatible — Wiles held one goal for seven years while changing proof paths continuously; Kepler held the Mars orbit problem while moving from circles to epicycles to an ellipse. Neither repeated one tactic three times.

`method_family` holds this boundary. If it is loose, "method" quietly inflates into "research program" and the rule bites the wrong thing.

## 12. Strategy Switching — Representation Is Conditionally First

Representational change is powerful when the impasse *is* representational. It is not the right first move when the cause is missing data, a tool failure, insufficient permissions, bad input, high external variance, or a direct counterexample. Those call for fixing the input, not rebuilding the representation.

The rule is therefore conditional:

> Representation switching takes highest priority when the same failure repeats under the same representation **and** there is an indication that the current representation is unduly constraining the search space.

For example: if the same contradiction recurs twice under a natural-language formulation, switch to a causal graph or a constraint formulation. Rebuilding the representation from the first failure onward just turns the Head into a remodeling contractor.

Other switching axes — decomposition to integration, forward to backward, direct to analogical, internal to external verification, state to time-series, cause-hunting to counterexample-hunting, solving to redefining — are selected against the observed `failure_signature`, not by fixed rank.

## 13. Cognitive Operator Research — Three Tracks

The research unit is not `operator presence` but **`operator policy` and `composition`**: when an operator is invoked, in what order operators compose, how fast branches are pruned, what counts as a signal, what is preserved after failure.

Decomposition, analogy, and thought experiment are used by novices and experts alike, so their presence has little discriminating power. A chess novice and a grandmaster both search; the difference is policy. This raises the unit of the research program rather than retiring it.

**Track A — candidate discovery.** Extreme success cases and cognitive research. Legitimate for search; this document makes no causal claim, so no control group is required.

**Track B — human process validation.** Sources with actual process data: think-aloud protocol studies, expert-novice problem representation research, protocol analysis. Invocation, pruning, and retention are recorded there. They are not recorded in memoirs and biographies — the chess analogy works precisely because game records exist.

**Track C — agent transfer validation.** Required, because a human expert's policy is not guaranteed to transfer to an LLM: memory structure, search behavior, and error modes all differ. Ablation on frontier models:

```text
base model
vs operator list only
vs operator list + invocation policy
vs invocation policy + stop rules
vs invocation policy + stop rules + inheritance
```

Measured on: success rate, total tokens, wall-clock time, tool calls, repeated-failure rate, false confidence, early-stop error, recovery rate after a strategy switch, reproducibility.

Without Track C, human cognition research stays a design source and never becomes a verified agent policy.

Representational change as the core of insight is retained as a conclusion, with its basis moved from the Archimedes anecdote to the representational-change literature on insight problem solving. The anecdote's only source is Vitruvius, roughly two centuries after Archimedes — a literary construction, not a record of cognition. The conclusion survived; the evidence did not.

## 14. v0.1 / v0.2 Split

**v0.1 — Head-internal policy and state, no new agents:**

- `attempt_spec` frozen and hashed before execution; `attempt_outcome` appended after.
- Three counters: `attempt_credit`, `same_failure_confirmations`, `strategy_switch_count`.
- Multi-dimensional budgets, profiled per task class.
- Local stop, suspend by default.
- Global stop with six reasons and a `stop_report`.
- Structured worker return of **raw `progress_claims`**; grades computed by rule.
- Grade 1, 2, and 4 active; **Grade 3 (structural) evaluation gated off** — claims recorded as `provisional` only, no registry-backed pre-registration exists yet (§6).
- Fixed independence axis enum (strong: `source_lineage`/`data_partition`/`verification_method`/`toolchain`; weak: `model_provider`/`prompt_context`/`sampling_path`/`evaluator_instance`) governing both grade-2 verification and `same_failure_confirmations` (§4, §6).
- `inheritance_record` accumulation.

None of this creates a new agent, a standing reviewer/tester chain, or dynamic agent creation, so it does not conflict with the non-scope list in the [analyzer-bot design](2026-08-01-analyzer-bot-main-agent-design.md). And if the Head is the only judging entity, minimum budget and retry control belong in v0.1 regardless — without them the Head is not an orchestrator, it is an infinite retry button.

Returning raw claims rather than grades also resolves a tension with bounded contracts: the Head derives the grade from typed return values instead of reading the worker's full context.

**v0.2:** activating Grade 3 evaluation — requires the `preregistered_claim_id` registry, immutable claim IDs, and a before/after verifier, none of which ship in v0.1 (§6); consuming `inheritance_record`; adaptive stopping thresholds tuned from logs; the operator policy library; evaluating expected-information-gain rules once there are logs to calibrate against; Track C ablation results feeding back into policy.

## Risks

| Risk | Mitigation |
| --- | --- |
| The Head manufactures its own signature labels | Frozen `attempt_spec` plus controlled IDs and mandatory lineage — the hash stops renaming, the registry stops invention |
| Grade 4 progress reappears as grade 3 | Grade 3 evaluation is disabled in v0.1 entirely — structural claims are recorded `provisional` and never counted, not gated by a field that could be spoofed |
| Worker returns a `generic_result` payload to dodge typed-schema evidence requirements | ASC treats every `generic` result as `blocked`/`partial` regardless of claimed status; never counts as progress, never justifies budget/credit extension (Caveman doc, "Worker Message Contract") |
| Workers award themselves grades | Workers return raw `progress_claims`; grades are derived by rule |
| Repeated representation changes evade the cap | Local credit resets, but `lineage_id` and `strategy_switch_count` do not |
| Elapsed time launders an exhausted lineage | `attempt_credit` never decays with time; only explicit events open a new lineage (§4) |
| Suspend-and-resume used to refill the budget | Restoration is scaled to the resume trigger, and the qualifying evidence must be grade 1 or 2 (§4) |
| Repeated approvals become a slow infinite retry | One grant per suspension episode, a cumulative lineage cap, and the system never solicits another (§4) |
| State cycled to fake a new suspension episode | A new episode requires grade 1 or 2 evidence or a real environment change (§4) |
| Weak differences pass as independent verification | Grade 2 needs one strong difference or two weak ones from a fixed axis enum, not a freely-described raw axis count (§6) |
| Same upstream source double-counted as two independent `source_lineage` paths | `source_lineage` credit requires genuinely separate upstream sources, not just divergence downstream of a shared one (§6) |
| One budget applied to every kind of task | Budgets are per task class, and the class is recorded in the lineage (§9) |
| Self-assessment scraps a viable method | Self-assessment produces reversible `suspend`; `destroy` needs hard failure |
| Exhausted alternatives read as support | `reopened_for_exploration` carries no confidence increase |
| Unattended runs wait forever | `stop_report` with explicit resume triggers; elapsed time is not one |
| "Method" inflates into "research program" | `method_family` defines the tactical unit (§11) |
| Human operator policy assumed to transfer to LLMs | Track C ablation (§13) |

## Open Questions

What remains open is parameter and operating policy, not structure.

- Actual per-dimension numbers for each task-class budget profile (§9), and how a task is classified in the first place.
- Whether `strategy_switch_count` is a fixed limit per task class or derived from the remaining budget.
- Concrete schema for the `core_assumption_ids` registry — how an assumption is named, deduplicated, and retired. Same open item blocks the `preregistered_claim_id` registry that Grade 3 needs (§6).
- Whether the §6 strong/weak axis enum (now fixed: `source_lineage`/`data_partition`/`verification_method`/`toolchain` vs `model_provider`/`prompt_context`/`sampling_path`/`evaluator_instance`) needs more axes once real operating logs show independence failures the current eight don't cover. The list is fixed to close the free-description loophole, not because it's assumed complete.
- The value of `max_cumulative_credit_per_method_lineage` (§4). The rule is settled; the number waits on operating logs.

## References

- [Mixed Framework Orchestration Plan](../README.md)
- [Analyzer-bot Main Agent Design](2026-08-01-analyzer-bot-main-agent-design.md)
- [LangGraph Head Reference Patterns](2026-07-31-langgraph-head-reference-patterns.md)
- [OpenClaw Runtime Skill Evolution Implementation Plan](superpowers/plans/2026-07-10-openclaw-runtime-skill-evolution.md)
- Representational change theory of insight — the basis for §13's retained conclusion
- Think-aloud protocol analysis and expert-novice problem representation studies — Track B source class
- Vitruvius, *De architectura* IX — the actual and only source of the Archimedes anecdote, cited here as what it is
