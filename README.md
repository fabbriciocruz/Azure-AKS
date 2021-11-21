# Azure AKS Step by Step

![image](https://user-images.githubusercontent.com/80425460/141885752-2b967f64-a50e-4e7b-a80b-2025a6b0fddb.png)

# Goals
1. Deploy AKS Cluster Using Azure CLI (az aks create)
2. Configure the Ingress Controller using the Application Gateway
3. Create a keyVault
4. Create a certificate; store it in the KeyVault; install the certicate in the Application Gateway
5. Deploy an example application (4-AKS-ingress_2048.yaml) which will be exposed via Application Gateway and redirecting http traffic to https

# Tests
All tests have been run in the Brazil South location

# Considerations
By default the command 'az aks create' creates the Vnet and subnet for the AKS Cluster. It assignes the CIDR 10.0.0.0/8 to the Vnet and the CIDR 10.240.0.0/16 to the subnet.

We will take this approach for now.

As the howto evolves new topics will be added.

Coming soon: 
* Prepare the network for the AKS Cluster
* Build an AKS Cluster on an existing Vnet/subnet

# Prerequisites
1. Three /24 private subnets
2. One /27 subnet for the Application Gateway
3. Enable the Bash environment in Azure Cloud Shell
4. Enable kubectl bash_completion

    ```sh
    kubectl completion bash >>  ~/.bash_completion
    . /etc/profile.d/bash_completion.sh
    . ~/.bash_completion
    ```

## AKS and Kubernetes Version Matrix
### Supported Kubernetes versions in Azure Kubernetes Service (AKS)
https://docs.microsoft.com/en-us/azure/aks/supported-kubernetes-versions?tabs=azure-cli

## Supported kubectl versions
You can use one minor version older or newer of kubectl (Client Version) relative to your kube-apiserver version (Server Version)

For example, if your kube-apiserver is at 1.17, then you can use versions 1.16 to 1.18 of kubectl with that kube-apiserver.

1. Get the current kubectl version
    ```sh
    kubectl version --short
    ```

2. To find out what Kubernetes versions (Server Version) are currently available for your subscription and region, use the az aks get-versions command. 
    ```sh
    az aks get-versions \
    --location <AZURE LOCATION> \
    --subscription <AZURE SUBSCRIPTION> # Optional
    ```

# Howto
## Create a Resource Group which will store the AKS Cluster
```sh
az group create --name <ResourceGroupName> --location <AzureLocation>
```

## Create some environment variables
```sh
AksClusterRG="<RESOURCE_GROUP_NAME>"
AksClusterName="<AKS_CLUSTER_NAME>"
AksClusterVersion="<AKS_CLUSTER_VERSION>"
AksAppGwName="<APPLICATION_GW_NAME>"
```

## Create a Cluster
1. Run the following command
    ```sh
    az aks create \
    --resource-group $AksclusterRG \
    --name $AksClusterName \
    --kubernetes-version $AksClusterVersion \
    --node-count 3 \
    --generate-ssh-keys \
    --node-vm-size Standard_B2s \
    --enable-managed-identity
    ```

2. Configure kubectl to connect to your Kubernetes cluster using the az aks get-credentials command
    ```sh
    az aks get-credentials --name $AksClusterName --resource-group $AksClusterRG
    ```

When you create an AKS cluster, a second resource group is automatically created to store the AKS resources needed to support your cluster, things like load balancers, public IPs, VMSS backing the node pools will be created here.

Create an environment variable getting the name of the second resource group:

```sh
AksClusterResourcesRG=$(az aks show --name $AksClusterName \
    --resource-group $AksClusterRG -o tsv \
    --query "nodeResourceGroup")
```

## Test the cluster getting some information 

1. List managed Kubernetes clusters.
    ```sh
    az aks list
    ```
2. Get kubectl client and server versions
    ```sh
    kubectl version --short
    ```

3. Get node information
    ```sh
    kubectl get nodes -o wide
    ```

4. Get cluster information
    ```sh
    kubectl cluster-info
    ```

5. Show the details for a managed Kubernetes cluster
    ```sh
    az aks show -g $RG -n Cluster01
    ```

## Application Gateway Ingress Controller - AGIC

Pay attention that AGIC only supports Application Gateway v2 SKUs

The App Gateway must be deployed in the same virtual network as AKS (To evolve the howto check this prerequisite carefully !!)

AKS — Different load balancing options. When to use what?

https://medium.com/microsoftazure/aks-different-load-balancing-options-for-a-single-cluster-when-to-use-what-abd2c22c2825

There’s three options when you enable the add-on:
* Have an existing Application gateway that you want to integrate with the cluster.
* Have the AGIC add-on create a new App Gateway to an existing subnet.
* Have the AGIC add-on create a new App Gateway and its subnet having that you provide the subnet CIDR you want to use.

Enable the AGIC add-on in the cluster using the third option listed above:

```sh
az aks enable-addons --resource-group $AksClusterRG \
--name $AksClusterName --addon ingress-appgw \
--appgw-subnet-cidr <CIDR_BLOCK> \
--appgw-name $AksAppGwName
```

# We'll need to check if when the addon is enabled the managed identity is created or if it is created only when the application using the AGIC is deployed. Then we need to declare a variable for its PrincipalID

## Deploy the application
1. Apply the configuration
    ```sh
    kubectl apply 4-AKS-ingress_2048.yaml
    ```
2. Check the ingress is deployed using the kubectl get ingress command
    ```sh
    kubectl get ingress -n game-2048
    ```
3. Copy the IP Address from the output of the command above and configure your DNS Server


## SSL/TLS: Configure a self-signed certificate from Key Vault to AppGw

* Appgw SSL/TLS redirect

    https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#ssl-redirect

    Add the followind annotation into the configuration yaml file of your example application (4-AKS-ingress_2048.yaml)
    ```sh
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    ```
    
* Appgw SSL/TLS certificate

    https://azure.github.io/application-gateway-kubernetes-ingress/features/appgw-ssl-certificate/

    https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#appgw-ssl-certificate

    Add the following annotation into the configuration yaml file of your example application (4-AKS-ingress_2048.yaml)
    ```sh
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: "examplegame03.operacaomulticloud.com"
    ```

1. Declare a variable for the AGIC managed identity`s name which has been created when the ingress controller addon was enabled

    ```sh
    AgicMgdIdentity=$(az aks addon show -n $AksClusterName -g $AksClusterRG -a ingress-appgw --query "identity.resourceId")
    ```

2. Declare a variable for the AGIC Managed Identity Principal ID
    ```sh
    AgicMngdIdentityPrincipalId=$(az identity show -g $AksClusterResourcesRG \
    --name $AgicMgdIdentity -o tsv --query "principalId")
    ```

3. Check if the subscription has already a Key Vault. If not, then create a new one using the following command:

    ```sh
    vaultName="<KeyVaultName>" && \
    az keyvault create --name $vaultName --resource-group $AksClusterResourcesRG  --location $Location
    ```
4. Create a user-assigned managed identity for the Application Gateway and store into variables its identityID and PrincipalId

    ```sh
    az identity create --name appgw-identity --resource-group $AksClusterResourcesRG --location $Location
    identityID=$(az identity show -n appgw-identity -g MC_aks-tests-fcruz_Cluster01_brazilsouth -o tsv --query "id")
    PrincipalId=$(az identity show -n appgw-identity -g MC_aks-tests-fcruz_Cluster01_brazilsouth -o tsv --query "principalId")
    ```
5. Assign AGIC identity to have operator access over AppGw identity

    ```sh
    az role assignment create --role "Managed Identity Operator" --assignee $AgicMngdIdentityPrincipalId --scope $identityID
    ```

6. Assign the managed identity to Application Gateway

    ```sh
    az network application-gateway identity assign \
    --gateway-name $AksAppGwName \
    --resource-group $AksClusterResourcesRG \
    --identity $identityID
    ```

7. Assign the managed identity GET secret access to Azure Key Vault

    ```sh
    az keyvault set-policy \
    --name $vaultName \
    --resource-group $AksClusterResourcesRG \
    --object-id $PrincipalId \
    --secret-permissions get
    ```

8. For each new certificate, Declare the variable for the certificate name, create a cert on keyvault and add unversioned secret id to Application Gateway

    ```sh
    CertificateName="examplegame03.operacaomulticloud.com"
    
    az keyvault certificate create \
    --vault-name $vaultName \
    --name $CertificateName \
    --policy "$(az keyvault certificate get-default-policy)"
    versionedSecretId=$(az keyvault certificate show -n $CertificateName --vault-name $vaultName --query "sid" -o tsv)
    unversionedSecretId=$(echo $versionedSecretId | cut -d'/' -f-5) # remove the version from the url
    ```

    8.1. For each new certificate, upload the certificate to the Application Gateway

    ```sh
    az network application-gateway ssl-cert create \
    --name $CertificateName \
    --gateway-name $AksAppGwName \
    --resource-group $AksClusterResourcesRG \
    --key-vault-secret-id $unversionedSecretId # ssl certificate with name "mykvsslcert" will be configured on AppGw
    ```
9. List the certificates uploaded to the Application Gateway

    ```sh
    az network application-gateway ssl-cert list --gateway-name $AksAppGwName --resource-group $AksClusterResourcesRG
    ```

Add the anottations bellow to the application configuration yaml file (4-AKS-ingress_2048.yaml)

```
appgw.ingress.kubernetes.io/appgw-ssl-certificate: "<Name of the Certificate which has been uploaded to the Application Gateway"
appgw.ingress.kubernetes.io/ssl-redirect: "true"
```


## Account Management on Azure AKS with AAD and AKS RBAC

https://ystatit.medium.com/account-management-on-azure-aks-with-aad-and-aks-rbac-fc178f90475b


## Useful commands

1. To delete a cluster
    ```sh
    az aks delete --name MyManagedCluster --resource-group MyResourceGroup
    ```

2. Stop an AKS Cluster
    ```sh
    az aks stop --name myAKSClusterCluster --resource-group myResourceGroup
    ```
3. Verify when your cluster is stopped
    ```sh
    az aks show --name myAKSCluster --resource-group myResourceGroup
    ```
4. Start an AKS Cluster
    ```sh
    az aks start --name Cluster01 --resource-group aks-tests-fcruz
    ```
5. Delete a certificate uploaded to the Application Gateway
    ```
    az network application-gateway ssl-cert delete --gateway-name $AksAppGwName --resource-group $AksClusterResourcesRG --name examplegame03.operacaomulticloud.com
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


# References

az aks

https://docs.microsoft.com/en-us/cli/azure/service-page/azure%20kubernetes%20service%20(aks)?view=azure-cli-latest

Quickstart: Deploy an Azure Kubernetes Service cluster using the Azure CLI
https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough


