# Installing an OpenShift Cluster on AWS

In this lab, you will install a fully functional OpenShift Container Platform cluster on Amazon Web Services. Rather than clicking through a web console, you will work with the OpenShift installer and a declarative YAML configuration file — the same approach used in production environments. By the end, you will understand every component the installer creates, why each piece is necessary, and how to connect to your running cluster.

## What You Will Build

The installer creates the following AWS infrastructure on your behalf:

- A **VPC** with public and private subnets across availability zones
- **EC2 instances** for the control plane (master) and compute (worker) nodes
- **Network Load Balancers** for the Kubernetes API (`api.<cluster>.<domain>`) and application traffic (`*.apps.<cluster>.<domain>`)
- **Route 53 DNS records** so that clients can resolve the API and application endpoints
- **IAM roles and instance profiles** so the nodes can interact with AWS services
- **Security groups** that control network access between nodes, load balancers, and the internet

Understanding this is important: the OpenShift installer is not just copying files to a server. It is provisioning an entire cloud environment designed for production workloads.

## Prerequisites

The following resources have already been prepared for this training environment:

| Resource | Details |
|----------|---------|
| **AWS Account** | An IAM user with `AdministratorAccess` has been created |
| **DNS Domain** | A Route 53 hosted zone for your base domain already exists |
| **Bastion Host** | An EC2 instance you will connect to from VS Code |
| **Pull Secret** | Already included in the configuration file — no Red Hat account needed |

If you were setting this up from scratch, you would need to register a domain in Route 53 (which automatically creates a hosted zone), create an IAM user with admin permissions, and launch a bastion host. These steps are documented in the reference section at the end of this lab.

## Set Up VS Code for Remote Development

Throughout this course, you will use **Visual Studio Code** as your primary tool for editing files and running terminal commands on the bastion host. VS Code's **Remote - SSH** extension lets you open a full editor session on a remote machine — you get syntax highlighting, file browsing, multi-file editing, and an integrated terminal, all running against the bastion host as if it were local.

This is a significant improvement over working with command-line text editors like `vi`. You can see the entire file, make precise edits visually, and keep multiple files open at once.

### Download the Lab SSH Key

At the top of the lab page, click **View on GitHub**. Click the green **Code** button in the top right corner, then click **Download as zip**. Extract the zip file and note the location of the `keys` directory — it contains the `lab.pem` file you will need.

### Set Key Permissions (Mac/Linux only)

SSH requires that private key files be unreadable by other users. Open a local terminal and run:

```bash
chmod 600 /path/to/keys/lab.pem
```

Replace `/path/to/keys/` with the actual path where you extracted the lab files. Windows users can skip this step — downloaded files on Windows have restricted permissions by default.

### Create an SSH Config File

An SSH config file tells VS Code (and the `ssh` command) how to connect to a host by name, without having to remember IP addresses, usernames, and key paths every time. This is the same file that system administrators use to manage dozens or hundreds of servers.

Open VS Code and press `Ctrl+Shift+P` (Windows/Linux) or `Cmd+Shift+P` (Mac) to open the **Command Palette**. Type `Remote-SSH: Open SSH Configuration File` and select it. VS Code will present a list of config file paths — select the **first option** in the list. This is the user-level SSH config file and is the correct choice on both Mac/Linux and Windows.

If the file does not exist, VS Code will create it for you. Add the following entry:

```
Host bastion
    HostName <BASTION_IP>
    User ec2-user
    IdentityFile /path/to/keys/lab.pem
```

Replace:
- `<BASTION_IP>` — The IP address provided by the instructor
- `/path/to/keys/lab.pem` — The full path to the SSH key you downloaded. Forward-slash paths (e.g., `C:/Users/YourName/Downloads/keys/lab.pem`) work on all platforms, including Windows.

Save the file.

### Connect to the Bastion Host

Now use the Remote Explorer to connect:

