# AKS Gitops (with Flux v2) setup guide

This repo contains instructions on how to setup GitOps with Flux v2 for AKS

## Prerequisites
* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/)
* [Helm](https://helm.sh/docs/intro/install/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [flux](https://fluxcd.io/flux/installation/)


## Create AKS cluster

1. Create a resource group using the `az group create` command.

    ```bash
    RG=aks-gitops
    REGION=eastus2
    az login
    az group create --name $RG --location $REGION
    ```

1. Create AKS cluster:
    ```bash
    CLUSTER=aks-gitops-cluster
    az aks create --resource-group $RG --name $CLUSTER
    ```
    
1. Log into AKS cluster:
    ```bash
    az aks get-credentials --resource-group $RG --name $CLUSTER --overwrite-existing
    ```

## Configure Azure Subscription and CLI to use Flux 

Register the following Azure resource providers if not already regitered:

```bash
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.KubernetesConfiguration
```

Enable CLI extensions:

```bash
az extension add -n k8s-configuration
az extension add -n k8s-extension
```

Install flux extension: 

```bash
az k8s-extension create -g $RG -c $CLUSTER -n flux-system \
--extension-type microsoft.flux -t managedClusters
```

## Deploy your Gitops (Flux) application

1. Export variablesto use:

    ```bash
    GIT_URL=https://github.com/yortch/aks-gitops
    BRANCH=main
    ```

1. Run the following command to initialize the flux system and configuration in the AKS cluster:

    ```
    az k8s-configuration flux create --resource-group $RG \
    --cluster-name $CLUSTER --cluster-type managedClusters \
    --name opa-config --scope namespace --namespace opa \
    --kind git --url=$GIT_URL --interval=1m --timeout=2m \
    --branch $BRANCH --kustomization name=opa-kustomize \
    path=./apps/opa/base interval=1m timeout=2m prune=true force=true
    ```

1. The `kustomization` above creates a `HelmRelease` using helm chart from `GitRepository` (https://github.com/yortch/opa-demo) which creates artifacts in the namespace `opa`. To validate application, run the following command to get the external IP:

   ```bash
   kubectl get service opa -n opa
   ```

1. Copy the `EXTERNAL_IP` value and navigate to `http://{EXTERNAL_IP}:8181` in a browser.

### Add additional environment deployments

Use the following command to add a dev flux config:

    ```bash
    az k8s-configuration flux create --resource-group $RG \
    --cluster-name $CLUSTER --cluster-type managedClusters \
    --name opa-dev-config --scope namespace --namespace opa-dev \
    --kind git --url=$GIT_URL --interval=1m --timeout=2m \
    --branch $BRANCH --kustomization name=opa-dev-kustomize \
    path=./apps/opa/dev interval=1m timeout=2m prune=true force=true retry-interval=1m
    ```

### Add second deployment on the same namespace

This shows how to add a new deployment using a separate helm chart on the same namespace
by adding a new `kustomization` resource:

    ```bash
    az k8s-configuration flux kustomization create --resource-group $RG \
    --cluster-name $CLUSTER --cluster-type managedClusters --name opa-config \
    --kustomization-name app2-kustomize --path ./apps/app2 \
    --prune true --force true --interval 1m --timeout 2m --retry-interval 30s
    ```