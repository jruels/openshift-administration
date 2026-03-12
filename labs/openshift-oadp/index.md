# Backing up and Restoring Applications with OADP

## Overview

OpenShift API for Data Protection (OADP) offers a robust solution for data protection by integrating the OADP Operator, Velero, and Restic to enable seamless backup and recovery of applications and data. This hands-on lab will guide you through the steps of deploying a sample WordPress application with a MariaDB database backed by a persistent volume, performing backups, simulating data loss by deleting the project, and restoring it to verify the backup's integrity and effectiveness.

## Deploying WordPress with Helm

In this section you will deploy a WordPress application with a MariaDB database using the Helm CLI. The application uses persistent volumes for both WordPress content and the MariaDB database, which makes it a good candidate for testing backup and restore.

### Add the Bitnami Helm repository

Add the Bitnami Helm chart repository and update your local cache:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

> **Note:** You may see a warning about Bitnami requiring a paid subscription. You can safely ignore this — the charts will still install and function normally for this lab.

### Create the dev project

```bash
oc new-project dev
```

### Install the WordPress chart

Deploy WordPress with a MariaDB database:

```bash
helm install wordpress bitnami/wordpress -n dev \
  --set wordpressUsername=user \
  --set wordpressPassword=password \
  --set wordpressBlogName='My Blog' \
  --set service.type=ClusterIP \
  --set mariadb.auth.rootPassword=password
```

### Wait for the pods to start

Monitor the pod status until both `wordpress` and `wordpress-mariadb` pods show `1/1 Running`:

```bash
oc get pods -n dev -w
```

Press `Ctrl+C` once both pods are running. This typically takes 1-2 minutes.

### Expose WordPress

Create a Route so WordPress is accessible from outside the cluster:

```bash
oc expose svc/wordpress -n dev
```

Get the Route URL:

```bash
oc get route wordpress -n dev
```

Open the URL from the `HOST/PORT` column in your browser to confirm WordPress is running.

### Create content to verify backup

Log into the WordPress admin panel by appending `/wp-admin` to the Route URL:

- Username = **user**
- Password = **password**

Create a new page called **OADP** and publish it. This page will be used to verify the backup and restore worked correctly.

## Setting Up OADP

### Install the OADP Operator

In the OpenShift Web Console:

- Expand **Ecosystem** in the left sidebar and click **Operators**.
- In the search box, type **OADP** and select the **Red Hat OADP Operator**.
- Click **Install**, accept the defaults, and click **Install** again.
- Wait for the operator to show **Succeeded** status. This may take a few minutes.

### Create the S3 Bucket

OADP stores backups in an S3 bucket. Create one on your bastion, replacing `<YOUR_INITIALS>` and `<YYYYMMDD>` with your initials and today's date (e.g., `jd-20260312`):

```bash
aws s3 mb s3://<YOUR_INITIALS>-<YYYYMMDD> --region us-west-1
```

Verify the bucket was created:

```bash
aws s3 ls | grep <YOUR_INITIALS>
```

### Create Cloud Credentials Secret

Create a secret containing the AWS credentials that OADP will use to access S3. Run this on your bastion:

```bash
oc create secret generic cloud-credentials -n openshift-adp \
  --from-file cloud=<(echo -e "[default]\naws_access_key_id=$(grep aws_access_key_id ~/.aws/credentials | awk -F= '{print $2}' | tr -d ' ')\naws_secret_access_key=$(grep aws_secret_access_key ~/.aws/credentials | awk -F= '{print $2}' | tr -d ' ')")
```

Verify the secret was created:

```bash
oc get secret cloud-credentials -n openshift-adp
```

### Create the DataProtectionApplication

Create the DataProtectionApplication (DPA) custom resource. **Replace `<YOUR_INITIALS>-<YYYYMMDD>` with the bucket name you created earlier:**

