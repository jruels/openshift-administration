# Kubernetes Operators

In this lab, you will learn what Kubernetes Operators are, how they extend the Kubernetes API using Custom Resource Definitions (CRDs), and how the Operator Lifecycle Manager (OLM) in OpenShift handles installing, upgrading, and managing Operators. By the end, you will have hands-on experience with CRDs, Custom Resources, and installing an Operator from OperatorHub.

## Why Operators Exist

Kubernetes Deployments, Services, and ReplicaSets handle stateless applications well — they can scale, self-heal, and roll out updates. But many real-world workloads (databases, message queues, monitoring systems) need **operational knowledge**: when to take backups, how to handle failover, how to resize storage, how to upgrade data schemas.

Operators encode this operational knowledge into software. An Operator is a custom controller that watches for changes to custom resources and takes action — just like the built-in Deployment controller watches Deployments and manages ReplicaSets, an Operator watches its own custom resources and manages the underlying application.

The **Operator pattern** has three components:
1. **Custom Resource Definition (CRD)** — Extends the Kubernetes API with a new resource type
2. **Custom Resource (CR)** — An instance of the CRD that declares the desired state
3. **Controller** — Software that watches CRs and reconciles actual state to match desired state

## Part 1: Custom Resource Definitions

Before diving into full Operators, you need to understand CRDs — the mechanism that lets anyone extend the Kubernetes API with new resource types.

### Explore Existing CRDs

OpenShift ships with many CRDs already installed. These power features like MachineConfigs, Routes, and the Operator Lifecycle Manager itself.

See how many CRDs exist on your cluster:

```bash
oc get crd | wc -l
```

You will see well over 100 CRDs. Each one adds a new resource type to the Kubernetes API.

Look at some of the built-in CRDs:

```bash
oc get crd | grep -E 'machineconfiguration|clusteroperator|machinesets'
```

Every OpenShift feature that goes beyond standard Kubernetes (MachineSets, MachineConfigs, ClusterOperators) is implemented as a CRD + controller — the same Operator pattern you are about to learn.

### Create a Custom Resource Definition

Now create your own CRD to understand the mechanics. You will define a new resource type called `WebApp` that could be used by an Operator to manage web applications.

Create a file called `webapp-crd.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.training.openshift.io
spec:
  group: training.openshift.io
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 10
              image:
                type: string
              port:
                type: integer
            required:
            - replicas
            - image
          status:
            type: object
            properties:
              availableReplicas:
                type: integer
    additionalPrinterColumns:
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Image
      type: string
      jsonPath: .spec.image
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    plural: webapps
    singular: webapp
    kind: WebApp
    shortNames:
    - wa
```

Let's understand the key fields:

- **group: training.openshift.io** — The API group for this resource. Combined with the version, it forms the `apiVersion` used in manifests (e.g., `training.openshift.io/v1`).
- **versions** — CRDs support multiple versions for evolution. Each version has its own schema.
- **schema.openAPIV3Schema** — Defines the structure and validation rules. The `minimum: 1` and `maximum: 10` on replicas means the API server will reject any value outside that range.
- **required** — Fields that must be present in every CR.
- **additionalPrinterColumns** — Custom columns shown when you run `oc get webapps`.
- **scope: Namespaced** — CRs exist within namespaces (as opposed to `Cluster` scope).
- **shortNames** — Lets you type `oc get wa` instead of `oc get webapps`.

Apply the CRD:

```bash
oc apply -f webapp-crd.yaml
```

Verify it was created:

```bash
oc get crd webapps.training.openshift.io
```

The Kubernetes API server now knows about the `WebApp` resource type. Wait a few seconds for the API server to fully register the new type, then use `oc explain` to explore it — just like any built-in resource:

```bash
oc explain webapp.spec
```

```
GROUP:      training.openshift.io
KIND:       WebApp
VERSION:    v1

FIELD: spec <Object>

FIELDS:
  image     <string> -required-
  port      <integer>
  replicas  <integer> -required-
```

This is the same `oc explain` you used for Deployments and Services — CRDs are truly first-class citizens in the API.

### Create a Custom Resource

Now create an instance of your CRD — a Custom Resource.

Create a project for this lab:

```bash
oc new-project operators-lab
```

Create a file called `my-webapp.yaml`:

```yaml
apiVersion: training.openshift.io/v1
kind: WebApp
metadata:
  name: my-webapp
spec:
  replicas: 3
  image: nginx:1.27
  port: 8080
```

Apply it:

```bash
oc apply -f my-webapp.yaml
```

List your WebApps:

```bash
oc get webapps
```

```
NAME        REPLICAS   IMAGE        AGE
my-webapp   3          nginx:1.27   5s
```

Notice the custom columns from `additionalPrinterColumns` — they show exactly the fields you defined. Try the short name:

