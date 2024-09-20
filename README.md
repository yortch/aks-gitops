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

## Deploy your Gitops (Flux) application

1. Export project name to use:

    ```bash
    NAMESPACE=opa-gitops
    ```

1. Run the following command to initialize the flux system in the AKS cluster:
    ```bash
    ```


 specify application git url and branch:

    ```
    GIT_URL=https://github.com/yortch/opa-demo
    BRANCH=gitops-aks
    az k8s-configuration flux create --resource-group $RG \
    --cluster-name $CLUSTER --cluster-type managedClusters \
    --name flux-config --scope cluster --namespace flux-system \
    --kind git --url=$GIT_URL \
    --branch $BRANCH
    ```

  flux create source git opa \
    --url=https://github.com/yortch/opa-demo \
    --branch=gitops-aks \
    --export



flux create helmrelease opa-gitops \
  --source GitRepository/opa-chart
  -- "https://github.com/yortch/opa-demo" \
  --chart ./iac/helm/opa-chart \
  --values ./iac/helm/opa-chart/values/dev/values.yaml \
  --namespace $NAMESPACE \
  --branch gitops-aks \
  --export