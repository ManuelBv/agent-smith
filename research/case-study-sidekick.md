# Case Study: How `sidedotdev/sidekick` Implements Agentic Workflows

Research date: 2026-07-09. Direct source-code investigation of [github.com/sidedotdev/sidekick](https://github.com/sidedotdev/sidekick) (default branch `main`), read via the GitHub API + raw file fetches. This is a companion to the `agent-smith` research set — it tests the thesis in [`synthesis-and-decision-framework.md`](synthesis-and-decision-framework.md) and [`hype-vs-evidence-disconnect.md`](hype-vs-evidence-disconnect.md) against a real, production-grade OSS coding agent that Manuel did not build.

**Files read** (all under `dev/` unless noted): `dev_agent.go`, `dev_agent_manager_workflow.go`, `basic_dev_workflow.go`, `planned_dev_workflow.go`, `llm_loop.go`, `build_dev_plan.go`, `follow_dev_plan.go`, `fulfillment.go`, plus `AGENTS.md` and the README. Line references below are to `main` at read time.

---

## 0. The one-sentence answer to Manuel's question

**Yes, sidekick uses "agents" — but almost nothing about its design matches the popular `.claude/agents/`-style "spawn a team of role-playing sub-agents" mental model.** Its "agent" is a *single durable LLM tool-calling loop*; its "manager" is a *Temporal supervisor workflow* (a dispatcher and human-in-the-loop signal router, **not** an LLM orchestrator); and its multi-role structure is expressed as **model-config-per-role keys** (`PlanningKey`, `CodingKey`, `JudgingKey`) inside a **strictly sequential pipeline** — the exact shape your `ceiling-designer` already has, and a near-perfect real-world instance of the three levers the `agent-smith` research says actually matter. It is the anti-hype architecture, shipped.

---

## 1. What "agent" actually means in sidekick

There are **three distinct things** the word could refer to, and only one of them is what the multi-agent hype means by "agent":

| Thing in the code | What it actually is | Hype-sense "agent"? |
|---|---|---|
| `DevAgent` (`dev_agent.go`) | A thin **client-side handle**. It doesn't reason. It finds-or-starts the manager workflow and forwards a `WorkRequest` to it via a Temporal Update. ~123 lines, zero LLM calls. | ❌ No |
| `DevAgentManagerWorkflow` (`dev_agent_manager_workflow.go`) | A long-lived **Temporal supervisor workflow**. Receives signals (cancel, user-response, request-for-user, workflow-closed), dispatches child dev workflows, updates task status, cleans up stale git worktrees hourly. **Contains no LLM reasoning whatsoever.** | ❌ No — it's a durable message router |
| `LlmLoop` (`llm_loop.go`) | The **actual agent**: a generic tool-calling loop that iterates the LLM, executes tool calls, and checks for pause/human-feedback each turn. | ✅ **This is the agent** |

**The critical finding:** the component named "manager" is *not* an LLM orchestrator. In the hype/Anthropic sense, an "orchestrator" is a *model* that plans and dispatches to *model* workers. Sidekick's manager is deterministic Go workflow code — it makes routing decisions with `if workRequest.FlowType == "basic_dev"` (`dev_agent_manager_workflow.go:320`), not with a prompt. **The coordination layer is code, not a model.** This is exactly the direction the vendor field is converging toward (see [`hype-vs-evidence-disconnect.md`](hype-vs-evidence-disconnect.md) §3: Microsoft/OpenAI retiring emergent GroupChat/Swarm for structured, supervised control flow) — sidekick just started there.

---

## 2. The architecture, top to bottom

```
Task created
   │
   ▼
DevAgent (client handle)  ──Temporal Update──▶  DevAgentManagerWorkflow (durable supervisor)
                                                        │  routes by FlowType, no LLM
                                                        ▼
                                        ExecuteChildWorkflow: PlannedDevWorkflow  (the pipeline)
                                                        │
        ┌───────────────────────────────────────────────┼───────────────────────────────────────────────┐
        ▼                     ▼                           ▼                          ▼                     ▼
  BuildDevRequirements   BuildDevPlan            FollowDevPlan (per step)     EnsureTestsPass       reviewAndResolve
  (PlanningKey model)    (PlanningKey model)     (CodingKey model list)       (tool-grounded)       (merge review)
        │                     │                           │                          │
        │                     │                    ┌──────┴───────┐                  │
        │                     │              performStep      CheckWorkMeetsCriteria │
        │                     │              (EditCode)       (JudgingKey model)     │
        │                     │              LlmLoop          forced-tool-call verdict│
        │                     │                                                       │
        └─────────────────────┴───────── all human-in-the-loop via pause/feedback signals ──────────────┘
```

Two flow types exist (`dev_agent_manager_workflow.go:320-341`):
- **`basic_dev`** — a lighter single-loop flow (`basic_dev_workflow.go`, ~900 lines).
- **`planned_dev`** — the full sequential pipeline (`planned_dev_workflow.go`). This is the interesting one.

### The `PlannedDevWorkflow` pipeline (`planned_dev_workflow.go:34-139`)
Strictly ordered, each stage feeding the next — a textbook sequential pipeline:
1. `EnsurePrerequisites` — environment setup.
2. `BuildDevRequirements` *(optional, gated by `DetermineRequirements`)* — refine the task into requirements.
3. `BuildDevPlan` — produce a structured multi-step plan.
4. `FollowDevPlan` — execute each plan step (the coding inner loop).
5. `EnsureTestsPassAfterDevPlanExecuted` — a **tool-grounded** gate: run the real test command, loop until pass or cap.
6. `AutoFormatCode`.
7. `reviewAndResolve` *(gated by workflow version + worktree env)* — final merge review.

This is `ceiling-designer`'s Speccer → Researcher → Planner → Implementer → Reviewer, with different names. The `agent-smith` decision framework would classify it as **"strict data dependency → sequential pipeline"** — and sidekick got the classification right: it did *not* parallelize the dependency chain.

---

## 3. The three levers — sidekick hits all three (this is the headline)

The `agent-smith` thesis says quality comes from **(1) tool-grounding, (2) a structurally distinct verifier, (3) bounded retry loops** — and cost from **model tiering + lean handoffs**. Sidekick independently implements every one of these.

### Lever 1 — Tool-grounding (strongest evidence in the research)
The test command is the ground truth, not an LLM opinion. `ensureTestsPassAfterDevPlanExecutedSubflow` (`planned_dev_workflow.go:147-201`) loops on the **actual** `RunTests` result — `testResult.TestsPassed` / `TestsSkipped` — including a separate integration-test gate. The LLM is only re-invoked (`completeDevStep`) *after* a real test failure, and the loop only exits on a real pass. This is the same load-bearing mechanism as `ceiling-designer`'s TDD loop.

### Lever 2 — A structurally distinct verifier, on its own model tier
This is the strongest match to the research's single-highest-leverage recommendation ("use a different/stronger model as verifier, and make the review a rubric verdict, not free-form"):

- The verifier is a **separate role with its own model config key**: `dCtx.GetModelConfig(common.JudgingKey, ...)` (`fulfillment.go:83, 133`) — distinct from `PlanningKey` and `CodingKey`. The judge model is *configurable independently* of the coder model, which is precisely the cross-tier verification the `agent-smith` research flags as the "single highest-leverage unbuilt idea" in Manuel's own systems. Sidekick built it.
- The review is **not free-form**. `CheckIfCriteriaFulfilled` forces a structured tool call — `determineCriteriaFulfillmentTool` → `CriteriaFulfillment{ IsFulfilled bool; Analysis string; FeedbackMessage string }` (`fulfillment.go:17-33, 145`). This is the rubric/verdict pattern that the self-preference-bias literature ([`hype-vs-evidence-disconnect.md`](hype-vs-evidence-disconnect.md) §5) recommends over "does this look right to me."
- The verifier re-derives from artifacts (git diff since last review, three-dot diff), not from the coder's self-report (`fulfillment.go:33-119`) — the "don't trust the generator's self-assessment" principle, enforced structurally.

### Lever 3 — Bounded retry loops with hard caps + human escape hatch
Caps are everywhere, and every one has a human-in-the-loop pressure valve:
- `LlmLoop` (`llm_loop.go:56-60`): `maxIterations: 17`, `autoIterations: 3` (later bumped to 8) — after N auto-iterations without finalizing, it **stops and asks a human** ("has looped %d times without finalizing. Please provide guidance", `llm_loop.go:129`).
- `ensureTestsPassAfterDevPlanExecutedSubflow`: `maxAttempts := 3` (`planned_dev_workflow.go:148`), then hands to a human *unless* `DisableHumanInTheLoop`.
- Per-step coding loop (`follow_dev_plan.go:171`): `maxAttempts := 17`, configurable via `repoConfig.MaxIterations`.
- Manager signal loop (`dev_agent_manager_workflow.go:275`): `count >= 1000` → `ContinueAsNew` (Temporal's unbounded-history mitigation).

The human-in-the-loop escape isn't a nicety — it's the *termination condition*, which is exactly the MAST "verification/termination gap" failure category the research warns about. Sidekick closes it by making "ask the human" the default terminal state rather than "loop forever."

### Cost lever — model tiering by role, plus a provider-failover ladder
- Three role keys with independent model configs: **`PlanningKey`** (`build_dev_plan.go:335, 532`), **`CodingKey`** (`follow_dev_plan.go:167`), **`JudgingKey`** (`fulfillment.go:83, 133`). This *is* "tier models by role, not by agent identity" — the research's #3 recommendation — implemented as first-class config.
- The coding role goes further: `GetModelsOrDefault(common.CodingKey)` returns a **list**, and the step loop **rotates through models** on repeated failure (`follow_dev_plan.go:217-241`), resetting chat history and `git checkout`-reverting the working tree between models (`goToNextModel`, lines 217-232). That's a resilience/cost ladder — try the cheaper/primary model, fall back to the next on exhaustion — not a quality-via-more-agents play.

---

## 4. Where sidekick's design *reinforces* the anti-hype thesis

- **No sub-agent fan-out for the core dev loop.** Despite being a serious agentic product, sidekick does **not** spawn parallel role-playing sub-agents to "improve quality." The parallelism it has (hourly worktree cleanup running concurrently with signal handling, `dev_agent_manager_workflow.go:50-81`) is *infrastructure* concurrency, not *reasoning* fan-out. This is a strong real-world data point for the thesis's core claim: a production coding agent chose single-loop + verification over agent-swarming.
- **The "orchestrator" is deterministic code.** Sidekick refutes the hype framing that you need an *LLM* orchestrator. Routing, dispatch, retries, and status bookkeeping are all plain Go inside a Temporal workflow. The model is reserved for the parts that genuinely need judgment (plan, code, judge). This maps to the `hype-vs-evidence-disconnect.md` §6 finding: quality comes from verification structure + tool-grounding, and the "orchestration" is mostly plumbing best done in code.
- **Durability replaces "memory agents."** Where naive multi-agent systems add agents/context-passing to preserve state, sidekick uses Temporal's durable execution + `NewVersionedChatHistory` + file/DB-persisted flow actions. Handoff-context-loss (MAST's ~37% failure category) is handled by the workflow engine's event history, not by agent-to-agent message passing. This is a *different, arguably stronger* answer to the same problem `ceiling-designer` solves with `N-agent-output.md` file handoffs.

## 5. Where sidekick is genuinely different from your `agent-smith` patterns

Not everything maps cleanly — three honest divergences worth noting:

1. **Temporal as the substrate changes the whole failure model.** Your `agent-smith` examples are prompt/markdown-defined agents in Claude Code; sidekick's are Go workflow functions with deterministic-replay constraints (see `AGENTS.md` lines 29-38: "Changes to workflow logic should be deterministic... add a new workflow version that gates new logic"). The `workflow.GetVersion(...)` calls littered throughout are *schema-migration-for-running-workflows* — a concern your file-based-handoff pattern doesn't have. **Lesson for `agent-smith`:** durable-execution engines give you crash-safety and bounded-history for free, at the cost of determinism discipline. Worth a note in the decision framework as a "when the pipeline must survive process restarts, reach for Temporal, not just files."
2. **The verifier is not confirmed to be a *stronger* model — only a *separately-configured* one.** `JudgingKey` can be pointed at any model; nothing in the code forces it to be stronger than the coder. So sidekick realizes the "distinct verifier" half of Lever 2 structurally, but whether it exploits generator-verifier *asymmetry* (stronger judge) is left to the operator's config. That's the same gap your research names — sidekick gives you the mechanism but doesn't mandate the strong-judge policy.
3. **No dual-reviewer voting (your example 03's pattern).** Sidekick uses a *single* judging pass (`CheckWorkMeetsCriteria`), then routes disagreement to a *human*, not to a second reviewer. This actually *agrees* with your research's caution ("don't add a third tie-breaking LLM; ground it or escalate to a human") — sidekick's escalation target is always the human, never a debate. Your example 03's narrow grounded voting is still a genuinely distinct, unbuilt-here pattern.

---

## 6. Verdict and what to take back into `agent-smith`

**Sidekick is a real-world existence proof of the `agent-smith` thesis.** A production coding agent, built independently, converged on: sequential pipeline + tool-grounded gates + a structurally distinct judging role on its own model tier + hard-capped loops with human escalation + deterministic (non-LLM) orchestration. It reached for *more agents* nowhere in the core loop. Every design choice lines up with "agent count is a latency lever, not a quality lever; quality is verification + grounding + bounded loops."

Concrete items to fold into the `agent-smith` docs:
1. **Add sidekick as a cited external case study** in `existing-patterns-analysis.md` — it's a third-party corroboration of the same pattern set, which is stronger evidence than two systems Manuel built himself.
2. **Add a "durable-execution substrate" row to the decision framework** — Temporal/workflow-engine as the answer when the pipeline must survive restarts and needs bounded history, distinct from lightweight file-handoff pipelines.
3. **Note the `JudgingKey` idiom as the concrete implementation** of "cross-tier verification" — it's the exact config-shaped realization of the research's #1 unbuilt recommendation, and a cleaner reference than describing it abstractly.
4. **Confirm example 03 (dual-reviewer voting) remains genuinely novel** — sidekick escalates to humans, not to a second reviewer, so your narrow grounded-voting pattern is not redundant with it.

---

## 7. Source map (files → claims)

| Claim | File:line |
|---|---|
| `DevAgent` is a thin client handle, no LLM | `dev/dev_agent.go:19-108` |
| Manager is a durable signal-routing supervisor, no LLM reasoning | `dev/dev_agent_manager_workflow.go:24-90, 239-280` |
| Manager routes by `FlowType` in code, not by prompt | `dev/dev_agent_manager_workflow.go:320-341` |
| Sequential pipeline: requirements → plan → follow → tests → format → review | `dev/planned_dev_workflow.go:34-139` |
| Tool-grounded test gate with bounded retry (max 3) + human escape | `dev/planned_dev_workflow.go:147-201` |
| The actual "agent" = generic `LlmLoop`, maxIterations 17, human check-in | `dev/llm_loop.go:55-192` |
| Planning role uses `PlanningKey` model config | `dev/build_dev_plan.go:335, 532` |
| Coding role uses `CodingKey` model *list* w/ failover ladder | `dev/follow_dev_plan.go:167, 217-241` |
| Verifier/judge = separate `JudgingKey` model, forced-tool-call rubric verdict | `dev/fulfillment.go:17-33, 83, 133-145` |
| Verifier re-derives from diffs, not coder self-report | `dev/fulfillment.go:33-119` |
| Deterministic-replay / workflow-versioning discipline | `AGENTS.md:29-38`; `workflow.GetVersion(...)` throughout |

**Investigation limits:** read the core `dev/` flow files and the manager, not the entire codebase (`basic_dev_workflow.go`, `llm2/` providers, and the `flow_action`/`persisted_ai` layers were only partially read). The `basic_dev` flow was inventoried but not line-audited; conclusions above are about the `planned_dev` pipeline specifically. Whether `JudgingKey` is *operationally* pointed at a stronger model is config-dependent and not determinable from source.
