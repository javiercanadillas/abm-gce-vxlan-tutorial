# Creating an Anthos on Bare Metal hybrid cluster on Compute Engine VMs

This tutorial is based on the Google Cloud public document [Create an Anthos on bare metal hybrid cluster on Compute Engine VMs](https://cloud.google.com/anthos/clusters/docs/bare-metal/latest/try/gce-vms). The tutorial has been adapted to be used in the Qwiklabs environment and to avoid some outdated commands in the original document as well as to work with GCE machines configured with OS Login (as is the case with Qwiklabs).

To try Anthos clusters on bare metal on Compute Engine VMs, complete the following steps:

- [Prerequisites](prerequisites)
- [Create six VMs in Compute Engine](create-six-vms-in-compute-engine)
- [Create a vxlan network with L2 connectivity between VMs](create-a-vxlan-network-with-l2-connectivity-between-vms)
- [Install prerequisites for Anthos clusters on bare metal](install-prerequisites-for-anthos-clusters-on-bare-metal)
- [Deploy an Anthos on bare metal hybrid cluster](deploy-an-anthos-on-bare-metal-hybrid-cluster)
- [Verify your cluster](verify-your-cluster)
- [Log into your cluster from the Google Cloud Console](log-into-your-cluster-from-the-google-cloud-console)

## Prerequisites

Have a Qwiklabs project (or equivalent GCP project with Project Owner role) and a Qwiklabs user and password. With this information, open an incognito window and log in to the GCP console at [https://console.cloud.google.com](https://console.cloud.google.com) with your Qwiklabs credentials. Accept the terms and conditions and select the Qwiklabs project you want to use for this lab.

Finally, open the Cloud Shell by clicking on the icon on the top right corner of the GCP console and proceed with the next steps of this tutorial.

## Create six VMs in Compute Engine
Complete these steps to create the following VMs:

- One VM for the admin workstation. An admin workstation hosts command-line interface (CLI) tools and configuration files to provision clusters during installation, and CLI tools for interacting with provisioned clusters post-installation. The admin workstation will have access to all the other nodes in the cluster via SSH.
- Three VMs for the three control plane nodes needed to run the Anthos clusters on bare metal control plane.
- Two VMs for the two worker nodes needed to run workloads on the Anthos clusters on bare metal cluster.

Setup the Qwiklabs project ID as an environment variable:

```bash
export PROJECT_ID=<qwiklabs_project_id>
echo "export PROJECT_ID=$PROJECT_ID" >> ~/.bashrc
```

Setup the rest of the environment variables:

```bash
export ZONE=europe-west1-b
export CLUSTER_NAME=abm
export BMCTL_VERSION=1.15.1
cat <<EOF >> ~/.bashrc
export ZONE=europe-west1-b
export CLUSTER_NAME=abm
export BMCTL_VERSION=1.15.1
EOF
```

Run the following commands to log in with your Google account and set your project as the default:

```bash
gcloud config set project $PROJECT_ID
gcloud config set compute/zone $ZONE
cat <<EOF >> ~/.bashrc
gcloud config set project $PROJECT_ID
gcloud config set compute/zone $ZONE
EOF
```

When asked, click on "Authorize". Create the baremetal-gcr service account:

```bash
gcloud iam service-accounts create baremetal-gcr
gcloud iam service-accounts keys create bm-gcr.json \
    --iam-account=baremetal-gcr@"${PROJECT_ID}".iam.gserviceaccount.com
```

Enable the following APIs:

```bash
gcloud services enable \
    anthos.googleapis.com \
    anthosaudit.googleapis.com \
    anthosgke.googleapis.com \
    cloudresourcemanager.googleapis.com \
    connectgateway.googleapis.com \
    container.googleapis.com \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    serviceusage.googleapis.com \
    stackdriver.googleapis.com \
    monitoring.googleapis.com \
    logging.googleapis.com \
    opsconfigmonitoring.googleapis.com
```

Give the baremetal-gcr service account additional permissions to avoid needing multiple service accounts for different APIs and services:
```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:baremetal-gcr@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/gkehub.connect" \
  --no-user-output-enabled

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:baremetal-gcr@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/gkehub.admin" \
  --no-user-output-enabled

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:baremetal-gcr@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter" \
  --no-user-output-enabled

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:baremetal-gcr@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/monitoring.metricWriter" \
  --no-user-output-enabled

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:baremetal-gcr@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/monitoring.dashboardEditor" \
  --no-user-output-enabled

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:baremetal-gcr@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/stackdriver.resourceMetadata.writer" \
  --no-user-output-enabled

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:baremetal-gcr@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/opsconfigmonitoring.resourceMetadata.writer" \
  --no-user-output-enabled
  ```

Create the variables and arrays needed for all the commands on this page:

```bash
MACHINE_TYPE=n1-standard-4
VM_PREFIX=abm
VM_WS=$VM_PREFIX-ws
VM_CP1=$VM_PREFIX-cp1
VM_CP2=$VM_PREFIX-cp2
VM_CP3=$VM_PREFIX-cp3
VM_W1=$VM_PREFIX-w1
VM_W2=$VM_PREFIX-w2
declare -a VMs=("$VM_WS" "$VM_CP1" "$VM_CP2" "$VM_CP3" "$VM_W1" "$VM_W2")
declare -a IPs=()
export WM_WS
```

```bash
cat <<EOF >> ~/.bashrc
MACHINE_TYPE=n1-standard-4
VM_PREFIX=abm
VM_WS=$VM_PREFIX-ws
VM_CP1=$VM_PREFIX-cp1
VM_CP2=$VM_PREFIX-cp2
VM_CP3=$VM_PREFIX-cp3
VM_W1=$VM_PREFIX-w1
VM_W2=$VM_PREFIX-w2
declare -a VMs=("$VM_WS" "$VM_CP1" "$VM_CP2" "$VM_CP3" "$VM_W1" "$VM_W2")
EOF
```

Use the following loop to create six VMs:

```bash
for vm in "${VMs[@]}"
do
    gcloud compute instances create "$vm" \
      --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud \
      --zone="${ZONE}" \
      --boot-disk-size 200G \
      --boot-disk-type pd-standard \
      --can-ip-forward \
      --network default \
      --tags http-server,https-server \
      --min-cpu-platform "Intel Haswell" \
      --enable-nested-virtualization \
      --scopes cloud-platform \
      --machine-type "$MACHINE_TYPE" \
      --metadata "cluster_id=${CLUSTER_NAME},bmctl_version=${BMCTL_VERSION}"
    IP=$(gcloud compute instances describe "$vm" --zone "${ZONE}" \
         --format='get(networkInterfaces[0].networkIP)')
    IPs+=("$IP")
done
```

```bash
cat <<EOF >> ~/.bashrc
declare -a IPs=("${IPs[@]}")
EOF
```

This command creates VM instances with the following names:

- **abm-ws**: The VM for the admin workstation.
- **abm-cp1**, **abm-cp2**, **abm-cp3**: The VMs for the control plane nodes.
- **abm-w1**, **abm-w2**: The VMs for the nodes that run workloads.

Also note that you've been saving all the environment variables in the `~/.bashrc` file. This is to make sure that you can use the same environment if the Cloud Shell VM disconnects and you lose your shell session.

Use the following loop to verify that ssh is ready on all VMs (select `y` if asked and hit `Enter` when asked to provide a passphrase):

```bash
for vm in "${VMs[@]}"
do
    while ! gcloud compute ssh "$vm" --zone "${ZONE}" --command "printf 'SSH to $vm succeeded\n'"
    do
        printf "Trying to SSH into %s failed. Sleeping for 5 seconds. zzzZZzzZZ" "$vm"
        sleep  5
    done
done
```

## Create a vxlan network with L2 connectivity between VMs

Use the standard `vxlan` functionality of Linux to create a network that connects all the VMs with L2 connectivity.

The following commands contains two loops that perform the following actions:

- ssh into each VM.
- Update and install needed packages.
- Execute the required commands to configure the network with vxlan.

Fist, update and install the needed packages on all VMs:

```bash
for vm in "${VMs[@]}"
do
  gcloud compute ssh "$vm" --zone "${ZONE}" --command "bash -s" -- <<EOF
    echo 'debconf debconf/frontend select Noninteractive' | sudo debconf-set-selections
    sudo apt-get -qq update > /dev/null
    sudo apt-get -qq install -y jq > /dev/null
EOF
done
```

Then, configure the network with `vxlan``:

```bash
i=2 # We start from 10.200.0.2/24
for vm in "${VMs[@]}"
do
  gcloud compute ssh "$vm" --zone "${ZONE}" --command "bash -s" -- <<EOF
    set -x
    sudo ip link add vxlan0 type vxlan id 42 dev ens4 dstport 0
    current_ip=\$(ip --json a show dev ens4 | jq '.[0].addr_info[0].local' -r)
    for ip in ${IPs[@]}; do
      if [ "\$ip" != "\$current_ip" ]; then
        echo "Current IP is \$current_ip and different to \$ip"
        echo "Adding a bridge for \$ip to vxlan0"
        sudo bridge fdb append to 00:00:00:00:00:00 dst \$ip dev vxlan0
      fi
    done
    sudo ip addr add 10.200.0.$i/24 dev vxlan0
    sudo ip link set up dev vxlan0
EOF
  i=$((i+1))
done
```

You now have L2 connectivity within the 10.200.0.0/24 network. The VMs have the following IP addresses:

- Admin workstation VM: 10.200.0.2
- VMs running the control plane nodes:
  - 10.200.0.3
  - 10.200.0.4
  - 10.200.0.5
- VMs running the worker nodes:
  - 10.200.0.6
  - 10.200.0.7

## Install prerequisites for Anthos clusters on bare metal

You need to install the following tools on the admin workstation before installing Anthos clusters on bare metal:

- `bmctl`
- `kubectl`
- `docker`

To install the tools and prepare for Anthos clusters on bare metal installation run the following commands to download the service account key to the admin workstation and install the required tools:

```bash
gcloud compute ssh root@$VM_WS --zone "${ZONE}" --command "bash -s " -- <<EOF
  set -x

  export PROJECT_ID=$(gcloud config get-value project)
  BMCTL_VERSION=\$(curl http://metadata.google.internal/computeMetadata/v1/instance/attributes/bmctl_version -H   "Metadata-Flavor: Google")
  export BMCTL_VERSION

  gcloud iam service-accounts keys create bm-gcr.json --iam-account=baremetal-gcr@\$PROJECT_ID.iam.gserviceaccount.com
  cp bm-gcr.json /root/bm-gcr.json

  export STABLE_RELEASE=\$(curl -L -s https://dl.k8s.io/release/stable.txt)
  curl -LO "https://dl.k8s.io/release/\$STABLE_RELEASE/bin/linux/amd64/kubectl"

  echo "Installing kubectl and bmctl"
  sudo install kubectl /usr/local/sbin/
  mkdir -p baremetal && cd baremetal
  gsutil cp gs://anthos-baremetal-release/bmctl/\$BMCTL_VERSION/linux-amd64/bmctl .
  sudo install bmctl /usr/local/sbin/

  cd \$HOME
  echo "Installing Docker"
  curl -fsSL https://get.docker.com -o get-docker.sh
  sudo sh get-docker.sh
  sudo groupadd docker
  sudo usermod -aG docker \$USER
  newgrp docker
EOF
```

Configure the ssh connectivity between the Admin workstation and the different cluster VMs. First, create a ssh key on the admin workstation:

```bash
gcloud compute ssh $VM_WS --zone "${ZONE}" --command "bash -s " -- <<EOF
  set -x
  echo "y" | sudo ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
  sudo cp /root/.ssh/id_rsa.pub ~/ssh-root
EOF
```

Then, copy it to the VMs so that the `root` user from the admin workstation can connect to them:

```bash
# Copy ssh key to Cloud Shell and back to the vms
gcloud compute scp --zone $ZONE $VM_WS:~/ssh-root $HOME/ssh-root
for vm in ${VMs[@]}; do
  # Skip the admin workstation
  [[ $vm == $VM_WS ]] && continue
  # Copy the ssh key to each of the VMs but the admin workstation
  echo "Copying key into $vm"
  gcloud compute scp --zone $ZONE ssh-root $vm:~/ssh-root
  # Set the ssh key as authorized key for root@abm-ws
  echo "Setting key as authorized key for root@$vm"
  gcloud compute ssh $vm --zone $ZONE --command "bash -s " -- <<EOF
    set -x
    sudo cp -- ssh-root /root/.ssh/authorized_keys
EOF
done
```

## Deploy an Anthos on bare metal hybrid cluster

Finally, connect as root to the admin workstation, as you'll be performing the final cluster creating it with the `bmctl` tool you installed there before:

```bash
gcloud compute ssh $VM_WS --zone "${ZONE}"
```

Open a shell as root and launch the following commands to prepare the cluster configuration:

```bash
sudo su -
export PROJECT_ID=$(gcloud config get-value project)
CLUSTER_NAME=$(curl http://metadata.google.internal/computeMetadata/v1/instance/attributes/cluster_id -H "Metadata-Flavor: Google")
BMCTL_VERSION=$(curl http://metadata.google.internal/computeMetadata/v1/instance/attributes/bmctl_version -H "Metadata-Flavor: Google")
export CLUSTER_NAME
export BMCTL_VERSION
bmctl create config -c "$CLUSTER_NAME"
```

Still inside the root shell of the admin workstation, create a cluster configuration file:
```bash
cat > bmctl-workspace/$CLUSTER_NAME/$CLUSTER_NAME.yaml << EOF
---
gcrKeyPath: /root/bm-gcr.json
sshPrivateKeyPath: /root/.ssh/id_rsa
gkeConnectAgentServiceAccountKeyPath: /root/bm-gcr.json
gkeConnectRegisterServiceAccountKeyPath: /root/bm-gcr.json
cloudOperationsServiceAccountKeyPath: /root/bm-gcr.json
---
apiVersion: v1
kind: Namespace
metadata:
  name: cluster-$CLUSTER_NAME
---
apiVersion: baremetal.cluster.gke.io/v1
kind: Cluster
metadata:
  name: $CLUSTER_NAME
  namespace: cluster-$CLUSTER_NAME
spec:
  type: hybrid
  anthosBareMetalVersion: $BMCTL_VERSION
  gkeConnect:
    projectID: $PROJECT_ID
  controlPlane:
    nodePoolSpec:
      clusterName: $CLUSTER_NAME
      nodes:
      - address: 10.200.0.3
      - address: 10.200.0.4
      - address: 10.200.0.5
  clusterNetwork:
    pods:
      cidrBlocks:
      - 192.168.0.0/16
    services:
      cidrBlocks:
      - 172.26.232.0/24
  loadBalancer:
    mode: bundled
    ports:
      controlPlaneLBPort: 443
    vips:
      controlPlaneVIP: 10.200.0.49
      ingressVIP: 10.200.0.50
    addressPools:
    - name: pool1
      addresses:
      - 10.200.0.50-10.200.0.70
  clusterOperations:
    # might need to be this location
    location: us-central1
    projectID: $PROJECT_ID
  storage:
    lvpNodeMounts:
      path: /mnt/localpv-disk
      storageClassName: node-disk
    lvpShare:
      numPVUnderSharedPath: 5
      path: /mnt/localpv-share
      storageClassName: local-shared
  nodeConfig:
    podDensity:
      maxPodsPerNode: 250
---
apiVersion: baremetal.cluster.gke.io/v1
kind: NodePool
metadata:
  name: node-pool-1
  namespace: cluster-$CLUSTER_NAME
spec:
  clusterName: $CLUSTER_NAME
  nodes:
  - address: 10.200.0.6
  - address: 10.200.0.7
EOF
```

Finally, again inside the root shell of the admin workstation, create the cluster:

```bash
bmctl create cluster -c "$CLUSTER_NAME"
```

## Verify your cluster

You can find your cluster's kubeconfig file on the admin workstation in the bmctl-workspace directory of the root account. To verify your deployment, complete the following steps.

If not already there, ssh into the admin workstation:

```bash
gcloud compute ssh abm-ws --zone "$ZONE"
```

Once inside, open a root session:

```bash
sudo su -
```

You can ignore any messages about updating the VM and complete this tutorial. If you plan to keep the VMs as a test environment, you might want to update the OS or upgrade to the next release as described in the Ubuntu documentation.

Set the KUBECONFIG environment variable with the path to the cluster's configuration file to run kubectl commands on the cluster.

```bash
export clusterid=abm
export KUBECONFIG=$HOME/bmctl-workspace/$clusterid/$clusterid-kubeconfig
kubectl get nodes
```

Set the current context in an environment variable:

```bash
export CONTEXT="$(kubectl config current-context)"
```

Run the following gcloud command, substituting your Qwiklabs account ID for <your-qwiklabs-account-id>:

```bash
GOOGLE_ACCOUNT_EMAIL=<your-qwiklabs-account-id>
gcloud container fleet memberships generate-gateway-rbac \
    --membership=abm \
    --role=clusterrole/cluster-admin \
    --users=$GOOGLE_ACCOUNT_EMAIL \
    --project=qwiklabs-gcp-03-f3b499ceeca2 \
    --kubeconfig=$KUBECONFIG \
    --context=$CONTEXT \
    --apply
```

This command:

- Grants your user account the Kubernetes `clusterrole/cluster-admin`` role on the cluster.
- Configures the cluster so that you can run `kubectl` commands on your local computer without having to ssh to the admin workstation.

When you are finished exploring, enter exit to log out of the admin workstation.

From the Cloud Shell VM, get the `kubeconfig`` entry that can access the cluster through the Connect gateway:

```bash
gcloud container fleet memberships get-credentials $CLUSTER_NAME
```

You can now run kubectl commands through the Connect gateway:

```bash
kubectl get nodes
kubectl get namespaces
```

## Log into your cluster from the Google Cloud Console

To observe your workloads on Anthos clusters on bare metal in the Google Cloud console, you need to log in to the cluster. Before you log in to the console for the first time, you need to configure an authentication method. The easiest authentication method to configure is Google identity. This authentication method lets you log in using the email address associated with your Google Cloud account.

The `gcloud container fleet memberships generate-gateway-rbac` command that you ran in the previous section configures the cluster so that you can log in with your Google identity.

1. In the Google Cloud console, either:

  - In the **GKE Clusters** page, click on the three vertical dots **Actions** next to the registered cluster, then click **Login**.

  or:

  - In the **Anthos Clusters** page, select the cluster you want to log in to in the list of clusters, then click **Login** in the information panel that displays.

2. Select Use your Google identity to log in.

3. Click Login.