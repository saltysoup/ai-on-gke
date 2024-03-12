# Pre-Requisites
1. GKE Standard cluster running version 1.28.3-gke.1098000 or later
1. 2 x Nodepools created (reserved pool and DWS pool with --enable-queued-provisioning)

# Instructions

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

5. Modify download-llama2_7b.yaml to include your HF token with access to Llama2 model repo and run the job
```shell
kubectl apply -f llama2_7b/download-llama2_7b.yaml
```

6. Run the llama2 finetuning job on the reserved queue by applying the manifest
```shell
kubectl apply -f llama2_7b/finetune-g2.yaml
```

7. Modify download-llama2_13b.yaml to include your HF token with access to Llama2 model repo and run the job
```shell
kubectl apply -f llama2_13b/download-llama2_13b.yaml
```

1. Run the llama2 finetuning job on the DWS queue by applying the manifest
```shell
kubectl apply -f llama2_13b/finetune-a3.yaml
```
