Updated: Sep 02, 2026

Overview: 
*This is my own guide to remind myself of the repetetive schemes I repeat but can't remember. 

Node Types: 
Currently only using A8 nodes as it has Linux and supports most of my python/shell deployments. 

Prerequisites: 
**SSH Key Authentication**
1. FiT-IoT uses SSH key authentication only — passwords are not accepted. Multiple keys can be registered simultaneously (e.g. separate keys for Linux and Windows).
2. Normally, I have to ssh into specific nodes and deploy my protocols there. Inability to do so is always a good sign to re-check the ssh situation whether I've wiped the .pubs clean. 
**Install CLI Tools**
```
pip install iotlabcli
iotlab-auth -u YOUR_USERNAME
```

==**Deployment Steps**==

**Step 1 - Reserve Nodes**
Submit reservation requests to reserve specific A8 nodes to deploy the protocol. A standard reservation looks as follows: 
```
iotlab-experiment submit -n qrng_run -d 280 -l grenoble,a8,122-131+197-212+214-228
```

One can check the status of the nodes with the following code: 
```
iotlab-experiment get -i EXPERIMENT_ID -s
```

In some exceptional cases, it is also essential to get a list of nodes/IDs of the nodes of the specific reservation: 
```
iotlab-experiment get -r | grep network_address
```

**Step 2 - Deploy Code**
The deployment is majorly taken care by the deploy.sh of every version of Beacon. The deployment step basically takes the protocol's full functioning and installs it onto the reserved nodes. So every node basically runs their own entire python directory and then later on they talk to each other to carry out the beacon protocol. 

**Step 2.1 - Upload from Local Machine**
It is essential to not deploy additional files that are unnecessary and will basically weigh things down. For example, it is essential to exclude files as shown below: 
```
cd ~                              # go UP, out of QBeacon_Sat
tar czf qbeacon.tgz \
    --exclude='.git' --exclude='.venv' --exclude='__pycache__' \
    --exclude='*.pyc' --exclude='logs' --exclude='results' \
    QBeacon_Sat/
scp qbeacon.tgz siddas@grenoble.iot-lab.info:~/
ssh siddas@grenoble.iot-lab.info "cd ~ && tar xzf qbeacon.tgz && rm qbeacon.tgz && ls QBeacon_Sat/main.py"
```

While only uploading single file do from the local system: 
```
scp ~/QBeacon_Sat/run_experiment.sh \
  siddas@grenoble.iot-lab.info:~/QBeacon_Sat/run_experiment.sh
```
After the new migration of A8 Grenoble nodes in August, the "rsync" hasn't been functioning within the specific A8 nodes. Another alternative was using "tar."
**Step 2.2 - Run deploy.sh from frontend**
The initial "tar" is to merely upload the entire working directory to the frontend of FitIoT. Afterwards, one must deploy the code directly to the nodes. "deploy.sh" basically takes care of it all: cleaning nodes, rsyncs/tar code, installs dependencies, and then verifies: 
```
bash deploy.sh 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50
```

**Step 2.3 - Verify Deployment**

```
ssh root@node-a8-1 "cd ~/QBeacon && python3 -c 'from beacon_config.settings import NODES; print(len(NODES))'"
```

