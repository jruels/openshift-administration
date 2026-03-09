# Automation Tools for OpenShift

In this lab, you will learn how to automate common OpenShift tasks using the `oc` CLI and Helm. Rather than clicking through the web console for every operation, automation lets you script repeatable workflows, extract exactly the data you need, and deploy complex applications with a single command. These are the skills that separate day-to-day operators from engineers who can manage clusters at scale.

## Part 1: oc CLI Power Features

The `oc` command is far more than `oc get` and `oc describe`. It includes powerful output formatting, filtering, and scripting capabilities that let you build automation scripts, monitoring checks, and operational tooling.

### Project Setup

Create a project for this lab:

```bash
oc new-project automation-tools
```

Grant the `anyuid` SCC so that nginx containers can run:

```bash
oc adm policy add-scc-to-user anyuid -z default -n automation-tools
```

Deploy a simple application to give us resources to work with:

```bash
oc create deployment web --image=nginx:1.27 --replicas=3
```

Wait for all Pods to be running:

```bash
oc rollout status deployment/web
```

Expose the Deployment with a Service and Route so we have a full resource stack:

```bash
oc expose deployment web --port=80
oc expose service web
```

Verify everything is running:

```bash
oc get all
```

You should see the Deployment, ReplicaSet, 3 Pods, Service, and Route.

### JSONPath Output

The default `oc get` output is designed for humans, but automation needs structured data. JSONPath lets you extract exactly the fields you need from any Kubernetes resource.

**Why this matters:** When you are writing scripts to monitor Pod health, generate reports, or feed data into other tools, you need machine-readable output — not a formatted table.

List just the Pod names and their current phase:

```bash
oc get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

Let's break down the JSONPath expression:

- `{range .items[*]}` — iterate over every item in the list
- `{.metadata.name}` — extract the Pod name
- `{"\t"}` — insert a tab character between fields
- `{.status.phase}` — extract the Pod status
- `{"\n"}` — insert a newline after each Pod
- `{end}` — close the range loop

Try extracting the cluster API server URL:

```bash
oc get infrastructure cluster -o jsonpath='{.status.apiServerURL}'
```

This is how automation scripts discover cluster endpoints without hardcoding them.

Extract all node names and their kubelet versions:

```bash
oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'
```

### Custom Columns

Custom columns give you table-formatted output with exactly the columns you choose. This is ideal for quick reporting and interactive use.

**Why this matters:** The default columns are often not what you need. Custom columns let you build purpose-specific views without writing a full script.

```bash
oc get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP
```

This produces a clean table with only the fields you care about:

```
NAME                   STATUS    IP
web-66d47686b4-xxxxx   Running   10.129.0.49
web-66d47686b4-yyyyy   Running   10.129.0.50
web-66d47686b4-zzzzz   Running   10.129.0.51
```

Try a more advanced example that shows node resource information:

```bash
oc get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory,OS:.status.nodeInfo.osImage
```

### Go Templates

Go templates are the most powerful output format. They support conditionals, loops, and functions — everything you need for complex reporting and automation logic.

**Why this matters:** When JSONPath is not expressive enough (for example, when you need conditional logic or string formatting), Go templates give you full control.

Map each Pod to the node it is running on:

```bash
oc get pods -o go-template='{{range .items}}{{.metadata.name}} -> {{.spec.nodeName}}{{"\n"}}{{end}}'
```

Show Deployments with their image versions:

```bash
oc get deployments -o go-template='{{range .items}}Deployment: {{.metadata.name}}{{"\n"}}{{range .spec.template.spec.containers}}  Image: {{.image}}{{"\n"}}{{end}}{{end}}'
```

### Shell Scripting with oc

The real power of these output formats shows when you combine them with shell scripting. Here are patterns you will use frequently in production.

**Check all Pods across all namespaces for non-Running status:**

```bash
oc get pods -A -o jsonpath='{range .items[?(@.status.phase!="Running")]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

The `?(@.status.phase!="Running")` is a JSONPath filter expression — it only returns items where the phase is NOT "Running". In a healthy cluster, this should return very little output.

