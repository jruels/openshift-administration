# Scaling OpenShift

In this lab, you will explore how OpenShift manages cluster infrastructure through MachineSets and MachineConfigs, and then configure autoscaling so the cluster automatically adds worker nodes when workloads demand more resources.

This lab has three parts:

1. **Explore cluster infrastructure** — Understand MachineSets, Machines, and MachineConfigs
2. **Configure autoscaling** — Create a ClusterAutoscaler and MachineAutoscaler
3. **Trigger and observe autoscaling** — Deploy a resource-intensive batch job and watch the cluster scale up

## Part 1: Explore Cluster Infrastructure

Before configuring autoscaling, you need to understand the resources that OpenShift uses to manage worker nodes.

### MachineSets and Machines

In OpenShift, **Machines** represent individual nodes (EC2 instances in AWS). **MachineSets** are controllers that manage a set of Machines, similar to how a ReplicaSet manages Pods. The MachineSet ensures the desired number of Machines are running and replaces any that fail.

List the MachineSets in your cluster:

```bash
oc get machinesets -n openshift-machine-api
```

```
NAME                              DESIRED   CURRENT   READY   AVAILABLE   AGE
trainer-xxxxx-worker-us-west-1b   1         1         1       1           7h
trainer-xxxxx-worker-us-west-1c   0         0                             7h
```

You have two MachineSets — one per availability zone. The `us-west-1b` MachineSet has 1 replica (one running worker node). The `us-west-1c` MachineSet has 0 replicas (no workers in that zone).

Store the active MachineSet name in a variable — you will use this throughout the lab:

```bash
MACHINESET=$(oc get machinesets -n openshift-machine-api -o jsonpath='{.items[?(@.spec.replicas>0)].metadata.name}')
echo "MachineSet: $MACHINESET"
```

Now list the Machines managed by this MachineSet:

```bash
oc get machines -n openshift-machine-api
```

```
NAME                                    PHASE     TYPE          REGION      ZONE         AGE
trainer-xxxxx-master-0                  Running   m6i.2xlarge   us-west-1   us-west-1b   7h
trainer-xxxxx-worker-us-west-1b-xxxxx   Running   m6i.xlarge    us-west-1   us-west-1b   7h
```

Each Machine maps to an EC2 instance. The **PHASE** column shows the Machine lifecycle: `Provisioning` (EC2 instance starting), `Provisioned` (instance up, node registering), `Running` (node joined and healthy), `Deleting` (node being drained and terminated).

Examine the MachineSet configuration:

```bash
oc get machineset $MACHINESET -n openshift-machine-api -o yaml | head -40
```

Key fields in the MachineSet spec:
- **spec.replicas** — How many Machines to maintain (like a ReplicaSet)
- **spec.template.spec.providerSpec** — Cloud-specific configuration (instance type, AMI, subnet, security groups)
- **metadata.annotations** — Resource capacity hints used by the autoscaler (`machine.openshift.io/vCPU`, `machine.openshift.io/memoryMb`)

### MachineConfigs

**MachineConfigs** define the operating system configuration for nodes — things like systemd units, kernel parameters, and file contents. The Machine Config Operator (MCO) applies these configurations to nodes and handles rolling reboots when configurations change.

List the MachineConfigs:

```bash
oc get machineconfigs
```

```
NAME                                               GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                          074b3c0953cb81283d7d129e4c3ba6b1a95eb090   3.5.0             7h
00-worker                                          074b3c0953cb81283d7d129e4c3ba6b1a95eb090   3.5.0             7h
01-master-container-runtime                        074b3c0953cb81283d7d129e4c3ba6b1a95eb090   3.5.0             7h
01-master-kubelet                                  074b3c0953cb81283d7d129e4c3ba6b1a95eb090   3.5.0             7h
01-worker-container-runtime                        074b3c0953cb81283d7d129e4c3ba6b1a95eb090   3.5.0             7h
01-worker-kubelet                                  074b3c0953cb81283d7d129e4c3ba6b1a95eb090   3.5.0             7h
...
rendered-master-xxxxx                              074b3c0953cb81283d7d129e4c3ba6b1a95eb090   3.5.0             7h
rendered-worker-xxxxx                              074b3c0953cb81283d7d129e4c3ba6b1a95eb090   3.5.0             7h
```

The naming convention tells you what each MachineConfig does:
- **00-master / 00-worker** — Base OS configuration for each role
- **01-master-kubelet / 01-worker-kubelet** — Kubelet settings
- **01-master-container-runtime / 01-worker-container-runtime** — Container runtime settings (CRI-O)
- **rendered-master-xxxxx / rendered-worker-xxxxx** — The MCO merges all MachineConfigs for a role into a single "rendered" config. This is what actually gets applied to nodes.

Check which rendered config the worker nodes are using:

```bash
oc get machineconfigpool worker
```

```
NAME     CONFIG                           UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   ...
worker   rendered-worker-xxxxx            True      False      False      1              1                   ...
```