1. Click the **Remote Explorer** icon in the VS Code sidebar (it looks like a monitor icon).
2. You should see **bastion** listed under the SSH section.
3. Click the connect icon next to **bastion** (or right-click and select **Connect in Current Window**).
4. VS Code will ask you to select the platform of the remote host — choose **Linux**.
5. The first connection takes a moment while VS Code installs its server component on the bastion. Once connected, the bottom-left corner of VS Code will show `SSH: bastion`.

You now have a full VS Code session running on the bastion host. The file explorer on the left shows the bastion's filesystem, and any terminal you open runs commands on the bastion.

### Open a Terminal

Open the integrated terminal by pressing `` Ctrl+` `` (backtick) or going to **Terminal > New Terminal** in the menu bar. This terminal is running on the bastion host — every command you type here executes remotely, just as if you had SSH'd in from a traditional terminal.

You will use this integrated terminal for all the commands in the rest of this lab.

## Generate an SSH Key Pair

The OpenShift installer embeds an SSH **public** key into every cluster node it provisions. This allows you to SSH directly into the nodes later for debugging — the key is added to the `authorized_keys` file for the `core` user on each node.

We use the Ed25519 algorithm because it produces shorter keys with stronger security than RSA, and it is the recommended key type for modern systems.

In the VS Code integrated terminal, run:

```bash
ssh-keygen -t ed25519 -N '' -f ${HOME}/.ssh/ocp4-aws-key
```

- **`-t ed25519`** — Use the Ed25519 algorithm.
- **`-N ''`** — Set an empty passphrase (acceptable for a lab; in production you would use a passphrase or a key agent).
- **`-f`** — Write the key pair to a specific file path.

View the public key — you will need this value in a later step. In the VS Code file explorer, navigate to `/home/ec2-user/.ssh/` and open `ocp4-aws-key.pub`. You can also open it from the terminal:

```bash
code ${HOME}/.ssh/ocp4-aws-key.pub
```

Copy the entire contents of this file and save it somewhere accessible (a new untitled tab in VS Code, a text file, etc.). It will look similar to:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ec2-user@ip-172-31-x-x.us-west-1.compute.internal
```

## Verify AWS Credentials

The OpenShift installer needs AWS API access to create all the infrastructure described earlier. It reads credentials from the standard AWS credentials file (`~/.aws/credentials`), the same file used by the AWS CLI. These credentials have already been configured on your bastion host.

Verify they work by running the following in the integrated terminal:

```bash
aws sts get-caller-identity
```

You should see output showing the account ID and IAM user ARN. If you get an error, notify the instructor.

## Verify the OpenShift Tools

Two tools from Red Hat have already been installed on your bastion host:

- **`oc`** — The OpenShift command-line client. This is a superset of `kubectl` that adds OpenShift-specific commands (projects, routes, builds, etc.). You will use `oc` for all cluster interactions after installation.
- **`openshift-install`** — The installer binary. It reads your configuration file, calls the AWS API to provision infrastructure, waits for the cluster to bootstrap, and reports the credentials when finished.

Verify they are available by running the following in the integrated terminal:

```bash
openshift-install version
```

```bash
oc version --client
```

You should see version numbers for both tools. Note the OpenShift version — this is the version of the cluster you are about to install.

## Understand the Installation Configuration

The OpenShift installer uses a file called `install-config.yaml` to determine exactly what to build. This is the single most important file in the installation process — it controls everything from how many nodes to create, to which AWS region to use, to what networking configuration the cluster will have.

Instead of running the installer's interactive prompts (which ask you the same questions one at a time), we will use a pre-built configuration file. This approach is repeatable, scriptable, and — most importantly — lets you see every setting in one place so you can understand what each one does.

A copy of `install-config.yaml` is included in this lab's directory. Let's examine it section by section.

### The Full Configuration File

```yaml
apiVersion: v1
baseDomain: labfun.org
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m6i.xlarge
  replicas: 1
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      type: m6i.2xlarge
  replicas: 1
metadata:
  name: studentN
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: us-west-1
publish: External
pullSecret: '<provided>'
sshKey: 'YOURSSHKEY'
```

