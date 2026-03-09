# ConfigMaps and Secrets

In this lab, you will learn how to externalize application configuration using ConfigMaps and how to manage sensitive data using Secrets. These two resources are the standard Kubernetes mechanism for separating configuration from container images — you build your image once and configure it differently for each environment (development, staging, production) without rebuilding.

This lab has two parts:

1. **ConfigMaps** — Store non-sensitive configuration data as key-value pairs or files
2. **Secrets** — Store sensitive data (passwords, API keys, certificates) with base64 encoding

## Why Externalize Configuration?

Hardcoding configuration values (database URLs, feature flags, cache sizes) into container images creates several problems:
- You must rebuild the image for every configuration change
- The same image cannot be used across environments
- Sensitive values like passwords get baked into image layers

ConfigMaps and Secrets solve this by injecting configuration at runtime — through environment variables or mounted files — so the same image works everywhere with different settings.

## Project Setup

Create a dedicated project for this lab:

```bash
oc new-project configmaps-secrets
```

## Part 1: ConfigMaps

A ConfigMap holds configuration data as key-value pairs. Pods can consume ConfigMaps in two ways:
- **Environment variables** — Each key becomes a variable name, and the value becomes its value
- **Volume mounts** — Each key becomes a file name, and the value becomes the file contents

### Create a ConfigMap from literal values

The simplest way to create a ConfigMap is with `--from-literal` flags:

```bash
oc create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=production
```

Verify the ConfigMap was created:

```bash
oc get configmap app-config
```

```
NAME         DATA   AGE
app-config   2      5s
```

The `DATA` column shows 2 key-value pairs. View the contents:

```bash
oc describe configmap app-config
```

```
Name:         app-config
Namespace:    configmaps-secrets

Data
====
APP_COLOR:
----
blue

APP_MODE:
----
production
```

### Use a ConfigMap as environment variables

The most common way to consume a ConfigMap is as environment variables. Create a Pod that loads all keys from the ConfigMap:

Create a file called `env-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ["sleep", "3600"]
    envFrom:
    - configMapRef:
        name: app-config
  restartPolicy: Never
```

The `envFrom` field loads **every** key from `app-config` as an environment variable. This is convenient when you want all values injected at once.

Apply it:

```bash
oc apply -f env-pod.yaml
```

Wait a few seconds for the Pod to start, then verify the environment variables:

```bash
oc exec env-pod -- env | grep APP_
```

```
APP_MODE=production
APP_COLOR=blue
```

Both values are available as environment variables inside the container.

You can also select individual keys if you only need specific values. Create a file called `env-pod-selective.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod-selective
spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ["sleep", "3600"]
    env:
    - name: COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_COLOR
  restartPolicy: Never
```

The `valueFrom.configMapKeyRef` syntax picks a single key from the ConfigMap and maps it to a custom environment variable name (`COLOR` instead of `APP_COLOR`).

Apply and verify:

```bash
oc apply -f env-pod-selective.yaml
```

```bash
oc exec env-pod-selective -- sh -c 'echo $COLOR'
```

```
blue
```

### Create a ConfigMap from a file

ConfigMaps can also hold entire configuration files. This is useful for applications that read their configuration from a file on disk (like Redis, Nginx, or any app with a `.conf` file).

Create a Redis configuration file:

```bash
cat > /tmp/redis.conf <<EOF
maxmemory 2mb
maxmemory-policy allkeys-lru
EOF
```

Create a ConfigMap from this file:

```bash
oc create configmap redis-config --from-file=/tmp/redis.conf
```

View the contents:

```bash
oc describe configmap redis-config
```

```
Data
====
redis.conf:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

The file name (`redis.conf`) becomes the key, and the file contents become the value.

### Mount a ConfigMap as a volume

Now deploy a Redis Pod that mounts this configuration file. Create a file called `redis-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:7-alpine
    command:
      - redis-server
      - "/redis-master/redis.conf"
    ports:
    - containerPort: 6379
    volumeMounts:
    - mountPath: /redis-master
      name: config
  volumes:
  - name: config
    configMap:
      name: redis-config
      items:
      - key: redis.conf
        path: redis.conf
