# Anchor Integration Journal — Classical + MinRoot Hardware Anchor

Date: 2026-09-10  
Board: Nordic nRF9151-SMA-DK  
Host: `sid@Gandalf`  
Project: `QBeacon_Sat`  
Firmware path: `~/QBeacon_Sat/anchor_fw`  
Beacon path: `~/QBeacon_Sat`  
Anchor serial port: `/dev/ttyACM0`Tags: `#QBeacon` `#Nordic` `#nRF9151` `#hardware-anchor` `#Wesolowski` `#MinRoot` `#FiTIoT` `#debug-log`

---

## 1. Purpose of this journal entry

This entry records the full hardware-anchor integration process for the QBeacon/AeroPQ beacon protocol:

1. classical Wesolowski anchor integration;
2. Nordic UART bring-up;
3. binary `QBA1`/`QBR1` protocol debugging;
4. SHA3 binding validation;
5. classical hardware verification;
6. 4-node local beacon integration;
7. debug flag hardening;
8. MinRoot firmware-side verifier implementation;
9. MinRoot direct anchor verification;
10. UART throughput tuning;
11. implications for FiT-IoT deployment.

I am keeping this note because developer journals are useful for recording decisions, attempted fixes, failures, final conclusions, and details that are easy to forget later 

[](https://ttkalcevic.medium.com/developer-journal-65a08cecb8ca?utm_source=you.com)

.

---

## 2. High-level result

Both hardware-anchor paths now work:

text

```text
Classical Wesolowski direct anchor: PASS
Classical 4-node local-test anchor: PASS
MinRoot direct anchor: PASS
MinRoot optimized UART setting: PASS at ANCHOR_UART_CHUNK=16, ANCHOR_UART_CHUNK_DELAY=0.001
```

The classical Nordic verification time is roughly:


```text
~0.86–0.95 seconds
```

The MinRoot Nordic verification time for the first tested parameter set was:


```text
~75 ms device-side
```

The MinRoot host roundtrip initially was:

text

```text
~4.5 seconds
```

but was reduced to:

text

```text
~1.0714 seconds with chunk=16 delay=0.002
~0.657 seconds with chunk=16 delay=0.001
```

The remaining gap between device-side MinRoot verification and host roundtrip is mostly UART transfer/throttling overhead, not cryptographic verification.

---

## 3. Core architectural invariants

### 3.1 Anchor must not affect beacon output

The anchor is strictly post-finalization. In `epoch.py`, the beacon output is computed first:

python

```python
beacon_output, entropy_seed = self._finalize_v2()
vdf_proof = self._delay_proof
t_finalize = time.time()
self.state = EpochState.FINALIZED
```

Only after that does the epoch dispatch the hardware anchor:

python

```python
if self.anchor is not None and self._delay_proof is not None:
    self._dispatch_anchor(self._delay_proof)
```

Therefore, anchor pass/fail cannot feed back into `beacon_output` or `entropy_seed` 

.

### 3.2 Delay input is bound to frozen commit transcript

The delay input is derived from:

text

```text
domain = AEROPQ/DELAY_INPUT/v1
epoch
previous_beacon
commit_digest
delay_mode
delay_T
```

as implemented in `_derive_delay_input()` 

.This matters because the hardware anchor verifies delay evidence against the same frozen context. The delay input is not derived from post-reveal entropy; it is derived after the commit set is frozen and before reveal processing completes 

.

### 3.3 Classical and post-quantum profile selection

In `BeaconEpoch.__init__()`, the active delay mode and delay parameter are selected from `CRYPTO_PROFILE`:

text

```text
classical      -> legacy:classical:wesolowski, T = VDF_DELAY_T
post_quantum   -> legacy:post_quantum:minroot, T = MINROOT_T
hybrid         -> legacy:hybrid:minroot, T = MINROOT_T
```

This ensures the recorded delay mode and delay parameter match the actual delay code path 

.

### 3.4 Anchor dispatch profiles

In `_dispatch_anchor()`, classical proofs are sent with `profile="classical"` and post-quantum/hybrid proofs are sent with `profile="post_quantum"` 

.For classical, Python computes:

python

```python
l = w._fiat_shamir_prime(proof.x, proof.y)
r = pow(2, proof.T, l)
```

and sends `l,r` to the anchor 

.For post-quantum, Python sends the MinRoot proof and context directly to the anchor bridge 

.

---

## 4. Hardware platform

Board used:

text

```text
nRF9151-SMA-DK
```

The UART path that ultimately worked was:

text

```text
/dev/ttyACM0
```

The working firmware UART device is:

c

```c
uart_dev = DEVICE_DT_GET(DT_NODELABEL(uart0));
```

as seen in `main.c` 

.A temporary `QBAHELLO` firmware was used to prove that `/dev/ttyACM0` was the actual UART/VCOM path. Seeing `QBAHELLO` continuously on `/dev/ttyACM0` confirmed the serial mapping.

---

## 5. Binary anchor protocol

The firmware uses a binary request/response protocol.Request magic:

text

```text
QBA1
```

Response magic:

text

```text
QBR1
```

The firmware scans for `QBA1` byte-by-byte before reading the fixed header. The scanner consumes bytes until it sees:

c

```c
const uint8_t magic[4] = {0x51, 0x42, 0x41, 0x31};
```

and deliberately does not emit `BADREQ` while scanning garbage/stale bytes 

.The firmware asserts the binary ABI sizes:

c

```c
BUILD_ASSERT(sizeof(qba_header_t) == 156, "qba_header_t must be 156 bytes");
BUILD_ASSERT(sizeof(qba_response_t) == 12, "qba_response_t must be 12 bytes");
```

This prevented silent Python/C structure mismatch 

.

---

## 6. Initial blockers and fixes

### 6.1 Missing `qba_sha3.c`

Initial problem:

text

```text
qba_sha3.c was missing.
```

The firmware `CMakeLists.txt` expected it, and the anchor verifier needed SHA3-256 to recompute context bindings. I added `qba_sha3.c` implementing:

c

```c
qba_sha3_256_init
qba_sha3_256_update
qba_sha3_256_final
```

The SHA3 implementation uses SHA3 padding:

c

```c
ctx->buf[ctx->pos] ^= 0x06;
ctx->buf[QBA_SHA3_256_RATE - 1] ^= 0x80;
```

matching the expected SHA3-256 domain padding style.

### 6.2 Board target confusion

Originally I was assuming nRF52840-style flashing. The actual board was:

text

```text
nRF9151-SMA-DK
```

The correct path was to use an nRF9151 Zephyr board target, e.g.:

bash

```bash
west build -b nrf9151dk/nrf9151 -p always .
west flash
```

with the exact target depending on installed NCS/Zephyr board names.

### 6.3 UART path was unknown

Both `/dev/ttyACM0` and `/dev/ttyACM1` initially showed no response. A temporary UART hello firmware was flashed. Seeing:

text

```text
QBAHELLO uart0
```

on `/dev/ttyACM0` proved that the correct UART was `uart0` and the correct Linux serial device was `/dev/ttyACM0`.

### 6.4 VS Code serial monitor interference

Opening the VS Code serial terminal was useful for `QBAHELLO`, but it must not be used during binary anchor tests.Reason:

text

```text
VS Code serial monitor consumes the same QBR1 binary response bytes that Python is waiting for.
```

Final rule:

text

```text
Do not open VS Code serial monitor during QBA binary anchor runs.
```

---

## 7. UART framing failures

### 7.1 Symptom

The board sometimes returned:

text

```text
status=2
```

which mapped to:

text

```text
QBA_STATUS_BADREQ
```

Sometimes Python saw:

text

```text
NO COMPLETE RESPONSE
```

At one point a supposedly valid empty-payload request produced:

text

```text
MAIN_PAYLOAD_TOO_BIG
```

even though Python had sent `payload_len=0`.This indicated the firmware was detecting `QBA1` but reading the rest of the 156-byte header incorrectly.

### 7.2 Root cause

The first UART read loop used a 1 ms sleep while waiting for bytes:

c

```c
while (uart_poll_in(uart_dev, b) != 0) {
    k_sleep(K_MSEC(1));
}
```

At 115200 baud, a byte arrives roughly every 87 µs. Sleeping for 1 ms between polls can overrun the UART/VCOM RX buffering and corrupt the fixed header.

### 7.3 Fix

The UART byte reader was changed to busy-wait briefly:

c

```c
while (uart_poll_in(uart_dev, b) != 0) {
    k_busy_wait(10);
}
```

This fixed header corruption and is now present in `main.c` 

.

### 7.4 Additional host-side mitigation

Python serial writes were chunked to avoid overrunning the polling UART receiver.Conservative initial setting:

bash

```bash
ANCHOR_UART_CHUNK=8
ANCHOR_UART_CHUNK_DELAY=0.005
```

Optimized working setting:

bash

```bash
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.001
```

---

## 8. Debug reason codes

To diagnose where requests failed, the firmware started returning reason codes in the `reserved` field of `qba_response_t`.Useful reason codes:

text

```text
0x1002 MAIN_BAD_VERSION
0x1003 MAIN_PAYLOAD_TOO_BIG
0x2007 ANCHOR_DELAY_INPUT
0x3006 WESO_BAD_R
0x400A MR_BAD_STEP
```

This made it possible to distinguish:

text

```text
bad UART framing
bad header version
payload length corruption
delay-input mismatch
Wesolowski precheck failure
MinRoot step failure
```

instead of seeing only generic `PASS`, `FAIL`, or `BADREQ`.

---

## 9. Debug bypass flags and production hardening

During bring-up, two debug flags existed:

c

```c
QBA_FLAG_DEBUG_EARLY_PASS
QBA_FLAG_DEBUG_SKIP_DELAY
```

They were useful to prove whether the request reached:

text

```text
main.c
qba_anchor.c
delay-input verification
crypto verification
```

But these flags are not acceptable in production if active, because they can bypass real verification.Final hardening:

c

```c
#ifndef QBA_ENABLE_DEBUG_FLAGS
#define QBA_ENABLE_DEBUG_FLAGS 0
#endif
```

The main-level early-pass block is now compiled only under:

c

```c
#if QBA_ENABLE_DEBUG_FLAGS
...
#endif
```

in `main.c` 

.The real anchor bridge sends `flags=0`, so normal requests do not use bypass behavior.Production behavior:

text

```text
QBA_ENABLE_DEBUG_FLAGS = 0
nonzero flags rejected
delay-input always verified
crypto path cannot be bypassed
```

---

## 10. Classical Wesolowski anchor path

### 10.1 What the classical anchor verifies

The classical Nordic firmware verifies:

text

```text
pi^l * x^r mod N == y
```

where:

text

```text
l = Fiat-Shamir challenge
r = 2^T mod l
```

The firmware also checks context binding:

text

```text
x matches delay_input
delay_output_digest matches y
r matches 2^T mod l
```

### 10.2 Important fix: `r = 2^T mod l`

The original `r` check used only a 16-byte accumulator and was unsafe when doubling overflowed 128 bits.It was replaced with a 17-byte modular-doubling accumulator so overflow is handled correctly.This fixed possible false rejection of valid Wesolowski proofs.

### 10.3 Direct classical test result

Direct classical anchor test passed:

text

```text
anchor passed: True
anchor us:     873840
```

This means the full request passed:

text

```text
binary framing
SHA3 delay-input binding
x binding
delay_output_digest binding
r = 2^T mod l check
Wesolowski equation
```

### 10.4 Integrated classical local-test

After fixing local-test pacing and anchor gating, classical local-test passed.Observed node 0 anchor results were around:

text

```text
~0.86–0.95 seconds
```

All four local nodes produced matching outputs per epoch, preserving determinism.

---

## 11. Local-test synchronization issues and fixes

### 11.1 Problem: all nodes tried to open `/dev/ttyACM0`

Initially, all four local worker processes tried to attach to the same hardware anchor.Fix:

bash

```bash
ANCHOR_NODE_ID=0
```

Only node 0 attaches the hardware anchor. Nodes 1–3 disable it.Expected behavior:

text

```text
node0 -> hardware anchor attached
node1 -> anchor configured but disabled
node2 -> anchor configured but disabled
node3 -> anchor configured but disabled
```

### 11.2 Problem: node 0 started late

Node 0 spent extra time opening the serial anchor, while nodes 1–3 entered epoch 0 earlier.Fix:

text

```text
construct QRNGBeacon before the local-test barrier
```

This ensures all workers wait after anchor initialization, not before.

### 11.3 Problem: node 0 drifted later over epochs

Anchor time is intentionally excluded from `total_latency_ms` because the anchor is post-finalize attestation. But using only `total_latency_ms` for epoch pacing caused node 0 to ignore the ~0.9s hardware-anchor wall time and drift later each epoch.Fix:

python

```python
epoch_wall_start = time.time()
result, epoch_handler = self._run_epoch()
elapsed_wall = time.time() - epoch_wall_start
sleep_s = max(0.0, EPOCH_DURATION_SEC - elapsed_wall)
```

This preserves latency metrics while using real wall-clock time for scheduling.

### 11.4 Anchor-aware local-test epoch duration

For anchor tests, local-test should use a larger epoch duration:

bash

```bash
EPOCH_DURATION_SEC=12
```

This gives node 0 enough time to perform hardware attestation without falling behind.

---

## 12. MinRoot anchor path

### 12.1 MinRoot verification model

The Nordic does not compute fifth roots.Instead, it verifies sampled adjacent trace relations:

text

```text
x_next^5 mod p == x_i + i mod p
```

This matches the Python-side MinRoot step check.

### 12.2 MinRoot field

MinRoot uses the secp256k1 field prime:

text

```text
p = 2^256 - 2^32 - 977
```

The firmware checks that the payload field prime matches this expected value.

### 12.3 What the MinRoot payload contains

The Python anchor bridge serializes a MinRoot audit payload containing:

text

```text
T
S
K
M
depth
p
x
y
trace_root
for each challenged segment:
    j
    step_start
    step_count
    states[0..step_count]
    path_start
    path_end
```

### 12.4 What the Nordic MinRoot verifier checks

The C verifier checks:

text

```text
payload length / parser bounds
T matches header delay_T
S, K, M, depth are within firmware limits
M == ceil(T / S)
p equals expected secp256k1 prime
x == int(delay_input) mod p, with zero mapped to one
delay_output_digest matches y
anchor context hash
challenge derivation
segment index matches derived challenge
Merkle path for segment start checkpoint
Merkle path for segment end checkpoint
every adjacent step relation:
    x_next^5 mod p == x_i + step_index mod p
no trailing bytes
```

### 12.5 Firmware implementation approach

Instead of writing a new optimized finite-field library from scratch, I reused the style of the existing C bignum/division scheme and implemented a smaller 256-bit field backend for MinRoot.This was acceptable because MinRoot verification only needs:

text

```text
256-bit modular add
256-bit modular multiply
x^5 mod p
SHA3-256
Merkle path verification
bounded payload parsing
```

No 2048-bit RSA arithmetic is needed for MinRoot.

### 12.6 MinRoot reason codes

Added MinRoot-specific failure reasons:

text

```text
0x4001 MR_BAD_PAYLOAD_LEN
0x4002 MR_BAD_T
0x4003 MR_BAD_PARAMS
0x4004 MR_BAD_FIELD
0x4005 MR_BAD_X_BINDING
0x4006 MR_BAD_OUTPUT
0x4007 MR_BAD_CHALLENGE
0x4008 MR_BAD_MERKLE_START
0x4009 MR_BAD_MERKLE_END
0x400A MR_BAD_STEP
0x400B MR_TRAILING_BYTES
```

These are essential for debugging failed MinRoot proofs.

---

## 13. Direct MinRoot test result

First direct MinRoot test:

bash

```bash
ANCHOR_PORT=/dev/ttyACM0 \
BEACON_SKIP_DISCOVERY=1 \
CRYPTO_PROFILE=post_quantum \
MINROOT_T=128 \
MINROOT_ANCHOR_TRACE=1 \
MINROOT_ANCHOR_S=16 \
MINROOT_ANCHOR_K=4 \
ANCHOR_UART_CHUNK=8 \
ANCHOR_UART_CHUNK_DELAY=0.005 \
python -u test_anchor_minroot.py
```

Result:

text

```text
T: 128
trace len: 129
S: 16
K: 4
opening anchor: /dev/ttyACM0
sending MinRoot anchor request...
roundtrip seconds: 4.518634796142578
anchor passed: True
anchor us:     75195
```

Conclusion:

text

```text
MinRoot firmware verifier works.
Device-side verification is ~75 ms.
Initial roundtrip was dominated by UART transfer/throttling.
```

---

## 14. UART optimization results

### 14.1 Conservative setting

Initial reliable setting:

bash

```bash
ANCHOR_UART_CHUNK=8
ANCHOR_UART_CHUNK_DELAY=0.005
```

MinRoot direct roundtrip:

text

```text
~4.5 seconds
```

### 14.2 Faster setting 1

Tried:

bash

```bash
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.002
```

Roundtrip:

text

```text
~1.0714 seconds
```

### 14.3 Faster setting 2

Tried:

bash

```bash
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.001
```

Roundtrip:

text

```text
~0.657 seconds
```

Stress tests passed for MinRoot at this setting.Current preferred setting:

bash

```bash
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.001
```

Fallback if FiT-IoT is flaky:

bash

```bash
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.002
```

Avoid for now:

bash

```bash
ANCHOR_UART_CHUNK_DELAY=0
```

because firmware RX is still polling-based, not interrupt/DMA ring-buffer based.

---

## 15. Does faster UART weaken security?

No.Reducing UART chunk delay does not weaken cryptographic verification. It only affects transport reliability.If UART is too fast, the likely failures are:

text

```text
BADREQ
timeout
malformed payload
serial read failure
```

not:

text

```text
false cryptographic PASS
```

If the board returns `PASS`, it means the firmware parsed the request and verified the proof/audit.The anchor still runs after beacon finalization, so it cannot affect beacon output 

.

---

## 16. `nrfjprog --reset` issue

Command:

bash

```bash
nrfjprog --reset --family NRF91 >/dev/null
```

sometimes reports:

text

```text
[error] [SeggerBackend] - JLinkARM.dll reported error -256 at line 736.
[error] [SeggerBackend] - JLinkARM.dll reported error -256 at line 629.
```

Observation:

text

```text
The error does not halt the test.
The pass check still gets printed.
```

Conclusion:

text

```text
Not currently blocking.
Do not reset before every anchor request unless recovering from a stuck state.
```

For normal testing:

bash

```bash
sudo fuser -k /dev/ttyACM0 2>/dev/null || true
sleep 1
```

is usually enough.Use `nrfjprog --reset` mainly after reflashing or when the serial state is suspect.

---

## 17. Current recommended direct test commands

### 17.1 Direct classical

bash

```bash
cd ~/QBeacon_Sat

sudo fuser -k /dev/ttyACM0 2>/dev/null || true

ANCHOR_PORT=/dev/ttyACM0 \
BEACON_SKIP_DISCOVERY=1 \
CRYPTO_PROFILE=classical \
VDF_DELAY_T=1024 \
ANCHOR_UART_CHUNK=16 \
ANCHOR_UART_CHUNK_DELAY=0.001 \
python -u test_anchor_classical.py
```

Expected:

text

```text
anchor passed: True
anchor us:     positive integer
```

### 17.2 Direct MinRoot

bash

```bash
cd ~/QBeacon_Sat

sudo fuser -k /dev/ttyACM0 2>/dev/null || true

ANCHOR_PORT=/dev/ttyACM0 \
BEACON_SKIP_DISCOVERY=1 \
CRYPTO_PROFILE=post_quantum \
MINROOT_T=128 \
MINROOT_ANCHOR_TRACE=1 \
MINROOT_ANCHOR_S=16 \
MINROOT_ANCHOR_K=4 \
ANCHOR_UART_CHUNK=16 \
ANCHOR_UART_CHUNK_DELAY=0.001 \
python -u test_anchor_minroot.py
```

Expected:

text

```text
anchor passed: True
anchor us:     around 75 ms for T=128,S=16,K=4
roundtrip:     around 0.65s on current setup
```

---

## 18. Current recommended local-test commands

### 18.1 Classical local-test

bash

```bash
cd ~/QBeacon_Sat

sudo fuser -k /dev/ttyACM0 2>/dev/null || true

ANCHOR_NODE_ID=0 \
ANCHOR_PORT=/dev/ttyACM0 \
BEACON_SKIP_DISCOVERY=1 \
CRYPTO_PROFILE=classical \
VDF_DELAY_T=1024 \
ANCHOR_UART_CHUNK=16 \
ANCHOR_UART_CHUNK_DELAY=0.001 \
EPOCH_DURATION_SEC=12 \
python -u main.py --local-test --crypto-profile classical 2>&1 | tee output_classical_anchor.txt
```

Expected:

text

```text
node0: hardware anchor attached
nodes1-3: anchor configured but disabled
node0: anchor verify=PASS
all nodes: same output per epoch
contributors=4/4 ideally
```

### 18.2 Post-quantum MinRoot local-test

bash

```bash
cd ~/QBeacon_Sat

sudo fuser -k /dev/ttyACM0 2>/dev/null || true

ANCHOR_NODE_ID=0 \
ANCHOR_PORT=/dev/ttyACM0 \
BEACON_SKIP_DISCOVERY=1 \
CRYPTO_PROFILE=post_quantum \
MINROOT_T=128 \
MINROOT_ANCHOR_TRACE=1 \
MINROOT_ANCHOR_S=16 \
MINROOT_ANCHOR_K=4 \
ANCHOR_UART_CHUNK=16 \
ANCHOR_UART_CHUNK_DELAY=0.001 \
EPOCH_DURATION_SEC=12 \
python -u main.py --local-test --crypto-profile post_quantum 2>&1 | tee output_minroot_anchor.txt
```

Expected:

text

```text
node0: anchor verify=PASS
nodes1-3: no hardware anchor
all nodes: same beacon output per epoch
contributors=4/4 ideally
```

---

## 19. Important environment variables

### Anchor ownership

bash

```bash
ANCHOR_NODE_ID=0
```

Only node 0 opens the physical board.

### Anchor serial port

bash

```bash
ANCHOR_PORT=/dev/ttyACM0
```

### Discovery bypass for local runs

bash

```bash
BEACON_SKIP_DISCOVERY=1
```

Prevents import-time FiT-IoT node discovery during local tests.

### Classical delay

bash

```bash
CRYPTO_PROFILE=classical
VDF_DELAY_T=1024
```

### MinRoot delay/audit

bash

```bash
CRYPTO_PROFILE=post_quantum
MINROOT_T=128
MINROOT_ANCHOR_TRACE=1
MINROOT_ANCHOR_S=16
MINROOT_ANCHOR_K=4
```

### UART tuning

bash

```bash
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.001
```

### Epoch pacing

bash

```bash
EPOCH_DURATION_SEC=12
```

---

## 20. Parameter scaling plan

Known-good MinRoot bring-up setting:

text

```text
T=128
S=16
K=4
```

Next possible settings:

text

```text
T=512,  S=32, K=8
T=1024, S=32, K=8
T=4096, S=64, K=16
```

Firmware limits to respect:

text

```text
S <= QBA_MR_MAX_SEG_STEPS
K <= QBA_MR_MAX_K
depth <= QBA_MR_MAX_DEPTH
```

Current intended firmware limits:

text

```text
QBA_MR_MAX_SEG_STEPS = 64
QBA_MR_MAX_K = 16
QBA_MR_MAX_DEPTH = 16
```

Device-side MinRoot verification cost should scale mostly with:

text

```text
K * S
```

not directly with full `T`, because the Nordic verifies sampled segments rather than the entire trace.

---

## 21. Security notes

### 21.1 No private entropy leakage through anchor

The anchor verifies delay evidence derived from the frozen public commit transcript and delay input.The delay input is computed before reveal-derived entropy is finalized 

.The MinRoot trace is deterministic from public `delay_input`, not from private QRNG contribution secrets.Therefore, sending MinRoot trace segments to the hardware anchor does not leak private contribution entropy.

### 21.2 Anchor is external attestation only

Because `_dispatch_anchor()` occurs after `_finalize_v2()`, the anchor cannot alter:

text

```text
beacon_output
entropy_seed
contributors
recovered set
commit digest
delay input
```

The anchor result is recorded as metadata:

python

```python
anchor_pass
anchor_us
```

in `EpochResult` 

.

### 21.3 MinRoot is sampled audit, not full succinct proof

The implemented MinRoot hardware verification checks sampled trace segments selected from the context and Merkle root.Therefore the correct wording is:

text

```text
hardware MinRoot audit attestation
```

not:

text

```text
full succinct MinRoot proof
```

unless parameters are chosen to cover all segments.

### 21.4 UART optimization does not reduce verification strength

UART chunking only changes how fast bytes are delivered to the MCU.It does not change:

text

```text
SHA3 context binding
Merkle path verification
step relation checks
Wesolowski equation checks
```

A too-aggressive UART setting can cause transport failure, but not a weaker cryptographic pass.

---

## 22. Current known caveats

### 22.1 Polling UART is still the transport bottleneck

Firmware currently uses polling UART RX:

c

```c
uart_poll_in()
```

with a short busy-wait 

.This works, but the better long-term solution is:

text

```text
interrupt-driven UART RX
ring buffer
or DMA-style buffered receive
```

That would allow Python to send full requests without artificial chunk delay.

### 22.2 Avoid stale multi-probe script

The old `probe_anchor_uart.py` should not be used for current validation unless cleaned up.Issues:

text

```text
may toggle DTR/RTS
may send multiple probes back-to-back
may not use chunked writes
may not include MinRoot reason names
```

Preferred validation scripts:

text

```text
test_anchor_classical.py
test_anchor_minroot.py
main.py --local-test
```

### 22.3 `nrfjprog --reset` warning

The Segger `-256` reset error appears non-fatal but should be noted.If it becomes disruptive on FiT-IoT, avoid resetting between runs unless necessary.

---

## 23. Current firmware design summary

### `main.c`

Responsible for:

text

```text
UART device selection
QBA1 magic scanning
fixed header reading
version check
payload length check
payload reading
dispatch to qba_handle_request()
QBR1 response writing
```

Key current properties:

text

```text
uses uart0
uses k_busy_wait(10)
does not sleep 1 ms between poll attempts
has qba_header_t/qba_response_t BUILD_ASSERTs
debug early-pass is compile-guarded
```

These are visible in `main.c` 

.

### `qba_anchor.c`

Responsible for:

text

```text
generic request validation
delay-input recomputation
profile/mode dispatch
classical Wesolowski dispatch
post-quantum MinRoot dispatch
status/reason response generation
```

### `qba_weso.c`

Responsible for:

text

```text
Wesolowski payload parsing
x binding
delay_output_digest check
r = 2^T mod l
RSA equation verification
```

### `qba_minroot.c`

Responsible for:

text

```text
MinRoot audit payload parsing
field prime check
x binding to delay_input
y binding to delay_output_digest
context hash
challenge derivation
Merkle path verification
x_next^5 == x_i + i step checks
```

### `qba_sha3.c`

Responsible for:

text

```text
SHA3-256 implementation shared by anchor binding, Merkle, and challenge logic
```

---

## 24. Current Python design summary

### `epoch.py`

Responsible for:

text

```text
commit phase
commit-set agreement
freeze commit transcript
derive delay input
evaluate VDF/MinRoot before reveal finalization
reveal/recovery
finalize beacon output
dispatch hardware anchor after finalization
record anchor_pass and anchor_us
```

The state machine is:

text

```text
IDLE -> COMMITTING -> REVEALING -> RECOVERING -> FINALIZED
```

with `FAILED` for skipped/failed epochs 

.

### `anchor_bridge.py`

Responsible for:

text

```text
build QBA header
build classical Wesolowski payload
build MinRoot payload
send chunked serial request
read QBR1 response
decode pass/fail/timing
log reason codes
```

### `test_anchor_classical.py`

Direct single-client classical anchor test.

### `test_anchor_minroot.py`

Direct single-client MinRoot anchor test.

---

## 25. FiT-IoT deployment considerations

For real FiT-IoT:

1. Use one physical anchor per selected node, or keep `ANCHOR_NODE_ID` so only one process opens the board.
2. Do not open serial monitors on the anchor port.
3. Start with conservative UART if remote environment is unstable:

bash

```bash
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.002
```

4. If stable, use:

bash

```bash
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.001
```

5. Keep `EPOCH_DURATION_SEC` large enough to absorb anchor wall time.
6. Avoid relying on `nrfjprog --reset` between every run.
7. Capture logs with `tee`.
8. Verify CSV columns:

text

```text
anchor_verify_pass
anchor_verify_us
beacon_output_hex
node_id
epoch_id
```

9. Confirm all honest nodes produce identical beacon output per epoch.
10. Treat MinRoot hardware result as sampled audit attestation unless coverage is complete.

---

## 26. Recommended stress tests before FiT-IoT

### 26.1 MinRoot direct stress

bash

```bash
cd ~/QBeacon_Sat

for i in $(seq 1 20); do
  echo "=== MinRoot direct run $i ==="
  sudo fuser -k /dev/ttyACM0 2>/dev/null || true
  sleep 1

  ANCHOR_PORT=/dev/ttyACM0 \
  BEACON_SKIP_DISCOVERY=1 \
  CRYPTO_PROFILE=post_quantum \
  MINROOT_T=128 \
  MINROOT_ANCHOR_TRACE=1 \
  MINROOT_ANCHOR_S=16 \
  MINROOT_ANCHOR_K=4 \
  ANCHOR_UART_CHUNK=16 \
  ANCHOR_UART_CHUNK_DELAY=0.001 \
  python -u test_anchor_minroot.py || exit 1
done
```

Pass condition:

text

```text
20/20 anchor passed: True
```

### 26.2 Classical direct stress

bash

```bash
cd ~/QBeacon_Sat

for i in $(seq 1 10); do
  echo "=== Classical direct run $i ==="
  sudo fuser -k /dev/ttyACM0 2>/dev/null || true
  sleep 1

  ANCHOR_PORT=/dev/ttyACM0 \
  BEACON_SKIP_DISCOVERY=1 \
  CRYPTO_PROFILE=classical \
  VDF_DELAY_T=1024 \
  ANCHOR_UART_CHUNK=16 \
  ANCHOR_UART_CHUNK_DELAY=0.001 \
  python -u test_anchor_classical.py || exit 1
done
```

Pass condition:

text

```text
10/10 anchor passed: True
```

### 26.3 Post-quantum local-test

bash

```bash
cd ~/QBeacon_Sat

sudo fuser -k /dev/ttyACM0 2>/dev/null || true
sleep 1

ANCHOR_NODE_ID=0 \
ANCHOR_PORT=/dev/ttyACM0 \
BEACON_SKIP_DISCOVERY=1 \
CRYPTO_PROFILE=post_quantum \
MINROOT_T=128 \
MINROOT_ANCHOR_TRACE=1 \
MINROOT_ANCHOR_S=16 \
MINROOT_ANCHOR_K=4 \
ANCHOR_UART_CHUNK=16 \
ANCHOR_UART_CHUNK_DELAY=0.001 \
EPOCH_DURATION_SEC=12 \
python -u main.py --local-test --crypto-profile post_quantum 2>&1 | tee output_minroot_local.txt
```

Pass condition:

text

```text
node0 anchor verify=PASS
nodes1-3 anchor disabled
all nodes same output per epoch
contributors ideally 4/4
```

---

## 27. Lessons learned

1. Always verify the actual board model before giving flash instructions.
2. nRF9151-SMA-DK UART/VCOM behavior differs from common nRF52840 assumptions.
3. A simple `QBAHELLO` firmware is the fastest way to prove UART mapping.
4. Binary UART protocols should not be monitored with a text serial terminal during active tests.
5. `uart_poll_in()` plus 1 ms sleep is too slow for receiving burst binary frames at 115200 baud.
6. Byte-by-byte magic scanning is necessary for resynchronization.
7. Fixed struct size assertions are extremely useful for Python/C binary protocols.
8. Reason codes in the response reserved field dramatically reduce debugging time.
9. Chunked UART writes are acceptable as a bring-up solution.
10. Interrupt-driven/ring-buffer UART RX is the right long-term fix.
11. Anchor initialization must happen before the local-test barrier.
12. Epoch pacing must include anchor wall-clock time even if anchor time is excluded from latency metrics.
13. Debug bypass flags must be compile-guarded and disabled by default.
14. Direct single-client tests should pass before local multi-node tests.
15. MinRoot sampled audit is much faster on-device than classical Wesolowski verification.
16. Host roundtrip can be dominated by transport even when device verification is fast.
17. The anchor must remain post-finalize to avoid changing beacon semantics.

---

## 28. Current best-known configuration

### Firmware

text

```text
Board: nRF9151-SMA-DK
UART: uart0
Linux port: /dev/ttyACM0
QBA_ENABLE_DEBUG_FLAGS=0
```

### Classical

bash

```bash
CRYPTO_PROFILE=classical
VDF_DELAY_T=1024
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.001
```

### MinRoot

bash

```bash
CRYPTO_PROFILE=post_quantum
MINROOT_T=128
MINROOT_ANCHOR_TRACE=1
MINROOT_ANCHOR_S=16
MINROOT_ANCHOR_K=4
ANCHOR_UART_CHUNK=16
ANCHOR_UART_CHUNK_DELAY=0.001
```

### Local-test

bash

```bash
ANCHOR_NODE_ID=0
ANCHOR_PORT=/dev/ttyACM0
BEACON_SKIP_DISCOVERY=1
EPOCH_DURATION_SEC=12
```

---

## 29. Open TODOs

### Short-term

- [ ]  Run post-quantum 4-node local-test with optimized UART.
- [ ]  Save `output_minroot_local.txt`.
- [ ]  Verify CSV anchor columns.
- [ ]  Verify all nodes have identical beacon outputs per epoch.
- [ ]  Try MinRoot `T=512,S=32,K=8`.
- [ ]  Try MinRoot `T=1024,S=32,K=8`.
- [ ]  Decide final FiT-IoT UART setting: `16/0.001` or `16/0.002`.

### Medium-term

- [ ]  Replace polling UART RX with interrupt-driven/ring-buffer RX.
- [ ]  Remove or archive stale probe scripts.
- [ ]  Add symbolic reason decoding everywhere logs show anchor failures.
- [ ]  Add automated direct-anchor regression tests.
- [ ]  Add automated local-test regression script.

### Paper/artifact wording

- [ ]  Call MinRoot hardware result “sampled audit attestation” unless all segments are covered.
- [ ]  Clearly separate device-side verification time from host roundtrip time.
- [ ]  Report UART throttling settings used for measurements.
- [ ]  State that anchor verification is post-finalize and does not affect beacon output.

---

## 30. Final state as of this journal

The anchor integration is now functionally successful.Classical path:

text

```text
Direct anchor: PASS
Local 4-node beacon anchor: PASS
```

MinRoot path:

text

```text
Direct anchor: PASS
Device-side verify: ~75 ms
Optimized roundtrip: ~0.657 s with chunk=16 delay=0.001
```

Architecture:

text

```text
Anchor is external attestation only.
Beacon output is finalized before anchor dispatch.
Delay input is bound to frozen commit transcript.
Debug bypasses are disabled by default.
UART transport is reliable with tuned chunking.
```

Next major step:

text

```text
Run and validate post-quantum MinRoot local-test, then prepare for FiT-IoT deployment.
```