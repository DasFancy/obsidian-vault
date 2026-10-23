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

Another important relevance. If I consider a system of N=100 IoT devices and m are my eligible devices m=10, then the Jain's index can be:
$$
J = \frac{10^{2}}{100(10)} = 0.1
$$
A perfectly honest, unifromly random lottery selecting 10 of 100 eligible devices produces exactly this result in every epoch. Even if the randomness is perfectly unpredictable and unbiased, the single-epoch fairness index is: 
$$
J_{epoch} = \frac{m}{N}
$$
Therefore, I must evaluate the fairness over various epochs (which is intuitive here). If I observe 100 scheduling epochs, define: 
$$
a_{i}(T) = \sum_{e=1}^{T}{A_{i,e}}
$$
where: 

$$
A_{i,e} = \begin{cases} 
1, & \text{device i receives an opportunity in epoch e},\\  
0,&  \text{otherwise} 
\end{cases} 
$$

Then: 
$$
J(T) = \frac{(\sum_{i=1}^{N}{a_{i}(T))^{2}}}{N\sum_{i=1}^{N}a_{i}(T)^{2}}
$$
This measures how equally resources have been distributed across devices during the observational interval. Under independent, uniformly random selections with a fixed eligible population, cumulative resource shares converge toward equality as the number of epochs grows. Cosnequently: 

$$
J(T) \xrightarrow[T\rightarrow \infty]{} 1
$$
Even after 100 epochs, those 10 devices receive every opportunity, while the remaining 90 receive none. Their fairness index remains J(T) = 0.1. That is why for my case, the cummulative fairness is more important than per-epoch fairness. 

==🟢Note==: perhaps a Jain's fairness index versus a number of scheduling epochs plot could be nice. 


> [!PDF|important] [[A_Quantitative_Measure_Of_Fairness_And_D_OCR.pdf#page=9&selection=194,0,196,2&color=important|Example 3:]]
> > Example 3:
> 
> 

An important example particularly for my work. E.g. some devices may be eligible during every epoch, while other participate only during particular contact window. Can be nice for the stringent/fast services. 

**Demand Satisfaction Fairness

For devices with positive demand $d_{i}(T)$ : 
$$x_{i}(T) = \min \left(\frac{a_{i}(T)}{d_{i}(T)}, 1 \right)$$
Then the Jain's formula can be applied to these $x_{i}(T)$ values.

**Entitlement-normalized fairness 

$$
x_{T} = \frac{a_{i}(T)}{q_{i}(T)}
$$
where $q_{i}(T)  > 0$ represents its expected allocation under the reference scheduling policy. 

IMP: An example scenario: 
Scheduler A uses an honest PRNG; Scheduler B uses a seed-grinding strategy to favor particular devices, but still produces apparently balanced cumulative allocations. 

They both can have similar Jain fairness indices. 

Yet scheduler B violates my intended selection policy. 

Therefore, merely depending upon Jain's fairness is not helpful in my evaluation. I need to establish expected selection probabilities and then measure whether adversarial behavior creates statistically meaningful deviations from them. This gives me a clean separation. 

$$J_{alloc} \; \Delta_{F} \; V_{e}$$
where $\Delta_{F}$ and $V_{e}$ are preliminary notations for quantities I will formally define during the security-model stage. 

Theorem 1 from section 7 is another important result worth retaining. In my analogy: 

Suppose device $k$ has received more allocation opportunities than device $j$. If we transfer $\Delta x$ from device $k$ to device $j$, Jain fairness increases when: 

$$ 0 < \Delta x < x_{k}-x_{j}$$
Here, allocations must remain non-negative. 

A malicious scheduler that systematically redirects opportunities from underserved devices toward already favored devices may reduce cumulative Jain fairness.

**Perhaps come back to Section 7 when the program/system model is developed further

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