The `UPDATED: True` and `UPDATING: False` columns confirm that all worker nodes are running the current configuration. If you add a new MachineConfig, the MCO will create a new rendered config and roll it out to nodes one at a time.

## Part 2: Configure Autoscaling

OpenShift autoscaling uses two resources that work together:
- **ClusterAutoscaler** — Cluster-wide policy: maximum number of nodes, CPU/memory limits, scale-down behavior
- **MachineAutoscaler** — Per-MachineSet policy: minimum and maximum replicas for a specific MachineSet

### Create the ClusterAutoscaler

The ClusterAutoscaler watches for Pods that cannot be scheduled due to insufficient resources. When it detects unschedulable Pods, it instructs a MachineSet to add more Machines.

Create a file called `cluster-autoscaler.yaml`:

```yaml
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  podPriorityThreshold: -10
  resourceLimits:
    maxNodesTotal: 3
    cores:
      min: 8
      max: 128
    memory:
      min: 4
      max: 256
  logVerbosity: 4
  scaleDown:
    enabled: true
    delayAfterAdd: 10m
    delayAfterDelete: 5m
    delayAfterFailure: 30s
    unneededTime: 5m
    utilizationThreshold: "0.4"
```

Key fields:
- **maxNodesTotal: 3** — The cluster will never exceed 3 total nodes (1 master + 2 workers max)
- **podPriorityThreshold: -10** — Only Pods with priority above this trigger scale-up
- **scaleDown.enabled: true** — Allow the autoscaler to remove underutilized nodes
- **scaleDown.unneededTime: 5m** — A node must be underutilized for 5 minutes before removal
- **scaleDown.utilizationThreshold: "0.4"** — A node is "underutilized" if resource usage is below 40%
- **logVerbosity: 4** — Detailed logging so you can see the autoscaler's decisions

Apply the manifest:

```bash
oc apply -f cluster-autoscaler.yaml
```

Verify it was created:

```bash
oc get clusterautoscaler default
```

```
NAME      AGE
default   5s
```

### Create the MachineAutoscaler

The ClusterAutoscaler needs a MachineAutoscaler to know *which* MachineSet to scale. The MachineAutoscaler sets the minimum and maximum replica count for a specific MachineSet.

Create a file called `machine-autoscaler.yaml`, replacing `MACHINESET_NAME` with your actual MachineSet name from Part 1:

```yaml
apiVersion: autoscaling.openshift.io/v1beta1
kind: MachineAutoscaler
metadata:
  name: MACHINESET_NAME
  namespace: openshift-machine-api
spec:
  minReplicas: 1
  maxReplicas: 2
  scaleTargetRef:
    apiVersion: machine.openshift.io/v1beta1
    kind: MachineSet
    name: MACHINESET_NAME
```

You can use `sed` to replace the placeholder with your actual MachineSet name:

```bash
sed -i "s/MACHINESET_NAME/$MACHINESET/g" machine-autoscaler.yaml
```

Apply the manifest:

```bash
oc apply -f machine-autoscaler.yaml
```

Verify it was created and references the correct MachineSet:

```bash
oc get machineautoscaler -n openshift-machine-api
```

```
NAME                              REF KIND     REF NAME                          MIN   MAX   AGE
trainer-xxxxx-worker-us-west-1b   MachineSet   trainer-xxxxx-worker-us-west-1b   1     2     5s
```

The autoscaler will maintain between 1 and 2 worker nodes. Currently you have 1 worker, so the autoscaler will add a second worker when the existing node cannot handle the workload.

### Verify the baseline

Before triggering autoscaling, record the current state:

```bash
oc get nodes
```

```
NAME                                        STATUS   ROLES                  AGE   VERSION
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    worker                 7h    v1.34.2
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    control-plane,master   7h    v1.34.2
```

You should see 2 nodes: 1 master and 1 worker.

## Part 3: Trigger and Observe Autoscaling

### Deploy a resource-intensive batch job

Create a project for the test workload:

```bash
oc new-project scaling-test
```

Create a batch Job that requests more resources than a single worker can handle. Each Pod requests 500m CPU (half a core), and the Job creates 50 Pods in parallel. Since a worker node has approximately 3500m allocatable CPU, only about 7 Pods fit on one node — the remaining Pods become unschedulable, triggering the autoscaler.

Create a file called `batch.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: work-queue-
spec:
  template:
    spec:
      containers:
      - name: work
        image: busybox
        command: ["sleep", "300"]
        resources:
          requests:
            memory: 500Mi
            cpu: 500m
      restartPolicy: Never
  backoffLimit: 4
  completions: 50
  parallelism: 50
```

Key fields:
- **parallelism: 50** — Launch 50 Pods simultaneously
- **completions: 50** — The Job finishes when 50 Pods complete successfully
- **resources.requests** — Each Pod needs 500m CPU and 500Mi memory. This deliberately exceeds what one node can provide.
- **generateName** — Each Job gets a unique name with a random suffix (since we use `oc create` instead of `oc apply`)

Submit the Job:

```bash
oc create -f batch.yaml
```

### Watch the autoscaler respond

Check the Job and Pod status:

