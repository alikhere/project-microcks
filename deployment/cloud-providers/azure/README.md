# **Deploying Keycloak on Azure AKS with Azure PostgreSQL**

## **Overview**
This guide provides step-by-step instructions for deploying **Keycloak** on **Azure Kubernetes Service (AKS)**, using **Azure PostgreSQL** as the external database. This deployment will include:

- Azure infrastructure setup (AKS, PostgreSQL, networking)
- Keycloak configuration
- Ingress setup with SSL certificates
- Connection between AKS and Azure PostgreSQL

The goal is to have a secure, scalable identity provider (Keycloak) deployed on AKS, integrated with Azure PostgreSQL for authentication.

## **Prerequisites**
Before you begin, ensure you have the following installed:

1. **Azure CLI**: For managing Azure resources.
   - [Azure CLI Installation Guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

2. **kubectl**: For interacting with your Kubernetes cluster.
   - [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

3. **Helm**: A package manager for Kubernetes that will help deploy Keycloak and other resources.
   - [Helm Installation Guide](https://helm.sh/docs/intro/install/)

4. **Proper permissions** on your Azure subscription to create and manage resources.

## **Deployment Steps**

## 1. Azure Authentication and Initial Setup

Login to Azure using the Azure CLI:

```sh
# Login to Azure
az login

# Set the active subscription
az account set --subscription "Azure subscription 1"

# Verify subscription ID
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
echo "Subscription ID: $SUBSCRIPTION_ID"

# Create resource group
az group create --name microcks-rg --location eastus
```

## 2. Create Service Principal for Deployment
The Service Principal (SP) will allow you to automate the deployment process. Use the following commands:

```sh
# Create service principal
SP_INFO=$(az ad sp create-for-rbac \
  --name http://microcks-sp \
  --role Owner \
  --scopes /subscriptions/$SUBSCRIPTION_ID/resourceGroups/microcks-rg)

# Extract credentials
APP_ID=$(echo $SP_INFO | jq -r '.appId')
PASSWORD=$(echo $SP_INFO | jq -r '.password')
TENANT_ID=$(echo $SP_INFO | jq -r '.tenant')

# Assign Contributor role to service principal
az role assignment create \
  --assignee $APP_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/microcks-rg"
```

## 3. Network Infrastructure Setup
Create a Virtual Network (VNet) and subnets for AKS and PostgreSQL:

```sh
# Create VNet
az network vnet create \
  --resource-group microcks-rg \
  --name microcks-vnet \
  --location eastus \
  --address-prefixes 10.0.0.0/16

# Create subnets
az network vnet subnet create \
  --resource-group microcks-rg \
  --vnet-name microcks-vnet \
  --name aks-subnet \
  --address-prefixes 10.0.1.0/24

az network vnet subnet create \
  --resource-group microcks-rg \
  --vnet-name microcks-vnet \
  --name postgres-subnet \
  --address-prefixes 10.0.2.0/24 \
  --service-endpoints Microsoft.Sql
```

## 4. Create Azure PostgreSQL Database
Now, create an Azure PostgreSQL flexible server and database for Keycloak:

```sh
# Create PostgreSQL flexible server
az postgres flexible-server create \
  --resource-group microcks-rg \
  --name microcks-postgresql \
  --location eastus \
  --admin-user keycloakadmin \
  --admin-password "StrongPassword123!" \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --version 13 \
  --storage-size 32 \
  --storage-auto-grow Enabled \
  --vnet microcks-vnet \
  --subnet postgres-subnet \
  --private-dns-zone microcks-private.postgres.database.azure.com

# Create database for Keycloak
az postgres flexible-server db create \
  --resource-group microcks-rg \
  --server-name microcks-postgresql \
  --database-name keycloak_db
```

## 5. Create AKS Cluster
Set up the Azure Kubernetes Service (AKS) cluster:

```sh
# Get subnet ID
SUBNET_ID=$(az network vnet subnet show \
  --resource-group microcks-rg \
  --vnet-name microcks-vnet \
  --name aks-subnet \
  --query id -o tsv)

# Create AKS cluster
az aks create \
  --resource-group microcks-rg \
  --name microcks-cluster \
  --location eastus \
  --node-count 2 \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --node-vm-size Standard_B2ms \
  --dns-service-ip 10.2.0.10 \
  --service-cidr 10.2.0.0/24 \
  --generate-ssh-keys \
  --service-principal $APP_ID \
  --client-secret $PASSWORD

# Get credentials for kubectl
az aks get-credentials --resource-group microcks-rg --name microcks-cluster

# Verify cluster access
kubectl get nodes
```

## 6. Install Ingress Controller
Install Ingress Nginx for routing and traffic management:

```sh
# Add ingress-nginx Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.config."proxy-buffer-size"="128k"

# Get external IP (wait a few minutes if not immediately available)
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
