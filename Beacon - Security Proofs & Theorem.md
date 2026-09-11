# QBeacon: Protocol Specification, Security Model, and Proofs

> Profile-parametric verifiable randomness beacon. All definitions, theorems, and proofs are grounded in the reference implementation (`epoch.py`, `metrics.py`, `main.py`). The protocol is instantiated under either the **classical** profile (Feldman VSS + Wesolowski VDF) or the **post-quantum** profile (Merkle-tree VSS + MinRoot VDF), selected by `CRYPTO_PROFILE`.

---

## 1. Notation and Setup

Let $\mathcal{N} = \{1, \dots, n\}$ be the node set. The fault/corruption threshold is
$$
t = \left\lfloor \frac{n-1}{3} \right\rfloor,
$$
and the reconstruction threshold is $k = t+1$ (`self.t = (self.n - 1) // 3`).

Let $H$ be a cryptographic hash (SHA3-256, `HASH_ALGORITHM`), modeled as a random oracle. Let $\mathbb{F}_p$ be the Shamir field (`ShamirSharing.P`). The beacon runs in discrete **epochs** $e = 0, 1, 2, \dots$, chained via the previous output $B_{e-1}$ (`previous_beacon`; genesis $B_{-1} = 0^{256}$).

Each epoch selects a deterministic **committee** $C_e \subseteq \mathcal{N}$ of size $c$ with
$$
t+1 \;\le\; c \;\le\; n,
$$
enforced by `if len(self._committee) < self.t + 1: raise ValueError`. When $c = n$ the protocol reduces to all-to-all dealing.

---

## 2. Cryptographic Assumptions

**A1 (Random Oracle).** $H$ is a random oracle. All digests — $\mathsf{CommitDigest}_e$, $\mathsf{DelayInput}_e$, the committee seed $\sigma_e$, the entropy digest $\eta_e$, and $B_e$ — are RO evaluations.

**A2 (Commitment binding & hiding).** $\mathsf{Commit}(e,i,s)$ (`Commitment.commit_contribution`) is computationally binding and hiding: no PPT adversary opens one commitment to two distinct values, and it reveals nothing about $s$ before opening.

**A3 (VSS correctness, binding, secrecy).** The threshold VSS (`ShamirSharing`/`MerkleVSS`) satisfies:
- **(i) correctness** — any $k=t+1$ valid shares reconstruct the dealt secret;
- **(ii) binding** — the public commitment fixes a unique degree-$t$ polynomial, so any $t+1$ verified shares reconstruct the *same* secret (`verify_share` + reconstruction re-check);
- **(iii) secrecy** — any $t$ shares are independent of the secret.

*Classical:* Feldman commitments $\{g^{a_k}\}$; security via discrete-log. *Post-quantum:* Merkle-root commitment over the share vector (`merkle_share_vss_v1`); security via collision-resistance of $H$ (A1).

**A4 (VDF — profile-parametric).** The delay function $\mathsf{VDF}(\cdot, T)$ satisfies:
- **(i) sequentiality** — evaluation requires $\ge (1-\epsilon)T$ sequential steps even with poly-parallelism;
- **(ii) uniqueness** — for fixed $(x, T)$ the output is unique.

*Classical (A4-C):* Wesolowski over $(\mathbb{Z}/N\mathbb{Z})^\ast$, $y = x^{2^T} \bmod N$; sequentiality from the RSA/low-order assumptions; **succinct verification** of proof $(\pi, \ell, r)$ via $\pi^\ell x^r \equiv y \pmod N$ in $O(\log T)$ ops (`_dispatch_anchor`).

*Post-quantum (A4-Q):* MinRoot over $\mathbb{F}_{p}$ (`MINROOT_PRIME`); sequentiality from the algebraic root-extraction assumption; verification is a **sampled audit** (see §6), *not* a succinct proof.

**A5 (Synchronous reveal/recover window).** The reveal-plus-recovery window (`REVEAL_WINDOW_SEC` + grace `RECONSTRUCTION_TIMEOUT + 2`) is long enough that every honest node reaches the *same* reveal/recover decision for each dealer $j \in A_e$. This is the standard synchronous-round assumption; it is the load-bearing timing hypothesis for strict agreement (Theorem 1, Step 5).

---

## 3. The Protocol (one epoch $e$)

Formalizes the state machine in `BeaconEpoch.run()`:
$$
\textsf{IDLE} \to \textsf{COMMIT} \to \textsf{AGREE} \to \textsf{FREEZE} \to \textsf{DELAY} \to \textsf{REVEAL} \to \textsf{RECOVER} \to \textsf{FINALIZE}.
$$

### Round 0 — Committee & parameters
All nodes compute $C_e = \mathsf{SelectCommittee}(e, \mathcal{N}, c)$ (`_select_committee`), where
$$
\sigma_e = H\big(\texttt{"COMMITTEE\_SELECT/v2"} \,\|\, e\big), \qquad
\mathsf{rank}(i) = H\big(\sigma_e \,\|\, i\big),
$$
and $C_e$ is the set of $c$ nodes of lowest $\mathsf{rank}$. **$\sigma_e$ depends only on $e$**, not on $B_{e-1}$ (deliberate; see Theorem 3 remark).

### Round 1 — Commit
Node $i$ samples entropy $r_i$, derives the canonical secret
$$
s_i = \mathsf{Contribution}(e, i, r_i) \in \mathbb{F}_p
$$
(`Commitment.contribution_from_raw`), forms $\gamma_i = \mathsf{Commit}(e, i, s_i)$, runs VSS to produce public commitment $V_i$ and shares $\{s_{i \to j}\}_{j \in C_e}$, and broadcasts $(\gamma_i, V_i)$ with **private shares routed only to committee members** $j \in C_e$ (`receiver_ids = sorted(self._committee)`).