```bash
oc get job
```

```
NAME               STATUS    COMPLETIONS   DURATION   AGE
work-queue-xxxxx   Running   0/50          5s         5s
```

Count how many Pods are in each state:

```bash
oc get pods --no-headers | awk '{print $3}' | sort | uniq -c
```

```
     47 Pending
      3 Running
```

Most Pods are Pending because the single worker node does not have enough CPU to schedule them all. The autoscaler detects these unschedulable Pods and triggers a scale-up.

### Monitor the scale-up

Watch the Machines to see the new node being provisioned:

```bash
oc get machines -n openshift-machine-api -w
```

Within 10–15 seconds, you should see a new Machine appear with the `Provisioning` phase. Press `Ctrl+C` once you see it.

```
NAME                                    PHASE          TYPE          REGION      ZONE         AGE
trainer-xxxxx-master-0                  Running        m6i.2xlarge   us-west-1   us-west-1b   7h
trainer-xxxxx-worker-us-west-1b-xxxxx   Running        m6i.xlarge    us-west-1   us-west-1b   7h
trainer-xxxxx-worker-us-west-1b-xxxxx   Provisioning   m6i.xlarge    us-west-1   us-west-1b   5s
```

The new Machine progresses through these phases:
1. **Provisioning** — The AWS EC2 instance is being created (~30 seconds)
2. **Provisioned** — The instance is running, the node is booting and registering with the cluster (~2 minutes)
3. **Running** — The node has joined the cluster and is Ready

You can also view the autoscaler's decision-making by checking its logs:

```bash
oc -n openshift-machine-api logs -l cluster-autoscaler=default --tail=5
```

Look for messages about "Scale-up" and "unschedulable pods".

### Verify the new node

Wait 3–4 minutes for the new node to fully join, then verify:

```bash
oc get nodes
```

```
NAME                                        STATUS   ROLES                  AGE     VERSION
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    worker                 3m      v1.34.2
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    worker                 7h      v1.34.2
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    control-plane,master   7h      v1.34.2
```

You now have 3 nodes — 1 master and 2 workers. The autoscaler added a worker to handle the batch workload.

Check the MachineSet to confirm it was scaled:

```bash
oc get machineset -n openshift-machine-api
```

```
NAME                              DESIRED   CURRENT   READY   AVAILABLE   AGE
trainer-xxxxx-worker-us-west-1b   2         2         2       2           7h
trainer-xxxxx-worker-us-west-1c   0         0                             7h
```

The DESIRED count went from 1 to 2 — the autoscaler updated the MachineSet replica count.

Check the Pod status again:

```bash
oc get pods --no-headers | awk '{print $3}' | sort | uniq -c
```

More Pods should now be Running since the new node provides additional CPU capacity. Some Pods may still be Pending because even 2 nodes cannot run all 50 Pods simultaneously at 500m CPU each (each node fits ~7 Pods).

## Cleanup

Delete the test workload and project:

```bash
oc delete project scaling-test
```

Delete the autoscalers:

```bash
oc delete clusterautoscaler default
oc delete machineautoscaler $MACHINESET -n openshift-machine-api
```

When you delete the MachineAutoscaler and ClusterAutoscaler, the MachineSet replica count stays at whatever the autoscaler set it to (2 in this case). The autoscaler does not scale back down on deletion. Scale the MachineSet back to 1 worker:

```bash
oc scale machineset $MACHINESET -n openshift-machine-api --replicas=1
```

Verify the extra Machine is being removed:

```bash
oc get machines -n openshift-machine-api
```

You should see one worker Machine in `Deleting` phase. The node is drained (existing Pods are evicted) and then the EC2 instance is terminated. This takes 2–3 minutes.

Wait until only the original master and one worker remain:

```bash
oc get nodes
```

```
NAME                                        STATUS   ROLES                  AGE   VERSION
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    worker                 10m   v1.34.2
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    control-plane,master   7h    v1.34.2
```

Clean up the YAML files:

```bash
rm -f cluster-autoscaler.yaml machine-autoscaler.yaml batch.yaml
```

## Congratulations

You have explored OpenShift cluster infrastructure and configured autoscaling:

- **Investigated MachineSets** to understand how OpenShift manages worker nodes as a set with a desired replica count
- **Reviewed MachineConfigs** to see how the Machine Config Operator applies OS-level configuration to nodes
- **Created a ClusterAutoscaler** with resource limits and scale-down policies
- **Created a MachineAutoscaler** that allows a MachineSet to scale between 1 and 2 replicas
- **Triggered autoscaling** by deploying a batch job that exceeded a single node's capacity
- **Observed the full lifecycle** — Provisioning, Provisioned, Running — as a new EC2 instance joined the cluster
- **Cleaned up** by scaling the MachineSet back to its original size

The key insight is that OpenShift autoscaling works at the infrastructure level, not just the Pod level. The ClusterAutoscaler adds actual compute capacity (new VMs) when existing nodes are full, and removes them when they are underutilized — automating what would otherwise be a manual process of launching and configuring new nodes.