### Field-by-Field Explanation

**`baseDomain: labfun.org`**

This is the DNS domain under which the cluster will be created. The installer creates a subdomain for your cluster — for example, if your cluster name is `trainer`, the API endpoint will be `api.trainer.labfun.org` and applications will be accessible at `*.apps.trainer.labfun.org`. A Route 53 hosted zone for this domain must already exist in the AWS account.

**`compute` (Worker Nodes)**

```yaml
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m6i.xlarge
  replicas: 1
```

- **`architecture: amd64`** — Use standard x86_64 processors. ARM-based nodes (`aarch64`) are also supported.
- **`hyperthreading: Enabled`** — Allow each physical CPU core to run two threads. This effectively doubles the available vCPUs and is the default for most production workloads.
- **`type: m6i.xlarge`** — The AWS EC2 instance type. An `m6i.xlarge` provides 4 vCPUs and 16 GiB of RAM — enough for a lab worker node. In production, you would choose a larger instance type based on workload requirements.
- **`replicas: 1`** — The number of worker nodes. For this lab, a single worker is sufficient. Production clusters typically run 3 or more workers for high availability.

Worker nodes run your application workloads. The control plane schedules Pods onto workers based on resource availability.

**`controlPlane` (Master Nodes)**

```yaml
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      type: m6i.2xlarge
  replicas: 1
```

- **`type: m6i.2xlarge`** — 8 vCPUs and 32 GiB of RAM. The control plane runs the Kubernetes API server, etcd (the cluster database), the scheduler, and the controller manager. These components are memory-intensive, which is why the master gets a larger instance type than the worker.
- **`replicas: 1`** — A single control plane node. Production clusters use 3 master nodes so that etcd can maintain quorum if one node fails. For a lab, 1 is sufficient.

**`metadata`**

```yaml
metadata:
  name: studentN
```

The cluster name. This becomes part of every DNS record and AWS resource tag. Replace `studentN` with the name assigned by the instructor (e.g., `student1`, `student2`). The cluster name must be lowercase and may contain hyphens. Keep it short — the installer combines it with a random suffix to name AWS resources, and some AWS resource types have character limits.

**`networking`**

```yaml
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
```

OpenShift uses three separate network ranges, and understanding why they are separate is important:

- **`clusterNetwork` (10.128.0.0/14)** — The IP range used for Pod networking. Every Pod in the cluster gets an IP from this range. The `hostPrefix: 23` means each node gets a `/23` subnet (512 addresses), which limits how many Pods can run on a single node.
- **`serviceNetwork` (172.30.0.0/16)** — The IP range for Kubernetes Services. When you create a Service, it gets a virtual IP (ClusterIP) from this range. These IPs are internal to the cluster.
- **`machineNetwork` (10.0.0.0/16)** — The IP range for the actual AWS VPC. This is the "real" network that the EC2 instances are connected to.

These three ranges must **not overlap** with each other or with your existing corporate networks. The defaults are safe for most environments.

- **`networkType: OVNKubernetes`** — The Container Network Interface (CNI) plugin. OVN-Kubernetes is the default for OpenShift 4.x and provides advanced features like network policy enforcement, egress IP management, and hardware offloading.

**`platform`**

```yaml
platform:
  aws:
    region: us-west-1
```

Specifies the cloud provider and region. The installer creates all infrastructure in this region.

**`publish: External`**

Makes the cluster's API and application endpoints publicly accessible via the internet. The alternative, `Internal`, restricts access to within the VPC — useful for private clusters that are only reachable through a VPN or Direct Connect.

**`pullSecret`**

JSON credentials that grant the cluster permission to pull container images from Red Hat's registries. This is already provided in the configuration file — you do not need to obtain your own.

**`sshKey`**

The public key from the key pair you generated earlier. Embedded into every node for emergency SSH access.

## Prepare the Installation