```bash
oc get wa
```

Get the full details:

```bash
oc describe webapp my-webapp
```

### Test API Validation

Try creating a WebApp with an invalid replica count:

```bash
oc apply -f - <<EOF
apiVersion: training.openshift.io/v1
kind: WebApp
metadata:
  name: invalid-webapp
spec:
  replicas: 50
  image: nginx:1.27
EOF
```

The API server rejects this because the schema specifies `maximum: 10`. This validation happens automatically — no Operator code needed.

### Understanding the Gap

You have a CRD and a CR, but nothing actually happens. The `my-webapp` resource exists in the API, but no Pods are running. That is because there is no **controller** watching for WebApp resources and creating the corresponding Deployments, Services, and Pods.

This is the gap that Operators fill. A real Operator would:
1. Watch for WebApp CRs
2. Create a Deployment with the specified image and replica count
3. Create a Service on the specified port
4. Continuously reconcile — if someone deletes a Pod, the Operator ensures it comes back

Delete the test resources before moving on:

```bash
oc delete webapp my-webapp
oc delete webapp invalid-webapp 2>/dev/null
```

## Part 2: Operator Lifecycle Manager (OLM)

Writing an Operator from scratch requires significant Go programming. Fortunately, the OpenShift ecosystem provides hundreds of pre-built Operators through **OperatorHub**, and the **Operator Lifecycle Manager (OLM)** handles installing, upgrading, and managing them.

### OLM Architecture

OLM consists of several components that work together:

**CatalogSources** — Define where Operators are available from. OpenShift ships with four default catalogs:

```bash
oc get catalogsource -n openshift-marketplace
```

```
NAME                  DISPLAY               TYPE   PUBLISHER   AGE
certified-operators   Certified Operators   grpc   Red Hat
community-operators   Community Operators   grpc   Red Hat
redhat-marketplace    Red Hat Marketplace   grpc   Red Hat
redhat-operators      Red Hat Operators     grpc   Red Hat
```

Each catalog contains an index of available Operators from a specific source:
- **redhat-operators** — Operators built and supported by Red Hat
- **certified-operators** — Partner Operators certified by Red Hat
- **community-operators** — Community-maintained Operators
- **redhat-marketplace** — Commercial Operators from Red Hat Marketplace

**OperatorGroups** — Control which namespaces an Operator can watch. OpenShift creates a default `global-operators` group in the `openshift-operators` namespace:

```bash
oc get operatorgroup -n openshift-operators
```

**Subscriptions** — Tell OLM to install a specific Operator from a catalog and keep it updated.

**ClusterServiceVersions (CSVs)** — Describe an Operator's capabilities, required permissions, and owned CRDs. The CSV is the "unit of deployment" in OLM.

**InstallPlans** — The concrete steps OLM takes to install or upgrade an Operator.

### Explore ClusterOperators

Before installing a new Operator, look at the Operators that are already managing your cluster. OpenShift itself is composed of Operators:

```bash
oc get clusteroperators
```

You will see operators for DNS, ingress, authentication, the image registry, and many more. Each one follows the same pattern: it watches custom resources and reconciles the actual state of its component to match.

Pick one and examine it:

```bash
oc describe clusteroperator ingress
```

This shows the ingress Operator's status, conditions, and the versions it has been managing. Every piece of OpenShift infrastructure — from networking to the web console — is managed by an Operator.

### Browse Available Operators

Use the `packagemanifests` resource to see what Operators are available for installation. PackageManifests are populated from the CatalogSources:

```bash
oc get packagemanifests -n openshift-marketplace | head -20
```

**Note:** If you see `No resources found`, the CatalogSources may not be fully synced yet. Check their status:

```bash
oc get catalogsource -n openshift-marketplace -o custom-columns=NAME:.metadata.name,STATE:.status.connectionState.lastObservedState
```

All catalogs should show `READY`. If any show `TRANSIENT_FAILURE`, the cluster's pull secret may need to be updated for registry.redhat.io access.

To search for a specific Operator:

```bash
oc get packagemanifests -n openshift-marketplace | grep -i web-terminal
```

Get details about an Operator package:

```bash
oc describe packagemanifest web-terminal -n openshift-marketplace
```

This shows the available channels, the current CSV version, and the CRDs the Operator provides.

### Install an Operator from OperatorHub

You will install the **Web Terminal** Operator, which provides a browser-based terminal directly in the OpenShift web console. This is a useful Operator because it is straightforward to install and you can immediately see the result in the web console. It also demonstrates **dependency resolution** — OLM will automatically install the DevWorkspace Operator that Web Terminal depends on.

#### Step 1: Create the Subscription

The Subscription tells OLM which Operator to install, from which catalog, and which update channel to follow.

