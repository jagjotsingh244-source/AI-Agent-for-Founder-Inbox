# Research

## Problem framing

Startups receive a large volume of emails from potential candidates, investors, partners, customers, and other sources. Manually reviewing these emails requires significant time and often involves researching the sender, verifying their claims, and determining whether the opportunity is actually relevant to the startup.

The problem is not simply classifying emails. The system needs to determine which emails deserve human attention and in what order.

Different categories require different signals to evaluate credibility. For example, a job candidate may require verification through their resume, LinkedIn, GitHub, and relevant experience, while an investor may require verification of their fund, investment history, and investment focus.

Even when an email is credible, it may not be relevant to the startup’s current needs. Therefore, the system must separate credibility/evidence evaluation from requirement matching.

## Design decisions

<img width="1472" height="1836" alt="image" src="https://github.com/user-attachments/assets/dc969a6b-ca53-40de-b9f5-fdf99ad5f36a" />

The initial implementation uses email as the primary input because it provides a programmatically accessible stream of incoming messages. LinkedIn messages were considered as an additional input source, but a suitable programmatic retrieval path was not identified during the initial research, so the system currently focuses on email while keeping the input layer extensible.
### Stage 1 — Intent classification
The agent first classifies each email based on its intent. This allows the owner to filter the inbox by category—for example, viewing only job-seeking candidates—and, more importantly, allows the system to apply different evaluation criteria to different types of emails.

The LLM classifies each email into one of five categories:

* Customer
* Investment
* Partnership
* Candidate
* Other

Along with the predicted category, the LLM returns a confidence estimate for each category based on the information in the email.

The cost of an incorrect classification can be high because the classification determines which evaluation process is used in the next stage. For example, an investment email incorrectly classified as a partnership would be evaluated using the wrong signals, potentially leading to an incorrect priority score.

To reduce the impact of uncertain classifications, the system uses a confidence-margin threshold. It compares the confidence of the highest-probability category with the second-highest category. If the difference is below the configured threshold, the classification is considered ambiguous and is escalated to a human reviewer instead of continuing automatically.

### Stage 2 - Category specific evaluation

Credibility cannot be evaluated using one universal set of questions because each category has different signals ,which can reveal potential value behind the email. Therefore, the agent uses a different set of questions and evidence for each category.

<img width="1472" height="3040" alt="image" src="https://github.com/user-attachments/assets/f262b698-cb0f-4a13-b9bf-a81338558f7f" />


When the LLM looks at one email (say, an investor email), you don't just want it to spit out a number like "0.72." You want to see how it got there — what it checked and what it found with that we can find the reason behind the low score.A simple binary yes or no per question will hide all the details. I created this structure to solve this problems

#### Piece 1: question_evaluations 
This is how llm will treat each question.It shows one entry per question that we gave it (from the list we wrote for candidate/investor/partner/customer).

For each question, it fills in three things:
```
  {"question": "which question this is answering",
  "finding": "In one plain sentence what did the model actually notice",
  "direction": "supports, contradicts, or insufficient_evidence"}
```
#### Piece 2: reasoning_summary 
After going through all the questions one by one, the model writes 2-3 plain sentences pulling it all together: "here's the overall picture I'm seeing." This step forces the model to actually think about the whole picture before locking in a number, instead of jumping straight to a score.

#### Piece 3: credibility_score  
The actual number (e.g., 0.72) that comes after all the reasoning above it. Because it's generated last, it's actually influenced by everything that came before it in the same response — not a guess made in isolation.

Overall structure
```
{
  "question_evaluations": [
    {
      "question": "Does the claimed fund exist as a real, active investment entity?",
      "finding": "Email claims 'Ridgeline Capital', a seed-stage fund. No verifiable public presence found in available context.",
      "direction": "contradicts",
    },
    {
      "question": "Is the sender's role at that fund verifiable?",
      "finding": "Sender claims 'Principal' title. Nothing in the email itself can confirm this without external lookup.",
      "direction": "insufficient_evidence",
    }
  ],
  "reasoning_summary": "The core identity claim — that Ridgeline Capital is a real, active fund — could not be verified, which is the single most load-bearing fact for an investor email. Absent that, downstream claims about check size and sector fit can't be meaningfully assessed even though they sound plausible on their own.",
  "credibility_score": 0.15
}
```


***Why the order matters:*** the model writes this top to bottom, in one shot. If you asked for credibility_score first, it would have to pick a number before explaining itself — meaning the explanation would just be an excuse for a number it already guessed, not the actual reasoning that produced it. By putting the questions and reasoning first, the score is forced to follow logically from what's above it.

Credibility score will act as are initial priority score on scale of 0-100

### Stage 3 - Requirement matching

This stage takes the requirements and preferences defined by the human and evaluates whether the sender or opportunity satisfies them.

The result of this comparison can have a positive or negative impact on the initial priority score from Stage 2.

Not all requirements are treated equally because some requirements are critical while others are simply preferences.

* Must-have → Failure to satisfy a must-have should significantly decrease the priority while satisfying it increases the priority.
* Preferred → Satisfying it increases priority, while failing to satisfy it has a smaller negative impact.
* Nice-to-have → Small impact. Satisfying it provides a small positive boost.
* Disqualifier → Largest negative impact. If a disqualifying condition is confirmed

Lets talk about its mathematical aspect 

A simple point-based model (start at a base score, add/subtract points per requirement) was the initial approach but was rejected for two reasons:

*Unbounded stacking. Multiple failed must-haves would subtract independently with no natural ceiling, meaning enough accumulated penalties could numerically out-rank a genuine disqualifier — inverting the priority ordering the tier system is meant to enforce.
*No principled connection to Stage 2. Stage 2's output is a probability (credibility score). An additive point system operating on that probability directly has no consistent interpretation — is +10 points the same "strength of evidence" at 90% confidence as it is at 20%? Probabilities compress near their bounds (0 and 1), so fixed-size adjustments behave inconsistently depending on where the score currently sits.

Log-odds resolves both. Odds (p / (1-p)) are the natural unit for combining independent pieces of evidence — this is the same math underlying Bayesian updating, which the Stage 2 → Stage 3 pipeline is already implicitly structured around (prior → evidence → posterior). Taking the log of the odds converts evidence-combination from multiplication into addition, which is what makes a "sum of deltas" model principled rather than ad hoc. It also produces automatic saturation: once a score is pushed toward an extreme, further evidence in the same direction has diminishing effect — solving the stacking problem without a hand-coded floor or cap.

Tier deltas (log-odds units):

Tier	Satisfied	Failed	Unable to verify
Must-have	+0.5	−3.0	−0.5
Preferred	+0.7	−1.0	0
Nice-to-have	+0.3	0	0
Disqualifier	—	gate (see below)	—

## Eval approach

TBD — needs a labeled seed set before this can be meaningfully evaluated. Options to weigh once labeling starts: founder manually tags a seed batch vs. bootstrapping from heuristics (sender domain reputation, known VIP list) and refining thresholds from review-stage corrections over time.
