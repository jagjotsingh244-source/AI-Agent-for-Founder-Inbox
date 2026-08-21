# Research

## Problem framing

"Priority" is not a single dimension — it collapses urgency, importance (sender stakes), and actionability. This project scores it in two stages rather than one composite pass, so the components stay legible: a credibility prior per category, then a requirement match that updates it.

## Design decisions

**Two-stage scoring instead of one LLM pass per email.**
A single classify-and-score call is cheaper but conflates "is this a credible sender" with "does this meet our bar." Splitting them lets each stage use signals suited to it — credibility signals differ wildly by category (investor: firm recognition, check-size language; candidate: domain match to claimed experience) while requirement matching is the same mechanical check across categories: does content satisfy a human-set rubric.

**Stage 1 → Stage 2 framed as prior → evidence, not arbitrary point adjustment.**
Considered: stage 2 as a flat +/- adjustment to the stage 1 score. Rejected — arbitrary adjustment weights require hand-tuning with no principled way to compare across categories. Framing stage 1 as a Bayesian prior and stage 2 as evidence that updates it gives a defensible way to combine them and makes the scoring auditable later.

**Confidence-gated human review instead of always-auto or always-human classification.**
Full automation risks silent miscategorization cascading into the wrong credibility model and wrong rubric. Full human review doesn't scale. Gating on classification margin (low margin = ambiguous = review) targets human attention at the cases where the model is actually uncertain, not just where it might be wrong.

**"Other" category skips stage 2 entirely.**
Considered: applying a generic requirement rubric to "other" as well. Rejected for now — "other" is a catch-all with no natural per-category rubric, and forcing one risks inventing false structure. It gets a lighter single-pass relevance score instead, on the assumption these emails are lower-priority by default but shouldn't be invisible.

## Open questions

- **Confidence threshold.** What margin between top two predicted categories counts as "low confidence" and routes to human review? No data-driven answer yet — needs to be calibrated once real emails are running through the classifier.
- **Score comparability across branches.** The "other" branch produces a single-pass relevance score; named categories produce a prior updated by evidence. These need to land on a comparable scale, or the unified ranked view will systematically favor one branch over the other regardless of actual priority.
- **Should human review corrections inform stage 2 rubric selection?** Currently a human-corrected category only affects stage 1 signal selection. It's plausible a human correction should also flag that rubric applies differently for this sender — not yet decided.
- **Requirement representation.** Freeform text requirements give the stage 2 LLM more nuance but are harder to eval and harder to explain to the founder when it scores something oddly. Structured checklists are more auditable but lose nuance. Not yet decided which to use, or whether it should differ by category.

## Eval approach

TBD — needs a labeled seed set before this can be meaningfully evaluated. Options to weigh once labeling starts: founder manually tags a seed batch vs. bootstrapping from heuristics (sender domain reputation, known VIP list) and refining thresholds from review-stage corrections over time.
