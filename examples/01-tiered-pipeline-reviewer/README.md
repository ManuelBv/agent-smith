# Example 1 — Tiered-Model Sequential Pipeline with Independent Reviewer

Demonstrates the single highest-leverage idea from the research: **model tiering by role + a structurally distinct, stronger-model reviewer that scores against an explicit rubric instead of free-form judgment.**

This is `ceiling-designer`'s pipeline shape, deliberately slimmed down and with the two gaps the research identified explicitly filled in:

1. Model tier is pinned per agent (see table below) — cheap for drafting, expensive reserved for the two steps that are irreversible or expensive to unwind.
2. The Reviewer is instructed to score a fixed rubric dimension-by-dimension (not "does this look right"), which the self-preference-bias literature shows cuts same-family bias by ~31.5%.

## Pipeline

```
Speccer (cheap) → Implementer (mid, TDD) → Reviewer (expensive, rubric-scored) ⇄ Implementer (max 4 rounds)
```

No separate Researcher/Planner stage here — this example is deliberately the *minimal* version of the tiered pattern. For a feature that genuinely needs research/planning, bolt `ceiling-designer`'s Researcher+Planner stages back in front, keeping the same tiering logic (Researcher = cheap, Planner = expensive, human-gated).

## Model tiering (the actual point of this example)

| Agent | Model | Why this tier |
|---|---|---|
| Speccer | Haiku-tier | Interviewing + writing a spec is drafting work; no irreversible decision is made here. Cheap model, cheap mistakes to catch. |
| Implementer | Sonnet-tier | Mid-complexity generation, but self-correcting via the TDD red-green-refactor loop — the tests themselves are the tool-grounded check, so the model doesn't need to be the strongest available. |
| Reviewer | Opus-tier (or whatever your strongest available model is) | The one step that's expensive to get wrong — a bad review either ships a defect or triggers a wasted correction round. This is where the research says to spend the money: verification is empirically cheaper to do well than generation, so a strong model spent *here* buys more correctness per dollar than the same spend on the Implementer. |

If you're running this in Claude Code with the Task tool, set each subagent's model in its frontmatter (see each agent file's `model:` field) rather than running the whole pipeline on one model throughout.

## Why the Reviewer is separate from the Implementer's self-check

The research is specific here: same-model self-review carries measurable self-preference bias, and a model that's mediocre at *generating* a correct solution can still be quite good at *judging* one (a documented gap of ~15% generation accuracy vs. ~85% verification accuracy in one study). So:

- The Implementer still self-checks before handoff (cheap, catches the obvious stuff) — see its output template.
- The Reviewer is a **separate invocation, on a stronger model, using a rubric** (`docs/features/[name]/review-rubric.md` is scored dimension-by-dimension with a verdict per line, not a paragraph of prose) — this is the part neither `ceiling-designer` nor `purrfect-blocks` does today.

## Bounded loop

Capped at **4 rounds** (slightly tighter than ceiling-designer's 5, since this pipeline has no Planner-stage human gate to catch a wrong direction early — tightening the loop compensates). On round 4 fail: stop, escalate to the human with the full failure log. This directly implements the MAST finding that centralized validation gates contain error amplification to ~4.4x vs. 17x for uncoordinated retry.

## Files

```
.claude/agents/
├── speccer.md       (Haiku-tier)
├── implementer.md   (Sonnet-tier)
└── reviewer.md       (Opus-tier, rubric-scored)
docs/features/[name]/
├── 1-spec.md
├── 2-implementation.md     (runs appended, never overwritten)
├── review-rubric.md         (the fixed scoring rubric, dimension-by-dimension)
└── 3-review.md               (PASS/FAIL + Review Runs Log, appended per round)
```

## When to reach for this pattern vs. the others in `agent-smith`

Use this when: the task is a single well-scoped feature, you know the shape of the work upfront (no research/architecture uncertainty), and you want the cheapest pipeline that still gets independent, high-quality verification. If there's real architectural uncertainty, use `ceiling-designer`'s full 5-stage version instead — don't strip the Researcher/Planner stages just to save money if the risk of picking the wrong approach is real.
