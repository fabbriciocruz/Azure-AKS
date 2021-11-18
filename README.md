# Azure AKS Step by Step

![image](https://user-images.githubusercontent.com/80425460/141885752-2b967f64-a50e-4e7b-a80b-2025a6b0fddb.png)

# Goals
## 1. Deploy AKS Cluster Using Azure CLI (az aks create)
## 2. Configure the Ingress Controller using the Application Gateway
## 3. Deploy an application example which will be exposed via https

# Prerequisites
# 1. Three /24 private subnets
# 2. One /27 subnet for the Application Gateway

# Create a Resource Group

# Use the Azure CLI

## Enable kubectl bash_completion
!
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
!

# Create a Resource Group
```
az group create --name <ResourceGroupName> --location <AzureLocation>
```

# Create a variable for the resource group name:
```
RG=<RESOURCE_GROUP_NAME>
```

# AKS and Kubernetes Version Matrix
## Supported Kubernetes versions in Azure Kubernetes Service (AKS)
### https://docs.microsoft.com/en-us/azure/aks/supported-kubernetes-versions?tabs=azure-cli

## Supported kubectl versions
### You can use one minor version older or newer of kubectl (Client Version) relative to your kube-apiserver version (Server Version)
### For example, if your kube-apiserver is at 1.17, then you can use versions 1.16 to 1.18 of kubectl with that kube-apiserver.

# 1. Get the current kubectl version
kubectl version --short

# 2. To find out what Kubernetes versions (Server Version) are currently available for your subscription and region, use the az aks get-versions command. 
```
az aks get-versions \
--location <AZURE LOCATION> \
--subscription <AZURE SUBSCRIPTION> # Optional
```

# Create a Cluster
```
az aks create \
 --resource-group $RG \
 --name Cluster01 \
 #--enable-private-cluster \
 #--disable-public-fqdn \
 --kubernetes-version <value from: `az aks get-versions`> \
 --node-count 3 \
 --generate-ssh-keys \
 --node-vm-size Standard_B2s \
 --enable-managed-identity
```

## 1. Configure kubectl to connect to your Kubernetes cluster using the az aks get-credentials command
```
az aks get-credentials --name Cluster01 --resource-group $RG
```

When you create an AKS cluster, a second resource group is automatically created to store the AKS resources needed to support your cluster, things like load balancers, public IPs, VMSS backing the node pools will be created here.

To get the name of the resource group, use the command below:

```
az aks show --name myAKSCluster \
    --resource-group myAKSResourceGroup \
    --query "nodeResourceGroup"
```

# Test the cluster getting some information 

## 1. List managed Kubernetes clusters.
az aks list

## 2. Get kubectl client and server versions
kubectl version --short

## 3. Get node information
kubectl get nodes -o wide

## 4. Get cluster information
kubectl cluster-info

## 5. Show the details for a managed Kubernetes cluster
az aks show -g $RG -n Cluster01


# Ingress Controller + Application Gateway
## Pay attention that AGIC only supports v2 SKUs.
## AKS — different load balancing options. When to use what?
### https://medium.com/microsoftazure/aks-different-load-balancing-options-for-a-single-cluster-when-to-use-what-abd2c22c2825


# Appgw ssl certificate
## https://azure.github.io/application-gateway-kubernetes-ingress/features/appgw-ssl-certificate/


# Account Management on Azure AKS with AAD and AKS RBAC
## https://ystatit.medium.com/account-management-on-azure-aks-with-aad-and-aks-rbac-fc178f90475b


# Useful commands

1. To delete a cluster
    ```
    az aks delete --name MyManagedCluster --resource-group MyResourceGroup
    ```

2. Stop an AKS Cluster
    ```
    az aks stop --name myAKSCluster --resource-group myResourceGroup
    ```
3. Start an AKS Cluster


#-----------DRAFT

az aks get-versions \
--location brazilsouth

az aks delete --name Cluster01 --resource-group $RG

az aks create \
 --resource-group $RG \
 --name Cluster01 \
 --kubernetes-version 1.21.2 \
 --node-count 3 \
 --generate-ssh-keys \
 --node-vm-size Standard_B2s \
 --enable-managed-identity


az aks enable-addons --addons ingress-appgw \
    --resource-group $RG \
    --name Cluster01 \
    --appgw-name AksCluster01 \
    --appgw-subnet-cidr 10.1.0.0/16

    [--appgw-subnet-id]

az aks get-credentials --name Cluster01 --resource-group $RG


az aks show --name Cluster01 \
--resource-group $RG \
--query "nodeResourceGroup"
