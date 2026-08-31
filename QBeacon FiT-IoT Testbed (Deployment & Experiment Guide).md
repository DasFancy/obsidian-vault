Updated: Aug 31, 2026

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
iotlab-experiment submit -n qbeacon_scaling -d 10080 -l grenoble,a8,1-16
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

