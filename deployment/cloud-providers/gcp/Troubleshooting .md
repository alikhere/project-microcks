# 🚀  Troubleshooting 

## Overview
This document provides a comprehensive troubleshooting guide for deploying Microcks on Google Kubernetes Engine (GKE) with External Keycloak and MangoDB. It includes common errors encountered during the deployment process and their solutions. Use this guide to resolve issues and ensure smooth deployment.

## 1. **Project Creation Error**

### Issue: 
gcloud projects create microcks-demo ERROR: (gcloud.projects.create) Project creation failed. The project ID you specified is already in use by another project. Please try an alternative ID.

### Solution:
The error occurs because the project ID should be globally unique so ensure you are using a unique project ID.

#### Command to Create a New Project:
```sh
gcloud projects create <unique-project-id>
```

## 2. Billing Account Not Found

### Issue:
gcloud services enable container.googleapis.com --project=microcks123
ERROR: (gcloud.services.enable) FAILED_PRECONDITION: Billing account for project '952413075526' is not found.
### Solution:
Billing must be enabled for the project to use Google Cloud services. Follow the steps below:
```sh
# Step 1: Check current billing status
gcloud beta billing projects describe <your-project-id>

# Step 2: List billing accounts
gcloud beta billing accounts list

# Step 3: Link billing account
gcloud beta billing projects link <your-project-id> --billing-account=<your-billing-account-id>

# Verify
gcloud beta billing projects describe <your-project-id>
```

## 3. Unrecognized Argument --service-account During Cluster Update

### Issue: 
unrecognized arguments: --service-account

### Solution:
The error occurs because the --service-account argument is not recognized for the command. Two possible solutions:

Option 1: Create a new node pool with service account
```sh
# List existing node pools
$ gcloud container node-pools list --cluster=<cluster-name> --zone=<zone>

# Create new pool
$ gcloud container node-pools create new-pool \
  --cluster=<cluster-name> --zone=<zone> \
  --num-nodes=3 --enable-autoscaling --min-nodes=2 --max-nodes=6 \
  --service-account=<service-account-email>

# Migrate workloads and delete old pool
$ gcloud container node-pools delete default-pool --cluster=<cluster-name> --zone=<zone>
```

Option 2: Recreate the cluster with correct config
```sh
# Delete existing cluster
$ gcloud container clusters delete <cluster-name> --zone=<zone>

# Create new cluster with service account
$ gcloud container clusters create <cluster-name> \
  --zone <zone> --num-nodes=3 --enable-autoscaling --min-nodes=2 --max-nodes=6 \
  --machine-type=e2-standard-4 \
  --enable-ip-alias --enable-network-policy \
  --service-account=<service-account-email>
```

## 4. Keycloak Pod Not Starting / CrashLoopBackOff

### Diagnosis:
```sh
$ kubectl get pods -n microcks -l app.kubernetes.io/name=keycloak
$ kubectl describe pod <pod-name> -n microcks
$ kubectl logs -n microcks <pod-name> --previous
```
### a. If Error is "Unable to Connect to Host":
```sh
# Enable public IP connectivity or set up private IP.

$ GKE_NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
$ gcloud sql instances patch microcks-cloudsql \
  --authorized-networks=$GKE_NODE_IP \
  --project=<Project-id>
```

### b. If Error is "Password Authentication Failed for User":
Make sure the values for DB_USER, DB_PASSWORD, and DB_NAME in your keycloak.yaml file exactly match the credentials you used when creating the Cloud SQL database and user. Any mismatch between these values and the actual database configuration will result in authentication failures during Keycloak startup.





