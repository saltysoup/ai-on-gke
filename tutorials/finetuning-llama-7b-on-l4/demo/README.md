As part of this tutorial, you will get to do the following:

*   Create a GKE cluster with an autoscaling L4 GPU nodepool
*   Run a Kubernetes Job to download Llama 2 7B and fine-tune using L4 GPUs
*   Run a Kubernetes Job to fine-tune Llama 2 7B using H100 GPUs obtained through Dynamic Workload Scheduler


# Pre-Requisites

1. GKE Standard cluster running version 1.28.3-gke.1098000 or later
2. 2 x Nodepools created (reserved pool and DWS pool with --enable-queued-provisioning)
3. L4 GPUs quota to be able to run additional 8 L4 GPUs
4. H100 GPUs quota to be able to run additional 8 H100 GPUs

# Instructions

## Set environment variables

Let’s start by setting a few environment variables that will be used throughout this post. You should modify these variables to meet your environment and needs. 

Download the code and files used throughout the tutorial:
```bash
git clone https://github.com/saltysoup/ai-on-gke.git --branch demo

cd ai-on-gke/tutorials/finetuning-llama-7b-on-l4/demo
```

Run the following commands to set the env variables and make sure to replace `<my-project-id>`:

```bash
gcloud config set project <my-project-id>
export PROJECT_ID=$(gcloud config get project)
export REGION=us-central1
export ZONE=us-central1-a
export BUCKET_NAME=${PROJECT_ID}-kueue-llama
export SERVICE_ACCOUNT="dws-demo@${PROJECT_ID}.iam.gserviceaccount.com"
export PREFIX="tcpx"
export GKE_VERSION=1.28.3-gke.1203001
export CLUSTER_NAME="gke-ml"
```

> Note: You might have to rerun the export commands if for some reason you reset your shell and the variables are no longer set. This can happen for example when your Cloud Shell disconnects.

## Create Multiple VPCs for A3
A3 VMs use multiple VPCs (one for each NIC) for high inter-node bandwidth throughput. Run the following to create VPCs.

```bash
for N in $(seq 1 4); do
  gcloud compute --project=${PROJECT_ID} \
    networks create \
    tcpx-net-$N \
    --subnet-mode=custom \
    --mtu=8244

  gcloud compute --project=${PROJECT_ID} \
    networks subnets create \
    tcpx-sub-$N \
    --network=tcpx-net-$N \
    --region=${REGION} \
    --range=192.168.$N.0/24

  gcloud compute --project=${PROJECT_ID} \
    firewall-rules create \
    tcpx-internal-$N \
    --network=tcpx-net-$N \
    --action=ALLOW \
    --rules=tcp:0-65535,udp:0-65535,icmp \
    --source-ranges=192.168.0.0/16
Done

```

## Creating the GKE cluster with GPU nodepools

Create the GKE cluster by running:
```bash
gcloud beta container clusters create ${CLUSTER_NAME} \
  --project ${PROJECT_ID} \
  --cluster-version ${GKE_VERSION} \
  --zone ${ZONE} \
  --enable-dataplane-v2 \
  --enable-multi-networking \
  --enable-ip-alias \
  --workload-pool=${PROJECT_ID}.svc.id.goog \
  --no-enable-autoupgrade \
  --addons GcsFuseCsiDriver
```

Authenticate to your new GKE cluster with the following
```bash
gcloud container clusters get-credentials ${CLUSTER_NAME} \
    --zone ${ZONE}
```

Configure cluster networks in NetDevice mode for the A3 VMs to use multiple VPCs
```bash
kubectl apply -f - <<EOF
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: vpc1
spec:
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: vpc1
  type: Device
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: vpc2
spec:
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: vpc2
  type: Device
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: vpc3
spec:
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: vpc3
  type: Device
---
apiVersion: networking.gke.io/v1
kind: Network
metadata:
  name: vpc4
spec:
  parametersRef:
    group: networking.gke.io
    kind: GKENetworkParamSet
    name: vpc4
  type: Device
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: vpc1
spec:
  vpc: tcpx-net-1
  vpcSubnet: tcpx-sub-1
  deviceMode: NetDevice
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: vpc2
spec:
  vpc: tcpx-net-2
  vpcSubnet: tcpx-sub-2
  deviceMode: NetDevice
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: vpc3
spec:
  vpc: tcpx-net-3
  vpcSubnet: tcpx-sub-3
  deviceMode: NetDevice
---
apiVersion: networking.gke.io/v1
kind: GKENetworkParamSet
metadata:
  name: vpc4
spec:
  vpc: tcpx-net-4
  vpcSubnet: tcpx-sub-4
  deviceMode: NetDevice
EOF
```

