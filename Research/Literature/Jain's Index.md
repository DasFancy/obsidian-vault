---
type: literature
status: queued
project:
  - IEEE-IoT
authors: Jain
year: "1984"
venue:
doi:
pdf: "[[_pdfs/IEEE-IoT/filename.pdf]]"
tags:
  - project/ieee-iot
  - domain/iot
  - concept/fairness
---

# Paper title: Building the mathematical foundation for beacon-derived IoT/NTN scheduler

## Why am I reading this?

In order to understand the applications I am conjuring up for IEEE IoT journal, I must understand fairness distribution in any form of distributed system.

## Triage

**Problem:**
In 1984, researchers were already investigating fair resource allocation in computer networks. However, fairness was frequently described qualitatively.

For instance, consider three allocation mechanisms:

- Algorithm A distributes resources equally among users.
- Algorithm B strongly favors a small number of users.
- Algorithm C distributes resources unequally, but not as severely as B.

We can say that A appears fairer than B.

> But how much fairer?

And how can we make comparisons reproducible across different experiments?

That is the problem the authors address.


**Main contribution:**

**Relevance to my work:**

In my case, if the beacon-derived scheduler selects devices to receive uplink opportunities, then should fairness mean: 
	- Equal number of uplink opportunities ?
	- Equal probabilities of being selected ? 
	- Equal waiting time ? 
	- Equal resource allocations relative to each device's entitlement 
	  
So basically, Jain's index can tell me about the fairness in resource allocation, ==🟠but it can't exactly tell me which fairness objective the scheduler should pursue.== 


**Decision:** Read

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

### Questions and disagreements

## Synthesis

**One-sentence takeaway:**

**What I can use:**

**Limitations:**

**Connections:**