Now that you understand what every field does, let's prepare the configuration file and run the installer.

### Create a Clean Installation Directory

The installer writes all generated assets (Kubernetes manifests, Ignition configs, credentials) into a directory you specify. It is critical that this directory starts clean — leftover files from a previous attempt will cause problems. In the VS Code integrated terminal, run:

```bash
rm -rf ~/ocp-install && mkdir -p ~/ocp-install
```

If this is your first time, there is nothing to remove, but this command is safe to run regardless.

### Get the Configuration File onto the Bastion

The `install-config.yaml` file is included in this lab's GitHub repository. Download it directly to the bastion using the terminal:

```bash
wget -q -O ~/ocp-install/install-config.yaml https://raw.githubusercontent.com/jruels/openshift-administration/main/labs/openshift-install-updated/install-config.yaml
```

Verify the file was downloaded:

```bash
ls ~/ocp-install/install-config.yaml
```

You should see the file path printed back. If you get "No such file or directory", re-run the `wget` command above.

### Edit the Configuration File

The configuration file has two placeholders that need to be replaced: the cluster name and the SSH public key. Run the following commands in the terminal, replacing `studentN` with your assigned cluster name (e.g., `student1`, `student2`):

```bash
sed -i "s/studentN/YOUR_STUDENT_NUMBER/" ~/ocp-install/install-config.yaml
```

For example, if you are student 3:

```bash
sed -i "s/studentN/student3/" ~/ocp-install/install-config.yaml
```

Next, inject your SSH public key (generated in the earlier step) into the configuration file:

```bash
sed -i "s|YOURSSHKEY|$(cat ~/.ssh/ocp4-aws-key.pub)|" ~/ocp-install/install-config.yaml
```

The pull secret is already included in the configuration file — you do not need to obtain one from the Red Hat Hybrid Cloud Console.

Verify your changes by opening the file in VS Code:

```bash
code ~/ocp-install/install-config.yaml
```

Confirm that `metadata.name` shows your assigned cluster name and that `sshKey` contains your public key (a long string starting with `ssh-ed25519`).

### Back Up the Configuration

This is a critical step that is easy to forget. The installer **consumes** (deletes) the `install-config.yaml` file when it runs. If the installation fails and you need to start over, you will need the original file. Always make a backup:

```bash
cp ~/ocp-install/install-config.yaml ~/install-config-backup.yaml
```

## Customize the Ingress Load Balancer

Before running the installer, you will customize one infrastructure setting: the type of load balancer used for application traffic (Routes). By default, OpenShift creates a Classic Load Balancer for the ingress router. You will change this to a **Network Load Balancer (NLB)**, which offers better performance, supports static IPs, and avoids the lower Classic LB quota in AWS.

To customize infrastructure settings, you first generate the installation manifests, modify them, and then run the installer. This is a common pattern in production environments where defaults need to be tuned.

### Generate Manifests

This command reads your `install-config.yaml` and generates all the Kubernetes manifests the installer will use. Note that it **consumes** the `install-config.yaml` file (this is why you made a backup earlier):

```bash
openshift-install create manifests --dir=$HOME/ocp-install
```

You will see output listing the generated manifest files. The directory now contains `manifests/` and `openshift/` subdirectories with YAML files for every cluster component.

### Configure NLB for the Ingress Controller

Open the ingress configuration manifest in VS Code:

```bash
code ~/ocp-install/manifests/cluster-ingress-02-config.yml
```

You will see a section that specifies the load balancer type as `Classic`:

```yaml
  loadBalancer:
    platform:
      aws:
        type: Classic
      type: AWS
```

Change `Classic` to `NLB`:

```yaml
  loadBalancer:
    platform:
      aws:
        type: NLB
      type: AWS
```

Save the file. This tells the ingress operator to create a Network Load Balancer instead of a Classic Load Balancer for the default router.

## Run the Installation

Now start the installer. Since the manifests already exist, the installer will use them directly:

