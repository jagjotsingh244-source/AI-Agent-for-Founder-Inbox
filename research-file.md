# Research & Design Notes — AI Agent for Founder Inbox Triage

**Project:** Week 1, AI-Native Engineering Sprint (Antern)
**Author:** Jagjot
**Working title:** AI Agent That Helps Founders Triage Cold Inbound

---

## 1. Problem Statement

Founders and CEOs receive a constant stream of unsolicited inbound messages — people looking for jobs, freelancers pitching gigs, investors/partners reaching out, and spam/promotional content. Manually triaging this is a time sink. This agent classifies incoming messages, evaluates the legitimate ones against the founder's actual needs, and routes only the high-value cases to a human reviewer — saving reviewer time without silently losing good opportunities.

---

## 2. Taxonomy

Every incoming message is first classified into one of three categories:

1. **Job-seeking / freelance gig** — someone pitching themselves as a candidate or freelancer
2. **Investment / partnership** — someone pitching a deal, funding, or collaboration
3. **Spam / promotion** — unsolicited sales, marketing, or irrelevant outreach

This classification is probabilistic, not a hard rule — the LLM outputs a probability distribution over the three categories rather than a single label. Ambiguous cases (e.g., a message that could be a partnership pitch or freelance pitch) are handled by taking the highest-probability category, with low-confidence cases (high entropy across categories) optionally flagged for human triage rather than forced into one bucket.

---

## 3. Two-Step Scoring Flow (per category)

Each category follows the same two-step shape, with different evidence and different cost asymmetries.

**Step 1 — Initial fit probability.** The LLM scores the message against the founder's current stated needs (open roles, freelance project needs, investment thesis, or "we don't want to see this" for spam), using whatever evidence is attached (resume, GitHub link, deal terms in the email, fund/portfolio info).

**Step 2 — Adjustment against past human-reviewed data.** The initial probability is adjusted by comparing the candidate/sender's profile against previously human-reviewed cases in the same category:
- If it resembles profiles a human previously **called for an interview / progressed**, the probability is adjusted upward.
- If it resembles profiles a human previously **rejected / disqualified**, the probability is adjusted downward.

This is a lightweight Bayesian update: the Step 1 score acts as a prior, and similarity to past labeled outcomes acts as the likelihood adjustment, producing a posterior fit probability.

**Step 3 — Threshold decision.** Based on the posterior probability, the agent either routes the message to human review or auto-disqualifies it. Every human review decision (call for interview / progress talks / disqualify) is logged back into that category's historical dataset, so the model improves over time.

---

## 4. Evidence Sources by Category

| Category | Step 1 evidence | Notes |
|---|---|---|
| Job-seeking / freelance | Resume text (pasted or PDF) + GitHub profile/repos | GitHub is used to verify skills claimed in the resume actually show up in shipped work |
| Investment / partnership | Message content (fund name, deal terms, thesis fit) — external lookup (fund site, past portfolio) is a stretch goal, not Week 1 scope | No resume/GitHub equivalent exists; evidence is mostly self-reported in the message itself |
| Spam / promotion | Message content only | No external evidence needed — pattern-matching against known spam signals is sufficient |

**Why LinkedIn is not scraped:** LinkedIn has no public API for messages or profile data, and scraping violates their ToS and triggers bot detection. Instead, the agent asks senders to paste their LinkedIn "About"/experience text directly, or submit a resume — same signal, no ToS risk.

**Why LinkedIn DMs aren't ingested automatically:** there is no legitimate API for reading a personal LinkedIn inbox. Realistic Week 1 intake is either (a) email via Gmail/Outlook API (legitimate OAuth-based access), or (b) a simple submission form where senders paste message text + links + resume. This sidesteps the "read someone's private inbox" problem entirely.

---

## 5. Cost Asymmetry (per category)

This is the most important design principle in the system: **the cost of a false negative and a false positive are not equal, and they're not equal in the same direction across categories.**

| Category | Costly mistake | Threshold implication |
|---|---|---|
| Job-seeking / freelance | Auto-disqualifying a genuinely good candidate (silent loss — no one ever notices) | Disqualify threshold should be conservative (high bar to disqualify); when uncertain, lean toward human review |
| Investment / partnership | Same as above — missing a real investor/partner is costly and hard to recover from | Same conservative disqualify threshold |
| Spam / promotion | Sending real spam to a human reviewer repeatedly (wastes the exact time this agent is meant to save) | Disqualify threshold can be more aggressive; when uncertain, lean toward auto-disqualify rather than escalate |