```

Key fields:
- **volumes** — Declares a volume backed by the `redis-config` ConfigMap
- **items** — Maps the ConfigMap key `redis.conf` to the file path `redis.conf` inside the volume
- **volumeMounts** — Mounts the volume at `/redis-master` inside the container
- **command** — Starts Redis using the mounted configuration file at `/redis-master/redis.conf`

Apply it:

```bash
oc apply -f redis-pod.yaml
```

Wait for the Pod to start:

```bash
oc get pod redis
```

Verify that Redis loaded the configuration from the ConfigMap:

```bash
oc exec redis -- redis-cli CONFIG GET maxmemory
```

```
maxmemory
2097152
```

The value `2097152` is 2 MB in bytes — matching the `maxmemory 2mb` setting from the ConfigMap.

```bash
oc exec redis -- redis-cli CONFIG GET maxmemory-policy
```

```
maxmemory-policy
allkeys-lru
```

The Redis server is running with configuration injected entirely from a ConfigMap. If you need to change the configuration, you update the ConfigMap — no image rebuild required.

### Clean up ConfigMap resources

```bash
oc delete pod redis env-pod env-pod-selective
oc delete configmap app-config redis-config
```

## Part 2: Secrets

Secrets are similar to ConfigMaps but designed for sensitive data like passwords, API keys, and TLS certificates. The key differences are:

- **Base64 encoding** — Secret values are stored as base64-encoded strings (not encrypted, but not plaintext)
- **Access control** — RBAC policies can restrict who can read Secrets separately from ConfigMaps
- **Memory storage** — Secrets can be stored in tmpfs on nodes, avoiding disk writes
- **Redacted output** — `oc get` and `oc describe` hide Secret values by default

### Create a Secret from literal values

```bash
oc create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=s3cur3P@ss
```

The `generic` type creates an Opaque Secret, which is the default type for arbitrary user data.

Examine the Secret:

```bash
oc get secret db-credentials
```

```
NAME             TYPE     DATA   AGE
db-credentials   Opaque   2      5s
```

Describe it:

```bash
oc describe secret db-credentials
```

```
Name:         db-credentials
Namespace:    configmaps-secrets

Type:  Opaque

Data
====
password:  10 bytes
username:  5 bytes
```

Notice that `describe` shows the **size** of each value but not the content. This protects Secrets from being exposed in terminal logs.

### View and decode Secret values

To see the actual values, you must request the YAML output:

```bash
oc get secret db-credentials -o yaml
```

The `data` section shows base64-encoded values:

```yaml
data:
  password: czNjdXIzUEBzcw==
  username: YWRtaW4=
```

Decode them using `base64 --decode`:

```bash
oc get secret db-credentials -o jsonpath='{.data.username}' | base64 --decode
```

```
admin
```

```bash
oc get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode
```

```
s3cur3P@ss
```

### Create a Secret from YAML

You can also create Secrets declaratively in YAML. The values must be base64-encoded.

First, encode the values:

```bash
echo -n "abcd1234567890" | base64
```

```
YWJjZDEyMzQ1Njc4OTA=
```

```bash
echo -n "supersecretvalue" | base64
```

```
c3VwZXJzZWNyZXR2YWx1ZQ==
```

**Important:** Always use `echo -n` (no newline) when encoding. Without `-n`, the newline character gets encoded into the value, which can cause authentication failures.

Create a file called `api-keys.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
type: Opaque
data:
  api-key: YWJjZDEyMzQ1Njc4OTA=
  api-secret: c3VwZXJzZWNyZXR2YWx1ZQ==
```

Apply it:

```bash
oc apply -f api-keys.yaml
```

Verify:

```bash
oc get secret api-keys
```

### Mount a Secret as a volume

Secrets can be mounted as files, just like ConfigMaps. This is how applications read certificates and key files.

Create a file called `secret-vol-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ["sleep", "3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
  restartPolicy: Never
```

The Secret keys become file names inside the mount path, and the values are automatically base64-decoded into the files.

Apply it:

```bash
oc apply -f secret-vol-pod.yaml
```

Wait a few seconds, then verify the files exist:

```bash
oc exec secret-vol-pod -- ls /etc/secrets
```

```
password
username
```

Read the values — they are already decoded:

```bash
oc exec secret-vol-pod -- cat /etc/secrets/username
```

```
admin
```

```bash
oc exec secret-vol-pod -- cat /etc/secrets/password
```

```
s3cur3P@ss
```

### Use a Secret as environment variables

Create a file called `secret-env-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: busybox
    image: busybox:1.36
    command: ["sleep", "3600"]
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
  restartPolicy: Never
```

Apply it:

```bash
oc apply -f secret-env-pod.yaml
```

Verify the environment variables:

```bash
oc exec secret-env-pod -- sh -c 'echo $DB_USERNAME'
```

```
admin
```

```bash
oc exec secret-env-pod -- sh -c 'echo $DB_PASSWORD'
```

```
s3cur3P@ss
```

The Secret values are decoded and injected as plain text environment variables inside the container.

## Cleanup

Delete the project and all resources:

```bash
oc delete project configmaps-secrets
```

## Congratulations

You have learned how to externalize application configuration in OpenShift:

**ConfigMaps:**
- Created from literal values (`--from-literal`) and from files (`--from-file`)
- Consumed as environment variables using `envFrom` (all keys) or `valueFrom.configMapKeyRef` (individual keys)
- Mounted as files using volumes — ideal for applications that read configuration from disk (like Redis)

**Secrets:**
- Created from literal values or from YAML with base64-encoded data
- Values are hidden by `oc get` and `oc describe` to prevent accidental exposure
- Mounted as volumes where keys become files with decoded values
- Injected as environment variables using `secretKeyRef`

The key pattern is the same for both: **define the data once, then inject it into Pods through environment variables or volume mounts.** ConfigMaps are for non-sensitive configuration, Secrets are for sensitive data. Together, they let you build images once and configure them differently for every environment.
