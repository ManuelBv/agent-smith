# Example 2 — Parallel Fan-out/Fan-in Research with Tiered Synthesis

Demonstrates Anthropic's own production pattern for their Research feature (Opus lead, Sonnet workers — 90.2% improvement over single-agent Opus on their internal eval) applied at solo-developer scale: **cheap workers fan out on independent angles, an expensive model synthesizes and is the only one that has to be right.**

This is the correct use of parallelism per the research: it wins on latency and coverage (parallel tool-calling cut Anthropic's own research time by up to 90%), not on hallucination-reduction — that's still the synthesis/verification step's job.

## Pipeline

```
Lead (expensive) — decomposes the question into 3-5 independent angles
    │
    ├── worker-1 (cheap) ─┐
    ├── worker-2 (cheap)  ├─→ Synthesizer (expensive) — cross-checks, resolves contradictions, cites sources
    ├── worker-3 (cheap)  │
    └── worker-N (cheap) ─┘
```

## When this pattern wins (and when it doesn't)

**Wins when**: the subtasks are genuinely independent — no worker's findings change what another worker should search for. Research angles on the same question (e.g. "performance", "accessibility", "bundle size", "existing prior art") are a clean fit — this is exactly `purrfect-blocks`' `architecture-researcher` + `legal-researcher` pairing, generalized to N workers.

**Fails when**: the "angles" are only superficially independent and actually need each other's evolving state — PlanCraft's benchmark showed up to 70% degradation forcing coordination onto genuinely sequential work. If you find a worker needs to know what another worker found before it can proceed, this is not fan-out — it's a sequential dependency wearing a parallel costume. Don't force it.

## Model tiering — the actual point of this example

| Role | Model | Why |
|---|---|---|
| Lead | Opus-tier | Decomposition quality determines everything downstream — Anthropic's own incident log includes the lead spawning 50 subagents for a simple query, so the decomposition step needs judgment, not just volume. Also the only agent that needs to know when to STOP (a MAST-identified failure category: agents not recognizing sufficient results already exist). |
| Workers (1..N) | Haiku or Sonnet-tier | Each covers one narrow, bounded angle. Cheap model is fine because each worker's job is breadth of search, not synthesis judgment — and a bad worker finding gets caught at synthesis, not silently trusted. |
| Synthesizer | Opus-tier | Must cross-reference contradictions across workers, weight by source authority, and decide what's actually true — this is the step where the generator-verifier asymmetry finding applies again: judging which of N conflicting claims is correct is cheaper and more reliable than having generated all N independently, so put the strong model where the judgment actually happens. |

## Explicit stop conditions (a documented gap in naive orchestrator designs)

The Lead must decide up front, not adaptively mid-run, how many workers a given request warrants — Anthropic's own postmortem names ungoverned scaling as a real failure mode. Use effort-scaling heuristics, stated explicitly in the Lead's prompt:

- Simple fact-finding: 1 worker, 3-10 tool calls
- Comparison across 2-4 options: 1 worker per option, 10-15 calls each
- Broad landscape survey: 4-5 workers max, each on a distinct angle

Do not let the Lead spawn additional workers mid-run just because a query "seems complex" — that's the exact failure Anthropic logged. If the initial decomposition turns out to be insufficient, that itself is a finding to report to the user, not a silent trigger for more spend.

## Grounding requirement

Every worker output must carry per-claim source citations (URL + what it supports). The Synthesizer's job includes a dedicated grounding pass — verifying that every claim in the final synthesis actually traces to a real citation in a worker's raw notes, not just that the synthesis reads plausibly. This mirrors Anthropic's own CitationAgent pattern: separate "can this be checked mechanically" from "is this a good synthesis."

## Files

```
.claude/agents/
├── lead.md          (Opus-tier, decomposes + sets stop conditions)
├── worker.md         (Haiku/Sonnet-tier, one instance per angle, template reused N times)
└── synthesizer.md    (Opus-tier, cross-checks + grounds + resolves contradictions)
docs/
├── decomposition.md   (Lead's angle breakdown + stop-condition reasoning)
├── worker-N-notes.md  (raw findings per worker, with citations)
└── synthesis.md        (final grounded output)
```

## Cost note

This pattern is *not* automatically cheap — Anthropic's own multi-agent research system runs ~15x the tokens of a single chat call. The savings here come specifically from (a) workers being on a cheap tier and (b) each worker's context being scoped to one narrow angle (~3-4k tokens) rather than the full problem context (~15-20k tokens) — not from "parallel is inherently cheap." If your task doesn't actually need the coverage or the wall-clock speedup, a single well-scoped agent call is very likely cheaper AND won't suffer the coordination overhead this pattern pays.
