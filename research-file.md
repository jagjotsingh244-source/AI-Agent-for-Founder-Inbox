# Research

## Problem framing

Startups receive a large volume of emails from potential candidates, investors, partners, customers, and other sources. Manually reviewing these emails requires significant time and often involves researching the sender, verifying their claims, and determining whether the opportunity is actually relevant to the startup.

The problem is not simply classifying emails. The system needs to determine which emails deserve human attention and in what order.

Different categories require different signals to evaluate credibility. For example, a job candidate may require verification through their resume, LinkedIn, GitHub, and relevant experience, while an investor may require verification of their fund, investment history, and investment focus.

Even when an email is credible, it may not be relevant to the startup’s current needs. Therefore, the system must separate credibility/evidence evaluation from requirement matching.

## Design decisions

<img width="1472" height="1836" alt="image" src="https://github.com/user-attachments/assets/22957aae-f43f-42da-8e18-0f60885b8d33" />



## Eval approach

TBD — needs a labeled seed set before this can be meaningfully evaluated. Options to weigh once labeling starts: founder manually tags a seed batch vs. bootstrapping from heuristics (sender domain reputation, known VIP list) and refining thresholds from review-stage corrections over time.