**A Standard Top-to-Bottom Example**
```
# ── ON YOUR LOCAL MACHINE ──────────────────────────────────────────
# Step 0: Upload clean working dir to frontend
rsync -av --exclude='.git/' --exclude='.gitignore' \
    --exclude='.venv/' --exclude='venv/' --exclude='env/' \
    --exclude='__pycache__/' --exclude='*.pyc' \
    --exclude='logs/' --exclude='results/' --exclude='.DS_Store' \
    -e ssh ~/path/to/QBeacon/ siddas@grenoble.iot-lab.info:~/QBeacon/

# ── ON THE FRONTEND (ssh siddas@grenoble.iot-lab.info) ─────────────
# Step 1: Reserve, EXCLUDING dead a8-6
iotlab-experiment submit -n pqfreeze -d 120 -l grenoble,a8,1-5+7-11
iotlab-experiment wait

# Step 2: BOOT CHECK (mandatory — "Running" ≠ "booted")
grep "Boot A8 succeeded" ~/.iot-lab/last/log/a8_*.log
#   Use ONLY node IDs that show this. Need ≥7 for healthy t margin.

# Step 3: CLOCK CHECK (the #1 silent cause of commit_digest divergence)
for n in 1 2 3 4 5 7 8 9 10 11; do
  echo -n "node-a8-$n: "; ssh -o StrictHostKeyChecking=no root@node-a8-$n "date -u +%T" 2>/dev/null
done
#   All within ~1-2s. With chaining ON, epoch-0 skew cascades. If skewed → STOP.

# Step 4: Deploy to nodes (confirmed-booted IDs only)
cd ~/QBeacon && bash deploy.sh 1 2 3 4 5 7 8 9 10

# Step 5: PQ RUN — export MINROOT_T (mandatory), no anchor, prng
export MINROOT_T=1024        # ~5-6s/epoch; use 4096 (~22s) for paper-relevant T
bash run_experiment.sh \
    --mode qrng --epochs 10 \
    --nodes "200 201 202 203 204 205 206 207 208 209 210 211 212 213 214 215 216 217 218 219 220 221 222 223 224 225 226 227 228" \
    --crypto-profile post_quantum --tag pqqrng
#   Do NOT export ANCHOR_PORT / ANCHOR_TCP.

# Step 6: VERIFY — the whole point
grep "FROZEN" ~/results/*_pqfreeze/node-*/logs/node_*.log | head
#   MUST show: delay_mode=legacy:post_quantum:minroot T=1024   (NOT wesolowski)
for i in 0 1 2 3 4 5 6 7 8; do
  echo "=== node-id $i ==="
  grep "FROZEN" ~/results/*_pqfreeze/node-*/logs/node_${i}.log 2>/dev/null | head -1
done
#   All nodes MUST print the SAME commit_digest= prefix per epoch.

```

**To Kill Processes running on Nodes**

```
for n in 175 176 177 178 179 180 181 182 183 184 185 197 198 199 200 201 202 203 204 205 206 207 208 209 210 211 212 213 214 215 216 217 218 219 220 221 222 223 224 225 226 227 228; do
  ssh -o StrictHostKeyChecking=no root@node-a8-$n "pkill -f 'python3 main.py'" 2>/dev/null
done
```

**Pulling folders down**

```
scp -r siddas@grenoble.iot-lab.info:'~/results/20260914_161334_qrng_n4_post_quantum_pq_anchor_tcp_test' ~/qbeacon_sat_results/
```

Usually with pulling data, I clunk a lot of garbage with log files. Meaningless/unwanted warnings can clunk the entire log files, so it is better to pull only the .csv files. 
```
for run in 20260916_154410_qrng_n21_post_quantum_pq_anchor_rtt_test_2; do
  ssh siddas@grenoble.iot-lab.info \
    "cd ~/results/${run} && tar czf /tmp/${run}.tgz \$(find . -name 'metrics_*.csv')"
  scp siddas@grenoble.iot-lab.info:/tmp/${run}.tgz ~/qbeacon_sat_results/
  mkdir -p ~/qbeacon_sat_results/${run}
  tar xzf ~/qbeacon_sat_results/${run}.tgz -C ~/qbeacon_sat_results/${run}/
done

```

markdown

````markdown
# QBeacon / AeroPQ — FiT-IoT Nordic Anchor Attachment

**Date:** 2026-09-14  
**Purpose:** Run QBeacon/AeroPQ on FiT-IoT A8 nodes while using the local Nordic nRF9151 board attached to my PC as the hardware anchor.

---

