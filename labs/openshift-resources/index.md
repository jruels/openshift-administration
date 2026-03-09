# Kubernetes Resources

In this lab, you will explore the core Kubernetes resource hierarchy — Pods, ReplicaSets, Deployments, and Services — and understand how each layer builds on the one below it. By the end, you will see how these resources work together to deliver self-healing, scalable, zero-downtime applications in OpenShift.

## Project Setup

Start by creating a dedicated project for this lab.

```bash
oc new-project k8s-resources
```

You should see output confirming the new project:

```
Now using project "k8s-resources" on server "https://api...."
```

The standard nginx image needs to run as root, which OpenShift restricts by default through Security Context Constraints (SCCs). Grant the `anyuid` SCC to the default service account so that Pods in this project can run as any user:

```bash
oc adm policy add-scc-to-user anyuid -z default -n k8s-resources
```

```
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "default"
```

This is a common pattern in OpenShift when running container images that were not designed to run as an arbitrary non-root user.

## Pods

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network namespace and storage volumes. Every container in a Pod gets the same IP address and can communicate with its neighbors over `localhost`.

We will start by creating a standalone Pod to understand the basics before moving on to controllers that manage Pods for us.

### Create a Pod

Create a file called `pod.yaml` with the following contents:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    tier: web
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

A few things to note about this manifest:

- **apiVersion: v1** — Pods are part of the core API group.
- **labels** — We assign two labels (`app: nginx` and `tier: web`). Labels are key-value pairs that other resources use to find and select Pods.
- **containerPort: 80** — This documents that the container listens on port 80. It does not publish the port externally.

Apply the manifest:

```bash
oc apply -f pod.yaml
```

### Examine the Pod

Use `oc get pods` to confirm the Pod is running:

```bash
oc get pods
```

You should see:

```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          10s
```

Get detailed information about the Pod, including its IP address, the node it was scheduled on, and recent events:

```bash
oc describe pod nginx-pod
```

### Access the Pod

You can run a command inside the Pod to verify nginx is serving content:

```bash
oc exec nginx-pod -- curl -s http://localhost
```

You should see the default nginx welcome page HTML.

### Delete the Pod

Now delete the Pod:

```bash
oc delete pod nginx-pod
```

Run `oc get pods` again:

```bash
oc get pods
```

```
No resources found in k8s-resources namespace.
```

The Pod is gone — permanently. There is no controller watching over it, so nothing recreates it. This is the fundamental limitation of standalone Pods: they have no self-healing. If the Pod crashes, gets deleted, or the node it runs on goes down, it is lost forever.

This limitation motivates the next section.

## ReplicaSets

A ReplicaSet is a controller that ensures a specified number of identical Pod replicas are running at all times. It continuously compares the **desired** state (how many replicas you asked for) against the **actual** state (how many are currently running) and creates or deletes Pods to reconcile the difference.

### Create a ReplicaSet

Create a file called `replicaset.yaml` with the following contents:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
    tier: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      tier: web
  template:
    metadata:
      labels:
        app: nginx
        tier: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

Key fields to understand:

- **replicas: 3** — The desired number of Pod copies.
- **selector.matchLabels** — The ReplicaSet only manages Pods whose labels match **both** `app: nginx` and `tier: web`. If either label is missing or different, the ReplicaSet will not count that Pod toward its replica total.
- **template** — The Pod template used to create new replicas. Notice the labels in the template must match the selector — if they don't, the API server will reject the manifest.

Apply the manifest:

```bash
oc apply -f replicaset.yaml
```

### Verify the Replicas

Confirm that three Pods are running:

```bash
oc get pods
```

