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
iotlab-experiment submit -n qrng_run -d 280 -l grenoble,a8,1-5+7-12
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

After the new migration of A8 Grenoble nodes in August, the "rsync" hasn't been functioning within the specific A8 nodes. Another alternative was using "tar."
**Step 2.2 - Run deploy.sh from frontend**
The initial "tar" is to merely upload the entire working directory to the frontend of FitIoT. Afterwards, one must deploy the code directly to the nodes. "deploy.sh" basically takes care of it all: cleaning nodes, rsyncs/tar code, installs dependencies, and then verifies: 
```
bash deploy.sh 1 2 3 4 5 7 8 9 10 11 12 13 14 15 16
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
for n in 1 3 4 5 8 9 10 16 17 18 19; do
  ssh -o StrictHostKeyChecking=no root@node-a8-$n "pkill -f 'python3 main.py'" 2>/dev/null
done
```

**Pulling folders down**

```
scp -r siddas@grenoble.iot-lab.info:'~/QBeacon_Sat/results/20260906_151700_qrng_n43_post_quantum_base' ~/qbeacon_sat_results/
```

Usually with pulling data, I clunk a lot of garbage with log files. Meaningless/unwanted warnings can clunk the entire data base, so it is better to pull only the .csv files. 
```
for run in 20260906_151700_qrng_n43_post_quantum_base \
           20260906_155236_qrng_n43_post_quantum_rec_wh3 \
           20260906_230450_qrng_n43_post_quantum_c15 \
           20260906_234006_qrng_n43_post_quantum_c25; do
  ssh siddas@grenoble.iot-lab.info \
    "cd ~/results/${run} && tar czf /tmp/${run}.tgz \$(find . -name 'metrics_*.csv')"
  scp siddas@grenoble.iot-lab.info:/tmp/${run}.tgz ~/qbeacon_sat_results/
  mkdir -p ~/qbeacon_sat_results/${run}
  tar xzf ~/qbeacon_sat_results/${run}.tgz -C ~/qbeacon_sat_results/${run}/
done

```