In other words: for job-seeking and investment categories, the system should be biased toward "when in doubt, ask a human." For spam, it should be biased toward "when in doubt, filter it out." This is not a single global threshold — each category needs its own calibrated cut-off reflecting its own cost table.

---

## 6. Data Staleness / Recency Handling

Human reviewer priorities change over time (open roles close, investment thesis shifts, hiring bar moves). Feeding all historical human decisions into the posterior update with equal weight risks anchoring the agent to outdated judgments.

**Week 1 approach (deliberately simple):** apply either
- a **recency window** — only use human decisions from the last N weeks / last N decisions per category, or
- **time-decay weighting** — recent decisions are weighted more heavily than older ones in the similarity-based adjustment step.

A recency window is simpler to implement and reason about for Week 1; time-decay weighting is a natural next iteration.

---

## 7. Known Limitations (Deliberately Deferred)

These are real problems worth naming explicitly rather than solving in Week 1:

- **Cold-start problem.** With zero or few human-reviewed examples, the Step 2 adjustment has nothing meaningful to compare against. Early on, the agent should rely primarily on Step 1 (static rubric) and only blend in the learned adjustment once a minimum sample size per category exists.
- **Selection bias in auto-disqualify.** The feedback loop only receives ground truth for candidates who reached a human reviewer. If the agent's Step 1 is systematically wrong about some profile type, it will never learn this, because those candidates never generate a labeled outcome. Mitigation (deferred): periodically route a small random sample of low-probability cases to human review anyway, purely to get ground truth on the disqualify bucket (an explore/exploit tradeoff).
- **Label noise from human reviewers.** "Called for interview" or "progressed to talks" is not a clean ground-truth signal — reviewers reject good candidates for reasons unrelated to fit (role filled, timing, gut feel, bias). The agent is really learning to match a specific reviewer's judgment, not some objective "true fit."
- **Proxy signals, not causal ones.** GitHub activity and resume content are proxies for skill, not guarantees. With small sample sizes the model can learn spurious correlations. This should be stated plainly rather than treated as solved.
- **Identity verification.** No mechanism confirms a GitHub link or resume actually belongs to the message sender.
- **Resume/PDF parsing failures.** Scanned resumes, unusual formatting, and tables can break text extraction — a common, boring failure mode worth testing against.
- **Multi-role scoring.** If multiple roles/projects are open at once, the agent needs to either score against each and pick best fit, or this becomes a genuinely harder multi-class matching problem — not addressed in Week 1.
- **Adversarial adaptation.** If senders learn what the filter rewards (certain keywords, GitHub activity patterns), signal quality degrades over time — a known phenomenon in any observable filtering system.
- **Fairness/compliance.** If this were ever used for real hiring decisions rather than as a class project, automated candidate filtering intersects with algorithmic hiring bias regulation (e.g., NYC Local Law 144 requires bias audits for automated hiring tools). Not a Week 1 concern, but worth acknowledging deployment isn't purely a technical problem.

---

## 8. Evaluation Plan

- Build a labeled set of ~30–50 example messages spanning all three categories (job-seeking, investment/partnership, spam), including deliberately ambiguous edge cases.
- **Classification metrics:** accuracy, precision, recall, confusion matrix — per category, since costs differ per category (see Section 5).
- **Calibration:** bucket predictions by stated confidence and check whether actual accuracy matches (e.g., do "0.87 confidence" predictions turn out correct ~87% of the time).
- **Entropy per decision:** high entropy (agent is genuinely unsure across categories, or unsure fit vs. no-fit) should be a signal to route to human review rather than force a decision.
- **Information gain from GitHub signal:** compare entropy of the fit decision using text-only vs. text + GitHub data, to quantify how much the additional evidence actually sharpens the decision (mutual information between GitHub signal and human-reviewed ground truth).

---

## 9. Week 1 Scope Decision

**Built this week:**
- Intake (form-based or Gmail API)
- Three-category classifier (probabilistic output)
- Step 1 fit scoring for job-seeking category (resume + GitHub)
- Evaluation: accuracy, precision/recall, confusion matrix, entropy, calibration

**Designed but deferred (documented here, not implemented):**
- Step 2 Bayesian adjustment against historical human decisions (needs real usage data to be meaningful — cold-start problem)
- Recency/decay weighting for stale data
- Investment/partnership and spam categories' full evidence-gathering pipelines
- Random-sampling correction for auto-disqualify selection bias
