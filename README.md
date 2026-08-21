# AI Agent for Founder Inbox

Classifies inbound startup emails by intent (customer, investor, candidate, partner, other), then scores priority through a two-stage pipeline: a per-category credibility prior, followed by a human-defined requirement match that updates that prior. Low-confidence classifications route to a human for correction before scoring continues.

## Current state

- [ ] Stage 1 intent classification (LLM, per-category prompts)
- [ ] Confidence check + human review routing
- [ ] Stage 1 credibility scoring (prior) — per category
- [ ] Stage 2 requirement matching (human rubric) — per category
- [ ] "Other" category relevance scoring (single-pass, no stage 2)
- [ ] Unified ranked inbox view

Mark items as they're built. Don't leave this list implying more is done than is.

## How it works

1. **Intent classification.** An LLM call reads the email and predicts one of: customer, investor, candidate, partner, other — along with a confidence signal (margin between top two predicted categories).
2. **Confidence check.** If the margin is low, the email goes to human review. The human assigns the correct category and can supply category-specific signals the model missed. If the margin is high, the predicted category is used as-is.
3. **Category routing.**
   - **Other** → single relevance score, no further scoring. Shown in the ranked view but not run through stage 2.
   - **Customer / investor / candidate / partner** → stage 1 credibility score computed from category-specific signals (this is the prior).
4. **Stage 2 requirement match.** For non-"other" categories, the email is checked against human-defined requirements for that category. Matches/mismatches update the stage 1 prior into a final priority score.
5. **Unified view.** All categories — including "other" — are shown together, ranked by priority score.

See `research.md` for why the pipeline is shaped this way.

## Setup

```bash
# fill in once the project has real setup steps
```

## Configuration

Per-category credibility signals and stage 2 requirements are human-defined, not learned. They live in [wherever you decide to put them — config file, DB table, etc.]. Document the exact format here once it's settled; this is the part someone other than you will need to edit without touching code.

## Known limitations

- Confidence-check threshold (what counts as "low margin") is not yet calibrated against real data.
- "Other" category gets a single relevance pass — its score scale needs to be validated against the two-stage score on named categories, or it will systematically over- or under-rank against them.
- Human review currently only feeds stage 1 signals. It does not influence which stage 2 rubric gets applied, even though a human category correction is itself a signal.
- No eval framework yet — see `research.md`.
