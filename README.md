# Agent Smith — Multi-Agent System Examples

A working reference for building multi-agent LLM pipelines in Claude Code that actually reduce hallucination/spec-drift, and actually control cost — grounded in deep research rather than intuition, and building on top of two systems already proven in this workspace (`apps/ceiling-designer`, `apps/purrfect-blocks`).

## Start here

1. [`research/multi-agent-systems-research.md`](research/multi-agent-systems-research.md) — the raw research: architectural patterns, what actually reduces hallucination (ranked by evidence strength), the real cost math behind "$100 one-shot vs $8-10 multi-agent," known failure modes, full annotated bibliography (Anthropic's own multi-agent writeup, MAST failure taxonomy, generator-verifier asymmetry papers, debate/self-consistency literature, framework comparisons).
2. [`research/existing-patterns-analysis.md`](research/existing-patterns-analysis.md) — what's already built and validated in this workspace: `ceiling-designer`'s sequential pipeline, `purrfect-blocks`' orchestrator+parallel model, and `neo/chats`' minimal 2-agent research→synthesis pipeline.
3. [`research/synthesis-and-decision-framework.md`](research/synthesis-and-decision-framework.md) — the distilled decision framework: which pattern to use for which task shape, how model tiering actually produces the cost savings, and the four concrete gaps the examples below fill in.

## The one-sentence finding

**Agent count is a latency/parallelism lever, not a quality lever.** Quality comes from asymmetric verification (a structurally distinct, ideally stronger-model check), tool-grounding, and hard-capped retry loops. Cost savings come from model tiering and lean file-based handoffs — not from adding more agents. Multi-agent decomposition without both of those levers tends to cost *more* than a single frontier-model call, not less (Anthropic's own system: ~15x the tokens of a single chat turn).

## Examples

| # | Pattern | Fills the gap | Use when |
|---|---|---|---|
| [`01-tiered-pipeline-reviewer`](examples/01-tiered-pipeline-reviewer/) | Sequential pipeline, model tiered by role, independent stronger-model reviewer scoring a fixed rubric | Neither existing system uses a different model for review than generation, or scores review dimension-by-dimension | Single well-scoped feature, no architectural uncertainty, want the cheapest pipeline with genuinely independent verification |
| [`02-parallel-fanout-research`](examples/02-parallel-fanout-research/) | Cheap-model workers fan out on independent angles, expensive model synthesizes + grounds | Generalizes `purrfect-blocks`' 2-way research pair to N angles with explicit stop-condition discipline (Anthropic's own "spawned 50 subagents" failure, avoided) | Genuinely independent research angles, coverage/latency matters more than depth on any one angle |
| [`03-grounded-dual-reviewer`](examples/03-grounded-dual-reviewer/) | Two independent reviewers vote on narrow, specific claims; disagreement resolved by tool-grounding or human escalation, never by a third "tie-breaking" opinion | Neither existing system does dual-reviewer voting at all — this is the one genuinely new pattern, scoped narrowly per the debate-drift/confident-convergence research | The highest-ambiguity, highest-stakes verdicts only (e.g. "does this meet WCAG 2.2 criterion X") — not a general-purpose review mechanism, doubles cost on the rows it's applied to |

Each example folder has its own README explaining the specific design decisions, model tiering rationale, and citations back to the research that justifies it.

## What NOT to do (validated failure modes, not opinions)

- **Don't parallelize a genuine dependency chain to look sophisticated.** PlanCraft benchmarks: up to 70% performance degradation forcing coordination onto sequential work. If step B needs step A's output to even know what to do, that's sequential — full stop.
- **Don't let a reviewer trust the generator's self-report**, even from the same model. Self-preference bias is measured, not theoretical (-38% to +90% swing in one benchmark).
- **Don't run unbounded retry/debate loops.** Every example here has a hard round cap and an explicit human-escalation path on exhaustion — this is a cost control and a correctness control simultaneously.
- **Don't add a third "tie-breaking" LLM opinion when two reviewers disagree.** That manufactures false consensus. Ground it in a tool if one exists; otherwise it's a human call.
- **Don't assume more agents means cheaper.** It's usually the opposite unless you deliberately tier models and keep handoffs lean (see the synthesis doc's cost section for the actual math).

## Prior art already in this workspace

- `apps/ceiling-designer` — 5-stage sequential pipeline (Speccer → Researcher → Planner[human gate] → Implementer ⇄ Reviewer, capped at 5 rounds), file-based handoffs, DDD/BDD rigor.
- `apps/purrfect-blocks` — Opus-tier orchestrator dispatching to Sonnet-tier specialized subagents, parallel research pair, parallel verification pair, JSON-structured review gate.
- `neo/chats/ai-general` — minimal 2-agent research→synthesis pipeline, the cheapest possible generator/verifier split.

These are not being replaced — the examples in `agent-smith` are deliberately built to fill the specific gaps the research identified in them (cross-tier review, narrow grounded voting, generalized N-way fan-out), not to duplicate what already works.