## 1. Current verified result

A 2-epoch FiT-IoT post-quantum anchor test succeeded on node 0. Both epochs recorded:

```text
anchor_verify_pass = 1
anchor_verify_us   = 76446
````

so the remote FiT-IoT node successfully reached the Nordic anchor through the TCP/tunnel path and received PASS twice [[9]].Anchor verification time was:


```text
76446 us ≈ 76.4 ms
```

for both epochs [[9]].Participation was:


```text
epoch 0: contributors = 4/4
epoch 1: contributors = 3/4
```

[[9]]This CSV is only node 0’s CSV, so it confirms node-0 anchor success, but full distributed agreement still requires comparing all node CSVs/logs.

---

## 2. Why the TCP anchor path is needed

The Nordic is attached to my local PC as:


```bash
/dev/ttyACM0
```

FiT-IoT beacon processes run on remote A8 nodes through SSH. `run_experiment.sh` launches each node remotely with `ssh root@node-a8-${node} ... python3 main.py` [[3]].Therefore, `/dev/ttyACM0` on my local PC is not visible to `node-a8-*`.So the working design is:


```text
FiT-IoT node 0
    |
    | TCP
    v
Local PC gateway
    |
    | UART /dev/ttyACM0
    v
Nordic nRF9151
```

The actual working tunnel path is:

```text
node-a8-FIRST:127.0.0.1:29100
    ↓
Grenoble frontend:127.0.0.1:19100
    ↓
Local PC:127.0.0.1:9100
    ↓
Nordic:/dev/ttyACM0
```

From QBeacon’s point of view, the anchor is:

```bash
ANCHOR_TCP=127.0.0.1:29100
```

on logical node 0.

---

## 3. Important implementation facts

`main.py` already supports TCP anchor mode: it reads `ANCHOR_TCP`, splits it into `host, port`, and creates `AnchorVerifier(tcp=(host, int(port)))` [[2]].`run_experiment.sh` already supports `--anchor-tcp`, stores it in `ANCHOR_TCP`, and forwards it only to the logical anchor node via `ANCHOR_ENV` [[3]].Logical node IDs are assigned by array index in `run_experiment.sh`: the first physical node in `--nodes` is launched with `--node-id 0` [[3]]. Therefore:

bash

```bash
--nodes "2 3 4 5" --anchor-node 0
```

means:

text

```text
logical node 0 = node-a8-2 = anchor node
```

The hardware anchor is called after beacon finalization in `epoch.py`, so it is external attestation and does not affect `beacon_output` [[5]].The Nordic firmware expects binary `QBA1` requests, scans for magic, reads a fixed 156-byte header, dispatches to the verifier, and returns a fixed 12-byte response [[1]].

---

## 4. File locations

### Local PC

The Nordic gateway should run on the local PC, because the Nordic is physically attached there:

text

```text
~/QBeacon_Sat/network/anchor_tcp_gateway.py
```

### Grenoble frontend

Create the tunnel helper here:

```text
~/QBeacon_Sat/anchor_tunnel.sh
```

This script is run from the Grenoble frontend, not from the A8 nodes.

### A8 nodes

No special tunnel script is needed on the A8 nodes. They only receive:


```bash
ANCHOR_TCP=127.0.0.1:29100
```

through `run_experiment.sh`.

---

## 5. Local PC terminal 1 — start Nordic TCP gateway

Run on the local PC:

bash

```bash
cd ~/QBeacon_Sat/network
```

```
```bash
cd ~/Work/QBeacon_Sat/network
python3 -u anchor_tcp_gateway.py --serial /dev/ttyACM0 \
  --listen-host 127.0.0.1 --listen-port 9100 \
  --chunk-size 16 --chunk-delay 0.003 --serial-timeout 120 \
  |& tee ~/hd_stress_gateway.log
```
  
  
```
Keep this terminal open.

````text

If Python says `No module named serial`, install:

```bash
python3 -m pip install --user pyserial
````

