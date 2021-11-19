# Azure AKS Step by Step

![image](https://user-images.githubusercontent.com/80425460/141885752-2b967f64-a50e-4e7b-a80b-2025a6b0fddb.png)

# Goals
1. Deploy AKS Cluster Using Azure CLI (az aks create)
2. Configure the Ingress Controller using the Application Gateway
3. Deploy an application example which will be exposed via https

# Prerequisites
1. Three /24 private subnets
2. One /27 subnet for the Application Gateway
3. Enable the Azure Cloud Shell
4. Enable kubectl bash_completion
    ```
    kubectl completion bash >>  ~/.bash_completion
    . /etc/profile.d/bash_completion.sh
    . ~/.bash_completion
    ```
# Howto
## Create a Resource Group
```
az group create --name <ResourceGroupName> --location <AzureLocation>
```

## Create a variable for the resource group name:
```
RG=<RESOURCE_GROUP_NAME>
```

## AKS and Kubernetes Version Matrix
### Supported Kubernetes versions in Azure Kubernetes Service (AKS)
https://docs.microsoft.com/en-us/azure/aks/supported-kubernetes-versions?tabs=azure-cli

## Supported kubectl versions
You can use one minor version older or newer of kubectl (Client Version) relative to your kube-apiserver version (Server Version)

For example, if your kube-apiserver is at 1.17, then you can use versions 1.16 to 1.18 of kubectl with that kube-apiserver.

1. Get the current kubectl version
    ```
    kubectl version --short
    ```

2. To find out what Kubernetes versions (Server Version) are currently available for your subscription and region, use the az aks get-versions command. 
    ```
    az aks get-versions \
    --location <AZURE LOCATION> \
    --subscription <AZURE SUBSCRIPTION> # Optional
    ```

## Create a Cluster
1. Run the following command
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

2. Configure kubectl to connect to your Kubernetes cluster using the az aks get-credentials command
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

## Test the cluster getting some information 

1. List managed Kubernetes clusters.
    ```
    az aks list
    ```
2. Get kubectl client and server versions
    ```
    kubectl version --short
    ```

3. Get node information
    ```
    kubectl get nodes -o wide
    ```

4. Get cluster information
    ```
    kubectl cluster-info
    ```

5. Show the details for a managed Kubernetes cluster
    ```
    az aks show -g $RG -n Cluster01
    ```

## Application GatewayIngress Controller
Pay attention that AGIC only supports v2 SKUs

AKS — Different load balancing options. When to use what?

https://medium.com/microsoftazure/aks-different-load-balancing-options-for-a-single-cluster-when-to-use-what-abd2c22c2825

There’s three options when you enable the add-on:
* Have an existing Application gateway that you want to integrate with the cluster.
* Have the AGIC add-on create a new App Gateway to an existing subnet.
* Have the AGIC add-on create a new App Gateway and its subnet having that you provide the subnet CIDR you want to use.

Enable the AGIC add on in the cluster using the third option listed above:

    ```
    az aks enable-addons --resource-group myAKSResourceGroup \
    --name myAKSCluster -a ingress-appgw \
    --appgw-subnet-cidr 192.168.2.0/24 \
    --appgw-name myAKSAppGateway
    ```
## Deploy the application
1. Apply the configuration
    ```
    kubectl apply 4-AKS-ingress_2048.yaml
    ```
2. Check the ingress is deployed using the kubectl get ingress command
    ```
    kubectl get ingress -n game-2048
    ```

# SSL/TLS 
1. Appgw ssl redirect

    https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#ssl-redirect

    1.1. Add the followind anotation into the configuration yaml file of your application
    ```
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    ```
    
2. Appgw SSL/TLS certificate

    https://azure.github.io/application-gateway-kubernetes-ingress/features/appgw-ssl-certificate/

    https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#appgw-ssl-certificate

## Account Management on Azure AKS with AAD and AKS RBAC

https://ystatit.medium.com/account-management-on-azure-aks-with-aad-and-aks-rbac-fc178f90475b


## Useful commands

1. To delete a cluster
    ```
    az aks delete --name MyManagedCluster --resource-group MyResourceGroup
    ```

2. Stop an AKS Cluster
    ```
    az aks stop --name myAKSClusterCluster --resource-group myResourceGroup
    ```
3. Verify when your cluster is stopped
    ```
    az aks show --name myAKSCluster --resource-group myResourceGroup
    ```
4. Start an AKS Cluster
    ```
    az aks start --name Cluster01 --resource-group aks-tests-fcruz
    ```



# DRAFT

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

az aks get-credentials --name Cluster01 --resource-group $RG

az aks show --name Cluster01 \
--resource-group $RG \
--query "nodeResourceGroup"

az aks enable-addons --addons ingress-appgw \
    --resource-group aks-tests-fcruz \
    --name Cluster01 \
    --appgw-name AksCluster01 \
    --appgw-subnet-cidr 10.241.0.0/16

    [--appgw-subnet-id]

# Comming soon: SSL/TLS Certificate (Mastigadinho...)
# Create a user-assigned managed identity
az identity create -g MC_aks-tests-fcruz_Cluster01_brazilsouth -n Aks-fcruz

# Configure your resources
```
appgwName="AksCluster01"

resgp="MC_aks-tests-fcruz_Cluster01_brazilsouth"

vaultName="aks-fcruz-3"

location="brazilsouth"

agicIdentityPrincipalId="fc7bab78-4133-454d-89f4-654515a599cc"
```

Obs.: Take a look at this: Argument 'enable_soft_delete' has been deprecated and will be removed in a future release.

```
az keyvault create -n aks-fcruz -g MC_aks-tests-fcruz_Cluster01_brazilsouth --enable-soft-delete -l $location
```

```
az identity create -n appgw-id -g MC_aks-tests-fcruz_Cluster01_brazilsouth -l $location
identityID=$(az identity show -n appgw-id -g MC_aks-tests-fcruz_Cluster01_brazilsouth -o tsv --query "id")
identityPrincipal=$(az identity show -n appgw-id -g MC_aks-tests-fcruz_Cluster01_brazilsouth -o tsv --query "principalId")
```
```
az role assignment create --role "Managed Identity Operator" --assignee $agicIdentityPrincipalId --scope $identityID
```
```
az network application-gateway identity assign \
  --gateway-name $appgwName \
  --resource-group MC_aks-tests-fcruz_Cluster01_brazilsouth \
  --identity $identityID
```
```
az keyvault set-policy \
-n $vaultName \
-g MC_aks-tests-fcruz_Cluster01_brazilsouth \
--object-id $identityPrincipal \
--secret-permissions get
```
```
az keyvault certificate create \
--vault-name $vaultName \
-n mycert \
-p "$(az keyvault certificate get-default-policy)"
versionedSecretId=$(az keyvault certificate show -n mycert --vault-name $vaultName --query "sid" -o tsv)
unversionedSecretId=$(echo $versionedSecretId | cut -d'/' -f-5) # remove the version from the url
```
```
az network application-gateway ssl-cert create \
-n mykvsslcert \
--gateway-name $appgwName \
--resource-group MC_aks-tests-fcruz_Cluster01_brazilsouth \
--key-vault-secret-id $unversionedSecretId # ssl certificate with name "mykvsslcert" will be configured on AppGw
```
```
az network application-gateway ssl-cert list --gateway-name AksCluster01 --resource-group MC_aks-tests-fcruz_Cluster01_brazilsouth
```



------------
az aks show --name Cluster01 \
--resource-group $RG \
--query "nodeResourceGroup"

appgwName="AksCluster01"
resgp="MC_1-13af75b4-playground-sandbox_Cluster01_centralus"

# generate certificate for testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -out test-cert.crt \
  -keyout test-cert.key \
  -subj "/CN=test"

openssl pkcs12 -export \
  -in test-cert.crt \
  -inkey test-cert.key \
  -passout pass:test \
  -out test-cert.pfx

# configure certificate to app gateway
az network application-gateway ssl-cert create \
  --resource-group $resgp \
  --gateway-name $appgwName \
  -n mysslcert \
  --cert-file test-cert.pfx \
  --cert-password "test"




