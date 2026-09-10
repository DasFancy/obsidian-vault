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

## Appendix: Symbol Table

| Sym
