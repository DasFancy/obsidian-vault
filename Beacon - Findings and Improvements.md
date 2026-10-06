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


# QBeacon / AeroPQ — Incident Log & Hardening Record

**Project:** QBeacon_Sat / AeroPQ **Testbed:** FiT-IoT Grenoble, 38 × A8 nodes, `10.0.12.x` **Hardware anchor:** Nordic nRF52840 via local PC serial → reverse SSH tunnel **Period:** 2026-09-23 → 2026-09-25 **Status:** 6 incidents identified, 4 fixed in code, 2 operational

---

## Contents

1. [H-01 — Async anchor wrapper signature mismatch](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-01)
2. [H-02 — Node-0 chain fork via pacing drift](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-02)
3. [H-03 — Tunnel port confusion](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-03)
4. [H-04 — `127.0.0.1` from node context](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-04)
5. [H-05 — Tunnel death at ~3.3 h](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-05)
6. [H-06 — Silent chain fork under threshold-tight committee](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-06)
7. [H-07 — Serial recovery cost scaling](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-07)
8. [Validation gates](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#gates)
9. [Launch checklist](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#checklist)

---

## H-01 — Async anchor wrapper signature mismatch

**Severity:** High · **Status:** Worked around (`ANCHOR_ASYNC=0`) · **Code:** still broken

### Symptom

```text
[main] hardware anchor attached (ASYNC) on node 0
[anchor] async verify error: AnchorContext is required
[Epoch 0] anchor error: cannot unpack non-iterable NoneType object
```

Repeated every epoch. Zero anchor rows written; all CSV anchor columns `-1`.

### Root cause

Two independent defects in `_AsyncAnchor` (`main.py`):

1. **Signature mismatch.** The wrapper defines `verify(self, proof, ctx, *args, **kwargs)`, but `epoch.py::_dispatch_anchor` calls `self.anchor.verify(proof=proof, ctx=ctx)` for the PQ path and `verify(proof=..., l=..., r=..., ctx=...)` for classical. The keyword `ctx` never reaches the inner `AnchorVerifier` correctly, which raises `AnchorContext is required`.
    
2. **Return contract mismatch.** `_dispatch_anchor` does `passed, us = self.anchor.verify(...)`, expecting a 2-tuple. The wrapper returns `pending`, which is `None` on epoch 0 and an `AnchorResult` object thereafter. Unpacking fails every epoch regardless of lag.
    

### Why the async path existed

To prevent anchor round-trip time from inflating node 0's `elapsed_wall` and shrinking its pacing sleep — see [H-02](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-02).

### Resolution

Run with `ANCHOR_ASYNC=0`. `main.py` then assigns `self.anchor = _real_anchor` directly.

The wrapper is **left in place but unused**. Do not enable it without first matching `_dispatch_anchor`'s signature and returning a `(bool, int)` tuple.

### Trap

`run_variety.sh` line:

```bash
export ANCHOR_ASYNC="${ANCHOR_ASYNC:-1}"
```

Default is **1**. `run_experiment.sh` forwards `ANCHOR_ASYNC=${ANCHOR_ASYNC:-1}`. **Always pass `ANCHOR_ASYNC=0` explicitly.**

---

## H-02 — Node-0 chain fork via pacing drift

**Severity:** High · **Status:** Fixed (`_resync_prev_beacon`)

### Symptom

`base_all`, `c_min`, `c_half`: agreement broken, `distinct_nodes=1`, `top=[(0, 47)]`. Node 0 alone diverged; other 37 achieved 50/50.

Log evidence: epoch 3, dealer 12 absent from node 0's AGREE set only.

### Root cause

In `main.py::start()`:

```python
elapsed_wall = time.time() - epoch_wall_start
sleep_s = max(0.0, EPOCH_DURATION_SEC - elapsed_wall)
```

Node 0 carries the anchor. With `EPOCH_SEC=30` and RTT ~2.7 s plus ~19 s protocol work, the epoch 2→3 gap reached **38.1 s** — `sleep_s` collapsed to zero. Node 0 drifted off the `EPOCH_ALIGN_SEC=10` grid and dealer 12's commit arrived outside its window.

Once the AGREE sets differ, `_freeze_commit_transcript` produces a different `commit_digest`, which feeds `derive_beacon_output_v2`. Because `_prev_beacon` chains forward, **one divergent epoch forks the node permanently.**

### Fix

`QRNGBeacon._resync_prev_beacon()` — after a skipped epoch, poll peers' `GET /beacon`, tally `beacon_output_hex` for `epoch_id == self._epoch_id - 1`, adopt if a value has ≥ `t+1` votes.

Requires `transport.py::publish_beacon_state()` plus `_last_beacon_hex` / `_last_beacon_epoch`, and a `do_GET` that returns them with `Content-Length`.

### Limitation

**Only fires on the `result is None` path.** In `base_all`, node 0 never failed — it completed and silently computed a different digest. Silent divergence is not covered here; see [H-06](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-06).

---

## H-03 — Tunnel port confusion

**Severity:** Medium · **Status:** Resolved (convention fixed)

### Symptom

```text
[anchor-tunnel] Checking Grenoble frontend tunnel 127.0.0.1:19100...
ConnectionRefusedError: [Errno 111] Connection refused
```

### Root cause

Three ports in play, mismatched across attempts:

|Port|Location|Meaning|
|---|---|---|
|`9100`|local PC|gateway listener|
|`19100`|frontend|reverse-tunnel endpoint|
|`29100`|node|forwarded anchor port|

`anchor_tunnel.sh` expects `Front port: 19100`, `Node port: 29100`. Earlier attempts used `29100` on the frontend.

### Correct topology

```text
node-a8-N:127.0.0.1:29100
    ↓  anchor_tunnel.sh (frontend → node forward)
frontend:127.0.0.1:19100
    ↓  reverse SSH from local PC
local PC:127.0.0.1:9100
    ↓  serial
Nordic nRF52840 /dev/ttyACM0
```

`ANCHOR_TCP` must be **`127.0.0.1:29100`** — it is evaluated on the node.

---

## H-04 — `127.0.0.1` evaluated in node context

**Severity:** High · **Status:** Resolved

### Symptom

Every node log:

```text
[main] anchor init failed: [Errno 111] Connection refused
```

Run used `--anchor-tcp 127.0.0.1:19100`.

### Root cause

`run_experiment.sh` injects `ANCHOR_TCP` into the **node's** environment via SSH. On node-a8-7, `127.0.0.1` is node-a8-7. The tunnel endpoint at `19100` lives on the frontend, not the node.

### Resolution

Either forward `19100` → node-local `29100` via `anchor_tunnel.sh` and set `ANCHOR_TCP=127.0.0.1:29100`, or use the frontend's `10.0.12.x` address directly.

### Note

The error appeared in all 38 node folders because FiT-IoT collection copies every logical node's log into each physical node directory. Only logical node 0 actually attaches the anchor.

---

## H-05 — Reverse tunnel death at ~3.3 hours

**Severity:** Critical · **Status:** Operational fix

### Symptom

`master_comparison.csv`, in run order:

```text
run                anchor_rtt_ms  pass_rate  anchor_board_us
c_75_v2                  2627       1.00          657465
wh1_all_v2              24696       0.20          132167
cmin_wh1_v2               269       0.00              -1
whlight_all_v2            295       0.00              -1
chalf_whlight_v2          277       0.00              -1
wht_all_v2                596       0.00              -1
chalf_wht_v2              312       0.00              -1
c75_wht_v2                288       0.00              -1
whover_all_v2             270       0.00              -1
```

Two runs at 120 s × 50 epochs ≈ **3.3 h**, matching the observed broken pipe.

### Reading the numbers

- `c_75_v2`: RTT 2,627 ms — healthy.
- `wh1_all_v2`: RTT **24,696 ms**, 20% pass — tunnel degrading, caught mid-run.
- All later: RTT 270–600 ms with **0%** pass — socket fails fast. This is connect-refused/EOF, **not** a board round-trip. Board time is ~660 ms; a real round-trip cannot complete in 270 ms.
- `wh1_all_v2`'s `board 132167 µs` is 10 passing epochs averaged with failures, not a measurement.

### Root cause

Version 2 of the tunnel used:

```bash
-o ServerAliveInterval=30 -o ServerAliveCountMax=120
```

That **detects** a dead peer but cannot **reconnect**. `ServerAliveCountMax=120` also means a 60-minute tolerance window before SSH even gives up.

### Fix

```bash
while true; do
  echo "[anchor-tunnel] starting $(date)"
  ssh -N -T \
    -o ExitOnForwardFailure=yes \
    -o ServerAliveInterval=15 \
    -o ServerAliveCountMax=8 \
    -o TCPKeepAlive=yes \
    -o ConnectTimeout=15 \
    -o Compression=no \
    -R 127.0.0.1:19100:127.0.0.1:9100 \
    siddas@grenoble.iot-lab.info
  echo "[anchor-tunnel] dropped $(date), restarting in 5s"
  sleep 5
done
```

`ServerAliveCountMax=8` → 2-minute detection instead of 60 minutes.

### Related: transport saturation

Earlier, while streaming video on the same link:

```text
board=660491us gateway=38525.96ms serial_write=1516.23ms serial_read=709.41ms
board=660552us gateway=89699.08ms serial_write=1516.26ms serial_read=709.53ms
```

Board 660 ms + serial 2.23 s ≈ 2.9 s accounted; **35–87 s** was `tcp_read_ms`. Idle RTT is ~2.6 s. **Do not saturate the uplink during runs.**

### Important: this did NOT break consensus

```text
run                anchor  strict_agree  epoch_skips  unrecovered
whlight_all_v2     dead     48/50            0             5
chalf_whlight_v2   dead     50/50            0             0
```

Both ran with the anchor completely offline. `chalf_whlight_v2` recovered 11,400/11,400 withheld contributions with perfect agreement.

`epoch.py::_dispatch_anchor` catches all exceptions; the anchor result is **not** an input to `derive_beacon_output_v2`. **Anchor availability and consensus liveness are independent** — and `chalf_whlight_v2` is direct evidence, worth reporting.

---

## H-06 — Silent chain fork under threshold-tight committee

**Severity:** Critical · **Status:** Fixed (`_finalize_v2` guard) · **Most serious finding**

### Symptom

```text
run           committee  withheld  strict_agree  epoch_skips  min_present  recovered  unrecovered
cmin_wh1_v2      13          1         0/50           0           38         1174        726
```

Zero skips. All 38 nodes present in every epoch. Anchor irrelevant. **Zero agreement.**

### Root cause

`C_MIN = KMIN = t + 1 = 13`. In `epoch.py`, PQ dealing targets only committee members:

```python
receiver_ids = sorted(self._committee)
```

Non-committee nodes hold **no shares** and cannot participate in reconstruction. With one withholder:

- Committee members (13) recover → dealer in `self._reveals`
- Non-committee nodes (25) fail → dealer absent from `self._reveals`

`_finalize_v2` built `contributions_by_node` directly from `self._reveals`, so two populations produced different `entropy_digest` → different `derive_beacon_output_v2` output.

1,174 recovered + 726 unrecovered = 1,900 = 38 nodes × 50 epochs. Consistent with a clean split.

**Contrast:** `chalf_whlight_v2` at c=19 with 6 withholders recovered 11,400/11,400 at 50/50. The failure is specific to `c = t+1`, which leaves zero margin.

### Fix

In `_finalize_v2`, after the `RuntimeError` guards, before `contributions_by_node`:

```python
missing = set(self._commits.keys()) - set(self._reveals.keys())
if missing:
    raise EpochSkip(
        f"[Epoch {self.epoch_id}] unrecovered dealers in frozen set: "
        f"{sorted(missing)} — aborting to prevent silent chain fork"
    )
```

`EpochSkip` propagates to `main.py::_run_epoch`, returns `(None, epoch)`, triggers `_resync_prev_beacon()`.

### Rationale

**A skip is recoverable. A silent fork is not.** The frozen AGREE set is the canonical transcript; a node that cannot reproduce it must refuse to emit output rather than emit a different one.

### Expected side-effect

`whover_all_v2` (13 withholders > t=12) will now skip every epoch instead of diverging silently. **This is correct** — it demonstrates the threshold bound empirically and should be reported as a positive result.

### Operational guard

`run_variety.sh` now warns when `C_MIN == KMIN`.

---

## H-07 — Serial recovery cost scaling

**Severity:** High · **Status:** Fixed (batched recovery)

### Symptom

```text
run                n_withheld  reveal_mean_ms  epoch_skips  strict_agree  min_present
wh1_all_v2              1           34374         3686          21/50         35
whlight_all_v2          6           78968            0          48/50         38
wht_all_v2             12          145757        27816           1/50         28
whover_all_v2          13          154574        35036           1/50         26
```

≈ **11.5 s per withheld dealer**, linear.

### Root cause

`_recover_withheld` called `collect_shares` once per dealer. Each call broadcasts `SHARE_REQUEST` to 37 peers and blocks up to `RECONSTRUCTION_TIMEOUT`. Twelve dealers → twelve sequential broadcast-and-wait cycles.

At 145 s reveal against `EPOCH_SEC=120`, `sleep_s` collapses, nodes drift off the `EPOCH_ALIGN_SEC=10` grid, `_wait_for_commits` raises `EpochSkip` for late arrivals.

**This is protocol-under-load, not anchor-related.** `chalf_whlight_v2` had a dead anchor and zero skips.

### Fix

**`transport.py`** — three additions:

- `collect_shares_batch(epoch_id, target_node_ids, timeout, min_shares)` — one `SHARE_REQUEST_BATCH` for all dealers, returns `{dealer_id: [share, ...]}`.
- `_handle_share_request_batch(req)` — serves all requested dealers in one `SHARE_BATCH`, with the same per-share validation as the single-dealer path.
- Router branches for `SHARE_REQUEST_BATCH` and `SHARE_BATCH`.

Legacy `SHARE_REQUEST` / `SHARE` handlers retained for mixed-version safety.

**`epoch.py`** — `_recover_withheld` now fetches once before the loop:

```python
batched = self.network.collect_shares_batch(
    epoch_id=self.epoch_id,
    target_node_ids=sorted(withheld),
    timeout=RECONSTRUCTION_TIMEOUT,
    min_shares=self.t + 1,
)

for target_nid in sorted(withheld):
    shares = batched.get(target_nid, [])
```

All Merkle/classical verification and reconstruction logic unchanged.

### Expected effect

Reveal phase at 12 withholders: **145 s → 5–15 s** (one timeout window plus verification). Note `RECONSTRUCTION_TIMEOUT` defaults to 4 s and is not scaled by dealer count — if batch fetches time out at high withholder counts, raise it rather than reverting to serial.

---

## Validation gates

Both must pass before any long sweep.

```bash
export NODES="7 8 9 10 11 12 13 14 15 16 17 44 45 46 47 48 49 50 51 52 53 54 122 123 124 125 126 127 128 129 130 222 223 224 225 226 227 228"
```

### Gate 1 — batched recovery at t=12

```bash
COMMITTEE_SIZE=0 WITHHOLD_NODES="3,6,9,12,15,18,21,24,27,30,33,36" \
ANCHOR_ASYNC=0 DELAY_SCHEME=hashdelay HASHDELAY_T=16384 \
HASHDELAY_ANCHOR_TRACE=1 HASHDELAY_ANCHOR_S=16 HASHDELAY_ANCHOR_K=12 \
BEACON_NET_PREFIX=10.0.12 MONITOR_SEC_PER_EPOCH=300 \
bash run_experiment.sh --mode qrng --epochs 8 --nodes "$NODES" \
  --crypto-profile post_quantum --anchor-node 0 \
  --anchor-tcp 127.0.0.1:29100 --epoch-sec 120 \
  --qrng-prefetch 1 --tag gate1_batch_wht
```

|Metric|Before|Pass|
|---|---|---|
|`reveal_mean_ms`|145,757|< 20,000|
|`epoch_skips`|27,816|0|
|`strict_agree`|1/50|8/8|
|`anchor_pass_rate`|0.00|1.00|

### Gate 2 — `cmin_wh1` correctness

```bash
COMMITTEE_SIZE=13 WITHHOLD_NODES="19" \
ANCHOR_ASYNC=0 DELAY_SCHEME=hashdelay HASHDELAY_T=16384 \
HASHDELAY_ANCHOR_TRACE=1 HASHDELAY_ANCHOR_S=16 HASHDELAY_ANCHOR_K=12 \
BEACON_NET_PREFIX=10.0.12 MONITOR_SEC_PER_EPOCH=300 \
bash run_experiment.sh --mode qrng --epochs 8 --nodes "$NODES" \
  --crypto-profile post_quantum --anchor-node 0 \
  --anchor-tcp 127.0.0.1:29100 --epoch-sec 120 \
  --qrng-prefetch 1 --tag gate2_cmin_wh1
```

**Pass:** `strict_agree = 8/8`, **or** explicit skips with `disagree = 0`. **Fail:** any epoch with `distinct_outputs > 1`. Silent divergence is the one unacceptable outcome.

### Check

```bash
python3 analyze_run_all.py ~/results/*gate1_batch_wht
python3 analyze_run_all.py ~/results/*gate2_cmin_wh1
```

---

## Pre-launch checklist

**Local PC, terminal 1 — gateway**

```bash
cd ~/Work/QBeacon_Sat/network
python3 -u anchor_tcp_gateway.py --serial /dev/ttyACM0 \
  --listen-host 127.0.0.1 --listen-port 9100 \
  --chunk-size 16 --chunk-delay 0.001 --serial-timeout 180 \
  |& tee ~/hd_gateway_$(date +%Y%m%d_%H%M).log
```

**Local PC, terminal 2 — reconnecting tunnel** (see [H-05](https://you.com/search?q=Anchor+Integration+Design&cfr=chat&cid=c0_4ac7fc0d-cc92-43e7-9daf-8d201859e791&chatMode=custom#h-05))

**Frontend**

```bash
python3 -c "import socket; socket.create_connection(('127.0.0.1',19100),timeout=3); print('OK')"
cd ~/QBeacon_Sat && ./anchor_tunnel.sh
ssh root@node-a8-7 "python3 -c \"import socket; socket.create_connection(('127.0.0.1',29100),timeout=3); print('OK')\""
```




# AeroPQ / QBeacon — Consensus Divergence Post-Mortem

> **Scope.** Root-cause analysis and remediation log for the multi-output ("fork") condition observed during the 38-node FIT IoT-LAB Grenoble campaign, 30 Sep – 2 Oct 2026. Written for future system-tuning reference. **Status.** Resolved. Paper submitted. **Tags.** `#qbeacon` `#postmortem` `#distributed-systems` `#consensus` `#iotlab` `#tuning`

---

## 1. Executive Summary

A 38-node post-quantum randomness beacon was producing **multiple distinct `beacon_output_hex` values for the same epoch** — a chain fork. Baseline runs showed `strict_complete_and_agree = 0/50` and `distinct_outputs = 2` from epoch 1 onward.

The root cause was **not** a cryptographic flaw. It was a **race between transcript freezing and commit arrival**: nodes froze non-identical commit sets, producing divergent `CommitDigest_e`, which propagated into `DelayInput_e` and `beacon_output`, and then chained forward irrecoverably.

Resolution required **five distinct fixes** applied across three files over four validation cycles. Final state: `max_distinct = 1` on every recorded epoch across six fork-free runs, with `sum_withheld_unrecovered = 0` across all withholding levels up to t+1.

**Key lesson:** the correct fix was _not_ to demand complete arrival (that caused availability collapse) but to let the **AGREE round** — not arrival completeness — enforce transcript identity, backed by a `2t+1` quorum floor.

---

## 2. System Under Test

|Parameter|Value|
|---|---|
|Nodes|n = 38 (IoT-LAB A8, Grenoble)|
|Byzantine threshold|t = ⌊(n−1)/3⌋ = **12**|
|Reconstruction threshold|k = t+1 = **13**|
|Quorum|2t+1 = **25**|
|Crypto profile|`post_quantum` — Merkle-tree VSS (`merkle_share_vss_v1`), SHA3-256|
|Delay function|HashDelay, `post_quantum:hashdelay:sha3_256`, **T = 16384**|
|Entropy|IDQ QRNG over HTTPS, `QRNG_PREFETCH=1`|
|Anchor|nRF52840, node 0 only, TCP bridge `127.0.0.1:29100`|
|Epoch state machine|`IDLE → COMMITTING → REVEALING → RECOVERING → FINALIZED \| FAILED`|

### Output binding chain

```
CommitDigest_e = H(domain ‖ epoch ‖ prev_beacon ‖ canonical_params
                   ‖ count ‖ ⋃_sorted(node_id ‖ commitment ‖ public_vss))

DelayInput_e   = H(domain ‖ epoch ‖ prev_beacon ‖ CommitDigest_e
                   ‖ delay_mode ‖ delay_T)

beacon_output  = derive_beacon_output_v2(epoch, prev_beacon, CommitDigest_e,
                                         entropy_digest, delay_mode, delay_output)
```

**Critical property:** `CommitDigest_e` is a function of the _frozen dealer set_. Any disagreement on that set — even by one dealer — yields a different digest, a different delay input, and a different beacon. And because `prev_beacon` feeds the next epoch, divergence is **permanent without resync**.

---

## 3. The Two Agreement Metrics

Understanding the whole post-mortem depends on this distinction. From `analyze_run_all.py → build_epoch_agreement()`:

|Column|Meaning|
|---|---|
|`output_agreement_among_present`|1 iff all nodes that _recorded a row_ produced one `beacon_output_hex` — **safety**|
|`strict_complete_and_agree`|1 iff agreement **AND** `len(nodes_present) == 38` — **safety + liveness**|
|`transcript_complete`|1 iff `min(n_contributors) ≥ 38`|
|`distinct_outputs`|unique outputs per epoch — **`max_distinct == 1` means no fork**|
|`sum_withheld_unrecovered`|dealers in frozen set never reconstructed — **must be 0**|

> **Reading rule.** Low `strict` with high `present_agree` is the **fail-stop guard working**, not degradation. A node that cannot see the full transcript raises `EpochSkip` and writes **no CSV row** — `metrics.record()` fires only when `result is not None`. Epoch counts are therefore _recorded_, not _attempted_.

---

## 4. Issue Log

### ISSUE-1 — Deaf node (inbound transport failure)

**Symptom.** Node 5 logged `Insufficient commits: 1/13` for three consecutive epochs while 37 nodes agreed perfectly with identical `commit_digest`.

**Diagnostic trail.**

```
20:30:19  [PeerWait] All 37 peers ready
20:30:33  Epoch 0 skipped — resyncing chain state with peers
20:30:40  [resync] no peer beacon state available
20:31:04  Epoch 1 SKIPPED: Insufficient commits: 1/13
20:31:16  [resync] no epoch reached t+1 agreement
20:31:38  [resync] chain desync repaired: 0000... -> 205335c1... (37/37 peers)
20:32:02  Epoch 1 SKIPPED: Insufficient commits: 1/13
```

**The asymmetry is the diagnosis.** Node 5 could _send_ to 37 peers (its commit appeared in everyone's AGREE set) but _received_ from zero. Outbound worked; inbound did not.

**Root cause.** `_peer_reachable` only tests outbound. Nothing verified that node 5's own `ThreadingHTTPServer` was reachable _from_ peers. Two contributing factors:

1. **Late bind.** `run_experiment.sh` launched 38 nodes via sequential `ssh -n -f`. Node 5 (6th launch) may have bound its port after peers had already fired epoch-0 commits. `transport._send` retried `MAX_RETRIES` then dropped — the commit was lost permanently.
2. **Port held by dying process.** Step 1 did `pkill` then `sleep 2`, but `main.py`'s `finally` block sleeps `RECONSTRUCTION_TIMEOUT + 2 = 27 s` before `network.stop()`. A node still in that window holds its socket.

**Secondary finding.** `setup_logging` sets the file handler to `WARNING`, so `logger.info` bind confirmations never reached `node_N.log` — the failure was invisible.

**Fix.**

```python
# transport.py — start()
def start(self):
    port = self.nodes[self.node_id]["port"]
    ThreadingHTTPServer.allow_reuse_address = True
    try:
        server = ThreadingHTTPServer(("0.0.0.0", port), BeaconRequestHandler)
    except OSError as e:
        logger.error(f"[Net] FATAL: node {self.node_id} cannot bind :{port} — {e}")
        raise
    ...
    logger.warning(f"[Net] Node {self.node_id} LISTENING on :{port}")   # WARNING, not INFO
```

```bash
# run_experiment.sh — Step 1
sleep 30        # was: sleep 2
```

**Verification.** `patchcheck2` onward: zero `Insufficient commits: 1/` across all node logs.

---

### ISSUE-2 — Serial resync poll (55-second blocking window)

**Symptom.** A skipped node's recovery attempt failed twice before succeeding on the third try.

**Root cause.** `_resync_prev_beacon` polled 37 peers **sequentially** with `requests.get(..., timeout=1.5)` → up to **55 s** of blocking. Peers advanced mid-poll; `publish_beacon_state` only updates after `_run_epoch` returns. By the time the poll completed, the sampled states were stale and inconsistent.

**Fix.**

```python
# main.py — _resync_prev_beacon()
from concurrent.futures import ThreadPoolExecutor

def _poll(p):
    try:
        r = req.get(f"http://{p['host']}:{p['port']}/beacon", timeout=1.5)
        return r.json() if r.status_code == 200 else None
    except Exception:
        return None

with ThreadPoolExecutor(max_workers=max(1, len(peers))) as ex:
    responses = list(ex.map(_poll, peers))
```

**Effect.** 55 s → ~2 s. Mirrors the threading already used in `_broadcast_all`.

---

### ISSUE-3 — Single-shot resync (no retry)

**Symptom.** Node adopted genesis `b"\x00"*32` as `prev_beacon` and chained forward on a private fork forever.

**Root cause.** One resync attempt, then the epoch counter advanced regardless.

**Fix.**

```python
# main.py — start(), skip branch
for attempt in range(5):
    if self._resync_prev_beacon():
        break
    self.logger.warning(f"[resync] attempt {attempt+1}/5 failed, retrying in 5s")
    time.sleep(5.0)
```

---

### ISSUE-4 — AGREE threshold too permissive ⭐ PRIMARY ROOT CAUSE

**Symptom.** `base_all`: `distinct_outputs = 2` from epoch 1 onward, with `mean_contributors = 38.0` on **both** branches. Every node claimed a complete 38-dealer transcript, yet produced different outputs.

**Root cause.** The AGREE round retained a dealer if **≥ t+1 = 13 voters** reported the same commitment hash. With `need` set below `self.n`, different nodes completed the AGREE round having heard from _different voter subsets_, and therefore froze _different dealer sets_ — while each still counted 38 contributors locally.

**This was the actual fork mechanism.** Not a crypto bug, not a transport bug — a **threshold-vs-membership race**.

**Fix (part A).**

```python
# epoch.py — collect_commit_sets()
need = self.n          # wait for all 38 voters before resolving the AGREE set
```

**Fix (part B) — make the AGREE set authoritative.** The original code only _dropped_ dealers; it never aborted when the agreed set named a dealer the local node lacked.

```python
# epoch.py — AGREE block in run()
missing = agreed - set(self._commits.keys())
if missing:
    self.state = EpochState.FAILED
    raise EpochSkip(
        f"[Epoch {self.epoch_id}] AGREE set includes dealers we lack: "
        f"{sorted(missing)} — skipping to preserve I6"
    )
```

**Cost.** AGREE round latency rose from ~3.7 s to ~15 s. Against the then-default `COMMIT_WINDOW_SEC=30`, this consumed half the window and produced late-commit rejections (see ISSUE-5).

---

### ISSUE-5 — Window variables never propagated to nodes

**Symptom.** `COMMIT_WINDOW_SEC=45` set on the command line had no effect. Logs showed `Ignoring late commit from 35 (transcript already frozen)` with `commit_phase_ms = 27,623` against a 30 s window.

**Root cause.** `run_experiment.sh` Step 3's SSH env block exported `COMMITTEE_SIZE`, `WITHHOLD_NODES`, `ANCHOR_ASYNC`, `DELAY_SCHEME`, `HASHDELAY_*` — but **not** the window variables. Nodes silently used defaults.

**Fix.**

```bash
# run_experiment.sh — Step 3 SSH env block
EPOCH_DURATION_SEC=${EPOCH_SEC} \
COMMIT_WINDOW_SEC=${COMMIT_WINDOW_SEC:-30} \
REVEAL_WINDOW_SEC=${REVEAL_WINDOW_SEC:-20} \
RECONSTRUCTION_TIMEOUT=${RECONSTRUCTION_TIMEOUT:-25} \
QRNG_PREFETCH=${QRNG_PREFETCH} \
```

Also recorded in `experiment_info.txt` so configuration is auditable post-hoc.

> **Generalised lesson.** Any tunable that does not appear in the remote-launch env block is a **silent default**. Audit the full env surface, and echo every effective value into the run manifest.

---

### ISSUE-6 — Over-strict commit gate → availability collapse ⭐ MOST INSTRUCTIVE

**Symptom.** After requiring all `self.n` commits, forks vanished — but **`base_all` died at epoch 31 of 50** with monotonic attrition:

```
epoch 0–4 : 38 present
epoch 5–6 : 37
epoch 7   : 36
epoch 8   : 35
epoch 9   : 34      ... and so on, never recovering
```

Skip counts across all runs: **186,793 `below quorum`**, **41,034 `unrecovered dealers in frozen set`**, **4,473 `AGREE set includes dealers we lack`**.

**The decisive signal:** at the time, `incomplete commit set` skips numbered **100,310** against **zero** AGREE-set violations. The AGREE round was not the bottleneck. **Arrival completeness was.**

**Root cause — the death spiral.**

```
node misses 1 of 38 commits
  → _wait_for_commits raises EpochSkip (n_commits < self.n)
  → writes no row, enters resync
  → but _resync_prev_beacon needs 2t+1 = 25 peers with current published state
  → during mass skipping, fewer than 25 have published
  → all 5 retries fail
  → node falls further behind
  → skips again next epoch  ⟲
```

Skips clustered on the **highest logical IDs** — node36 (108,781), node33 (99,421), node34 (98,537), node35 (85,006). These are launched last by the sequential SSH loop, start latest, and never catch up.

**Fix — relax to quorum, let AGREE enforce identity.**

```python
# epoch.py — _wait_for_commits()
n_commits = len(self._commits)
quorum = 2 * self.t + 1          # 25 of 38

if n_commits < quorum:
    self.state = EpochState.FAILED
    raise EpochSkip(
        f"[Epoch {self.epoch_id}] below quorum: {n_commits}/{quorum}"
    )
```

**Why this is safe.** I6 is enforced by the AGREE round's `t+1`-vote threshold **plus** the `missing` abort — nodes must _agree_ on the frozen set, not all receive it simultaneously. Requiring universal arrival over HTTP at n=38 is neither necessary nor achievable.

**Fix — degraded-quorum resync fallback.** `2t+1` is correct for fork rejection but unreachable during mass skipping:

```python
# main.py — _resync_prev_beacon()
if votes is None:
    for ep in sorted(by_epoch.keys(), reverse=True):
        c = by_epoch[ep]
        top_hex, top_n = c.most_common(1)[0]
        if top_n >= self.t + 1 and (
            len(c) == 1 or c.most_common(2)[1][1] < self.t + 1
        ):
            self.logger.warning(
                f"[resync] degraded quorum accept at epoch {ep}: {top_n} votes"
            )
            votes = c
            self._epoch_id = ep + 1
            break

if votes is None:
    self.logger.warning("[resync] no epoch reached quorum agreement")
    return False
```

> **Note on the double `if votes is None`.** Intentional. The first block _may assign_ `votes` via the fallback; the second checks whether the fallback also failed. Collapsing them would return `False` before the fallback ran. A `for...else` is the equivalent single-block form.

---

### ISSUE-7 — Wrong delay scheme (silent default)

**Symptom.** `patchcheck2` reported `board = 68,359 µs` — 10× faster than the established ~660,000 µs population.

**Root cause.** `delay_mode = legacy:post_quantum:minroot T=128`. `run_experiment.sh` defaults `DELAY_SCHEME=${DELAY_SCHEME:-minroot}`; invoking it directly (not via `run_variety.sh`) never set HashDelay.

**Impact.** Those board times are **not comparable** to the HashDelay population. Run excluded from all figures.

**Fix.** Always pass explicitly:

```bash
DELAY_SCHEME=hashdelay HASHDELAY_T=16384
```

---

### ISSUE-8 — Bash substring-expansion bug

```bash
export ANCHOR_ASYNC="${ANCHOR_ASYNC: -0}"      # WRONG — substring with offset -0
export ANCHOR_ASYNC="${ANCHOR_ASYNC:-0}"       # RIGHT — default value
```

`${VAR: -0}` is _substring expansion_, returning the whole string (or empty if unset) — **not** a default. With `ANCHOR_ASYNC` unset, `run_experiment.sh` fell back to `ANCHOR_ASYNC=1`, meaning `_AsyncAnchor.verify()` returned the **previous** epoch's result. Anchor numbers would be lagged by one epoch.

> **Tuning rule.** The space in `${VAR: -0}` changes the operator entirely. Grep every config script for `: -` versus `:-`.

---

### ISSUE-9 — Analyzer averaging failure rows

**Symptom.** `summary.json` reported `board_us = 330,316` for `wht_all_v2` while the console printed the correct ~660,000.

**Root cause.** Failed anchor rows carry `anchor_verify_us = -1`. `main()` filtered with `> 0`; `write_summary()` did not:

```python
anchor["board_us"] = safe_stats(ag["anchor_verify_us"])                       # WRONG
anchor["board_us"] = safe_stats(ag[ag["anchor_verify_us"] > 0]["anchor_verify_us"])  # RIGHT
```

At ~50% pass rate the mean halves. **Rule: sentinel values must be filtered at every aggregation site, not just the one you're looking at.**

---

## 5. Remediation Summary

|#|File|Location|Change|
|---|---|---|---|
|1|`transport.py`|`start()`|Catch `OSError` on bind; log at `WARNING`|
|2|`run_experiment.sh`|Step 1|`sleep 2` → `sleep 30`|
|3|`main.py`|`_resync_prev_beacon()`|`ThreadPoolExecutor` parallel poll|
|4|`main.py`|`start()` skip branch|5× retry loop, 5 s backoff|
|5|`main.py`|`_resync_prev_beacon()`|Degraded-quorum fallback at `t+1` unambiguous|
|6|`epoch.py`|`collect_commit_sets()`|`need = self.n`|
|7|`epoch.py`|AGREE block in `run()`|`missing` → `EpochSkip`|
|8|`epoch.py`|`_wait_for_commits()`|`self.n` → `2t+1 = 25` quorum|
|9|`run_experiment.sh`|Step 3 SSH env|Propagate window vars + record in manifest|
|10|`run_variety.sh`|line 48|`${VAR: -0}` → `${VAR:-0}`|
|11|`analyze_run_all.py`|`write_summary()`|Filter `anchor_verify_us > 0`|

### Deployment verification (mandatory — one unpatched node reforks the chain)

```bash
for n in $NODES; do
  a=$(ssh -o ConnectTimeout=5 root@node-a8-$n \
      "grep -c 'below quorum' ~/QBeacon_Sat/beacon_core/epoch.py" 2>/dev/null)
  b=$(ssh -o ConnectTimeout=5 root@node-a8-$n \
      "grep -c 'ThreadPoolExecutor' ~/QBeacon_Sat/main.py" 2>/dev/null)
  c=$(ssh -o ConnectTimeout=5 root@node-a8-$n \
      "grep -c 'degraded quorum accept' ~/QBeacon_Sat/main.py" 2>/dev/null)
  printf "node-a8-%-4s quorum=%s pool=%s fallback=%s\n" "$n" "${a:-X}" "${b:-X}" "${c:-X}"
done
```

---

## 6. Validation Cycles

|Cycle|Config|Result|
|---|---|---|
|`patchcheck`|MinRoot T=128, 5 ep|Node 5 deaf, `Insufficient commits: 1/13` ×3|
|`patchcheck2`|MinRoot T=128, 5 ep|No skips; late-commit rejections from nodes 1, 35|
|`patchcheck3`|HashDelay T=16384, 5 ep|ep0 clean; nodes 33 then 1 forked (3 distinct at ep3/4)|
|`patchcheck4`|+`self.n` gate, 5 ep|**`max_distinct = 1` all 5 epochs**; ep4 node 1 skipped (guard working)|
|`patchcheck5`|+`2t+1` relax, 5 ep|**5/5 strict, 5/5 present_agree, 5/5 anchor PASS**|

> `patchcheck4` epoch 4 is the canonical illustration: node 1 absent, yet all 37 present nodes show `n_recovered = 1`, `n_contributors = 38`. Node 1 skipped rather than finalizing a divergent digest. Cost visible — `total_latency_ms` 14,500 → 45,000, `reveal_phase_ms` ≈ 24,000 as `collect_shares_batch` ran against `RECONSTRUCTION_TIMEOUT = 25`.

---

## 7. Final Campaign Results

### Fork-free runs

|Run|Recorded|Strict|Present-agree|max_distinct|Board µs|RTT ms|
|---|---|---|---|---|---|---|
|`base_all`|31|5|**31/31**|**1**|660,503|2,697|
|`c_full` (c=38)|50|25|**50/50**|**1**|659,752|2,653|
|`wh1_all`|20|4|**20/20**|**1**|660,675|2,760|
|`wht_all` (wh=12)|20|3|**20/20**|**1**|661,468|3,257|
|`whover_all` (wh=13)|21|5|**21/21**|**1**|660,961|3,249|
|`chalf_wht`|50|**0**|**50/50**|**1**|659,128|5,125|

### Withholding degradation — zero unrecovered at every level

|Run|Withheld|Recovered|**Unrecovered**|min_present|
|---|---|---|---|---|
|`wh1_all`|1|1,336|**0**|37|
|`whlight_all`|||||