```bash
cat <<EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-sample
  namespace: openshift-adp
spec:
  backupLocations:
  - name: default
    velero:
      config:
        profile: default
        region: us-west-1
      credential:
        key: cloud
        name: cloud-credentials
      default: true
      objectStorage:
        bucket: <YOUR_INITIALS>-<YYYYMMDD>
        prefix: oadp
      provider: aws
  configuration:
    nodeAgent:
      enable: true
      uploaderType: restic
    velero:
      defaultPlugins:
      - openshift
      - aws
  snapshotLocations:
  - velero:
      config:
        profile: default
        region: us-west-1
      provider: aws
EOF
```

## Verifying the Installation

Run the following commands to verify OADP is set up correctly:

- **Verify Resources**: Check that all the correct resources have been created:
   ```bash
   oc get all -n openshift-adp
   ```
   You should see a `velero` pod, a `node-agent` pod (one per worker node), and the `openshift-adp-controller-manager` pod, all in `Running` state.

- **Verify the DPA** has been reconciled:
   ```bash
   oc get dpa dpa-sample -n openshift-adp -o jsonpath='{.status}'
   ```
   The output should contain `"type":"Reconciled"`.

- **Verify the BackupStorageLocation**:
   ```bash
   oc get backupStorageLocation -n openshift-adp
   ```
   The **PHASE** should be **Available**. If it shows **Unavailable**, double-check your cloud credentials secret and S3 bucket name.

## Backup WordPress

Now that OADP is configured, create a backup of the `dev` namespace.

In the OpenShift Web Console:

* Expand **Ecosystem** in the left sidebar and click **Installed Operators**.
* Select the **OADP Operator** and click the **Backup** tab.
* Click **Create Backup**.
* Switch to **YAML view** and replace the contents with:

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: backup
  namespace: openshift-adp
spec:
  includedNamespaces:
  - dev
  snapshotVolumes: true
  storageLocation: default
```

* Click **Create**.

Monitor the status of the backup. Refresh the page until the **Phase** changes to **Completed**. You can also check from the command line:

```bash
oc get backup backup -n openshift-adp -o jsonpath='{.status.phase}'
```

## Delete the WordPress app and data

Now simulate a disaster by deleting the entire `dev` project:

```bash
oc delete project dev
```

Wait for the project to be fully deleted, then confirm all resources are gone:

```bash
oc get pods -n dev
```

You should see `No resources found in dev namespace.`

## Restore the dev namespace

Use OADP to restore the deleted `dev` namespace from your backup.

In the OpenShift Web Console:

* Go to the **OADP Operator** (Ecosystem → Installed Operators → OADP Operator).
* Click the **Restore** tab, then **Create Restore**.
* Switch to **YAML view** and replace the contents with:

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore
  namespace: openshift-adp
spec:
  backupName: backup
  includedNamespaces:
  - dev
  restorePVs: true
```

* Click **Create**.

Monitor the status until **Phase** changes to **Completed**:

```bash
oc get restore restore -n openshift-adp -o jsonpath='{.status.phase}'
```

Once the restore completes, verify the resources are running:

```bash
oc get pods -n dev
```

Wait for both pods to show `1/1 Running`, then confirm the WordPress site loads in your browser and shows the **OADP** page you created earlier.

## Schedule backups

It is best practice to schedule backups to run automatically. For this lab, create a schedule that runs every five minutes.

* In the OADP Operator, click the **Schedule** tab, then **Create Schedule**.
* Switch to **YAML view** and replace the contents with:

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: dev-backup-schedule
  namespace: openshift-adp
spec:
  schedule: "*/5 * * * *"
  template:
    includedNamespaces:
    - dev
    snapshotVolumes: true
    storageLocation: default
```

* Click **Create**.

After five minutes, check the **Backup** tab. You should see a new backup created automatically by the schedule. Confirm its **Phase** is **Completed**.

## Congratulations

---

Congratulations on completing the lab! You have successfully deployed a WordPress application with persistent volumes using Helm, installed and configured the OADP operator with S3 storage, created a backup, simulated a disaster by deleting the application, restored it from backup, and set up automated scheduled backups. These skills are essential for managing data protection in an OpenShift environment.