You should see three Pods with names that start with `nginx-replicaset-`:

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-replicaset-abc12   1/1     Running   0          15s
nginx-replicaset-def34   1/1     Running   0          15s
nginx-replicaset-ghi56   1/1     Running   0          15s
```

You can also inspect the ReplicaSet itself:

```bash
oc get replicaset nginx-replicaset
```

```
NAME               DESIRED   CURRENT   READY   AGE
nginx-replicaset   3         3         3       30s
```

### Self-Healing Demo

Delete one of the Pods (replace the Pod name with one from your output):

```bash
oc delete pod nginx-replicaset-abc12
```

Immediately list Pods again:

```bash
oc get pods
```

You will still see three Pods — the ReplicaSet detected that the actual count dropped below the desired count and immediately created a replacement. This is the **self-healing** behavior that standalone Pods lack.

### Scale the ReplicaSet

Scale up to 5 replicas:

```bash
oc scale replicaset nginx-replicaset --replicas=5
```

Verify:

```bash
oc get pods
```

You should now see five Pods running. Scale back down to 3:

```bash
oc scale replicaset nginx-replicaset --replicas=3
```

### Clean Up the ReplicaSet

Delete the ReplicaSet before moving on:

```bash
oc delete replicaset nginx-replicaset
```

This also deletes all of the Pods it was managing.

ReplicaSets give us self-healing and scaling, but they have an important limitation: they provide no mechanism for rolling updates. If you want to change the container image, you would have to delete the old ReplicaSet and create a new one, causing downtime. Deployments solve this problem.

## Deployments

A Deployment is the standard way to run stateless applications in Kubernetes. It manages ReplicaSets on your behalf and adds the ability to perform **rolling updates** and **rollbacks** with zero downtime.

When you update a Deployment (for example, changing the container image), it creates a **new** ReplicaSet with the updated Pod template and gradually scales it up while scaling the old ReplicaSet down. The old ReplicaSet is kept around (with zero replicas) so you can roll back instantly if something goes wrong.

### Create a Deployment

Create a file called `deployment.yaml` with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
    tier: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      tier: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
        tier: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

The new field here is **strategy**:

- **type: RollingUpdate** — Pods are replaced incrementally rather than all at once.
- **maxSurge: 1** — At most 1 extra Pod can exist above the desired replica count during the update.
- **maxUnavailable: 0** — No Pods may be unavailable during the update. This means the new Pod must be Ready before the old one is terminated, ensuring zero downtime.

Apply the manifest:

```bash
oc apply -f deployment.yaml
```

### Examine the Resource Chain

One of the most important things to understand is the naming chain that connects Deployments, ReplicaSets, and Pods. List all three:

```bash
oc get deployment,replicaset,pods
```

You will see something like:

```
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           30s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-5d5cf8d7fc   3         3         3       30s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-5d5cf8d7fc-abc12   1/1     Running   0          30s
pod/nginx-deployment-5d5cf8d7fc-def34   1/1     Running   0          30s
pod/nginx-deployment-5d5cf8d7fc-ghi56   1/1     Running   0          30s
```

Notice the naming pattern:
- **Deployment:** `nginx-deployment`
- **ReplicaSet:** `nginx-deployment-5d5cf8d7fc` (Deployment name + template hash)
- **Pods:** `nginx-deployment-5d5cf8d7fc-abc12` (ReplicaSet name + random suffix)

This chain makes it easy to trace any Pod back to the Deployment that created it.

### Rolling Update

Now perform a rolling update by changing the image from `nginx:1.25` to `nginx:1.27`:

```bash
oc set image deployment/nginx-deployment nginx=nginx:1.27
```

Watch the rollout progress:

```bash
oc rollout status deployment/nginx-deployment
```

You should see messages indicating Pods are being replaced one at a time:

```
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

List ReplicaSets to see what happened behind the scenes:

```bash
oc get replicasets
```

```
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-5d5cf8d7fc   0         0         0       2m
nginx-deployment-7bc4f9875d   3         3         3       30s
```

The old ReplicaSet has been scaled to 0 but still exists. The new ReplicaSet now manages all 3 Pods. This is how Deployments enable instant rollbacks.

### Rollout History and Rollback

View the rollout history:

```bash
oc rollout history deployment/nginx-deployment
```

If you need to revert to the previous version, perform a rollback:

```bash
oc rollout undo deployment/nginx-deployment
```

Watch the rollback complete:

```bash
oc rollout status deployment/nginx-deployment
```

Verify the image is back to `nginx:1.25`:

```bash
oc describe deployment nginx-deployment | grep Image
```

```
    Image:        nginx:1.25
```

Before moving on, update the Deployment back to `nginx:1.27` so we have the latest image for the Services section:

