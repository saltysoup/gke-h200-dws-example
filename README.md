# Instructions

## Overview

To enable efficient distributed training on a3 ultragpu VMs (H200 GPUs) on GCP, the recommended method is to use GPUDirect RDMA using RoCE to enable 3200Gbps of GPU to GPU traffic. To enable capacity assurance of in-demand GPUs such as H200, we'll be using DWS Flex-Start provisioning, which provides a greater access to capacity at 50% reduced price vs on-demand. To see the difference between spot vs DWS Flex-Start vs Reservation provisioned GPUs, [refer to this documentation](https://cloud.google.com/ai-hypercomputer/docs/consumption-models)

## Pre-requisites

1. Create 1 x RDMA VPC and 1 x "Standard" VPC (for north south traffic on VM's primary NIC)
2. Create a GKE cluster with multi-networking enabled
3. Create GKE network objects on the cluster
4. Install RDMA binary and configure NCCL on the node
5. (Optional but recommended) Install Kueue to automate request and provisioning of jobs with DWS Flex Start
6. IAM credentials to create above cloud resources

## Instructions

1. Set env var for our cloud resources - ref https://cloud.google.com/ai-hypercomputer/docs/create/gke-ai-hypercompute-custom#create-vpcs-and-subnets

```
export REGION="us-east4"
export ZONE="us-east4-b"
export PROJECT="gpu-launchpad-playground"
export GVNIC_NETWORK_PREFIX="a3u-gvnic"
export RDMA_NETWORK_PREFIX="a3u-rdma"
export GKE_VERSION="1.32.2-gke.1652000"
export CLUSTER_NAME="injae-h200-dws1"
export NODEPOOL_NAME="h200-dws"
export GPU_TYPE="nvidia-h200-141gb"
export AMOUNT=8 # Number of GPUs to attach per VM
export MACHINE_TYPE="a3-ultragpu-8g"
export NUM_NODES=0 # Must be set to 0 for flex-start to initialise 0 sized nodepool
export TOTAL_MAX_NODES=4 # Max number of nodes that can scale up in nodepool for flex-start. Could be upto 1000 VMs (8k GPUs)
export DRIVER_VERSION="latest"

```

2. Create RDMA VPC and a Standard VPC (non RDMA)

```
# Create a VPC for the additional Google Titanium CPU NIC
gcloud compute --project=${PROJECT?} \
  networks create \
  ${GVNIC_NETWORK_PREFIX?}-net \
  --subnet-mode=custom

gcloud compute --project=${PROJECT?} \
  networks subnets create \
  ${GVNIC_NETWORK_PREFIX?}-sub \
  --network=${GVNIC_NETWORK_PREFIX?}-net \
  --region=${REGION?} \
  --range=192.168.0.0/24

gcloud compute --project=${PROJECT?} \
  firewall-rules create \
  ${GVNIC_NETWORK_PREFIX?}-internal \
  --network=${GVNIC_NETWORK_PREFIX?}-net \
  --action=ALLOW \
  --rules=tcp:0-65535,udp:0-65535,icmp \
  --source-ranges=192.168.0.0/16

# Create HPC VPC for the RDMA NICs with 8 subnets.
gcloud beta compute --project=${PROJECT?} \
  networks create ${RDMA_NETWORK_PREFIX?}-net \
  --network-profile=${ZONE?}-vpc-roce \
  --subnet-mode=custom

# Create subnets for the HPC VPC.
for N in $(seq 0 7); do
  gcloud compute --project=${PROJECT?} \
    networks subnets create \
    ${RDMA_NETWORK_PREFIX?}-sub-$N \
    --network=${RDMA_NETWORK_PREFIX?}-net \
    --region=${REGION?} \
    --range=192.168.$((N+1)).0/24 &  # offset to avoid overlap with gvnics
done
```

3. Create a GKE Standard cluster with multi networking enabled - ref https://cloud.google.com/ai-hypercomputer/docs/create/gke-ai-hypercompute-custom#create-cluster-and-node-pool-rdma-multi-net

```
# Check minimum GKE version for accelerators here - https://cloud.google.com/ai-hypercomputer/docs/create/gke-ai-hypercompute-custom#requirements
gcloud container clusters create $CLUSTER_NAME \
  --region=us-east4 \
  --cluster-version=$GKE_VERSION \
  --enable-dataplane-v2 --enable-ip-alias --enable-multi-networking

```

4. Create a node pool with DWS flex-start provisioning and queued provisioning enabled (latter provides atomic, gang scheduling of nodes to avoid idle GPU waste) 

```
gcloud container node-pools create $NODEPOOL_NAME \
  --region $REGION \
  --cluster $CLUSTER_NAME \
  --node-locations $ZONE \
  --accelerator type=$GPU_TYPE,count=$AMOUNT,gpu-driver-version=$DRIVER_VERSION \
  --machine-type $MACHINE_TYPE \
  --num-nodes=$NUM_NODES \
  --flex-start --num-nodes=0 --enable-autoscaling \
  --total-max-nodes $TOTAL_MAX_NODES \
  --no-enable-autorepair --location-policy=ANY \
  --reservation-affinity=none \
  --enable-queued-provisioning \
  --additional-node-network network=${GVNIC_NETWORK_PREFIX}-net,subnetwork=${GVNIC_NETWORK_PREFIX}-sub \
  --additional-node-network network=${RDMA_NETWORK_PREFIX}-net,subnetwork=${RDMA_NETWORK_PREFIX}-sub-0 \
  --additional-node-network network=${RDMA_NETWORK_PREFIX}-net,subnetwork=${RDMA_NETWORK_PREFIX}-sub-1 \
  --additional-node-network network=${RDMA_NETWORK_PREFIX}-net,subnetwork=${RDMA_NETWORK_PREFIX}-sub-2 \
  --additional-node-network network=${RDMA_NETWORK_PREFIX}-net,subnetwork=${RDMA_NETWORK_PREFIX}-sub-3 \
  --additional-node-network network=${RDMA_NETWORK_PREFIX}-net,subnetwork=${RDMA_NETWORK_PREFIX}-sub-4 \
  --additional-node-network network=${RDMA_NETWORK_PREFIX}-net,subnetwork=${RDMA_NETWORK_PREFIX}-sub-5 \
  --additional-node-network network=${RDMA_NETWORK_PREFIX}-net,subnetwork=${RDMA_NETWORK_PREFIX}-sub-6 \
  --additional-node-network network=${RDMA_NETWORK_PREFIX}-net,subnetwork=${RDMA_NETWORK_PREFIX}-sub-7
```

5. Once provisioned, connect to your cluster to run `kubectl` commands to configure the GKE cluster
```
gcloud container clusters get-credentials $CLUSTER_NAME --location=$REGION
```

6. Create the GKE network objects, which will allow k8 jobs to use the RDMA NICs

```
kubectl apply -f - <<EOF
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: gvnic-1
spec:
  vpc: ${GVNIC_NETWORK_PREFIX}-net
  vpcSubnet: ${GVNIC_NETWORK_PREFIX}-sub
  deviceMode: NetDevice
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: gvnic-1
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: gvnic-1
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: rdma-0
spec:
  vpc: ${RDMA_NETWORK_PREFIX}-net
  vpcSubnet: ${RDMA_NETWORK_PREFIX}-sub-0
  deviceMode: RDMA
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: rdma-0
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: rdma-0
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: rdma-1
spec:
  vpc: ${RDMA_NETWORK_PREFIX}-net
  vpcSubnet: ${RDMA_NETWORK_PREFIX}-sub-1
  deviceMode: RDMA
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: rdma-1
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: rdma-1
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: rdma-2
spec:
  vpc: ${RDMA_NETWORK_PREFIX}-net
  vpcSubnet: ${RDMA_NETWORK_PREFIX}-sub-2
  deviceMode: RDMA
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: rdma-2
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: rdma-2
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: rdma-3
spec:
  vpc: ${RDMA_NETWORK_PREFIX}-net
  vpcSubnet: ${RDMA_NETWORK_PREFIX}-sub-3
  deviceMode: RDMA
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: rdma-3
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: rdma-3
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: rdma-4
spec:
  vpc: ${RDMA_NETWORK_PREFIX}-net
  vpcSubnet: ${RDMA_NETWORK_PREFIX}-sub-4
  deviceMode: RDMA
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: rdma-4
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: rdma-4
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: rdma-5
spec:
  vpc: ${RDMA_NETWORK_PREFIX}-net
  vpcSubnet: ${RDMA_NETWORK_PREFIX}-sub-5
  deviceMode: RDMA
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: rdma-5
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: rdma-5
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: rdma-6
spec:
  vpc: ${RDMA_NETWORK_PREFIX}-net
  vpcSubnet: ${RDMA_NETWORK_PREFIX}-sub-6
  deviceMode: RDMA
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: rdma-6
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: rdma-6
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: rdma-7
spec:
  vpc: ${RDMA_NETWORK_PREFIX}-net
  vpcSubnet: ${RDMA_NETWORK_PREFIX}-sub-7
  deviceMode: RDMA
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: rdma-7
spec:
  type: "Device"
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: rdma-7
EOF
```

7. Install the RDMA binary and configure NCCL

```
# For GKE standard cluster
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/refs/heads/master/gpudirect-rdma/nccl-rdma-installer.yaml
```

8. Install Kueue to automate scheduling of jobs with DWS flex start provisioned VMs. Although it is possible to use queued provisioning without Kueue, it requires using your own scheduling tool to create and manage the Provisioning Request (which sends DWS API for node creation). Ref https://cloud.google.com/kubernetes-engine/docs/how-to/provisioningrequest#prepare_your_environment

```
VERSION=v0.11.4
kubectl apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/download/$VERSION/manifests.yaml
```

9. Create Kueue resources including a cluster-level queue called `dws-cluster-queue` and a localqueue namespace `dws-local-queue`. Jobs that refer to `dws-cluster-queue` will use flex-start to get the GPU resources. 

```
kubectl apply -f kueue-dws-queues.yaml
```

You should see an output similar to following
```
resourceflavor.kueue.x-k8s.io/default-flavor created
admissioncheck.kueue.x-k8s.io/dws-prov created
provisioningrequestconfig.kueue.x-k8s.io/dws-config created
clusterqueue.kueue.x-k8s.io/dws-cluster-queue created
localqueue.kueue.x-k8s.io/dws-local-queue created
```

10. Create a local dummy service to allow pods to share DNS domain for running NCCL test (alternatively can use hostNetwork or enableDNSHostnames) 

```
kubectl apply -f nccl-service.yaml
```

11. Deploy an example NCCL test to verify that the RDMA network is being used between the GPU VMs

```
kubectl apply -f nccl-dws-example.yaml
```

once you submit the job, you'll notice that the job is in a suspended state. This will automatically unsuspend and begin once the nodes are provisioned and ready for the workload pods

```
Events:
  Type    Reason                 Age    From                        Message
  ----    ------                 ----   ----                        -------
  Normal  Suspended              6m35s  job-controller              Job suspended
  Normal  CreatedWorkload        6m35s  batch/job-kueue-controller  Created Workload: default/job-nccl-dws-job-1dc1f
  Normal  UpdatedAdmissionCheck  6m9s   batch/job-kueue-controller  dws-prov: Waiting for resources. Currently there are not enough resources available to fulfill the request.
  Normal  Started                3m14s  batch/job-kueue-controller  Admitted by clusterQueue dws-cluster-queue
  Normal  SuccessfulCreate       3m14s  job-controller              Created pod: nccl-dws-job-vlz6t
  Normal  SuccessfulCreate       3m14s  job-controller              Created pod: nccl-dws-job-9v2k7
  Normal  Resumed                3m14s  job-controller              Job resumed
```

```
# You can optionally check provision request to query DWS has accepted/rejected the flex start request
kubectl get provreq
NAME                                ACCEPTED   PROVISIONED   FAILED   AGE
job-nccl-dws-job-1dc1f-dws-prov-1   True       True                   7m18s
```

After node scales up, the workload pod will run

```
NAME                 READY   STATUS              RESTARTS   AGE
nccl-dws-job-9v2k7   0/1     ContainerCreating   0          6m41s
nccl-dws-job-vlz6t   0/1     ContainerCreating   0          6m41s
```


