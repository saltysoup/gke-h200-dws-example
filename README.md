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
export ZONE="us-east4-"b
export PROJECT="gpu-launchpad-playground"
export GVNIC_NETWORK_PREFIX="a3u-gvnic"
export RDMA_NETWORK_PREFIX="a3u-rdma"
export CLUSTER_NAME="injae-h200-dws"
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
  --cluster-version=1.31.7-gke.1390000 \
  --enable-dataplane-v2 --enable-ip-alias --enable-multi-networking

```

4. Create a node pool with DWS flex-start provisioning and queued provisioning enabled (latter provides atomic, gang scheduling of nodes to avoid idle GPU waste) 

```
gcloud container node-pools create $NODE_POOL_NAME \
  --region COMPUTE_REGION --cluster $CLUSTER_NAME \
  --node-locations $COMPUTE_ZONE \
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
gcloud container clusters get-credentials $CLUSTER_NAME --location=$COMPUTE_REGION
```