**Loop through all Deployments and show their replica counts:**

```bash
for deploy in $(oc get deployments -o jsonpath='{.items[*].metadata.name}'); do
  replicas=$(oc get deployment "$deploy" -o jsonpath='{.spec.replicas}')
  ready=$(oc get deployment "$deploy" -o jsonpath='{.status.readyReplicas}')
  echo "$deploy: $ready/$replicas ready"
done
```

### Exploring the API with oc explain

When writing YAML manifests, you often need to know what fields are available and what they mean. `oc explain` is your built-in reference — it documents every field of every resource directly from the API server's OpenAPI schema.

**Why this matters:** Instead of searching the web for YAML examples, you can get authoritative field documentation instantly. This is especially useful for CRDs (Custom Resource Definitions) that may not have external documentation.

```bash
oc explain deployment.spec.strategy
```

```
GROUP:      apps
KIND:       Deployment
VERSION:    v1

FIELD: strategy <Object>

DESCRIPTION:
    The deployment strategy to use to replace existing pods with new ones.
    ...
FIELDS:
  rollingUpdate  <Object>
    ...
  type  <string>
    Type of deployment. Can be "Recreate" or "RollingUpdate".
```

Drill deeper into nested fields with dot notation:

```bash
oc explain deployment.spec.strategy.rollingUpdate
```

Use `--recursive` to see the entire field tree at a glance:

```bash
oc explain pod.spec --recursive | head -40
```

### Cluster Identity Commands

These commands are useful in scripts that need to verify context before making changes.

Show the current user:

```bash
oc whoami
```

Show the API server URL:

```bash
oc whoami --show-server
```

Show the current project:

```bash
oc project
```

These three commands are often the first lines in operational scripts — they confirm you are connected to the right cluster, as the right user, in the right project before making any changes.

## Part 2: Helm — Application Package Management

Helm is the standard package manager for Kubernetes. It lets you define, install, upgrade, and roll back complex applications as a single unit called a **chart**. A chart bundles all the Kubernetes manifests an application needs (Deployments, Services, ConfigMaps, etc.) along with configurable values.

**Why Helm matters:** Without Helm, deploying an application means managing many individual YAML files. Helm wraps them into a versioned, configurable package — like `apt` or `yum` for your cluster.

### Verify Helm Installation

Confirm Helm is installed:

```bash
helm version
```

You should see version information for Helm v3.

### Create a Helm Chart

Rather than installing a chart from a remote repository, you will create your own chart to understand what is inside one.

Switch to a working directory:

```bash
cd /tmp
```

Create a new chart scaffold:

```bash
helm create my-webapp
```

This generates a complete chart directory:

```
my-webapp/
  Chart.yaml          # Chart metadata (name, version, description)
  values.yaml         # Default configuration values
  charts/             # Dependencies (sub-charts)
  templates/          # Kubernetes manifest templates
    deployment.yaml
    service.yaml
    serviceaccount.yaml
    hpa.yaml
    ingress.yaml
    ...
```

### Explore the Chart Structure

Look at the chart metadata:

```bash
cat my-webapp/Chart.yaml
```

Key fields:
- **name** — The chart's name
- **version** — The chart version (follows Semantic Versioning)
- **appVersion** — The version of the application being deployed (nginx 1.16.0 by default)

Now look at the default values:

```bash
head -20 my-webapp/values.yaml
```

The `values.yaml` file defines every configurable parameter. Users can override any of these values at install time without modifying the templates. Notice that the default image is `nginx` — this chart will deploy a working nginx application out of the box.

Look at one of the templates to see how values are injected:

```bash
head -30 my-webapp/templates/deployment.yaml
```

The `{{ .Values.replicaCount }}` and `{{ .Values.image.repository }}` syntax pulls values from `values.yaml`. This is how a single chart can be reused across environments — development might use 1 replica while production uses 10, but the same chart serves both.

### Create a Project and Install the Chart

Create a project for the Helm deployment:

```bash
oc new-project helm-demo
```