### Round 2 — Agree (commit-set certification)
Each node $i$ forms its seen-set with per-dealer hash
$$
h_j \;=\; H\big(\gamma_j \,\|\, \bar V_j\big), \qquad \bar V_j = \texttt{\_canonical\_public\_vss}(V_j),
$$
broadcasts $S_i = \{(j, h_j)\}$, collects peers' sets, and computes the **intersection-agreed set**
$$
A_e \;=\; \Big(\,\bigcap_{i' \in \text{voters}} \big\{\, j : (j, h_j^{(i')}) \in S_{i'} \wedge h_j^{(i')} = h_j^{(i)} \,\big\}\Big) \;\cap\; \{\, j : j \in S_i \,\}.
$$
A dealer survives iff it appears with a **matching** hash in **every** collected set *and* in $i$'s own set (`agreed &= {int(d) ... if own_seen.get(int(d)) == h}`). Nodes drop any dealer $\notin A_e$.

### Round 3 — Freeze
Let $P_e = \texttt{\_canonical\_params}$ (protocol version, $n$, $t$, $k$, sorted $\mathcal{N}$, $c$, sorted $C_e$, `crypto_profile`, `delay_mode`, `delay_T`). Compute
$$
\mathsf{CommitDigest}_e \;=\; H\!\Big(\texttt{"COMMIT\_DIGEST/v1"} \,\|\, e \,\|\, B_{e-1} \,\|\, P_e \,\big\|\, \big\{\,(j,\, \gamma_j,\, \bar V_j)\,\big\}_{j \in A_e}^{\text{sorted}}\Big)
$$
(`_freeze_commit_transcript`). This must precede reveal.

### Round 4 — Delay
Derive
$$
\mathsf{DelayInput}_e \;=\; H\!\big(\texttt{"DELAY\_INPUT/v1"} \,\|\, e \,\|\, B_{e-1} \,\|\, \mathsf{CommitDigest}_e \,\|\, \texttt{delay\_mode} \,\|\, T\big)
$$
(`_derive_delay_input`), then evaluate the VDF **on the frozen input, before any reveal**:
$$
(\omega_e, \pi_e) \;=\; \mathsf{VDF}\big(\mathsf{DelayInput}_e,\, T\big)
$$
(`maybe_eval_vdf`, step 5b). In the classical profile with an anchor, `_dispatch_anchor` externally verifies $\pi_e$ (step 5c).

### Round 5 — Reveal & Recover
Nodes broadcast openings $s_i$. For any dealer $j \in A_e$ that did not reveal, committee members serve verified shares; a node reconstructs
$$
s_j \;=\; \mathsf{Reconstruct}\big(\{\,(x_\ell, y_\ell)\,\}_{\ell=1}^{t+1}\big)
$$
from $t+1$ shares that pass `verify_share` **and** re-open $\gamma_j$ via `verify_contribution` (`_recover_withheld`).

### Round 6 — Output
Let $R_e$ = revealed $\cup$ recovered (the contributor set). Compute the entropy digest and bound output:
$$
\eta_e = H\big(e \,\|\, \{s_j\}_{j \in R_e}\big) \quad(\texttt{hash\_entropy\_by\_node}),
$$
$$
B_e \;=\; H\!\big(\texttt{"bind"} \,\|\, e \,\|\, B_{e-1} \,\|\, \mathsf{CommitDigest}_e \,\|\, \eta_e \,\|\, \texttt{delay\_mode} \,\|\, \omega_e\big)
$$
(`derive_beacon_output_v2`, `_finalize_v2`). Chain $B_e$ into epoch $e+1$.

---

## 4. Security Properties and Theorems

### Definition 1 (Agreement / Determinism — invariant I6)
For every epoch $e$, all honest nodes output the identical $B_e$.

### Theorem 1 (Agreement)
Under **A1–A3, A5**, if all honest nodes complete Round 2, they compute identical $\mathsf{CommitDigest}_e$, hence identical $\mathsf{DelayInput}_e$, $\omega_e$ (A4-ii), $\eta_e$, and $B_e$.

**Proof.** $B_e$ is a deterministic function of $(e, B_{e-1}, \mathsf{CommitDigest}_e, \eta_e, \texttt{delay\_mode}, \omega_e)$. Each is shown identical across honest nodes.

1. **$A_e$ is common.** The intersection rule keeps $j$ iff $(j, h_j)$ with a *fixed* hash appears in *every* collected voter set and in the node's own set — both conditions depend only on broadcast data, not arrival order. Two honest nodes collecting the same voter sets compute the same intersection. Under $\ge 2t+1$ voters (`need = 2t+1`), any two honest nodes share all honest voters; a dealer surviving in one intersection survives in the other. Hence $A_e$ is identical. $\square_1$

2. **$P_e$ is identical.** All fields of `_canonical_params` are epoch-deterministic; $C_e$ is identical by Theorem 3. Serialization is canonical (`json.dumps(sort_keys=True, separators=(",",":"))`), so $P_e$ is byte-identical.

3. **$\mathsf{CommitDigest}_e$ is identical.** It hashes $P_e$, $B_{e-1}$ (identical by chain induction), and $\{(j, \gamma_j, \bar V_j)\}_{j \in A_e}$ in sorted order. By A2-binding, honest nodes hold the same $(\gamma_j, \bar V_j)$ for each $j \in A_e$ (equivocators are excluded by the hash-match in Step 1). Byte-identical serialization $\Rightarrow$ identical digest.

4. **$\omega_e$ is identical.** $\mathsf{DelayInput}_e$ is a function of the above; A4-ii (uniqueness) gives a unique $\omega_e$.

5. **$\eta_e$ is identical.** $R_e \subseteq A_e$. For $j \in R_e$, $s_j$ is either revealed (opens $\gamma_j$; unique by A2-binding) or recovered (re-opens $\gamma_j$; checked by `verify_contribution`), so both yield the *same* $s_j$. By **A5**, honest nodes agree on membership of $R_e$. Hence $\eta_e = H(e \,\|\, \{s_j\}_{j \in R_e})$ is identical. $\blacksquare$

> **Empirical witness.** I6 held (byte-identical `beacon_output_hex` across all nodes, every epoch) on **all** N=43 post-quantum runs.

---

### Definition 2 (Unpredictability / Bias-resistance)
No PPT adversary controlling $\le t$ nodes can predict $B_e$ before the delay elapses, nor bias its distribution non-negligibly, provided $\ge 1$ honest contribution survives in $R_e$.

