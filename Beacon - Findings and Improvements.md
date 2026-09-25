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

**Verify before launch**

- [ ]  `ANCHOR_ASYNC=0` on the command line — **not** relying on defaults
- [ ]  `ANCHOR_TCP=127.0.0.1:29100` (node-local, not `19100`)
- [ ]  Node 0 log shows `hardware anchor attached (SYNC)`
- [ ]  Uplink not saturated



