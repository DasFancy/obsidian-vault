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






## Reading notes

### Claims
Four important properties  > ([[A_Quantitative_Measure_Of_Fairness_And_D_OCR.pdf#page=5&selection=201,0,217,10&color=important|A_Quantitative_Measure_Of_Fairness_And_D_OCR, p.5]])
> We feel the desired properties should be the following:

1. Population size: Applicable to different numbers of users. In relevant to my work, the example can be drawn as : my network may need 50, 100, or 500 eligible IoT devices.
2. Scale independence: should be unaffected by changing measurement units. I may measure allocations in slots, bytes, or airtime.
3. Boundedness: A normalized index between 0 and 1. Enables a clear comparison between schedulers. 
4. Continuity: Responds smoothly to changes in allocation. Help detects gradual changes in allocation equality.


### Method

### Results / evidence
I should only consider Jain's index as a measure distribution of allocations, it does not establish that the mechanism producing those allocations was honest. It does not provide a model for malicious schedulers, adversarial seed selection, randomness manipulation, or public verification. 

So, high Jain's fairness does not necessarily means verifiably honest scheduling. The index alone does not measure waiting-time tails, starvation probability, or deadline satisfaction.

This could be by way of extension with security and operational performance metrics.

==🟢Another figure idea:== Adversarial fairnessSeed-grinding attempts \(k\) Jain fairnessMeasure how manipulation affects allocation equality


### Questions and disagreements

## Synthesis

**Takeaway:**
Jain et al. (1984), Section 5 — Research notes
1. The allocation metric must be selected according to the application; Jain's index does not define what fairness should mean.
2. For beacon-derived IoT/NTN lottery scheduling, primary fairness is cumulative per-device allocation of scheduling opportunities across epochs.
3. Fraction-of-demand fairness uses x_i = min(a_i/d_i, 1) for d_i > 0 and is appropriate when requested service differs across devices.
4. Dynamic eligibility may require entitlement-normalized service x_i = a_i/q_i, with q_i defined by an explicit reference policy. This is our adaptation.
5. Jain's index measures allocation equality, not the honesty of the scheduler, public verifiability, absolute efficiency, waiting-time quality, or eligibility integrity.

Section 7: The section establishes how Jain's index responds to resource exchanges and additional allocations, and gives maximization and bounded-allocation properties. Theorem 1 is particularly relevant: transferring resources from an over-served device to an under-served device increases fairness when the transfer is sufficiently small. For beacon-derived scheduling, systematic favoritism may reduce cumulative Jain fairness, but Jain fairness alone cannot detect every manipulation of randomized selection. No Section 7 theorem needs to be incorporated directly into the initial scheduling algorithm.

**What I can use:**

**Limitations:**

**Connections:**