### Theorem 2 (Bias-resistance)
Under **A1–A5**, an adversary corrupting $\le t$ nodes cannot bias $B_e$ beyond a detectable abort.

**Proof (sketch).** Two adversarial levers:

**(i) Adaptive choice of $s_i$ — defeated by freeze-before-delay.** The VDF runs on $\mathsf{DelayInput}_e$, fixed at Round 3, *before any reveal* (`run()` step 5b precedes step 6). By A4-i, $\omega_e$ cannot be computed until $\ge (1-\epsilon)T$ sequential steps after freezing, by which time all commitments are binding (A2). Thus the adversary commits $s_i$ before it can compute $\omega_e$ and cannot correlate its contribution with the output. In the RO model (A1), given any single honest, unpredicted $s_j \in R_e$,
$$
B_e = H(\cdots \,\|\, \eta_e \,\|\, \cdots) \;\overset{d}{\approx}\; \mathsf{Uniform}\{0,1\}^{256}.
$$

**(ii) Selective withholding — defeated by recovery.** A committed $s_j$ cannot be erased: by A3-i the honest committee reconstructs $s_j$ from $t+1$ shares (`_recover_withheld`), *provided the responsive committee margin meets the threshold* (Theorem 4). If recovery succeeds, $s_j$ is included regardless of the adversary. If it fails, $j$ is **deterministically dropped at all honest nodes** (Theorem 1, Step 5 + A5), so no node gains a last-revealer advantage — only entropy is lost, not biased. $\blacksquare$

---

### Definition 3 (Committee Consistency)
All honest nodes select the identical committee $C_e$.

### Theorem 3 (Committee determinism)
Under **A1**, $C_e$ is identical across all honest nodes and independent of adversarial behavior in epoch $e$.

**Proof.** $C_e$ is the $c$ lowest-$\mathsf{rank}$ nodes under $\mathsf{rank}(i) = H(\sigma_e \,\|\, i)$ with $\sigma_e = H(\texttt{"..."} \,\|\, e)$. Since $\sigma_e$ depends only on $e$, every honest node computes identical ranks, hence identical $C_e$. $\blacksquare$

> **Design remark.** Seeding $\sigma_e$ on $B_{e-1}$ would raise committee unpredictability but induces a **cascade failure**: one divergent $B_{e-1} \Rightarrow$ divergent $C_e \Rightarrow$ scattered shares $\Rightarrow$ permanent desync. Excluding $B_{e-1}$ trades committee unpredictability for robust liveness; since the committee affects only *who reconstructs*, not the output value, this does **not** weaken Theorem 2. Restoring beacon-seeded rotation is safe only once the AGREE round guarantees per-epoch agreement (future refinement).

---

### Definition 4 (Recovery Liveness)
A withheld committed contribution $s_j$ ($j \in A_e$) is recoverable iff $\ge t+1$ valid committee shares are collectable.

### Theorem 4 (Recovery-threshold — the committee-margin lemma)
Let $f_C$ be the number of committee members unresponsive during the recovery window. Recovery of withheld dealer $j$ succeeds iff
$$
\boxed{\,c \;-\; f_C \;-\; [\,j \in C_e\,] \;\ge\; t+1\,}
$$
where $[\cdot]$ is the Iverson bracket. In particular, at the minimum committee $c = t+1$, recovery fails whenever **any** committee share-holder is unresponsive or is $j$ itself.

**Proof.** Shares of $s_j$ exist only on $C_e$ (Round 1 routes to $j' \in C_e$). Reconstruction requires $t+1$ verified, distinct-$x$ shares (A3-i; `len(valid_pairs) >= self.t + 1`). The available pool is $C_e$ minus unresponsive members ($f_C$) minus $j$'s own withheld share if $j \in C_e$. Recovery succeeds iff this pool $\ge t+1$. $\blacksquare$

**Corollary 4.1 (design rule).** Choose
$$
c \;\ge\; t + 1 + \phi,
$$
where $\phi$ upper-bounds expected committee churn.

> **Empirical witness (two-point proof).** With 3 withheld nodes at $N=43$ ($t=14$):
> - $c = t+1 = 15$ (margin $\phi = 0$): recovery of the withheld set succeeded in only $\approx 14\%$ of epochs ($\overline{n_{\text{rec}}} = 0.56$).
> - $c = 25$ (margin $\phi = 10$): recovery succeeded on **every** epoch ($n_{\text{withheld}} = 0$ across all 50 epochs; $\overline{n_{\text{rec}}} = 4.70$).

---

### Definition 5 (Chain Integrity)
The sequence $\{B_e\}$ is tamper-evident: altering any past record changes all subsequent outputs.

### Theorem 5 (Chain integrity)
Under **A1**, altering any $B_{e'}$ (or its $\mathsf{CommitDigest}_{e'}$, entropy set, or delay output) changes all $B_e$, $e \ge e'$, except with negligible probability.

**Proof.** $B_e = H(\cdots \,\|\, B_{e-1} \,\|\, \mathsf{CommitDigest}_e \,\|\, \eta_e \,\|\, \omega_e)$ includes $B_{e-1}$. For a change at $e'$ not to propagate, some $H$-collision at $e \in [e', \cdot]$ must occur; A1 collision-resistance makes this negligible. $\blacksquare$

---

## 5. Complexity

**Communication (per node, per epoch).** Each dealer sends commitment + public VSS + one share to each of $c$ committee members, plus the public root broadcast; aggregated over the $n$ dealers a node hears (`metrics.py`):
$$
\mathsf{CommitBytes} \;=\; \big(32 + 32 + (64 + t\cdot 32)\big)\cdot c \;+\; 32\,n.
$$
At $t = 14$: $\mathsf{CommitBytes} = 576\,c + 32n$, i.e. $O(cn)$ versus $O(n^2 t)$ for all-to-all Feldman ($c = n$).

| $c$ | $\mathsf{CommitBytes}$ (bytes) |
|----:|-------------------------------:|
| $15$  | $10016$ |
| $25$  | $15776$ |
| $43$  | $26144$ |

> **Model caveat.** $\mathsf{CommitBytes}$ is **analytical**, computed from `committee_size` in `metrics.py`, not wire-measured. Determinism (Thm 1) and recovery (Thm 4) results are empirical; bandwidth is a derived model.

