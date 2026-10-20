---
type: literature
status: queued
project:
  - IEEE-IoT
authors: Waldspurger and Weihl
year: "1984"
venue:
doi:
pdf: "[[_pdfs/IEEE-IoT/filename.pdf]]"
tags:
  - project/ieee-iot
  - domain/iot
---

# Paper title: Lottery Scheduling: Flexible Proportional-Share Resource Management

## Why am I reading this?

This paper is particularly important for me because it provides the scheduling foundation on which I can build the beacon-derived allocation mechansim. 

With Jain et al., I learned how to measure fairness. Now I need to understand how a randomized scheduler can "produce" fair resource allocations. 

## Triage

**Problem:**
The authors attempted to a solve a problem involving the allocation of limited computing resources among competing applications with different priorities or resource requirements. 

Conventional priority-based schedulers do not always provide precise control over the relative amount of service each application receives.  


**Main contribution:**
A lotter-based scheduling: a randomized mechanism that allocates resources in proportion to the number of lottery tickets assigned to each participant. 
**Relevance to my work:**

The authors describe the lottery tickets having three properties: > ([[waldspurger.pdf#page=2&selection=42,0,65,39&color=yellow|waldspurger, p.2]])
> Lottery tickets encapsulate resource rights that are abstract, relative, and uniform. They are abstract because they quantify resource rights independently of machine details. Lottery tickets are relative, since the fraction of a resource that they represent varies dynamically in proportion to the contention for that resource. Thus, a client will obtain more of a lightly contended resource than one that is highly contended; in the worst case, it will receive a share proportional to its share of tickets in the system. Finally, tickets are uniform because rights for heterogeneous resources can be homogeneously represented as tickets


Waldspurger and Weihl (1994), Sections 1–2

1. Lottery scheduling provides probabilistic proportional-share allocation using resource tickets.
2. Resource rights are abstract, relative, and uniform. Relative entitlement depends on the total active ticket pool.
3. For a single-winner lottery, $p_i = t_i / sum_j t_j.$ 
4. For n independent identical lotteries: $E[W_i] = np_i$ and $Var(W_i) = np_i(1-p_i)$.
5. The expected number of draws until the first win is $1/p_i$ ; finite waiting times are not guaranteed.
6. For our equal-weight batch allocation of m distinct winners among N eligible devices, the individual inclusion probability is $m/N$, but device outcomes within one epoch are dependent.
7. Our proposed extension is not a new proportional-share lottery: it is a publicly reproducible, bias-resistant instantiation with eligibility commitments and externally verifiable selection.


Observation 1 — Resource rights affect fairness independently of randomness.

Ticket inflation demonstrates that an unbiased lottery can still produce an undesirable allocation if ticket rights are improperly manipulated.

In our system, the corresponding concern is manipulation of the eligible-device set or allocation weights.

Observation 2 — Freezing eligibility is necessary but insufficient.

Our commitment $C_e$ can bind the scheduler to a particular eligibility set before the beacon output becomes known. However, it does not establish that the committed set was legitimate. We still need a defined eligibility policy and evidence sufficient to verify compliance with that policy.

The ticket-currency mechanism illustrates how the original authors separate resource rights across trust boundaries. We do not need to reproduce their currency system.

## Reading notes

### Claims


### Method

### Results / evidence


### Questions and disagreements

## Synthesis

**Takeaway:**

Waldspurger and Weihl (1994), Section 5.1 — Fairness

1. The paper experimentally validates lottery scheduling by comparing intended ticket-allocation ratios with observed execution-rate ratios.
2. Figure 4 measures allocation accuracy across different ticket ratios. Finite randomized scheduling windows produce deviations from expected proportions.
3. Figure 5 studies fairness over shorter time windows, showing that overall proportional allocation can be maintained despite local fluctuations.
4. Lottery fairness means agreement with intended proportional resource rights, not necessarily equal allocation.
5. For our beacon scheduler, compare expected and observed per-device selection probabilities across epochs, evaluate convergence over observation windows, and separately quantify favored-group advantage under adversarial seed grinding.
6. A verifiable beacon should match honest PRNG scheduling in statistical fairness; its distinct contribution is publicly reproducible, manipulation-resistant random selection.

**What I can use:**

**Limitations:**

**Connections:**