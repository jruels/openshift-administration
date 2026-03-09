# Scheduling in OpenShift

In this lab, you will learn how the Kubernetes scheduler decides where to place Pods and how to influence those decisions. You will work with four scheduling mechanisms:

1. **Node Selectors** — The simplest way to constrain Pods to specific nodes
2. **Node Affinity** — A more expressive version of node selectors with required and preferred rules
3. **Taints and Tolerations** — Allow nodes to repel Pods unless they explicitly opt in
4. **Pod Affinity and Anti-Affinity** — Place Pods relative to other Pods, not just nodes

Understanding these tools is essential for production clusters where you need to control workload placement for performance, isolation, compliance, or hardware requirements.

## Cluster Topology

Before making scheduling decisions, you need to know what you are working with. List the nodes in your cluster:

```bash
oc get nodes
```

You should see two nodes — a worker and a control-plane/master:

```
NAME                                        STATUS   ROLES                  AGE   VERSION
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    worker                 ...   v1.34.2
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    control-plane,master   ...   v1.34.2
```

Check for taints, which restrict what can be scheduled on a node:

```bash
oc describe nodes | grep -A1 "Taints:"
```

The master node has a `node-role.kubernetes.io/master:NoSchedule` taint, which means regular workloads cannot run on it. Only the worker node is available for your Pods. This is the default in OpenShift — control-plane nodes are reserved for cluster infrastructure.

## Project Setup

Create a dedicated project for this lab:

```bash
oc new-project scheduling-lab
```

Grant the `anyuid` SCC so that nginx containers can run:

```bash
oc adm policy add-scc-to-user anyuid -z default -n scheduling-lab
```

## Part 1: Node Selectors

A `nodeSelector` is the simplest way to constrain Pods to nodes with specific labels. The scheduler will only place the Pod on nodes whose labels match every key-value pair in the selector.

### Label a node

First, identify your worker node name:

```bash
WORKER_NODE=$(oc get nodes -l node-role.kubernetes.io/worker -o jsonpath='{.items[0].metadata.name}')
echo $WORKER_NODE
```

Add a custom label to simulate a node with SSD storage:

```bash
oc label nodes $WORKER_NODE disktype=ssd
```

Verify the label was applied:

```bash
oc get nodes -l disktype=ssd
```

You should see your worker node listed.

### Deploy a Pod with nodeSelector

Create a file called `ssd-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  labels:
    app: ssd-test
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
  nodeSelector:
    disktype: ssd
```

The `nodeSelector` tells the scheduler: only place this Pod on nodes that have the label `disktype=ssd`.

Apply it:

```bash
oc apply -f ssd-pod.yaml
```

Verify the Pod was scheduled to the labeled node:

```bash
oc get pod ssd-pod -o wide
```

The `NODE` column should show your worker node — the only node with the `disktype=ssd` label.

### Clean up

```bash
oc delete pod ssd-pod
oc label nodes $WORKER_NODE disktype-
```

The trailing `-` after `disktype` removes the label.

## Part 2: Node Affinity

Node affinity is a more expressive version of `nodeSelector`. It supports two rule types:

- **requiredDuringSchedulingIgnoredDuringExecution** — Hard requirement. The Pod will not schedule unless a matching node exists. It stays `Pending` indefinitely until one becomes available.
- **preferredDuringSchedulingIgnoredDuringExecution** — Soft preference. The scheduler tries to find a matching node but will schedule the Pod elsewhere if no match exists.

The "IgnoredDuringExecution" part means that if a node's labels change after a Pod is already running, the Pod is not evicted.

### Required node affinity

Create a file called `affinity-required.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-required
spec:
  replicas: 2
  selector:
    matchLabels:
      app: affinity-required
  template:
    metadata:
      labels:
        app: affinity-required
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values:
                - production
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

This Deployment requires nodes with the label `env=production`. The `operator: In` means the node's `env` label must have a value that is in the list `[production]`. Other operators include `NotIn`, `Exists`, `DoesNotExist`, `Gt`, and `Lt`.

Apply it:

```bash
oc apply -f affinity-required.yaml
```

Check the Pods:

```bash
oc get pods -l app=affinity-required
```

```
NAME                                 READY   STATUS    RESTARTS   AGE
affinity-required-85dd58687d-xxxxx   0/1     Pending   0          5s
affinity-required-85dd58687d-yyyyy   0/1     Pending   0          5s
```

Both Pods are `Pending` because no node has the label `env=production`. Investigate with `oc describe`:

```bash
oc describe pod -l app=affinity-required | grep -A3 "Events:"
```

You will see a `FailedScheduling` event explaining that no nodes match the Pod's node affinity.

### Fix the scheduling issue

Add the required label to the worker node:

```bash
oc label nodes $WORKER_NODE env=production
```

Within seconds, the scheduler detects the matching node and places the Pods:

```bash
oc get pods -l app=affinity-required -o wide
```

Both Pods should now be `Running` on the worker node.

### Preferred node affinity

Create a file called `affinity-preferred.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-preferred
spec:
  replicas: 2
  selector:
    matchLabels:
      app: affinity-preferred
  template:
    metadata:
      labels:
        app: affinity-preferred
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: gpu
                operator: In
                values:
                - "true"
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