**Rounds.** $O(1)$ synchronous message rounds (commit, agree, reveal) plus one VDF epoch of wall-time $\Theta(T)$.

---

## 6. What Is Proven vs. Assumed (honesty statement)

- **Profile-independent theorems** (identical proof for classical *and* post-quantum): **Thm 1** (Agreement/I6), **Thm 3** (Committee determinism), **Thm 4** (Recovery-threshold), **Thm 5** (Chain integrity). These rely only on the RO-hash structure (A1), commitment binding (A2), and the abstract $(t+1)$-threshold VSS interface (A3) — satisfied by both Feldman and Merkle instantiations.
- **Profile-dependent theorem:** **Thm 2** (Bias-resistance) — its VDF assumption is A4-C (Wesolowski) in the classical profile and A4-Q (MinRoot) in the post-quantum profile.
- **A5 (synchrony)** is the load-bearing timing assumption; the protocol targets a synchronous reveal/recover window. **No asynchronous safety is claimed.** I6 = PASS on all runs is its empirical support.
- **Anchor verification differs by profile:**
  - *Classical:* the Wesolowski proof $(\pi, \ell, r)$ admits **succinct cryptographic verification** ($\pi^\ell x^r \equiv y \pmod N$).
  - *Post-quantum:* MinRoot has no native succinct proof; the anchor performs a **sampled elapsed-work audit** over a Merkle-committed trace. For $k$ challenges and a fraction $\varepsilon$ of inconsistent trace segments, soundness error is
  $$
  \Pr[\text{cheat undetected}] \;=\; (1-\varepsilon)^k.
  $$
  This is **audit-based evidence**, not a succinct proof of elapsed work; full succinctness (e.g. a folding/SNARK proof over the MinRoot circuit) is stated as future work.

---


# QBeacon — Post-Quantum Profile: Specification, Security Model, and Proofs

> Companion to the classical-profile document. This note specifies the **post-quantum instantiation** (`CRYPTO_PROFILE = "post_quantum"`): **Merkle-tree VSS** replaces Feldman VSS, and the **MinRoot VDF** replaces Wesolowski. All definitions, theorems, and proofs are grounded in `epoch.py`, `metrics.py`, `main.py`. Theorems that are profile-independent are stated with a cross-reference; theorems that change are proven in full here.

---

## 0. Relationship to the Classical Profile

The protocol is **profile-parametric**: one state machine (`BeaconEpoch.run()`) instantiated with different primitives, selected at construction time via `CRYPTO_PROFILE`. The security surface factors cleanly:

| Theorem | Statement | Post-quantum status |
|---|---|---|
| **T1** Agreement / I6 | all honest nodes output identical $B_e$ | **holds verbatim** (hash + abstract-VSS only) |
| **T2** Bias-resistance | $\le t$ adversary cannot bias $B_e$ | **holds under revised VDF assumption A4-Q** |
| **T3** Committee determinism | identical $C_e$ | **holds verbatim** |
| **T4** Recovery-threshold | recover iff margin $\ge t{+}1$ | **holds verbatim** (Merkle-VSS threshold) |
| **T5** Chain integrity | tamper-evident chain | **holds verbatim** |

Only **T2** carries a profile-dependent assumption, and only the **anchor** changes from *succinct proof* to *sampled audit*. Everything else is identical because the frozen transcript and the beacon binding are **hash-based** and agnostic to which VSS/VDF is used.

---

## 1. Notation (deltas from the classical profile)

Let $p$ = `MINROOT_PRIME` (a large prime; the MinRoot field). Let $H$ = SHA3-256 (`HASH_ALGORITHM`), modeled as a random oracle. Threshold and committee are unchanged:
$$
t = \left\lfloor \tfrac{n-1}{3} \right\rfloor, \qquad k = t+1, \qquad t+1 \le c \le n.
$$

**Merkle-VSS commitment (public).** For a dealer $i$ with $c$ shares:
$$
V_i \;=\; \big(\mathsf{root}_i,\; n_{\text{shares}},\; k,\; \texttt{hash\_name}\big),
$$
where $\mathsf{root}_i$ is the Merkle root over the salted share-leaves (`merkle_share_vss_v1`, `_serialize_merkle_commitment`). This is the **only** public commitment — it contains no coefficients, no shares, no per-receiver salts.

**Merkle-VSS share (private).** The package to receiver $j$:
$$
\pi_{i\to j} \;=\; \big(i,\; j,\; x_j,\; y_j,\; \mathsf{salt}_j,\; \mathsf{path}_j\big),
$$
with a Merkle authentication path $\mathsf{path}_j$ (`_serialize_merkle_share`). Verification `MerkleVSS.verify_share` recomputes the leaf $H(\mathsf{salt}_j \,\|\, x_j \,\|\, y_j)$ and checks $\mathsf{path}_j$ against $\mathsf{root}_i$.

---

## 2. Cryptographic Assumptions (post-quantum)

**A1 (Random Oracle).** $H$ is a random oracle. *(Unchanged.)*

**A2 (Commitment binding & hiding).** `Commit(e,i,s)` is binding and hiding. *(Unchanged — this is the per-node contribution commitment, hash-based in both profiles.)*

**A3-Q (Merkle-VSS correctness, binding, secrecy).** The Merkle-share VSS satisfies:
- **(i) correctness** — any $k=t+1$ shares whose $(x,y)$ verify against $\mathsf{root}_i$ reconstruct the dealt secret via Lagrange interpolation (`MerkleVSS.reconstruct`);
- **(ii) binding** — under collision-resistance of $H$ (A1), $\mathsf{root}_i$ binds a fixed multiset of leaves $\{H(\mathsf{salt}_j \| x_j \| y_j)\}$; a dealer cannot present two distinct valid $(x_j, y_j)$ for the same $j$ without an $H$-collision, so any $t+1$ verified shares reconstruct the **same** secret;
- **(iii) secrecy** — the shares are Shamir shares of a degree-$t$ polynomial, so any $\le t$ of them are information-theoretically independent of the secret; the salted Merkle leaves leak nothing further (RO).

