# QBeacon — Journal Extension: Architecture Log & Decisions
_Last updated: 2026-09-04 | Status: local logic validated (5/5 PASS), NOT yet anchor-ready_

## 0. TL;DR / Current State
- Post-quantum verifiable randomness beacon on FiT-IoT ARM Cortex-A8 nodes. Journal extension of accepted conference paper ("Beaqon", IoMT, DFG/CeTI).
- Stack: per-node QRNG entropy → MerkleVSS (share-vector Merkle commitment + Shamir) → MinRoot VDF (post-quantum) → chained beacon output.
- Protocol: `commit → collect → AGREE (new) → FREEZE → delay → reveal → recover → extract`.
- **Local test (COMMITTEE_SIZE=0, WITHHOLD_NODES=1): clean 5/5 PASS** — all 4 nodes agree every epoch, node 1 recovered every epoch, contributors=4/4, recovery_rate=0.25, avg_bandwidth_bytes=1224.
- **NOT ready for anchor yet.** Two blockers remain (see §7).

---

## 1. Core Invariants (load-bearing — never break these)
- **I6 — Freeze agreement:** all honest nodes must freeze the IDENTICAL accepted commit set → identical `CommitDigest_e`. Everything downstream (DelayInput, VDF, output, next committee) depends on this.
- **I7 — Output binding:** `beacon_output = derive_beacon_output_v2(previous_beacon, commit_digest, entropy_digest, delay_mode, delay_output)`. Beacon is chained epoch-to-epoch via `previous_beacon`.
- **Threshold:** `t = floor((n-1)/3)`, reconstruction needs `t+1`. Committee needs `c ≥ t+1` (use `c ≥ 2t+1` for Byzantine safety).
- **Ordering is load-bearing:** AGREE runs AFTER collect, BEFORE freeze. Complaint-round VSS (future) will merge into this same "agree on qualified set before freeze" step.

---

## 2. The Central Bug We Chased: Implicit-Agreement Freeze
### The disease
`_wait_for_commits` freezes "whatever commits arrived by the deadline," and *hopes* every node saw the same set. That hope was the bug. Under any timing skew, nodes freeze different sets → different `CommitDigest_e` → different DelayInput → different beacon output → (with beacon-seeded committee) permanent cascade.

### Why the conference paper "worked" anyway
Table I shows full participation collapsing to ~1% at N=32, ~0% at N=64. It only survived because: classical primitives + no committee rotation + generous windows meant divergence never cascaded. The journal design (committee + PQ + recovery + rotation) removes that luck — so the freeze flaw becomes fatal and MUST be fixed honestly.

### Why it matters for the anchor
The Nordic anchor attests the VDF proof computed over `CommitDigest_e`. If nodes disagree on that digest, the anchor attests DIFFERENT things on different nodes → its guarantee is void. **Agreement must precede anchoring. Non-negotiable.**

---

## 3. Bugs Found & Fixed (chronological, with root cause)

| # | Symptom | Root cause | Fix |
|---|---------|-----------|-----|
| 1 | KeyError on self-commit at COMMITTEE_SIZE=3 | Non-committee dealer deals to committee, not itself → `share_map[self.node_id]` missing | `self_share = share_map.get(self.node_id)`; include only if not None (mirrors `broadcast_commit` guard) |
| 2 | Non-committee node rejects committee-dealt commits → breaks I6 | `on_commit` returned when it held no share | Added non-committee branch: accept commit for freeze, store `shares={}`, **replay buffered early reveals before return** |
| 3 | Pasted pseudocode crashed all nodes | Literal `broadcast_reveal(...)` / `on_reveal({...})` with `...` ellipsis | Replaced with real gated reveal block (withholder skips both broadcast AND own on_reveal) |
| 4 | Committee cascade (divergent output → divergent committee → scattered shares) | `_select_committee` seeded on `previous_beacon` | **Seed on `epoch_id` ONLY** (bumped domain tag v1→v2). `previous_beacon` still bound in freeze/output (I7 intact) — only committee SELECTION decoupled. Restore beacon-seeding only after AGREE proven at scale. |
| 5 | Serial-stall `Read timed out` storm; divergent commit sets | `broadcast_commit` was SYNCHRONOUS — one dead peer blocks 6s (2s×3 retries) | **Threaded `broadcast_commit`** (per-peer thread + bounded join, mirrors `_broadcast_all`). Per-peer unicast unchanged → confidentiality preserved. |
| 6 | Recovery: `only 0/1 shares available`; same-epoch output divergence | `HTTPServer` is SINGLE-THREADED — a node blocked in `collect_shares` can't answer peers' `SHARE_REQUEST` | **`ThreadingHTTPServer`** — share requests served concurrently |
| 7 | Fast (recovered) nodes race into next epoch alone → different commit sets | `EPOCH_DURATION_SEC=1` in local test | Set to **12** so all nodes wait to same wall-clock boundary |
| 8 | Epoch-4 (final) divergence: `Connection refused` mid-recovery | Fast finisher on LAST epoch exits process (no next epoch to keep server alive), killing its HTTP server while peers still recover it | **Grace `sleep(RECONSTRUCTION_TIMEOUT + 2)` before `network.stop()`** in `QRNGBeacon.start` finally |
| 9 | Lagging peers get no share mid-run | `_run_epoch` finally unregistered current epoch immediately | **2-epoch handler retention:** unregister `self._epoch_id - 1`, not current |