Let’s create a nodepool for our finetuning which will use 8 L4 GPUs per VM.
Create the `g2-standard-96` nodepool by running:

```bash
gcloud beta container node-pools create g2-standard-96 \
  --cluster ${CLUSTER_NAME} \
  --zone ${ZONE} \
  --project ${PROJECT_ID} \
  --accelerator "type=nvidia-l4,count=8,gpu-driver-version=latest" \
  --machine-type "g2-standard-96" \
  --ephemeral-storage-local-ssd count=8 \
  --location-policy=ANY \
  --reservation-affinity=none \
  --enable-autoscaling \
  --enable-image-streaming \
  --num-nodes=0 \
  --total-max-nodes 2 \
  --enable-gvnic \
  --scopes "https://www.googleapis.com/auth/cloud-platform" \
  --node-locations ${ZONE}
```

Let’s create a nodepool for our finetuning which will use 8 H100 GPUs per VM via Dynamic Workload Scheduler.
Create the `dws-nodepool` nodepool by running:

```bash
gcloud beta container node-pools create dws-nodepool \
  --cluster ${CLUSTER_NAME} \
  --zone ${ZONE} \
  --project ${PROJECT_ID} \
  --enable-autoupgrade \
  --enable-autorepair \
  --accelerator "type=nvidia-h100-80gb,count=8,gpu-driver-version=latest" \
  --machine-type "a3-highgpu-8g" \
  --ephemeral-storage-local-ssd "count=16" \
  --enable-queued-provisioning \
  --location-policy=ANY \
  --reservation-affinity=none \
  --enable-autoscaling \
  --num-nodes=0 \
  --total-max-nodes 1  \
  --enable-gvnic \
  --scopes "https://www.googleapis.com/auth/cloud-platform" \
  --additional-node-network network=${PREFIX}-net-1,subnetwork=${PREFIX}-sub-1 \
  --additional-node-network network=${PREFIX}-net-2,subnetwork=${PREFIX}-sub-2 \
  --additional-node-network network=${PREFIX}-net-3,subnetwork=${PREFIX}-sub-3 \
  --additional-node-network network=${PREFIX}-net-4,subnetwork=${PREFIX}-sub-4 
```

> Note the --enable-queued-provisioning parameter which enables DWS for the nodepool

