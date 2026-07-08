# Synthesis & Decision Framework

Distills [`multi-agent-systems-research.md`](multi-agent-systems-research.md) (the raw research) and [`existing-patterns-analysis.md`](existing-patterns-analysis.md) (what's already validated in `ceiling-designer` / `purrfect-blocks` / `chats/ai-general`) into a single decision framework for building and extending `agent-smith` examples.

## The one-sentence version

**Agent count is a latency/parallelism lever, not a quality lever. Quality comes from asymmetric verification (a structurally distinct, ideally stronger-model check) plus tool-grounding plus a hard-capped retry loop — and cost savings come from model tiering + lean file-based handoffs, not from adding more agents.**

## The three levers, in priority order

1. **Tool-grounding wherever a deterministic check exists.** Tests, type-checkers, linters, accessibility audits, citation-existence checks. This removes the LLM from the judgment loop entirely for anything checkable. Already the load-bearing mechanism in `ceiling-designer`'s TDD loop — keep doing this first, before reaching for more agents.
2. **A structurally distinct, ideally stronger/different-model verifier.** Verifying is empirically much cheaper than generating (a model at ~15% accuracy *generating* proofs hit ~85% *judging* them). Same-model self-review carries measurable self-preference bias (-38% to +90% swings in some benchmarks). Concretely: don't let the Implementer's model also be the Reviewer's model if you can afford not to; at minimum, switch the reviewer prompt from free-form ("does this look right") to an explicit rubric scored dimension-by-dimension (cuts self-preference bias ~31.5%).
3. **Bounded retry loops with a hard cap and a visible log.** Validated twice over: MAST shows uncoordinated systems amplify errors up to 17x vs. ~4.4x when a centralized validation gate exists; unbounded debate/self-refine loops are shown to drift or converge confidently on a shared wrong answer. Both existing systems already do this (5-round cap, JSON approval gate) — never remove it.

Everything else (parallel fan-out, debate, hierarchical subagents, swarms) is a *latency or coverage* tool, not a hallucination-reduction tool. Reach for them only when the task graph actually has independent branches.

## Pattern selection — pick by task graph shape, not novelty

| Task graph shape | Pattern | Existing example |
|---|---|---|
| Strict data dependency, each step needs the last (spec → plan → build) | Sequential pipeline | `ceiling-designer` |
| Independent subtasks that don't share mutable state (two research angles; code review + test coverage check) | Parallel fan-out/fan-in | `purrfect-blocks` (research pair, verification pair) |
| Need a decomposer + specialized workers, task complexity varies per request | Orchestrator-supervisor | `purrfect-blocks` (orchestrator agent) |
| Two answers already exist and disagree, need to surface *where* they disagree (not to invent new truth) | Bounded debate/voting, paired with a grounding fallback | Gap — build in `agent-smith` |
| Problem has genuine multi-domain decomposition (frontend/backend/infra as separate concerns within one feature) | Hierarchical/nested subagents | Not needed yet at current project scale |
| Routing is dynamic and unknowable in advance (support triage) | Swarm/decentralized/handoffs | Not applicable to Manuel's project shape — avoid; the field itself is moving away from this (AutoGen → maintenance mode, Agents SDK favors bounded agents-as-tools over open handoffs) |

**Rule of thumb**: default to sequential. Only add parallelism where you can point at two steps and say "neither's output changes the other's input." Never parallelize a dependency chain to look more sophisticated — PlanCraft shows up to 70% degradation from doing this.

## Cost tiering — the actual mechanism for the $8-10 target

Model tiering by **role**, not by agent identity:

| Role | Why | Tier |
|---|---|---|
| Speccer (interview), Researcher (drafting/gathering) | No irreversible decision made here; volume work | Cheap/fast (Haiku-tier) |
| Implementer | Mid-complexity generation, but self-correcting via TDD | Mid (Sonnet-tier) |
| Planner (human-gated architecture decision), Reviewer (final quality gate) | Irreversible or expensive-to-unwind if wrong | Expensive (Opus-tier) |

Combine with **lean file-based handoffs** — each agent reads a distilled `N-agent-output.md` artifact, not the full prior conversation. This is the actual multiplier: tiering alone gets you to ~$25-35 from a $100 baseline; tiering + lean context gets you toward $8-10. Multi-agent-ness *by itself*, without both levers, tends to cost **more** than a single frontier call (Anthropic's own system: ~15x tokens vs. a single chat turn).

**Watch for the cost traps** (all three are "more agents, same or worse quality, higher bill"):
- Redundant context re-derivation (incomplete handoff artifact forces re-reading everything)
- Unbounded retry loops (no hard cap)
- Orchestration overhead that doesn't buy back wall-clock time or coverage (a lead agent's own planning/synthesis calls are pure overhead unless parallelization or context-window-exceeding coverage justifies them)

## What's already right in `ceiling-designer` / `purrfect-blocks` — don't refactor these

- File-based handoff as the memory model (matches Anthropic's own "artifact pattern" recommendation and directly addresses the top two MAST failure categories: spec/design flaws ~42%, inter-agent context loss ~37%)
- Independent reviewer that doesn't trust the generator's self-report
- Bounded correction loop with a visible log
- Parallel dispatch only where independence is real (research pair, verification pair)
- Tool-scoping per agent (orchestrator can't write code; researchers can't touch source)

## What's genuinely missing — the actual gaps to fill with new `agent-smith` examples

1. **Cross-model/tiered verification** — neither existing system uses a different model for review than for generation. This is the single highest-leverage unbuilt idea.
2. **Rubric-based (not free-form) review scoring** — both existing reviewers use checklists already, which is good, but neither explicitly scores dimension-by-dimension with a number/verdict per dimension the way the self-preference-bias mitigation research recommends.
3. **Grounded, narrowly-scoped dual-reviewer voting** — two independent reviewer passes voting on specific, narrow acceptance criteria (not open-ended "is this good"), escalating to a human or tool-grounded check on disagreement rather than letting them debate to resolution.
4. **Explicit model-tiering by role documented as a first-class design decision**, not an incidental implementation detail.

These four map directly to the three example projects built in `agent-smith/examples/` (see each example's README for how).

## Sources
See the full annotated bibliography in [`multi-agent-systems-research.md`](multi-agent-systems-research.md#7-annotated-bibliography). Anchor citations: [Anthropic's multi-agent research system writeup](https://www.anthropic.com/engineering/multi-agent-research-system) (architecture + cost data), [MAST failure taxonomy](https://arxiv.org/html/2503.13657v3) (failure modes), [generator-verifier asymmetry](https://tobysimonds.com/research/2025/09/29/Proofs.html) (verification cheaper than generation), [self-preference bias survey](https://arxiv.org/html/2410.21819v1) (why same-model review is risky).