```bash
openshift-install create cluster --dir=$HOME/ocp-install --log-level=info
```

- **`--log-level=info`** — Show informational messages. You can use `debug` for more verbose output if you need to troubleshoot.

**Important:** Do not close VS Code or the terminal tab while the installer is running. If the connection drops, the installer process will continue on the bastion host, but you will not see its output. If this happens, reconnect to the bastion and check the log file at `~/ocp-install/.openshift_install.log`.

### What Happens During Installation

The installation takes approximately 30-45 minutes. Here is what the installer does at each stage:

**Stage 1: Infrastructure Creation (~5-10 minutes)**

The installer calls the AWS API to create:
- A VPC with public and private subnets
- Internet gateway and NAT gateways
- Security groups for master, worker, and load balancer traffic
- IAM roles and instance profiles for the nodes
- Network Load Balancers for the API and application ingress (NLB, as configured in the manifest step)
- Route 53 DNS records pointing to the load balancers
- A bootstrap EC2 instance and the master EC2 instance

**Stage 2: Bootstrap (~15-20 minutes)**

The bootstrap machine starts first. Its job is to bring up a temporary control plane that the master node uses to configure itself. You will see messages like:

```
INFO Waiting up to 20m0s for the Kubernetes API at https://api.<cluster>.<domain>:6443...
INFO API v1.x.x up
INFO Waiting up to 45m0s for bootstrapping to complete...
```

Once the master is self-sufficient, the bootstrap machine is automatically destroyed. This is a key design principle — the bootstrap node is **temporary** and is not part of the final cluster.

**Stage 3: Cluster Initialization (~10-15 minutes)**

The master and worker nodes finish configuring themselves. Cluster Operators (the components that manage DNS, networking, storage, monitoring, the web console, etc.) start up one by one. You will see:

```
INFO Waiting up to 40m0s for the cluster to initialize...
```

**Stage 4: Completion**

When all Cluster Operators are healthy, the installer prints the credentials:

```
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run
INFO     export KUBECONFIG=/home/ec2-user/ocp-install/auth/kubeconfig
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.<cluster>.<domain>
INFO Login to the console with user: "kubeadmin", and password: "<generated-password>"
```

**Write down the kubeadmin password** — you will need it to log into the web console.

## Connect to Your Cluster

### Command Line Access

The installer generates a `kubeconfig` file containing the credentials needed to authenticate to the cluster API. Export it as an environment variable so that `oc` knows where to find it. In the VS Code integrated terminal, run:

```bash
export KUBECONFIG=$HOME/ocp-install/auth/kubeconfig
```

To make this permanent so it persists across terminal sessions:

```bash
echo 'export KUBECONFIG=$HOME/ocp-install/auth/kubeconfig' >> ~/.bashrc
source ~/.bashrc
```

Verify you can reach the cluster:

```bash
oc get nodes
```

You should see your master and worker nodes in `Ready` status:

```
NAME                                        STATUS   ROLES                  AGE   VERSION
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    control-plane,master   10m   v1.x.x
ip-10-0-xx-xx.us-west-1.compute.internal    Ready    worker                 5m    v1.x.x
```

Check that all Cluster Operators are healthy:

```bash
oc get clusteroperators
```

Every operator should show `AVAILABLE: True`, `PROGRESSING: False`, and `DEGRADED: False`. If any operator is still progressing, wait a few minutes and check again — some operators take a few minutes to fully stabilize after the installer reports completion.

Verify the cluster version:

```bash
oc get clusterversion
```

### Web Console Access

Open a browser and navigate to the console URL printed by the installer:

```
https://console-openshift-console.apps.<cluster>.<domain>
```

Log in with:
- **Username:** `kubeadmin`
- **Password:** The password printed at the end of the installation

The web console provides a graphical interface for managing your cluster. You can view workloads, monitor resource usage, manage users, and much more. For this course, we will primarily use the `oc` command line in VS Code's integrated terminal, but the web console is useful for visualization and quick exploration.