---

## 4. Architectural Decisions Made

### 4a. Two-tier committee dealing (the "committee, but better")
- **Decouple entropy-contribution from VSS-dealing.** A node can contribute entropy without all-to-all dealing, yet still be recoverable if it deals to a small RECONSTRUCTOR SET (the committee).
- **Every node still COMMITS** (broadcasts commitment + public root). Committee only restricts WHO RECEIVES SHARES.
- Committee members deal to committee; non-committee nodes deal to committee too (opt-in recoverability at O(c), independent of N).
- `MerkleVSS.make_shares` already takes `receiver_ids` → pass `receiver_ids = sorted(self._committee)`. Nearly the whole change.
- **Bandwidth spectrum (3 measurable operating points):**
  - all-to-all `O(N²t)` — everyone recoverable, security-max (baseline; c=N)
  - committee-only `O(cNt)` — only committee recoverable (naive)
  - **two-tier `O(cNt)` — EVERYONE recoverable at committee cost (new sweet spot)**
- **Why novel:** server-class beacons (drand, SPURT, RandHound, Choi–Manoj–Bonneau SoK) treat committee as THE participant set — they don't have many independent QRNG entropy sources. Our MANY-CONTRIBUTOR IoT regime does (ties to Proposition 1: ≥1 honest QRNG contributor sets the entropy floor). This is a genuine, unexplored contribution.

### 4b. Committee selection
- Deterministic function of `epoch_id` only (currently), bound into `_canonical_params` (`committee`, `committee_size`) → all nodes agree → I6-safe.
- **Frozen as `sorted(self._committee)`** (order-canonical) in the digest; rank-ordered list used for dealing. Membership matters for digest, not rank.
- Rotates per epoch (via epoch_id). Restore `previous_beacon`-seeding for unpredictable rotation ONLY after AGREE guarantees per-epoch agreement.

### 4c. Classical path left UNCHANGED (deliberate)
- Classical stays all-to-all Feldman/Shamir for bit-for-bit conference reproducibility. Committee dealing is PQ-only. Test committee with `--crypto-profile post_quantum`.