```bash
oc set image deployment/nginx-deployment nginx=nginx:1.27
oc rollout status deployment/nginx-deployment
```

## Services

Pod IP addresses are ephemeral — they change every time a Pod is recreated. A Service provides a **stable IP address** and **DNS name** that automatically routes traffic to the current set of healthy Pods matching its selector. It also load-balances traffic across multiple replicas.

### Create a Service

Create a file called `service.yaml` with the following contents:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
    tier: web
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  type: ClusterIP
```

Key fields:

- **selector** — The Service will route traffic to any Pod with labels `app: nginx` AND `tier: web`. This must match the labels on the Deployment's Pod template.
- **port: 80** — The port the Service listens on.
- **targetPort: 80** — The port on the Pod to forward traffic to.
- **type: ClusterIP** — This is the default type. The Service gets a cluster-internal IP address that is only reachable from within the cluster.

Apply the manifest:

```bash
oc apply -f service.yaml
```

### Examine the Service and Endpoints

View the Service:

```bash
oc get service nginx-service
```

```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   172.30.45.123   <none>        80/TCP    10s
```

The CLUSTER-IP is stable — it will not change as long as the Service exists. Now examine the endpoints, which are the actual Pod IPs that the Service is routing to:

```bash
oc describe service nginx-service
```

In the output, look for the **Endpoints** line — you should see three Pod IPs listed, one for each replica. When Pods are added or removed, the endpoint list updates automatically.

### Test with DNS

Every Service gets a DNS entry in the format `<service-name>.<namespace>.svc.cluster.local`. You can test this from inside the cluster using a temporary Pod:

```bash
oc run test-dns --image=busybox --rm -it --restart=Never -- wget -qO- http://nginx-service.k8s-resources.svc.cluster.local
```

You should see the nginx welcome page HTML. The DNS name resolved to the Service's ClusterIP, which then load-balanced the request to one of the three Pods.

After the command completes, the temporary Pod is automatically removed (thanks to `--rm`).

### Expose the Service Externally

In OpenShift, the idiomatic way to expose a Service externally is by creating a Route. This gives you a public URL backed by the cluster's router:

```bash
oc expose service nginx-service
```

Retrieve the Route URL:

```bash
oc get route nginx-service
```

```
NAME            HOST/PORT                                              PATH   SERVICES        PORT   TERMINATION   WILDCARD
nginx-service   nginx-service-k8s-resources.apps.example.com                  nginx-service   80                   None
```

You can now visit the URL from the `HOST/PORT` column in your web browser and see the nginx welcome page served from one of your Pods.

## Putting It All Together

At this point, you have built the complete resource hierarchy that underpins nearly every application in OpenShift. Here is how the pieces connect:

```
Route (external URL)
  └── Service (stable IP + DNS + load balancing)
        └── Deployment (rolling updates + rollback)
              └── ReplicaSet (desired replica count + self-healing)
                    └── Pods (running containers)
```

View everything in the project:

```bash
oc get all
```

This shows your Deployment, ReplicaSet, Pods, Service, and Route in a single view.

### Final Self-Healing Test

Delete one of the Pods and verify the system heals itself end-to-end:

```bash
oc delete $(oc get pods -o name | head -1)
```

Watch the Pods:

```bash
oc get pods
```

Within seconds, you should see a replacement Pod starting up. The Service automatically adds the new Pod to its endpoint list, and the Route continues to serve traffic without interruption. This is the power of the Kubernetes resource hierarchy — each layer handles a different concern, and together they deliver a resilient, self-healing system.

## Cleanup

Delete the project and all resources within it:

```bash
oc delete project k8s-resources
```

## Congratulations

You have built and explored the full Kubernetes resource hierarchy from the ground up:

- **Pods** are the atomic unit — but they have no self-healing.
- **ReplicaSets** add self-healing and scaling by continuously reconciling desired vs actual state.
- **Deployments** manage ReplicaSets to provide rolling updates and instant rollbacks.
- **Services** decouple networking from the Pod lifecycle with stable IPs, DNS names, and load balancing.
- **Routes** (OpenShift-specific) expose Services to external traffic.

These are the foundational building blocks that every other OpenShift resource builds upon.