This Deployment prefers nodes with `gpu=true` (weight 80 out of 100), but no node in the cluster has that label.

Apply it:

```bash
oc apply -f affinity-preferred.yaml
```

Check the Pods:

```bash
oc get pods -l app=affinity-preferred -o wide
```

Both Pods are `Running` — even though no node has the `gpu=true` label. This is the key difference between `required` and `preferred`: preferred rules are best-effort, not mandatory. The scheduler places the Pods on the best available node.

### Clean up

```bash
oc delete deployment affinity-required affinity-preferred
oc label nodes $WORKER_NODE env-
```

## Part 3: Taints and Tolerations

Taints and tolerations work together to control scheduling from the **node's** perspective. A taint on a node says "do not schedule here unless you tolerate me." A toleration on a Pod says "I can handle that taint."

This is the opposite of affinity: affinity attracts Pods to nodes, while taints repel Pods from nodes.

Each taint has three components:
- **key** — A label-like identifier (e.g., `dedicated`)
- **value** — An optional value (e.g., `special`)
- **effect** — What happens to Pods that do not tolerate the taint:
  - `NoSchedule` — New Pods will not be scheduled (existing Pods remain)
  - `PreferNoSchedule` — The scheduler tries to avoid the node but may still place Pods there
  - `NoExecute` — New Pods are rejected AND existing Pods without a matching toleration are evicted

Your cluster already demonstrates taints — the master node has `node-role.kubernetes.io/master:NoSchedule`, which is why your workloads only run on the worker.

### Taint the worker node

Apply a custom taint to the worker node:

```bash
oc adm taint nodes $WORKER_NODE dedicated=special:NoSchedule
```

**Note:** OpenShift uses `oc adm taint` instead of `kubectl taint`.

Verify the taint:

```bash
oc describe node $WORKER_NODE | grep Taints
```

```
Taints:             dedicated=special:NoSchedule
```

Now both nodes have taints — the master with its built-in taint and the worker with your custom one. No regular Pod can schedule anywhere.

### Deploy without a toleration

Create a file called `no-toleration.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-toleration
spec:
  replicas: 2
  selector:
    matchLabels:
      app: no-toleration
  template:
    metadata:
      labels:
        app: no-toleration
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Apply it:

```bash
oc apply -f no-toleration.yaml
```

Check the Pods:

```bash
oc get pods -l app=no-toleration
```

```
NAME                             READY   STATUS    RESTARTS   AGE
no-toleration-64c84d75cd-xxxxx   0/1     Pending   0          5s
no-toleration-64c84d75cd-yyyyy   0/1     Pending   0          5s
```

Both Pods are `Pending`. Inspect the scheduling failure:

```bash
oc describe pod -l app=no-toleration | grep -A2 "Warning"
```

The event message says `2 node(s) had untolerated taint(s)` — neither node will accept these Pods.

### Deploy with a toleration

Create a file called `with-toleration.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: with-toleration
spec:
  replicas: 2
  selector:
    matchLabels:
      app: with-toleration
  template:
    metadata:
      labels:
        app: with-toleration
    spec:
      tolerations:
      - key: dedicated
        operator: Equal
        value: special
        effect: NoSchedule
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

The `tolerations` section says: "This Pod can tolerate nodes with the taint `dedicated=special:NoSchedule`." The toleration must match the taint's key, value, and effect exactly.

Apply it:

```bash
oc apply -f with-toleration.yaml
```

Check both Deployments side by side:

```bash
oc get pods -o wide
```

You should see the `with-toleration` Pods running on the worker node, while the `no-toleration` Pods remain `Pending`:

```
NAME                               READY   STATUS    RESTARTS   AGE   IP            NODE
no-toleration-64c84d75cd-xxxxx     0/1     Pending   0          30s   <none>        <none>
no-toleration-64c84d75cd-yyyyy     0/1     Pending   0          30s   <none>        <none>
with-toleration-6bb8b9fd94-xxxxx   1/1     Running   0          10s   10.129.0.91   ip-10-0-xx-xx...
with-toleration-6bb8b9fd94-yyyyy   1/1     Running   0          10s   10.129.0.92   ip-10-0-xx-xx...
```

The toleration acts as a key that unlocks the tainted node. Without it, Pods are locked out.

### Clean up

Remove the custom taint from the worker and delete the Deployments:

```bash
oc adm taint nodes $WORKER_NODE dedicated-
oc delete deployment no-toleration with-toleration
```

The trailing `-` after `dedicated` removes the taint. Verify:

```bash
oc describe node $WORKER_NODE | grep Taints
```

```
Taints:             <none>
```

## Part 4: Pod Affinity and Anti-Affinity

Pod affinity and anti-affinity let you place Pods relative to **other Pods** rather than nodes. This is useful when:

- **Pod affinity** — You want related services on the same node for lower latency (e.g., a web server and its cache)
- **Pod anti-affinity** — You want replicas spread across nodes for high availability (e.g., database replicas on separate nodes)

Like node affinity, pod affinity supports both `required` and `preferred` variants.

### Pod affinity: co-locate related services

Deploy a cache and a web server that must run on the same node:

Create a file called `pod-affinity.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

The `web` Deployment has a `podAffinity` rule: it **requires** that web Pods run on the same `kubernetes.io/hostname` (same node) as Pods with label `app=cache`. The `topologyKey` defines what "same location" means — `kubernetes.io/hostname` means the same node, while `topology.kubernetes.io/zone` would mean the same availability zone.

Apply it:

```bash
oc apply -f pod-affinity.yaml
```

Verify the Pods are co-located:

```bash
oc get pods -o wide
```

All three Pods (1 cache + 2 web) should be on the same node. In this cluster with one schedulable worker, this is the only option, but in a larger cluster the scheduler would place web Pods on whichever node the cache Pod happens to land on.

### Pod anti-affinity: require separation

Anti-affinity is the opposite — it prevents Pods from being co-located. This is how you ensure replicas are spread across nodes for high availability.

Create a file called `pod-anti-affinity.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-colocate
spec:
  replicas: 2
  selector:
    matchLabels:
      app: no-colocate
  template:
    metadata:
      labels:
        app: no-colocate
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - no-colocate
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

This rule says: no two Pods with label `app=no-colocate` may run on the same node. With 2 replicas but only 1 schedulable node, one Pod will be stuck.

Apply it:

```bash
oc apply -f pod-anti-affinity.yaml
```

Check the Pods:

```bash
oc get pods -l app=no-colocate -o wide
```

```
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE
no-colocate-79f7cfdcb-xxxxx   0/1     Pending   0          8s    <none>        <none>
no-colocate-79f7cfdcb-yyyyy   1/1     Running   0          8s    10.129.0.96   ip-10-0-xx-xx...
```

One Pod is `Running` and one is `Pending`. The first Pod scheduled on the worker, but the second cannot go there (anti-affinity) and the master is tainted, so it has nowhere to go.

This demonstrates an important production lesson: **required anti-affinity with more replicas than available nodes will leave Pods unschedulable.** In production clusters with many nodes this is not a problem, but it is something to plan for.

### Pod anti-affinity: prefer separation

The safe alternative is `preferred` anti-affinity, which tries to spread Pods but allows co-location when necessary:

Create a file called `pod-anti-affinity-preferred.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prefer-spread
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prefer-spread
  template:
    metadata:
      labels:
        app: prefer-spread
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - prefer-spread
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Apply it:

```bash
oc apply -f pod-anti-affinity-preferred.yaml
```

Check the Pods:

```bash
oc get pods -l app=prefer-spread -o wide
```

Both Pods are `Running` on the same node. The scheduler tried to separate them (weight 100 is the maximum preference), but since there is only one schedulable node, it placed them together anyway. In a cluster with multiple nodes, the scheduler would spread them across different nodes.

This is the recommended approach for most production workloads — `preferred` anti-affinity provides the best possible spreading without risking unschedulable Pods.

### Clean up

```bash
oc delete deployment cache web no-colocate prefer-spread
```

## Cleanup

Remove all custom labels from nodes and delete the project:

```bash
oc label nodes $WORKER_NODE disktype- env- 2>/dev/null
oc delete project scheduling-lab
```

## Congratulations

You have learned four scheduling mechanisms that control where Pods run in OpenShift:

- **Node Selectors** are the simplest constraint — Pods only run on nodes with exact label matches.
- **Node Affinity** adds expressiveness with `required` (hard) and `preferred` (soft) rules, plus operators like `In`, `NotIn`, and `Exists`.
- **Taints and Tolerations** work from the node's perspective — taints repel all Pods except those with matching tolerations. OpenShift uses this to protect control-plane nodes.
- **Pod Affinity** co-locates related Pods (e.g., web + cache on the same node), while **Pod Anti-Affinity** separates them for high availability.

The key design decision in production is choosing between `required` and `preferred` rules. Required rules guarantee placement constraints but risk leaving Pods unschedulable. Preferred rules are best-effort but ensure your application always runs, even in degraded conditions.
