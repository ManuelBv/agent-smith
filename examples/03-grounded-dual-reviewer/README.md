# Example 3 — Grounded Dual-Reviewer Voting (narrow scope, hard grounding fallback)

Demonstrates the one pattern genuinely missing from `ceiling-designer` and `purrfect-blocks`: **two independent reviewer instances voting on narrow, specific acceptance criteria — used to surface disagreement, not to invent truth.**

This directly implements the research's most-hedged recommendation. Debate/voting among LLM agents is real but risky: the foundational 2023 debate paper showed gains on math/reasoning tasks, but newer work documents two failure modes — **problem drift** across rounds, and **confident convergence on a shared wrong answer** when neither participant has grounding to correct the other. The mitigation the research lands on: keep the scope narrow (one specific, checkable claim per vote — not "is this whole feature good"), cap it at one round (no back-and-forth debate, just independent parallel verdicts), and **always fall back to a tool-grounded check or a human on disagreement** rather than letting the two reviewers argue to resolution.

## Pipeline

```
Implementer → [Reviewer-A ∥ Reviewer-B] (independent, parallel, same rubric, no communication)
                        │
                  Agreement?
                   ├── YES → verdict stands
                   └── NO  → escalate to tool-grounded check if one exists,
                             else escalate to human — never let A and B debate it out
```

## Why this is NOT open-ended debate

The two reviewers:
- Never see each other's output before submitting their own verdict (true independence, not sequential "here's what the other reviewer said, do you agree?")
- Score the **same fixed, narrow rubric row** — e.g. "does this scenario's Given/When/Then hold, yes or no" — not an open-ended "is this good" judgment
- Get **exactly one round each**. If they disagree, the system does not ask them to argue — it escalates. This is the deliberate fix for the drift/confident-convergence failure modes: unbounded back-and-forth is where those failures happen, not in a single independent pass.

## When to use this vs. a single Reviewer (Example 1's pattern)

Use dual-reviewer voting specifically for **the highest-stakes, most ambiguity-prone verdicts** — e.g., "does this satisfy WCAG 2.2 criterion X" or "does this scenario's acceptance criteria actually hold" where a single reviewer's mistake is expensive and the judgment call is genuinely non-mechanical. Don't apply it to everything — it doubles review cost for a benefit that only shows up when the underlying judgment is actually ambiguous. For anything with a deterministic check available (tests, linters, type-checkers), skip voting entirely and just run the tool — that's strictly cheaper and strictly more reliable than any number of LLM opinions.

## Model choice for the two reviewers

Ideally **A and B are different model families or at least different model tiers**, not two calls to the same model/prompt. The research is specific that same-model review carries measurable self-preference bias; two instances of the identical model/prompt voting on the same rubric mostly just re-samples the same blind spot rather than providing genuine independent perspective. If only one model family is available, at minimum vary the prompt framing meaningfully between A and B (e.g., one reviews "does this pass," the other reviews "find a reason this fails") to reduce correlated error — but note this is a weaker substitute for genuine model diversity, not equivalent to it.

## Disagreement handling — the load-bearing design decision

On disagreement between A and B:
1. **First choice**: if a tool-grounded check exists for the disputed row (run the actual test, the actual axe-core accessibility scan, the actual type-checker), run it and let it decide. This is always preferred over more LLM opinions.
2. **Fallback**: if no tool-grounded check exists (e.g., a genuinely subjective spec-fidelity question), escalate to the human with both verdicts and both reviewers' stated reasoning side by side. Do not have a third LLM "tie-break" — that just adds a third opinion to a disagreement that's already shown the question is genuinely ambiguous; a human call is more honest than manufacturing false consensus.

## Files

```
.claude/agents/
├── reviewer-a.md
├── reviewer-b.md
└── arbiter.md        (only runs the disagreement-handling logic above — never a "third opinion" tie-break)
docs/features/[name]/
├── vote-rubric.md      (the narrow, specific claims being voted on — kept small, on purpose)
├── reviewer-a-verdict.md
├── reviewer-b-verdict.md
└── vote-result.md       (agreement/disagreement + escalation outcome)
```

## Cost note

This pattern costs roughly 2x a single-reviewer pass on the rows it's applied to — which is exactly why it should be scoped narrowly (a handful of high-ambiguity rows) rather than applied to an entire review. Applying full dual-review to everything defeats the cost-tiering logic in Example 1; use it as a scalpel for the specific judgment calls that are genuinely hard, not as the default review mechanism.
