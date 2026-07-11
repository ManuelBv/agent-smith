# Example 1 — Tiered-Model Sequential Pipeline with Independent Reviewer

Demonstrates the single highest-leverage idea from the research: **model tiering by role + a structurally distinct, checklist-driven reviewer that re-verifies independently instead of trusting the implementer's self-check.**

This is `ceiling-designer`'s actual pipeline shape — Full and Short routes, plan/progress tracking, and per-feature numbered briefs — now with real model tiers assigned to each agent (ceiling-designer's own agent files didn't originally pin a model per role; this example adds that).

**Canonical agent files**: `.claude/agents/tiered/` at the repo root (`speccer.md`, `researcher.md`, `planner.md`, `implementer.md`, `reviewer.md`). This README explains the rationale; the root folder is what you actually copy into a project or reference via the `tiered` shorthand.

## Pipeline

Two routes, chosen by the Speccer based on the size/risk of the feature:

```
Full pipeline (new interaction pattern, architectural decision, unknown territory):
Speccer (mid) → Researcher (expensive) → Planner (mid) [human approval] → Implementer (mid, TDD) → Reviewer (expensive, checklist) ⇄ Implementer (max 5 rounds)

Short pipeline (small, well-understood feature):
Speccer (mid) → Implementer (mid, TDD) → Reviewer (expensive, checklist) ⇄ Implementer (max 5 rounds)
```

The Speccer decides which route a feature takes and states its rationale explicitly in the spec output — this routing logic is not optional or example-specific, it's how ceiling-designer actually works.

## Model tiering

| Agent | Model | Why this tier |
|---|---|---|
| Speccer | Sonnet-tier | Interviewing + writing a spec is drafting work with real judgment calls (DDD terms, BDD scenario completeness, routing decision) — mid-tier balances quality against the fact that mistakes here are still cheap to catch downstream. |
| Researcher | Opus-tier | Only runs on the Full route, for features with genuine architectural/technical uncertainty — the quality of the options surfaced here directly bounds the quality of the Planner's decision, so it's worth the strongest available model. |
| Planner | Sonnet-tier | Chooses the approach, but a human explicitly reviews and approves the choice before it proceeds — the human-in-the-loop gate is the real safety net here, so mid-tier is sufficient. |
| Implementer | Sonnet-tier | Mid-complexity generation, but self-correcting via the TDD red-green-refactor loop — the tests themselves are the tool-grounded check, so the model doesn't need to be the strongest available. |
| Reviewer | Opus-tier (or whatever your strongest available model is) | The one step that's expensive to get wrong — a bad review either ships a defect or triggers a wasted correction round. This is where the research says to spend the money: verification is empirically cheaper to do well than generation, so a strong model spent *here* buys more correctness per dollar than the same spend on the Implementer. |

If you're running this in Claude Code with the Task tool, set each subagent's model in its frontmatter (see each agent file's `model:` field in `.claude/agents/tiered/`) rather than running the whole pipeline on one model throughout.

## Why the Reviewer is separate from the Implementer's self-check

The research is specific here: same-model self-review carries measurable self-preference bias, and a model that's mediocre at *generating* a correct solution can still be quite good at *judging* one — this is the generation/verification asymmetry finding the literature documents (magnitude varies by benchmark; treat it as directional, not a fixed ratio). So:

- The Implementer still self-checks before handoff against Farley's 8 quality gates (cheap, catches the obvious stuff) — see its output template.
- The Reviewer is a **separate invocation, on a stronger model**, doing a full independent re-check every run against a fixed checklist (BDD scenarios, DDD ubiquitous language, the same 8 Farley gates, type-checking/linting, tests, accessibility, performance) — not trusting the implementer's self-assessment. This is ceiling-designer's actual review discipline, not an invention of this example.

Note: ceiling-designer's real Reviewer produces PASS/FAIL checklist results with a written summary per item, not a numeric dimension-by-dimension score. If you want rubric-style numeric scoring instead of a checklist, see the `dual-review` pattern's rubric format — don't conflate the two; this pattern's Reviewer is checklist-based by design, matching its source.

## Bounded loop

Capped at **5 rounds** (matches ceiling-designer's actual cap). On round 5 fail: stop, escalate to the human with the full failure log. This follows multi-agent failure-mode research showing that centralized validation gates contain error amplification far better than uncoordinated retry loops left to run unchecked.

## Docs structure

Matches ceiling-designer's actual structure — this pattern is not just a list of agents, it's a doc-tracking discipline:

```
docs/
├── general/
│   ├── plan.md        ← architecture decisions, tech stack, step list — written collaboratively first session
│   └── progress.md    ← live step tracker — read at the start of every session, updated after each confirmed feature
└── features/
    └── YYYY-MM-DD-NNN-feature-name/
        ├── 1-speccer-output.md      ← spec: domain terms + BDD scenarios + pipeline routing decision
        ├── 2-researcher-output.md   ← options + comparison table (Full pipeline only)
        ├── 3-planner-output.md      ← chosen approach + implementation plan + Review Runs Log (Full pipeline only)
        ├── [2 or 4]-implementer-output.md  ← implementation notes, runs appended (never overwritten)
        └── [3 or 5]-reviewer-output.md      ← checklist results, PASS/FAIL per run
```

Files are updated in place — new sections appended per run, never overwritten. On the Full pipeline the Review Runs Log lives in `3-planner-output.md`; on the Short pipeline (no Planner stage) it's appended directly in the reviewer-output file.

## Files

```
.claude/agents/tiered/    (canonical — at repo root, not duplicated here)
├── speccer.md       (Sonnet-tier)
├── researcher.md    (Opus-tier, Full pipeline only)
├── planner.md       (Sonnet-tier, Full pipeline only, human-approval gate)
├── implementer.md   (Sonnet-tier)
└── reviewer.md       (Opus-tier, checklist-based)
```

## When to reach for this pattern vs. the others in `agent-smith`

Use this when: you're building out a project's core feature pipeline and want independent, high-quality verification with real progress tracking across sessions — this is a general-purpose sequential build pattern, not a stripped-down toy version. The Speccer's Full/Short routing decision already handles the "is this simple enough to skip research/planning" judgment call per feature, so you don't need a separate minimal variant — use the same pattern for both small and large features and let the routing logic do the work.