> **Contrast with classical A3.** Feldman binding rests on **discrete-log**; Merkle-VSS binding rests only on **hash collision-resistance** (A1). This is the source of the post-quantum claim: no algebraic hardness assumption vulnerable to Shor is used in the commit/VSS layer.

**A4-Q (MinRoot sequentiality & uniqueness).** MinRoot over $\mathbb{F}_p$ (`MINROOT_T` iterations) satisfies:
- **(i) sequentiality** — evaluating the MinRoot permutation for $T$ steps requires $\ge (1-\epsilon)T$ sequential field operations; each step computes a fifth-root (equivalently an inverse-power) that is inherently serial and not parallelizable, under the **iterated-root sequentiality assumption** in $\mathbb{F}_p$;
- **(ii) uniqueness** — for fixed input $x$ and step count $T$, the output $\omega$ is unique (MinRoot is a permutation).

> MinRoot's security rests on an **algebraic sequentiality assumption over a prime field**, not on factoring/RSA. It is the post-quantum delay primitive.

**A5 (Synchronous reveal/recover window).** *(Unchanged.)* The reveal-plus-grace window (`REVEAL_WINDOW_SEC` + `RECONSTRUCTION_TIMEOUT + 2`) is long enough that all honest nodes reach the same reveal/recover decision per dealer $j \in A_e$.

---

## 3. The Protocol (post-quantum deltas)

The state machine is identical to the classical profile:
$$
\textsf{IDLE} \to \textsf{COMMIT} \to \textsf{AGREE} \to \textsf{FREEZE} \to \textsf{DELAY} \to \textsf{REVEAL} \to \textsf{RECOVER} \to \textsf{FINALIZE}.
$$
Only the boxed rounds differ.

### Round 1 — Commit *(Merkle-VSS, committee-scoped)*
Node $i$ derives secret $s_i = \mathsf{Contribution}(e,i,r_i)$, commits $\gamma_i = \mathsf{Commit}(e,i,s_i)$, and runs Merkle-VSS **dealing only to the committee**:
$$
\texttt{receiver\_ids} = \mathsf{sort}(C_e), \qquad
(V_i, \{\pi_{i\to j}\}_{j\in C_e}) = \mathsf{MerkleVSS.make\_shares}(s_i,\, c,\, t,\, e,\, i,\, C_e).
$$
It broadcasts $(\gamma_i, V_i)$ (public) and routes each $\pi_{i\to j}$ **only** to committee member $j$ (`share_map`). This is the **two-tier** step: per-dealer share cost is $O(c)$, not $O(n)$.

### Round 2 — Agree
Identical intersection rule (`agreed &= {...}`), but the per-dealer hash binds the **Merkle** public commitment:
$$
h_j \;=\; H\big(\gamma_j \,\|\, \bar V_j\big), \qquad \bar V_j = \texttt{json}(V_j;\ \text{sort\_keys}) \quad (\texttt{\_canonical\_public\_vss}).
$$
Since $\bar V_j$ is the canonicalized `merkle_share_vss_v1` dict (root + counts), the AGREE rule is byte-identical in structure to the classical case.

### Round 3 — Freeze
Identical. $\mathsf{CommitDigest}_e$ hashes $P_e$ (which now records `crypto_profile = "post_quantum"`, `delay_mode = "legacy:post_quantum:minroot"`, `delay_T = MINROOT_T`, `committee_size = c`, `committee = C_e`) and the sorted $\{(j, \gamma_j, \bar V_j)\}_{j\in A_e}$.

### Round 4 — Delay *(MinRoot)*
$$
(\omega_e, \pi_e^{\text{MR}}) \;=\; \mathsf{MinRoot}\big(\mathsf{DelayInput}_e,\, T\big), \qquad T = \texttt{MINROOT\_T}.
$$
Evaluated on the frozen input, before reveal (`maybe_eval_vdf`, step 5b). **The anchor dispatch (step 5c) does NOT fire in this profile** in the current implementation — the gate is `CRYPTO_PROFILE == "classical"`. Post-quantum anchoring uses the sampled-audit scheme of §6.

### Round 5 — Reveal & Recover *(Merkle-share reconstruction)*
Withheld dealer $j$ is recovered by collecting $\ge t+1$ share packages, each of which must (a) match `dealer_id = j`, (b) have distinct $x$, (c) pass `MerkleVSS.verify_share` against $\mathsf{root}_j$; then $s_j = \mathsf{MerkleVSS.reconstruct}(\{(x,y)\}_{t+1})$, re-checked to re-open $\gamma_j$ via `verify_contribution` (`_recover_withheld`, Merkle branch).

### Round 6 — Output
Identical binding (`derive_beacon_output_v2`):
$$
B_e = H\big(\texttt{"bind"} \,\|\, e \,\|\, B_{e-1} \,\|\, \mathsf{CommitDigest}_e \,\|\, \eta_e \,\|\, \texttt{delay\_mode} \,\|\, \omega_e\big).
$$

---

## 4. Theorems (post-quantum)

### Theorem T1-Q (Agreement / I6)
Under **A1, A2, A3-Q, A5**, all honest nodes output identical $B_e$.

**Proof.** Identical in structure to the classical T1; the only profile-dependent step is Step 3 (identity of $\{(j,\gamma_j,\bar V_j)\}$). Here $\bar V_j = \texttt{json}(V_j; \text{sort\_keys})$ where $V_j$ is the Merkle commitment `{root, n_shares, threshold_k, hash_name}`. By A3-Q-(ii), a dealer surviving the intersection-AGREE presents a **single** $\mathsf{root}_j$ (equivocation on the root produces a distinct $h_j$ and is dropped by the matching-hash intersection). Hence all honest nodes hold byte-identical $\bar V_j$, the sorted serialization is identical, $\mathsf{CommitDigest}_e$ is identical, and by A4-Q-(ii) $\omega_e$ is identical. Steps 1, 2, 4, 5 are hash/VSS-generic and carry over unchanged. $\blacksquare$

> **Empirical witness.** I6 = PASS (byte-identical `beacon_output_hex` across all 43 nodes, every epoch) on **all** post-quantum N=43 runs — this is the *primary* validated profile.

---