Grant the `anyuid` SCC so nginx can run:

```bash
oc adm policy add-scc-to-user anyuid -z default -n helm-demo
```

The chart creates a custom ServiceAccount, so grant it access too:

```bash
oc adm policy add-scc-to-user anyuid -z my-webapp -n helm-demo
```

Install the chart:

```bash
helm install my-webapp ./my-webapp --set service.type=ClusterIP
```

The `--set` flag overrides a value from `values.yaml` without editing the file. Here we set the Service type to ClusterIP (the default for OpenShift).

You should see output confirming the release:

```
NAME: my-webapp
LAST DEPLOYED: ...
NAMESPACE: helm-demo
STATUS: deployed
REVISION: 1
```

### Verify the Deployment

Check what Helm created:

```bash
oc get all -n helm-demo
```

You should see a Deployment, ReplicaSet, Pod, Service, and ServiceAccount — all created from a single `helm install` command.

List Helm releases:

```bash
helm list
```

```
NAME     	NAMESPACE	REVISION	UPDATED                               	STATUS  	CHART          	APP VERSION
my-webapp	helm-demo	1       	...                                   	deployed	my-webapp-0.1.0	1.16.0
```

Get detailed status:

```bash
helm status my-webapp
```

### Upgrade a Release

One of Helm's most powerful features is upgrades. You can change configuration values and Helm will apply only the differences — just like a Deployment rolling update.

Scale to 3 replicas:

```bash
helm upgrade my-webapp ./my-webapp --set replicaCount=3 --set service.type=ClusterIP
```

Verify:

```bash
oc get pods -n helm-demo
```

You should now see 3 Pods running.

View the release history:

```bash
helm history my-webapp
```

```
REVISION	UPDATED                 	STATUS    	CHART          	APP VERSION	DESCRIPTION
1       	...                     	superseded	my-webapp-0.1.0	1.16.0     	Install complete
2       	...                     	deployed  	my-webapp-0.1.0	1.16.0     	Upgrade complete
```

Each upgrade creates a new revision. Helm keeps track of every change, making it easy to audit what changed and when.

### Rollback a Release

If an upgrade causes problems, you can instantly revert to any previous revision:

```bash
helm rollback my-webapp 1
```

```
Rollback was a success! Happy Helming!
```

Verify the rollback:

```bash
oc get pods -n helm-demo
```

You should see only 1 Pod again (the original configuration). Check the history:

```bash
helm history my-webapp
```

```
REVISION	UPDATED                 	STATUS    	CHART          	APP VERSION	DESCRIPTION
1       	...                     	superseded	my-webapp-0.1.0	1.16.0     	Install complete
2       	...                     	superseded	my-webapp-0.1.0	1.16.0     	Upgrade complete
3       	...                     	deployed  	my-webapp-0.1.0	1.16.0     	Rollback to 1
```

Notice that rollback creates a **new** revision (3), not a revert. The full history is preserved.

### Uninstall a Release

Remove the application and all its resources:

```bash
helm uninstall my-webapp
```

Verify everything is gone:

```bash
helm list
oc get all -n helm-demo
```

The release is removed and all Kubernetes resources it created are deleted.

## Cleanup

Delete the projects:

```bash
oc delete project automation-tools
oc delete project helm-demo
```

## Congratulations

You have learned two essential automation approaches for OpenShift:

**oc CLI Power Features:**
- **JSONPath** — Extract specific fields for machine-readable output
- **Custom Columns** — Build purpose-specific table views
- **Go Templates** — Full-power formatting with loops and conditionals
- **Shell Scripting** — Combine oc output with shell logic for operational automation
- **oc explain** — Discover API fields without leaving the terminal

**Helm:**
- **Charts** package all of an application's Kubernetes resources into a single, versioned unit
- **Values** make charts configurable without modifying templates
- **Upgrades** apply configuration changes incrementally
- **Rollbacks** instantly revert to any previous revision
- **History** provides a complete audit trail of every change

Together, these tools transform OpenShift from a system you operate manually into one you can manage programmatically and at scale.
