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

When the LLM looks at one email (say, an investor email), you don't just want it to spit out a number like "0.72." You want to see how it got there — what it checked and what it found with that we can find the reason behind the low score. That's why I created this structure.



## Eval approach

TBD — needs a labeled seed set before this can be meaningfully evaluated. Options to weigh once labeling starts: founder manually tags a seed batch vs. bootstrapping from heuristics (sender domain reputation, known VIP list) and refining thresholds from review-stage corrections over time.