### Theorem T2-Q (Bias-resistance under MinRoot)
Under **A1, A2, A3-Q, A4-Q, A5**, an adversary corrupting $\le t$ nodes cannot bias $B_e$ beyond a detectable abort, provided $\ge 1$ honest contribution survives in $R_e$.

**Proof (sketch).**

**(i) Adaptive-choice bias — defeated by freeze-before-delay + MinRoot sequentiality.** The MinRoot evaluation runs on $\mathsf{DelayInput}_e$, fixed at Round 3, *before any reveal* (step 5b precedes step 6). By A4-Q-(i), $\omega_e$ cannot be computed until $\ge (1-\epsilon)T$ sequential field steps after the freeze. All contribution commitments $\gamma_i$ are binding (A2) at freeze time. Thus the adversary commits $s_i$ before it can know $\omega_e$, and cannot correlate its contribution with the output. In the RO model, given any unpredicted honest $s_j \in R_e$:
$$
B_e = H(\cdots \| \eta_e \| \cdots) \;\overset{d}{\approx}\; \mathsf{Uniform}\{0,1\}^{256}.
$$

**(ii) Selective-withholding bias — defeated by Merkle-VSS recovery.** A committed $s_j$ cannot be erased: by A3-Q-(i) the honest committee reconstructs $s_j$ from $t+1$ verified Merkle shares, provided the responsive committee margin meets the threshold (Theorem T4-Q). If recovery succeeds, $s_j$ is included regardless of the adversary; if it fails, $j$ is deterministically dropped at all honest nodes (T1-Q + A5), yielding entropy loss but **no bias**. $\blacksquare$

> **Assumption boundary (state explicitly in the paper).** T2-Q's unpredictability rests on the **MinRoot iterated-root sequentiality assumption (A4-Q-i)**, which is *less classically established* than RSA-group sequentiality. This is the honest cost of post-quantum positioning: the delay is Shor-resistant but relies on a newer algebraic assumption.

---

### Theorem T3-Q (Committee determinism)
Under **A1**, $C_e$ is identical across honest nodes and independent of adversarial behavior in epoch $e$.

**Proof.** Identical to classical T3: $C_e$ is the $c$ lowest-$\mathsf{rank}$ nodes under $\mathsf{rank}(i)=H(\sigma_e \| i)$, $\sigma_e = H(\texttt{"...COMMITTEE\_SELECT/v2"} \| e)$, depending only on $e$. $\blacksquare$

> **Design remark (unchanged).** $\sigma_e$ deliberately **excludes** $B_{e-1}$ to prevent cascade desync (divergent $B_{e-1} \Rightarrow$ divergent $C_e \Rightarrow$ scattered shares $\Rightarrow$ permanent desync, observed in early testing). Restoring beacon-seeded rotation is safe only after AGREE guarantees per-epoch agreement.

---

### Theorem T4-Q (Recovery-threshold — committee-margin lemma)
Let $f_C$ = number of committee members unresponsive during the recovery window. Recovery of withheld dealer $j$ succeeds iff
$$
\boxed{\,c \;-\; f_C \;-\; [\,j \in C_e\,] \;\ge\; t+1\,}.
$$
At the minimum committee $c = t+1$, recovery fails whenever any committee share-holder is unresponsive or is $j$ itself.

**Proof.** Merkle shares of $s_j$ exist only on $C_e$ (Round 1 routes only to $C_e$). Reconstruction requires $t+1$ **distinct-$x$, root-verified** shares (`len(valid_pairs) >= self.t + 1`, with `MerkleVSS.verify_share`). The available pool is $C_e \setminus (\text{unresponsive}) \setminus \{j \text{ if } j\in C_e\}$; success iff $\ge t+1$. $\blacksquare$

**Corollary T4.1-Q (design rule).** Choose $c \ge t+1+\phi$, $\phi$ bounding expected committee churn.

> **Empirical witness (two-point proof, both post-quantum).** At $N=43$, $t=14$, 3 withheld nodes:
> - $c = t+1 = 15$ ($\phi=0$): recovery of the withheld set succeeded in only $\approx 14\%$ of epochs ($\overline{n_{\text{rec}}} = 0.56$).
> - $c = 25$ ($\phi=10$): recovery succeeded on **every** epoch ($n_{\text{withheld}}=0$ on all 50 epochs, $\overline{n_{\text{rec}}}=4.70$; `metrics_n43_qrng_post_quantum.csv`).

---