### 4d. The AGREE round (commit-set certification) — the proper fix for the freeze flaw
- New phase between `_wait_for_commits` and `_freeze_commit_transcript`.
- Each node broadcasts `own_seen = {dealer_id: H(commitment || public_vss)}` (hash catches equivocation).
- Gather ≥ (2t+1) peers' sets via `collect_commit_sets`.
- **INTERSECTION rule (chosen):** keep only dealers present WITH MATCHING HASH in EVERY collected set. Guarantees no node is ever missing an agreed dealer → no `COMMIT_FETCH` round needed → byte-identical transcript.
- Prune `self._commits` to agreed set; `_freeze_commit_transcript` already iterates `self._commits` → freezes over agreed set automatically (no change there).
- **This is the gather/common-subset primitive** the protocol was missing and that SPURT/RandHound/drand all have.
- **Intersection vs ≥t+1 tally:** tally has a latent divergence bug (a dealer agreed by ≥t+1 but not held locally → `missing` branch → some include, some don't). Intersection eliminates it. Upgrade to tally+`COMMIT_FETCH` (variant A) later for higher participation.

---

## 5. NEW IDEA — Deferred VSS Research (separate conversation / possible patent)
> Handoff prompt already saved separately. Summary:
- **Problem A (verifiability — GRAVE):** current MerkleVSS is NOT verifiable. `verify_share` only checks Merkle inclusion, NOT that shares lie on a single degree-t polynomial. A malicious dealer can commit inconsistent shares that each verify but reconstruct differently for different subsets. No DISQUALIFICATION exists.
  - **Fix:** Pedersen-style hash-VSS + COMPLAINT ROUND (blinding poly, public coeff hash commitments, receiver-side degree-t check, broadcast complaint, disqualify dealer BEFORE freeze). Hash-based → PQ-safe, A8-friendly. NO FRI/lattice (too heavy). Complaints off-path under honest majority → zero amortized cost.
- **Problem B (efficiency/fairness):** the two-tier committee work above.
- **Key composition insight:** the complaint round and the AGREE round are the SAME step — "agree on the qualified dealer set before freeze." Merge them → zero extra rounds. Fixing agreement fixes both determinism AND verifiability.
- **Caveat to state:** two-tier concentrates reconstruction trust in committee; >t colluding committee members could reconstruct one pre-reveal contribution early (confidentiality break of a one-time QRNG sample) — does NOT break unbiasability (VDF+freeze bind output). Bounded by t < c/3.

---

## 6. Metrics Fix (done)
- `metrics.py` bandwidth WAS hardcoded to `n_nodes` → committee benefit invisible (both c=0 and c=3 showed 1096/1064 identically before fix).
- **Fixed:** added `committee_size` to `EpochResult` (`=len(self._committee)`), and `commit_bytes` now scales shares by `c`: `(32+32+(64+t*32))*c + 32*n_nodes`.
- **Verified:** c=4 → 1224; c=3 → 1064. Committee benefit now measurable → this is the data for the conference Eq.(5) reviewer rebuttal (O(N²t) objection).
- **Caveat:** this is an ANALYTICAL estimate (formula), not wire-measured. Label as such. Optionally instrument `_send` later for true byte counts.

---

## 7. BLOCKERS before adding the Nordic anchor
1. **AGREE round is UNVALIDATED.** It's a NO-OP on localhost (`All 4 commits in` every epoch → intersection == full set). Its whole purpose (surviving PARTIAL commit arrival) is untestable locally. **Must validate on testbed N≥16 under induced loss: pass = identical `beacon_output_hex` across all nodes per epoch EVEN WHEN `n_contributors < n_nodes`.**
2. **Anchor path is classical/Wesolowski-ONLY.** `_dispatch_anchor` uses `WesolowskiVDF._fiat_shamir_prime` and is gated `CRYPTO_PROFILE == "classical"`. The journal beacon runs MinRoot (post_quantum) → **NO MinRoot anchor-verify path exists.** This is net-new work, not wiring.

---

## 8. Remaining cleanups (non-blocking)
- `epoch.py` AGREE block: still has the dead `missing = agreed - ...` block + `if missing:` warning ("not seen by >= t+1" text) — WRONG for intersection (missing is always empty). DELETE it. Fix drop-log text → "not in intersection".
- `epoch.py`: move `import os` (inside `run()` step 6) to module top.
- `_select_committee`: check for any leftover duplicated v1 `h_seed` lines (dead, v2 overwrites) — delete if present.
- `settings.py`: **`QRNG_URL` stale** (`1ce4-…`); live endpoint resolved as `3965-…`. Local uses prng so harmless, but **qrng testbed runs will FAIL** — update before testbed.
- `run_experiment.sh` does NOT forward `COMMITTEE_SIZE` — add to per-node SSH env (mirror `CRYPTO_PROFILE`) for committee dealing on hardware.
- NEVER set `WITHHOLD_NODES` in a real data-collection run (it suppresses a reveal every epoch) — dropout experiment only.

---

## 9. Local test config (reference)
- `_early_local_test_bootstrap` + `_local_test_worker`: COMMIT=6, REVEAL=8, RECONSTRUCTION=5, EPOCH_DURATION=12, MINROOT_T=512, EPOCH_ALIGN=0, mode=prng (forced), spawn context, 4 nodes on 9000–9003.
- Spawn children run `_local_test_worker` (NOT `_early_local_test_bootstrap`) — window env must be set THERE for children.
- Committee env DOES propagate (verified: c=3 → digest differs from c=0, bandwidth 1064 vs 1224).

---

## 10. NEXT STEPS (in order)
1. **Run FiT-IoT testbed evaluation** (N≥16, qrng, post_quantum) — the real AGREE-round validation under partial arrival. Fix `QRNG_URL` + forward `COMMITTEE_SIZE` first.
2. Clean up §8 items (delete dead `missing` block etc.).
3. **Build MinRoot anchor-verify path** (Nordic) — anchor the AGREE-guaranteed transcript.
4. (Later / separate) VSS complaint-round verifiability — merge into AGREE step.
5. (Later) Upgrade AGREE intersection → tally + `COMMIT_FETCH` for higher participation.

---

## What actually went wrong in c15_wh3 — the tight-committee recovery trap [Important Findings]

The root issue is a **conflict between committee dealing and the recovery threshold**. Here's the mechanism, traced through `epoch.py`:**1. You deal shares only to the committee.** At `COMMITTEE_SIZE=15`, the dealer sends Merkle shares to `receiver_ids = sorted(self._committee)` — only **15 nodes hold any share** of a given secret. Non-committee nodes hold nothing.**2. Recovery needs `t+1 = 15` valid shares.** `_recover_withheld` calls `collect_shares(min_shares=self.t + 1)`, and the Merkle path requires `len(valid_pairs) >= self.t + 1` = 15 before it can `MerkleVSS.reconstruct`.**3. The fatal squeeze:** committee size = 15, threshold = 15. So you need **every single committee member's share** to recover a withheld node. But:

- The withheld node itself may be a committee member (it holds its own share, but it's withholding, so it won't serve it).
- Any committee member that dropped/timed out (real FiT-IoT churn) removes a share.
- `collect_shares` has a `RECONSTRUCTION_TIMEOUT=4s` window — slow members don't respond in time.

**Result: you almost never get all 15.** That's why the CSV shows `n_recovered=0, n_withheld=2` on most epochs — recovery _tried_ and _failed_ because it could gather at most 14 shares.