Create a file called `web-terminal-subscription.yaml`:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: web-terminal
  namespace: openshift-operators
spec:
  channel: fast
  installPlanApproval: Automatic
  name: web-terminal
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

Key fields:
- **channel: fast** — The update channel. Operators offer different channels (e.g., `stable`, `fast`) with different update cadences.
- **installPlanApproval: Automatic** — OLM will automatically install and upgrade the Operator. Set to `Manual` if you want to approve each update.
- **source: redhat-operators** — The CatalogSource to pull from.

Apply it:

```bash
oc apply -f web-terminal-subscription.yaml
```

#### Step 2: Monitor the Installation

Watch the Subscription status:

```bash
oc get subscription web-terminal -n openshift-operators
```

Wait for the CSVs to appear and reach the `Succeeded` phase. This may take a minute or two:

```bash
oc get csv -n openshift-operators
```

```
NAME                            DISPLAY                 VERSION   REPLACES                   PHASE
devworkspace-operator.v0.39.0   DevWorkspace Operator   0.39.0    devworkspace-operator...   Succeeded
web-terminal.v1.13.0            Web Terminal            1.13.0    web-terminal.v1.12.1...    Succeeded
```

Notice that **two** CSVs appeared — OLM automatically detected that Web Terminal depends on the DevWorkspace Operator and installed both. This is OLM's dependency resolution in action. The `Succeeded` phase means the Operator is installed and running.

#### Step 3: Verify the Installation

Check the InstallPlan that OLM created:

```bash
oc get installplan -n openshift-operators
```

This shows the concrete steps OLM took — which CRDs it created, which RBAC permissions it granted, and which Deployment it created for the Operator.

Check the Operator's Deployments:

```bash
oc get deployments -n openshift-operators
```

You should see three Deployments — the Web Terminal controller and two DevWorkspace components:

```
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
devworkspace-controller-manager   1/1     1            1           60s
devworkspace-webhook-server       2/2     2            2           55s
web-terminal-controller           1/1     1            1           60s
```

#### Step 4: Explore the Operator's CRDs

Every Operator brings its own CRDs. See what the Web Terminal Operator added:

```bash
oc get crd | grep devworkspace
```

Use `oc explain` to explore the new resource types:

```bash
oc explain devworkspace.spec
```

These CRDs are the Operator's API — you interact with the Operator by creating instances of its CRDs, just like you created a WebApp CR earlier.

#### Step 5: Use the Operator

Open the OpenShift web console in your browser. You should now see a terminal icon (`>_`) in the top-right toolbar. Click it to open a web-based terminal session directly in the console.

This terminal is a **Custom Resource** managed by the Web Terminal Operator. The Operator detected the CR, provisioned a Pod with a terminal container, and connected it to the web console. This is the Operator pattern in action — you declare what you want (a terminal), and the Operator handles the how.

### Understanding the OLM Resource Chain

Just as Kubernetes has a resource hierarchy (Deployment → ReplicaSet → Pod), OLM has its own:

```
CatalogSource (index of available Operators)
  └── PackageManifest (a single Operator's metadata)
        └── Subscription (request to install the Operator)
              └── InstallPlan (concrete installation steps)
                    └── ClusterServiceVersion (Operator deployment + permissions)
                          └── CRDs (the Operator's API)
                                └── Custom Resources (instances managed by the Operator)
```

Each layer handles a different concern:
- **CatalogSource** answers: *what Operators are available?*
- **Subscription** answers: *which Operator do I want, and how should it update?*
- **InstallPlan** answers: *what steps are needed to install it?*
- **CSV** answers: *what permissions and Deployments does the Operator need?*
- **CRDs** answer: *what API does the Operator expose?*
- **CRs** answer: *what do I want the Operator to manage?*

## Cleanup

Delete the CRD you created:

```bash
oc delete crd webapps.training.openshift.io
```

Delete the lab project:

```bash
oc delete project operators-lab
```

The Web Terminal Operator can remain installed — it is useful for the rest of the course.

## Congratulations

You have learned the Operator pattern from the ground up:

- **CRDs** extend the Kubernetes API with new resource types, complete with validation and custom columns
- **Custom Resources** are instances of CRDs that declare desired state
- **Operators** are controllers that watch CRs and reconcile actual state to match
- **OLM** manages the full lifecycle of Operators — installation, upgrades, and dependency resolution
- **OperatorHub** provides a catalog of hundreds of pre-built Operators from Red Hat, partners, and the community
- **Subscriptions** tell OLM which Operators to install and how to handle updates

The Operator pattern is the foundation of OpenShift's architecture. Every cluster component — from networking to the image registry to the web console — is managed by an Operator. Understanding this pattern is essential for operating and extending OpenShift.