### Theorem T5-Q (Chain integrity)
Under **A1**, altering any $B_{e'}$ (or its $\mathsf{CommitDigest}_{e'}$, entropy set, or $\omega_{e'}$) changes all $B_e$, $e \ge e'$, except with negligible probability.

**Proof.** Identical to classical T5 — pure hash chain via $B_{e-1}$ inclusion; A1 collision-resistance. $\blacksquare$

---

## 5. Complexity (post-quantum)

**Communication (per node, per epoch).** Each dealer sends: contribution commitment (32 B) + public Merkle commitment (32 B root) + one Merkle share per committee receiver, each share carrying $(x,y,\mathsf{salt},\mathsf{path})$ with path length $\approx \log_2 c$ (modeled as $64 + t\cdot 32$ B in `metrics.py`), plus the public root broadcast. Aggregated over the $n$ dealers a node hears:
$$
\mathsf{CommitBytes} \;=\; \big(32 + 32 + (64 + t\cdot 32)\big)\cdot c \;+\; 32\,n \;=\; 576\,c + 32n \quad (t{=}14).
$$

| $c$ | $\mathsf{CommitBytes}$ (bytes) |
|----:|-------------------------------:|
| $15$  | $10016$ |
| $25$  | $15776$ |
| $43$  | $26144$ |

This is $O(cn)$, versus $O(n^2 t)$ for all-to-all dealing ($c=n$).

> **Model caveat.** $\mathsf{CommitBytes}$ is **analytical** (computed from `committee_size` in `metrics.py`), not wire-measured. **T1-Q determinism and T4-Q recovery are empirical**; bandwidth is a derived model. The Merkle-path term $t\cdot 32$ is an upper-bound approximation; the true path is $\lceil \log_2 c \rceil$ hashes.

**Rounds.** $O(1)$ synchronous rounds + one MinRoot delay epoch of wall-time $\Theta(T)$.

> **Empirical delay.** Recorded MinRoot evaluation is $\overline{\texttt{vdf\_eval\_ms}} \approx 10.2\text{–}10.4\,\text{s}$ across all N=43 runs, i.e. **$T = 4096$** (the `run_variety.sh` `MINROOT_T=1024` export did not propagate to nodes; `run_experiment.sh` step 3 fell back to `MINROOT_T=${MINROOT_T:-4096}`). Report the post-quantum delay parameter as **$T = 4096$**.

---

## 6. The Post-Quantum Anchor: Sampled Elapsed-Work Audit

MinRoot admits **no native succinct proof** of elapsed work (unlike Wesolowski, whose $(\pi, \ell, r)$ verifies in $O(\log T)$). The post-quantum anchor therefore performs a **sampled hardware audit** of sequential execution — *not* a succinct proof.

**Prover (on the A8 node):**
1. Evaluate MinRoot for $T$ steps, recording $O(\sqrt T)$ (or fixed-stride) checkpoint states.
2. Merkle-commit the checkpoint vector → one root $\rho$ (32 B). *(Same Merkle-commitment technique as the VSS layer — succinct commitment to the trace.)*
3. Derive non-interactive challenges (Fiat–Shamir):
$$
\{i_1, \dots, i_k\} \;=\; H\big(\mathsf{DelayInput}_e \,\|\, \rho \,\|\, \omega_e\big).
$$
4. Open, for each challenge $i_m$, the bracketing checkpoints + Merkle paths.

**Verifier (on the Nordic MCU, in C):**
1. Verify each opened checkpoint against $\rho$ ($k \cdot \lceil\log(\#\text{checkpoints})\rceil$ hashes — no big-int, no pairings).
2. Verify the **local MinRoot step relation** between bracketing checkpoints (a bounded number of $\mathbb{F}_p$ operations, 256-bit modmul — fits nRF flash/RAM).
3. Re-derive $\{i_m\}$ from $H(\mathsf{DelayInput}_e \| \rho \| \omega_e)$ and confirm the openings match (prevents adversarial challenge selection).
4. Return $(\texttt{PASS/FAIL}, \mu s)$ → CSV `anchor_verify_pass`, `anchor_verify_us`.

**Soundness.** If a fraction $\varepsilon$ of the trace segments is inconsistent, then for $k$ independent challenges:
$$
\Pr[\text{cheat undetected}] \;=\; (1-\varepsilon)^k.
$$
This is **audit-based elapsed-work evidence** with tunable soundness (increase $k$), bound to $\mathsf{DelayInput}_e$ so a PASS certifies *this epoch*.

> **Honest claim.** Classical anchor = **succinct cryptographic proof** verification ($\pi^\ell x^r \equiv y \pmod N$). Post-quantum anchor = **sampled sequential-execution audit**, soundness $(1-\varepsilon)^k$. Full post-quantum succinctness (a folding/SNARK proof over the MinRoot circuit) is stated as **future work**.

---

## 7. What Is Proven vs. Assumed (post-quantum honesty statement)

- **Proven under A1, A2, A3-Q, A5 — hash + Merkle-VSS only, NO discrete-log, NO factoring:**
  **T1-Q** (Agreement / I6), **T3-Q** (Committee determinism), **T4-Q** (Recovery-threshold), **T5-Q** (Chain integrity). These rely solely on the random-oracle hash structure (A1), commitment binding/hiding (A2), and the abstract $(t{+}1)$-threshold Merkle-VSS interface (A3-Q). **This is the core post-quantum claim: the agreement, committee, recovery, and chain guarantees carry no assumption vulnerable to Shor-type quantum attack.**

- **T2-Q (Bias-resistance)** additionally requires **A4-Q (MinRoot iterated-root sequentiality)** over $\mathbb{F}_p$ — a newer algebraic assumption, not reducible to hashing. This is the honest cost of post-quantum delay: the unpredictability guarantee rests on MinRoot sequentiality, which is less classically studied than RSA-group sequentiality. **State this assumption boundary explicitly in the paper.**

- **A5 (synchrony)** is the load-bearing timing assumption. The protocol targets a **synchronous reveal/recover window** (`REVEAL_WINDOW_SEC` + grace `RECONSTRUCTION_TIMEOUT + 2`, per `epoch.py`). **No asynchronous safety is claimed.** Empirical support: I6 = PASS on **all** N=43 post-quantum runs (byte-identical `beacon_output_hex` every epoch).

- **Anchor verification is a SAMPLED AUDIT, not a succinct proof (post-quantum).** Unlike the classical Wesolowski anchor (which admits $O(\log T)$ succinct verification $\pi^\ell x^r \equiv y \pmod N$), MinRoot has no native succinct proof. The post-quantum anchor performs a Merkle-committed, Fiat–Shamir-challenged, $k$-sample audit of sequential execution with soundness error
$$
\Pr[\text{cheat undetected}] = (1-\varepsilon)^k,
$$
bound to $\mathsf{DelayInput}_e$. **Describe this as audit-based elapsed-work evidence with tunable soundness**, and cite folding/SNARK-over-MinRoot (e.g. a Nova-style proof) as the path to full post-quantum succinctness — **explicitly future work.**

- **In the current implementation the post-quantum anchor does NOT fire:** `_dispatch_anchor` is gated `CRYPTO_PROFILE == "classical"` (`epoch.py`), and the post-quantum CSV records `anchor_verify_pass = -1`, `anchor_verify_us = -1` (`metrics_n43_qrng_post_quantum.csv`). Removing this gate and implementing the §6 audit is the anchor-integration task; the *protocol* proofs above hold independently of the anchor (the anchor is post-finalize attestation only and never alters $B_e$).

- **Bandwidth (§5) is an ANALYTICAL model**, computed from `committee_size` in `metrics.py` — **not** wire-measured. T1-Q determinism and T4-Q recovery are **empirical**; the Merkle-path term $t\cdot 32$ over-approximates the true $\lceil \log_2 c \rceil$ hashes.

- **Empirical delay parameter is $T = 4096$** (recorded `vdf_eval_ms ≈ 10.2$–$10.4$ s across all runs), not the $T = 1024$ intended by `run_variety.sh`: the `export MINROOT_T=1024` did not propagate through `run_experiment.sh` step 3's `MINROOT_T=${MINROOT_T:-4096}` fallback (only `COMMITTEE_SIZE`/`WITHHOLD_NODES` were passed via `env "$@"`). **Report the post-quantum delay as $T = 4096$.**

- **⚠️ Data-provenance note for the uploaded run.** The CSV `metrics_n43_qrng_post_quantum.csv` reports `commit_bytes_sent = 15776`, which by the §5 model $576c + 32n$ (at $t=14$, $n=43$) implies **committee size $c = 25$**, and the recovery trace is clean ($n_{\text{withheld}} = 0$ every epoch, $\overline{n_{\text{rec}}} = 4.70$). However, `run_variety.sh` invoked this run with tag `c15_wh3` and `COMMITTEE_SIZE=15` (which would give $10016$). **Resolve this tag-vs-data discrepancy before citing $c$**, since Corollary T4.1-Q's two-point argument (fragile at $c=t+1$, reliable at $c>t+1$) depends on which committee actually ran. The clean $n_{\text{withheld}}=0$ behavior is consistent with $c=25$, not $c=15$.

---

## 8. Summary Table — Assumption Dependence

| Theorem                        | RO (A1) | Commit (A2) | VSS (A3-Q) | VDF (A4-Q) | Sync (A5) | Shor-safe? |
| ------------------------------ | :-----: | :---------: | :--------: | :--------: | :-------: | :--------: |
| **T1-Q** Agreement / I6        |    ✔    |      ✔      |     ✔      |     —      |     ✔     |  **Yes**   |
| **T2-Q** Bias-resistance       |    ✔    |      ✔      |     ✔      |     ✔      |     ✔     |    Yes*    |
| **T3-Q** Committee determinism |    ✔    |      —      |     —      |     —      |     —     |  **Yes**   |
| **T4-Q** Recovery-threshold    |    ✔    |      ✔      |     ✔      |     —      |     ✔     |  **Yes**   |
| **T5-Q** Chain integrity       |    ✔    |      —      |     —      |     —      |     —     |  **Yes**   |

\* T2-Q is post-quantum *provided* the MinRoot iterated-root sequentiality assumption (A4-Q) holds; this is the sole non-hash assumption in the post-quantum profile.

---

## Appendix A: Symbol Table

| Symbol | Meaning | Code reference |
|---|---|---|
| $n$ | number of nodes | `self.n = len(all_node_ids)` |
| $t$ | fault/corruption threshold, $\lfloor (n{-}1)/3 \rfloor$ | `self.t = (self.n - 1) // 3` |
| $k = t+1$ | reconstruction threshold | `threshold_k` |
| $c$ | committee size, $t{+}1 \le c \le n$ | `_committee_size`, `COMMITTEE_SIZE` |
| $C_e$ | committee for epoch $e$ | `_select_committee` |
| $\sigma_e$ | committee seed $= H(\texttt{"...v2"} \| e)$ | `_select_committee` |
| $H$ | SHA3-256 random oracle | `HASH_ALGORITHM` |
| $p$ | MinRoot field prime | `MINROOT_PRIME` |
| $\mathbb{F}_p$ | Shamir / MinRoot field | `ShamirSharing.P` |
| $r_i$ | node $i$'s raw QRNG entropy | `entropy_src.get_entropy` |
| $s_i$ | canonical field contribution | `Commitment.contribution_from_raw` |
| $\gamma_i$ | contribution commitment | `Commitment.commit_contribution` |
| $V_i$ | public Merkle-VSS commitment | `_serialize_merkle_commitment` |
| $\rho_i / \mathsf{root}_i$ | Merkle root of share vector | `MerkleVSSCommitment.root` |
| $\pi_{i\to j}$ | private share package to $j$ | `_serialize_merkle_share` |
| $A_e$ | intersection-agreed dealer set | AGREE block in `run()` |
| $P_e$ | canonical parameter header | `_canonical_params` |
| $\mathsf{CommitDigest}_e$ | frozen commit transcript hash | `_freeze_commit_transcript` |
| $\mathsf{DelayInput}_e$ | delay function input | `_derive_delay_input` |
| $T$ | delay step count | `MINROOT_T` / `delay_T` |
| $\omega_e$ | delay output | `_delay_output` |
| $\eta_e$ | entropy digest over $R_e$ | `hash_entropy_by_node` |
| $R_e$ | contributor set (revealed ∪ recovered) | `contributors` |
| $B_e$ | beacon output for epoch $e$ | `derive_beacon_output_v2` |
| $f_C$ | unresponsive committee members | (recovery-window churn) |
| $\varepsilon$ | fraction of inconsistent trace segments | §6 audit |
| $k$ | number of audit challenges | §6 audit |

---

## Appendix B: Profile Diff (classical ↔ post-quantum)

| Layer | Classical | Post-quantum |
|---|---|---|
| VSS | Feldman ($g^{a_k}$) | Merkle-tree ($\mathsf{root}$) |
| VSS binding | discrete-log | hash collision-resistance (A1) |
| Public commitment | list of group elements | one 32-byte root + counts |
| VDF | Wesolowski (RSA group) | MinRoot ($\mathbb{F}_p$) |
| VDF assumption | RSA / low-order (A4-C) | iterated-root sequentiality (A4-Q) |
| Anchor | succinct proof $\pi^\ell x^r \equiv y$ | sampled audit, soundness $(1-\varepsilon)^k$ |
| Shor-vulnerable? | **Yes** (factoring/DLog) | **No** (hash) except A4-Q delay |
| Selected by | `CRYPTO_PROFILE="classical"` | `CRYPTO_PROFILE="post_quantum"` |

> **One-line thesis.** The post-quantum profile removes every factoring/discrete-log dependency from the *agreement, committee, recovery, and chain* guarantees (T1-Q, T3-Q, T4-Q, T5-Q proven under hashing alone), leaving a single, clearly-stated algebraic assumption (MinRoot sequentiality, A4-Q) for the *delay/unpredictability* guarantee (T2-Q) — with the anchor providing a tunable-soundness sampled audit of that delay.


