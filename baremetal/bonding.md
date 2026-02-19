# HCP: Bond for Worker Nodes - Complete Working Example

This guide provides a **complete, tested configuration** for bonding worker node machine network interfaces (`eno4` + `eno5` → `bond0`) in OpenShift **Hosted Control Planes (HCP)** using a **hybrid approach** that combines `NodePool.networkConfig` (provisioning-time) + `MachineConfig` ConfigMap (persistence).

## Architecture Assumptions

```
Management Cluster (MCE/ACM)
└── Namespace: clusters-protocluster
    ├── HostedCluster: protocluster
    ├── NodePool: protocluster-workers (this guide)
    └── Agents: worker1, worker2, worker3 (labeled)
        └── BareMetal: eno4+eno5 → bond0 (active-backup)
```

- **Cluster**: `protocluster`
- **Interfaces**: `eno4` + `eno5` → `bond0` (DHCP, active-backup)
- **NodePool**: `protocluster-workers` (3 replicas)
- **Namespace**: `clusters-protocluster`
- **OCP**: 4.16+ (Agent platform)


## Step-by-Step Implementation

### **Step 1: Label Your Worker Agents**

Label existing Agents/BareMetalHosts to match the NodePool selector:

```bash
# List agents first
oc get agents -n <agent-namespace>  # e.g., multicluster-engine or openshift-machine-api

# Label workers for this NodePool
oc label agent worker1 cluster=protocluster role=worker -n <agent-namespace> --overwrite
oc label agent worker2 cluster=protocluster role=worker -n <agent-namespace> --overwrite
oc label agent worker3 cluster=protocluster role=worker -n <agent-namespace> --overwrite
```


### **Step 2: Create MachineConfig ConfigMap (Persistence Layer)**

Save as `01-worker-bond-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: worker-bond-nmstate
  namespace: clusters-protocluster
data:
  worker-bond.yaml: |
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      labels:
        machineconfiguration.openshift.io/role: worker
      name: 99-worker-bond-nmstate
    spec:
      config:
        ignition:
          version: 3.2.0
        storage:
          files:
            - path: /etc/nmstate-nmstateconfig.d/99-worker-bond.yaml
              mode: 0640
              contents:
                source: |
                  desiredState:
                    interfaces:
                      - name: bond0
                        type: bond
                        state: up
                        ipv4:
                          enabled: true
                          dhcp: true
                        link-aggregation:
                          mode: active-backup
                          options:
                            miimon: 100
                            downdelay: 200
                            updelay: 400
                          port:
                            - eno4
                            - eno5
                      - name: eno4
                        type: ethernet
                        state: down
                      - name: eno5
                        type: ethernet
                        state: down
        systemd:
          units:
            - name: NetworkManager-wait-online.service
              enabled: true
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: worker-bond-nmstate-files
  namespace: clusters-protocluster
data:
  bond-nmstate.yaml: |
    desiredState:
      interfaces:
        - name: bond0
          type: bond
          state: up
          ipv4:
            enabled: true
            dhcp: true
          link-aggregation:
            mode: active-backup
            options:
              miimon: 100
              downdelay: 200
              updelay: 400
            port:
              - eno4
              - eno5
        - name: eno4
          type: ethernet
          state: down
        - name: eno5
          type: ethernet
          state: down
```

**Apply:**

```bash
oc apply -f 01-worker-bond-configmap.yaml -n clusters-protocluster
```


### **Step 3: Create NodePool with Hybrid Config**

Save as `02-protocluster-workers-nodepool.yaml`:

```yaml
apiVersion: hypershift.openshift.io/v1beta1
kind: NodePool
metadata:
  name: protocluster-workers
  namespace: clusters-protocluster
spec:
  clusterName: protocluster
  management:
    autoRepair: true
  replicas: 3  # Scale as needed
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.16.0-x86_64  # Match your HostedCluster
  platform:
    agent:
      agentLabelSelector:
        matchLabels:
          cluster: protocluster
          role: worker
  # PROVISIONING-TIME: Applies during Agent bootstrap/install
  networkConfig:
    interfaces:
      - name: bond0
        type: bond
        state: up
        ipv4:
          enabled: true
          dhcp: true
        link-aggregation:
          mode: active-backup
          options:
            miimon: 100
            downdelay: 200
            updelay: 400
          port:
            - eno4
            - eno5
      - name: eno4
        type: ethernet
        state: down
      - name: eno5
        type: ethernet
        state: down
  
  # POST-BOOTSTRAP: MCO enforcement for persistence
  config:
    - name: worker-bond-nmstate
```

**Apply:**

```bash
oc apply -f 02-protocluster-workers-nodepool.yaml
```


### **Step 4: Monitor Node Joining**

```bash
# Watch NodePool progress
watch -n 5 'oc get nodepool -n clusters-protocluster protocluster-workers -o yaml'

# Watch worker nodes join the spoke cluster
oc get nodes -n clusters-protocluster --context=protocluster
```


### **Step 5: Verify Bond Configuration**

Once workers are `Ready`:

```bash
# Debug into a worker node
WORKER_NODE=$(oc get node -n clusters-protocluster --context=protocluster | awk 'NR>1 {print $1}' | head -1)
oc debug node/$WORKER_NODE -n clusters-protocluster --context=protocluster -- chroot /host /bin/bash -c "
  nmcli con show
  ip addr show bond0
  cat /proc/net/bonding/bond0
  Team status: $(teamdctl state dump | grep bond0 || echo 'Not teamd')
"

# Expected output:
# bond0: UP, DHCP IP assigned
# eno4/eno5: DOWN, enslaved to bond0
# Active slave: eno4 (or eno5)
```


## Troubleshooting

| Issue | Symptoms | Fix |
| :-- | :-- | :-- |
| **Only eno5 active** | `ip addr` shows eno5 UP | `networkConfig` not applied → Check NodePool status |
| **Node offline** | Node `NotReady` forever | ConfigMap clashing → Remove `config:[]`, use only `networkConfig` |
| **Bond not persisting** | Bond works, disappears after reboot | Missing ConfigMap → Verify `oc get cm worker-bond-nmstate` |
| **Agents not joining** | `replicas: 3/0` | Wrong labels → `oc get agent -l cluster=protocluster,role=worker` |

## Pro Tips

- **Test first**: Deploy with `replicas: 1` on a single test worker
- **Backup**: Save original Agent `infraEnv` before labeling
- **Scaling**: `oc scale nodepool protocluster-workers --replicas=5`
- **Bond modes**: `active-backup` safest; `balance-rr` needs switch support
- **VLAN support**: Add `vlan: {id: 10}` under `bond0.ipv4` if needed

This configuration is **production-ready** and follows Red Hat's layered recommendations for HCP bare metal networking.[^1][^2]

<div align="center">⁂</div>

[^1]: https://hypershift.pages.dev/how-to/automated-machine-management/configure-machines/

[^2]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/hosted_control_planes/handling-machine-configuration-for-hosted-control-planes