### Alternative: Token-Based Login

You can also obtain a login token from the web console, which is useful if you want to authenticate `oc` on a different machine:

1. In the web console, click your username (**kube:admin**) in the upper-right corner.
2. Select **Copy login command**.
3. Click **Display Token**.
4. Copy the `oc login` command and paste it into your VS Code terminal.

## Congratulations

You now have a running OpenShift cluster on AWS. You understand:

- **Why** the installer needs DNS, IAM, and a pull secret before it can start
- **What** each field in `install-config.yaml` controls and how the three network ranges are used
- **How** the installer progresses through infrastructure creation, bootstrap, and cluster initialization
- **How** to connect to the cluster using both the `oc` command line and the web console

For the remaining labs, keep your VS Code session connected to the bastion host. You will use the integrated terminal for `oc` commands and the editor for viewing and creating YAML manifests.

---

## Reference: Prerequisite Setup (Instructor Use)

The steps below document how the prerequisites for this lab were created. Students do not need to perform these steps — they are included for completeness and for instructors setting up new training environments.

### Register a DNS Domain in Route 53

OpenShift requires a DNS domain. Route 53 is the simplest option when running on AWS because the installer can automatically create DNS records in the hosted zone.

1. In the AWS Console, navigate to **Route 53 > Registered domains**.
2. Click **Register domains**, search for your desired domain, and purchase it.
3. Route 53 automatically creates a hosted zone for the new domain.
4. Verify the hosted zone exists under **Route 53 > Hosted zones**.

### Create an IAM User

The installer needs an IAM user with permissions to create VPCs, EC2 instances, load balancers, Route 53 records, IAM roles, and S3 buckets.

1. In the AWS Console, navigate to **IAM > Users > Create user**.
2. Create a group named `admin` with the `AdministratorAccess` policy.
3. Add the user to the `admin` group.
4. Under the user's **Security credentials** tab, create an access key for **CLI** use.
5. Save both the **Access Key ID** and **Secret Access Key** — the secret is only shown once.

### Increase AWS Service Quotas

When running many clusters simultaneously (e.g., 45 student clusters), several AWS default quotas are too low. Request increases **before** the class begins — some increases are auto-approved, others may take hours.

| Quota | Service Code | Default | Recommended | Quota Code |
|-------|-------------|---------|-------------|------------|
| Gateway VPC Endpoints per Region | vpc | 20 | 60 | L-1B52E74A |
| Network Load Balancers per Region | elasticloadbalancing | 100 | 150 | L-69A177A2 |

Each OpenShift cluster creates 1 Gateway VPC Endpoint and 3 Network Load Balancers (2 for the API + 1 for the ingress router when using NLB mode). For 45 clusters plus an instructor cluster, that requires 46 VPC endpoints and 138 NLBs.

Request increases using the AWS CLI:

```bash
aws service-quotas request-service-quota-increase --service-code vpc --quota-code L-1B52E74A --desired-value 60
aws service-quotas request-service-quota-increase --service-code elasticloadbalancing --quota-code L-69A177A2 --desired-value 150
```

The lab instructs students to configure NLB for the ingress router (instead of the default Classic Load Balancer). This is essential — the Classic LB default quota is only 20 per region, which would be exhausted before half the class finishes installing.

### Launch a Bastion Host

The bastion host is where the installer runs. It needs outbound internet access to reach the AWS API and Red Hat registries.

1. Navigate to **EC2 > Launch instance**.
2. Configure:
   - **AMI:** Amazon Linux 2023
   - **Instance type:** `c5.2xlarge` (or smaller — the installer is not resource-intensive)
   - **Key pair:** Create a new RSA key pair and download the `.pem` file
   - **Network:** Default VPC, auto-assign public IP enabled
   - **Security group:** Allow SSH (port 22) inbound
   - **Storage:** 50 GiB gp3 (the installer downloads several GB of data)
3. Launch the instance and note its public IP address.