> Note: The `--node-locations` flag might have to be adjusted based on which region you choose. Please check which zones the [GPUs are available](https://cloud.google.com/compute/docs/gpus/gpu-regions-zones) if you change the region to something other than `us-central1`.

The nodepool has been created and is scaled down to 0 nodes. So you are not paying for any GPUs until you start launching Kubernetes Pods that request GPUs.


## Configuring GCS and required permissions

Create a GCS bucket to store our models:
```bash
gcloud storage buckets create gs://${BUCKET_NAME}
```

The model loading Job will write to GCS. So let’s create a Google Service Account that has read and write permissions to the GCS bucket. Then create a Kubernetes Service Account named `l4-demo` that is able to use the Google Service Account.

To do this, first create a new Google Service Account:
```bash
gcloud iam service-accounts create ${SERVICE_ACCOUNT}
```

Assign the required GCS permissions to the Google Service Account:
```bash
gcloud storage buckets add-iam-policy-binding gs://${BUCKET_NAME} \
  --member="serviceAccount:${SERVICE_ACCOUNT}" --role=roles/storage.admin
```

Allow the Kubernetes Service Account in the `default` namespace to use the Google Service Account:
```bash 
gcloud iam service-accounts add-iam-policy-binding ${SERVICE_ACCOUNT} \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${PROJECT_ID}.svc.id.goog[default/${SERVICE_ACCOUNT}]"
```

Create a new Kubernetes Service Account:
```bash
kubectl create serviceaccount workload-identity-k8-sa
kubectl annotate serviceaccount workload-identity-k8-sa iam.gke.io/gcp-service-account=l4-demo@${PROJECT_ID}.iam.gserviceaccount.com
```

Hugging face requires authentication to download the[ Llama 2 7B HF](https://huggingface.co/meta-llama/Llama-2-7b-hf) model, which means an access token is required to download the model.

You can get your access token from [huggingface.com > Settings > Access Tokens](https://huggingface.co/settings/tokens). Make sure to copy it and then use it in the next step when you create the Kubernetes Secret.

Create a Secret to store your HuggingFace token which will be used by the Kubernetes job:
```bash
kubectl create secret generic llama2-hf-token \
  --from-literal="HF_TOKEN=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"
```


## Install Kueue on your GKE Cluster

1. Install Kueue in your cluster with necessary configuration to enable Provisioning Request integration:
```shell
kubectl apply --server-side -f kueue/kueue-manifests.yaml
```

2. Verify update has applied to Kueue and is now ready to use:
```shell
kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n kueue-system
```

Successful output:
```shell
pod/kueue-controller-manager-778b9f45db-2jbvn condition met
```

3. Create a LocalQueue with a backing ClusterQueue to use DWS. The ClusterQueue uses a AdmissionCheck hook, which utilises the provisioning-request for obtaining and provisioning the desired GPUs. This will result in resizing of the target nodepool to run jobs.
```shell
kubectl apply -f kueue/dws-queue.yaml
```

4. Create a second LocalQueue with a backing ClusterQueue. This uses the reserved nodepool without using DWS or admission check:
```shell
kubectl apply -f kueue/l4-queue.yaml
```

## Download Llama2 model from HuggingFace

Let's use Kubernetes Job to download the Llama 2 7B model from HuggingFace.
The file `download-llama2_7b.yaml` in folder llama2_7b in this repo shows how to do this:

[embedmd]:# (llama2_7b/download-llama2_7b.yaml)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: model-loader-llama2-7b
  namespace: default
  labels:
    kueue.x-k8s.io/queue-name: l4-local-queue
spec:
  backoffLimit: 2
  suspend: true
  template:
    metadata:
      annotations:
        kubectl.kubernetes.io/default-container: loader
        gke-gcsfuse/volumes: "true"
        gke-gcsfuse/memory-limit: 400Mi
        gke-gcsfuse/ephemeral-storage-limit: 30Gi
    spec:
      restartPolicy: OnFailure
      containers:
      - name: loader
        image: python:3.11
        command:
        - /bin/bash
        - -c
        - |
          pip install huggingface_hub
          mkdir -p /gcs-mount/llama2-7b
          python3 - << EOF
          from huggingface_hub import snapshot_download
          model_id="meta-llama/Llama-2-7b-chat-hf"
          snapshot_download(repo_id=model_id, local_dir="/gcs-mount/llama2-7b",
                            local_dir_use_symlinks=False, revision="main",
                            ignore_patterns=["*.safetensors", "model.safetensors.index.json"])
          EOF
        imagePullPolicy: IfNotPresent
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: llama2-hf-token
              key: HF_TOKEN
        volumeMounts:
        - name: gcs-fuse-csi-ephemeral
          mountPath: /gcs-mount
      serviceAccountName: workload-identity-k8-sa
      volumes:
      - name: gcs-fuse-csi-ephemeral
        csi:
          driver: gcsfuse.csi.storage.gke.io
          volumeAttributes:
            bucketName: ${BUCKET_NAME}
            mountOptions: "implicit-dirs"
```

Run the Kubernetes Job to download the the Llama 2 7B model to the bucket created previously:
```bash
envsubst < llama2_7b/download-llama2_7b.yaml | kubectl apply -f -
```
> Note: `envsubst` is used to replace `${BUCKET_NAME}` inside `download-model.yaml` with your own bucket.

Give it a minute to start running, once up you can watch the logs of the job by running:
```bash
kubectl logs -f -l job-name=model-loader-llama2-7b
```

Once the job has finished you can verify the model has been downloaded by running:
```bash
gcloud storage ls -l gs://$BUCKET_NAME/llama2-7b/
```

### Fine-tuning Llama2

1. Run the llama2 finetuning job on the reserved queue by applying the manifest
```shell
kubectl apply -f llama2_7b/finetune-g2.yaml
```

2. Modify download-llama2_13b.yaml to include your HF token with access to Llama2 model repo and run the job
```shell
kubectl apply -f llama2_13b/download-llama2_13b.yaml
```

3. Run the llama2 finetuning job on the DWS queue by applying the manifest
```shell
kubectl apply -f llama2_13b/finetune-a3.yaml
```
