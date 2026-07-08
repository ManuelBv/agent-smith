# Analysis of Existing Multi-Agent Patterns (ceiling-designer & purrfect-blocks)

Before designing new examples, this documents what Manuel already built and validated in two prior projects. Both live in `neo/apps/`. Read this before designing new agent-smith examples — don't reinvent what already works.

## Pattern 1: Sequential Pipeline with Human Gate (`ceiling-designer`)

**Structure**: 5 agents, strictly sequential, file-based handoff.

```
Speccer → Researcher → Planner [HUMAN APPROVAL GATE] → Implementer ⇄ Reviewer (max 5 rounds)
```

- **Speccer**: interviews the user one question at a time, produces a spec using DDD ubiquitous language (domain terms table) and BDD Given/When/Then scenarios written from the *user's* observable perspective only (no implementation details in acceptance criteria). Also decides pipeline routing: "Full" (needs research+planning) vs "Short" (skip straight to implementer) based on whether the feature is architecturally novel.
- **Researcher**: investigates options against a fixed rubric (performance, browser compat, accessibility, bundle size, stack fit, gotchas) with a strict source-authority hierarchy (official docs > benchmarks > GitHub activity > respected blogs > SO). Explicitly forbidden from making the final call — only the Planner decides. This separation (research ≠ decision) is a deliberate anti-bias device.
- **Planner**: scores options against a priority-ordered rubric, then **stops and asks the user to confirm before any code is written**. This is the single most load-bearing human-in-the-loop checkpoint in the whole system — it exists specifically to prevent the agent from committing to an architecturally wrong direction before expensive implementation work happens.
- **Implementer**: strict TDD (red-green-refactor, never implementation before a failing test). Self-checks against Farley's 8 quality gates (readability, changeability, testability, modularity, cohesion, separation of concerns, abstraction, loose coupling) *before* handing off — this is a cheap self-review pass that catches obvious issues before the expensive independent reviewer runs.
- **Reviewer**: does a **full** re-check every single run (not just a diff of what changed) against BDD scenarios, DDD term usage, the 8 Farley gates independently (does not trust implementer's self-assessment — this is the generator/verifier asymmetry principle), TypeScript/lint, tests, WCAG 2.2, performance. On fail: appends structured failure notes to a "Review Runs Log" section inside the planner's output file (append-only, never overwritten) and hands back to Implementer. **Bounded at 5 rounds** — on round 5 fail, it stops and escalates to the human rather than looping forever.

**Key mechanisms for reducing hallucination/drift**:
1. File-based state (`docs/features/[name]/N-agent-output.md`) instead of one long context — each agent re-reads only what it needs, so errors don't silently compound through context dilution.
2. Independent reviewer that never trusts the generator's self-report (asymmetry: cheap self-check by implementer, then expensive independent re-check by a fresh reviewer invocation).
3. A hard bound on the retry loop (5 rounds) with mandatory human escalation on exhaustion — prevents infinite-loop cost blowup and silent failure.
4. Human approval gate placed at the *cheapest possible point* — after planning, before implementation — so redirection costs are minimal.
5. Domain language (DDD) enforced verbatim in code by the reviewer — reduces spec drift by making the vocabulary itself a checkable artifact, not just prose.

## Pattern 2: Orchestrator + Parallel Fan-out (`purrfect-blocks`)

**Structure**: one Opus-tier orchestrator agent dispatches to specialized subagents via the Task tool. Some subagents run in parallel, others sequentially, chosen per-workflow.

```
[architecture-researcher ∥ legal-researcher] → implementation-planner → implementer (TDD loop) → [code-reviewer ∥ unit-tester]
```

- **Orchestrator**: model = opus (stronger model reserved for coordination judgment, not raw generation). Has only `Read, Write, Task, Glob, Grep` — deliberately cannot edit code itself, forcing all implementation through subagents. Chooses one of 3 modes (Full Feature Workflow / Quick Implementation / Individual Agent direct route) based on request shape, and explicitly parallelizes independent research and independent verification steps in the same message (multiple Task calls at once).
- **architecture-researcher**: model = sonnet (cheaper than orchestrator). Tools restricted to `Read, Grep, Glob, WebSearch` — cannot write code, only research output. Forced to run 4-8 distinct web searches across different angles before synthesizing, explicitly told not to stop at one result.
- **code-reviewer**: outputs strict JSON schema (severity levels, approval/changes_requested gate) rather than prose — makes the review machine-parseable and forces the reviewer to commit to a binary gate decision rather than hedging.
- Parallel dispatch is used specifically where subagents are **independent of each other's output** (two different research angles; two different verification angles) — never for steps with a data dependency.

**Key mechanisms**:
1. Model tiering by role: opus for orchestration/judgment, sonnet for research/generation — cost control by matching model strength to task difficulty, not using the biggest model everywhere.
2. Tool-scoping per agent (principle of least privilege) — the orchestrator literally cannot write code, so all code changes are forced through an agent whose entire job is to follow TDD.
3. Structured (JSON) review output with an explicit approval gate, rather than free-form prose that can be misread as "probably fine."
4. Parallel dispatch is a first-class decision the orchestrator makes explicitly ("Run independent agents in the same message with multiple Task calls") — not implicit.

## Cross-cutting techniques used in both

- **CLAUDE.md-level ground rules** (both projects): forbid unsubstantiated quality claims ("fully functional", "high quality") without measured evidence (build output, test results); require explaining *why* something works, not just asserting it.
- **TDD as a structural anti-hallucination device**: forcing a failing test before implementation code exists is itself a grounding mechanism — it converts a fuzzy spec into an executable, falsifiable check before generation happens.
- **Skills** (`.claude/skills/` or `.agents/skills/`) encode reusable procedural knowledge (TDD loop mechanics, browser automation, session archival) outside of any single agent definition, so multiple agents/pipelines can invoke the same verified procedure instead of re-deriving it.
- **Memory files** (`.agents/memory/*.md`) separate stable identity/context (who Manuel is, project history) from ephemeral working state (current feature, review run count) — avoids re-deriving context every session while keeping it out of the hot path of every agent invocation.

## Pattern 3: Lean 2-Agent Research→Synthesis Pipeline (`neo/chats/ai-general`)

A minimal, non-coding example worth noting because it's the simplest possible instance of generator/synthesizer separation, and it's used for research-writing rather than software:

```
research-agent → response-agent
```

- **research-agent**: parses the request via CTFCV (Context/Task/Format/Constraints/Verification), then executes a structured source-prioritization protocol (primary academic/official > peer-reviewed > authoritative secondary > verified tertiary), runs 3-5 parallel search queries per topic, cross-references every claim against 2+ sources, and explicitly flags contradictions rather than silently picking one. Every source gets a confidence rating (H/M/L) and every finding must carry a citation. Explicitly forbidden from fabricating sources — must say "no data found" rather than guess.
- **response-agent**: takes the research-agent's raw notes and synthesizes them into a final document, but has its own quality checklist run *before* finalizing (all claims cited, quantitative conclusions show working, limitations explicitly stated, no fabricated data). Uses hedged epistemic language by rule ("evidence suggests" not "it is proven").

**Why this matters for agent-smith**: it's the cleanest illustration that the generator/verifier split isn't specific to code — the same "one agent produces, a second independently checks against a fixed rubric before anything is called done" structure applies to research synthesis too. It's also the cheapest possible multi-agent system (2 agents, no loop, no orchestrator) — useful as a minimal baseline example against the heavier 5-agent and orchestrator patterns.

## Open questions for agent-smith to answer via research

- Is 5 the right bound for correction rounds, or should it be adaptive (e.g. stop early if the last 2 rounds show no delta)?
- Both examples use single-model-family review (Claude reviewing Claude). Is cross-model review (e.g. a cheaper/different model as adversarial critic) meaningfully better at catching same-family blind spots?
- Neither example does explicit self-consistency/voting (N parallel generations + majority vote) — is that worth the cost for the highest-stakes steps (e.g. the Planner's architecture choice)?
- Where exactly does the $8-10 vs $100 cost delta come from in practice — model tiering, shorter per-call context, or avoiding wasted full-regenerations from a single giant one-shot attempt that goes wrong?
