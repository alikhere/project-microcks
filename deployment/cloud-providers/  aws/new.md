# Deploying Microcks on Google Kubernetes Engine (GKE)

## Prerequisites
Before deploying Microcks on GKE, ensure that both the root user (admin) and the developer have installed the following tools:

1. **Google Cloud SDK (gcloud):** Required for interacting with GCP resources (e.g., GKE, Cloud SQL, Firestore). [Install Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
2. **Helm:** Required for deploying Microcks on Kubernetes using Helm charts. [Install Helm](https://helm.sh/docs/intro/install/)
3. **Kubernetes CLI (kubectl):** Required for interacting with Kubernetes clusters (e.g., deploying Microcks, viewing pods). [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
4. **Docker (Optional):** Required if deploying locally or testing with Docker Compose. [Install Docker](https://docs.docker.com/get-docker/)

---



## 1. Authenticate and Set Up GCP Project

### Authenticate and Configure GCP Project
Run the following commands to log in, create a project, and set it as active. Replace `<PROJECT-ID>` and `<PROJECT-NAME>` with your details.
```sh
$ gcloud auth login
$ gcloud projects create <PROJECT-ID> --name="<PROJECT-NAME>"
$ gcloud projects list
$ gcloud config set project <PROJECT-ID>
```
For example, if your project name is **cncf-microcks** and project ID is **microcks333**, use these values accordingly.

### Enable Billing for Your Project
Your Google Cloud project must be linked to a billing account. If you haven’t set up billing, follow these steps:
- Go to **Google Cloud Console** → **Billing**.
- Link an existing billing account or create a new one.
- Once set up, link the billing account to your project.

Alternatively, enable billing using the command-line:
```sh
$ gcloud beta billing projects describe <PROJECT-ID>
$ gcloud beta billing accounts list
$ gcloud beta billing projects link <PROJECT-ID> --billing-account=<BILLING-ACCOUNT-ID>
$ gcloud beta billing projects describe <PROJECT-ID>
```
Once billing is enabled, you're ready to proceed with the next steps.


## 2. Create Service Account and Assign Required Roles

Create a service account for deploying Microcks. Replace `<SERVICE-ACCOUNT-NAME>`, `<SERVICE-ACCOUNT-DESCRIPTION>`, and `<PROJECT-ID>` with your values. For example, the service account name could be `microcks-deployer`, and the project ID could be `microcks333`.

```sh
$ gcloud iam service-accounts create <SERVICE-ACCOUNT-NAME> \
  --display-name "<SERVICE-ACCOUNT-DESCRIPTION>" \
  --project <PROJECT-ID>
```

### Assign required roles to the service account:

```sh
$ gcloud projects add-iam-policy-binding <PROJECT-ID> \
  --member="serviceAccount:<SERVICE-ACCOUNT-NAME>@<PROJECT-ID>.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

$ gcloud projects add-iam-policy-binding <PROJECT-ID> \
  --member="serviceAccount:<SERVICE-ACCOUNT-NAME>@<PROJECT-ID>.iam.gserviceaccount.com" \
  --role="roles/cloudsql.admin"

$ gcloud projects add-iam-policy-binding <PROJECT-ID> \
  --member="serviceAccount:<SERVICE-ACCOUNT-NAME>@<PROJECT-ID>.iam.gserviceaccount.com" \
  --role="roles/serviceusage.serviceUsageAdmin"

$ gcloud projects add-iam-policy-binding <PROJECT-ID> \
  --member="serviceAccount:<SERVICE-ACCOUNT-NAME>@<PROJECT-ID>.iam.gserviceaccount.com" \
  --role="roles/datastore.user"

$ gcloud projects add-iam-policy-binding <PROJECT-ID> \
  --member="serviceAccount:<SERVICE-ACCOUNT-NAME>@<PROJECT-ID>.iam.gserviceaccount.com" \
  --role="roles/container.developer"

$ gcloud projects add-iam-policy-binding <PROJECT-ID> \
  --member="serviceAccount:<SERVICE-ACCOUNT-NAME>@<PROJECT-ID>.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```

### Create a Service Account Key
Generate a key file for the service account. Replace `<PROJECT-ID>` with your actual GCP project ID before running the command.

```sh
$ gcloud iam service-accounts keys create /path/to/microcks-deployer-key.json \
  --iam-account=microcks-deployer@<PROJECT-ID>.iam.gserviceaccount.com
```

## 3. Create a GKE Cluster

## Enable Kubernetes Engine API

Before creating a GKE cluster, enable the Kubernetes Engine API. This process takes 1-2 minutes.

```sh
$ gcloud services enable container.googleapis.com --project=<PROJECT-ID>
```

For example, if your project ID is **microcks333**, replace `<PROJECT-ID>` accordingly.

## Create a GKE Cluster

Create a Kubernetes cluster in the specified region with autoscaling enabled.

```sh
$ gcloud container clusters create <CLUSTER-NAME> \
  --zone us-central1-a \
  --machine-type e2-medium \
  --num-nodes 2 \
  --disk-size 50 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 4 \
  --enable-ip-alias \
  --enable-network-policy \
  --service-account <SERVICE-ACCOUNT>@<PROJECT-ID>.iam.gserviceaccount.com
```

Wait for 7-8 minutes for the cluster to be provisioned.

## Authenticate with the Service Account

Authenticate using the service account key to access the GKE cluster.

```sh
$ gcloud auth activate-service-account --key-file=/path/to/microcks-deployer-key.json

