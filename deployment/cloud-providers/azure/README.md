hi# Deploying Microcks on Azure Kubernetes Service (AKS) with External Keycloak, PostgreSQL, and Cosmos DB

This document provides a step-by-step guide for deploying Microcks on Azure Kubernetes Service (AKS). The deployment integrates with an external Keycloak for authentication, Azure Database for PostgreSQL for Keycloak's data, and Azure Cosmos DB (MongoDB API) for Microcks' data.

## Overview
In this guide, we will deploy Microcks on AKS with the following components:
- **Keycloak**: An open-source identity and access management solution.
- **Azure Database for PostgreSQL**: A relational database service for managing Keycloak’s data.
- **Azure Cosmos DB (MongoDB API)**: A NoSQL database used for storing Microcks' data.
  
We will walk through the steps to prepare your environment, create necessary resources, configure the AKS cluster, and deploy both Keycloak and Microcks.

## Prerequisites
Before beginning the deployment, ensure the following tools are installed and configured for both the root user and any developer involved:

- **Azure CLI**: Used to interact with Azure resources.  
    Install Azure CLI [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).
  
- **Helm**: For deploying Microcks and Keycloak on Kubernetes.  
    Install Helm [here](https://helm.sh/docs/intro/install/).

- **Kubectl**: Kubernetes command-line tool.  
    Install kubectl [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

- **Docker** (Optional): For local deployment or testing with Docker Compose.  
    Install Docker [here](https://docs.docker.com/get-docker/).

## 1. Login to Azure
Log in to your Azure account using the Azure CLI:

```bash
az login
```

## 2. Create the Service Principal (SP)

A Service Principal (SP) is used to automate Azure resource management. The SP will be assigned necessary permissions to manage resources.

### 2.1 Create the Service Principal

Run the following command, replacing `{subscription-id}` with your Azure Subscription ID and `{resource-group-name}` with the name of your resource group:

```bash
az ad sp create-for-rbac --name http://microcks-sp --role Owner --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group-name}
```
# Create the Service Principal (SP)

## 2.2 Assign Roles to the Service Principal

```bash
# Assign Contributor role for general resource management
az role assignment create --assignee <appId> --role "Contributor" --scope /subscriptions/{subscription-id}/resourceGroups/{resource-group-name}

# Assign role for AKS cluster interaction
az role assignment create --assignee <appId> --role "Azure Kubernetes Service Cluster User Role" --scope /subscriptions/{subscription-id}/resourceGroups/{resource-group-name}

# Assign role for Cosmos DB management
az role assignment create --assignee <appId> --role "Cosmos DB Account Contributor" --scope /subscriptions/{subscription-id}/resourceGroups/{resource-group-name}

# Assign role for PostgreSQL server management
az role assignment create --assignee <appId> --role "Azure Database for PostgreSQL Server Contributor" --scope /subscriptions/{subscription-id}/resourceGroups/{resource-group-name}
```

# 3. Log in with the Service Principal
Log in using the Service Principal to authenticate and perform actions from the terminal:

```bash
az login --service-principal -u <appId> -p <password> --tenant <tenant>
```