`requirements.txt` includes `pyserial`, so the project dependency is correct [[6]].

---

## 6. Local PC terminal 2 — reverse tunnel to Grenoble

Run on the local PC:

```bash
ssh -N -o ExitOnForwardFailure=yes \
  -R 19100:127.0.0.1:9100 \
  siddas@grenoble.iot-lab.info
	  
	  OR
	  


ssh -N -T \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=120 \
  -o TCPKeepAlive=yes \
  -o ConnectTimeout=15 \
  -o Compression=no \
  -R 127.0.0.1:19100:127.0.0.1:9100 \
  siddas@grenoble.iot-lab.info
 

  
  Longer Tunnel
  
  while true; dodo
  echo "[anchor-tunnel] starting $(date)"

  ssh -N -T \
    -o ExitOnForwardFailure=yes \
    -o ServerAliveInterval=30 \
    -o ServerAliveCountMax=120 \
    -o TCPKeepAlive=yes \
    -o ConnectTimeout=15 \
    -o Compression=no \
    -o BatchMode=yes \
    -R 127.0.0.1:19100:127.0.0.1:9100 \
    siddas@grenoble.iot-lab.info

  echo "[anchor-tunnel] dropped $(date), restarting in 5s"
  sleep 5
done

```

Keep this terminal open. This creates:


```text
Grenoble 127.0.0.1:19100 → local PC 127.0.0.1:9100 → Nordic
```

Test on Grenoble:

```bash
python3 - <<'PY'
import socket
s = socket.create_connection(("127.0.0.1", 19100), timeout=5)
print("OK: Grenoble can reach local Nordic gateway tunnel")
s.close()
PY
```

Expected:

```text
OK: Grenoble can reach local Nordic gateway tunnel
```

If this fails with `ConnectionRefusedError`, restart the reverse SSH tunnel.

---

## 7. Grenoble — create `anchor_tunnel.sh`

Create:

bash

```bash
cd ~/QBeacon_Sat
nano anchor_tunnel.sh
```

Paste:

bash

```bash
#!/usr/bin/env bash
set -euo pipefail
```


## 8. Grenoble frontend: prepare the tunnel for a reservation

For a reservation with nodes `2 3 4 5`:

bash

```bash
cd ~/QBeacon_Sat

NODES="2 3 4 5"
./anchor_tunnel.sh
```

Expected output should include:


```text
OK: Grenoble can reach local Nordic gateway tunnel
OK: node-a8 can reach anchor tunnel
```

The first node in the list becomes the anchor node. So for:

```bash
NODES="2 3 4 5"
```

the physical anchor node is:

```text
node-a8-2
```

and it is logical node `0`.

## 9. Run a short anchor experiment

After the tunnel is ready, run:

bash

```bash
cd ~/QBeacon_Sat

NODES="30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50"
./anchor_tunnel.sh

bash run_experiment.sh \
  --mode qrng \
  --epochs 30 \
  --nodes "$NODES" \
  --crypto-profile post_quantum \
  --anchor-node 0 \
  --anchor-tcp 127.0.0.1:29100 \
  --mr-trace 1 \
  --mr-s 16 \
  --mr-k 4 \
  --uart-chunk 16 \
  --uart-delay 0.001 \
  --epoch-sec 12 \
  --tag pq_anchor_rtt_test

```

This is the correct invocation because `run_experiment.sh` parses `--anchor-tcp`, stores it in `ANCHOR_TCP`, and forwards it only to the selected logical anchor node. Non-anchor nodes receive no `ANCHOR_TCP` or `ANCHOR_PORT` through `ANCHOR_ENV` 

**While using the anchor_tunnel.sh make sure that the first node is pointed to the device properly. I made the script more general but this is a point to note that if I change node ordering randomly for example, if the inital node is 30, instead of 2, the IP should be redirected properly !! **

