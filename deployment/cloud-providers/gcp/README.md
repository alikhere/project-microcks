# Deploying Microcks on Google Kubernetes Engine (GKE)

## Prerequisites
Before deploying Microcks on GKE, ensure that both the root user (admin) and the developer have installed the following tools:

1. **Google Cloud SDK (gcloud):** Required for interacting with GCP resources (e.g., GKE, Cloud SQL, Firestore). [Install Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
2. **Helm:** Required for deploying Microcks on Kubernetes using Helm charts. [Install Helm](https://helm.sh/docs/intro/install/)
3. **Kubernetes CLI (kubectl):** Required for interacting with Kubernetes clusters (e.g., deploying Microcks, viewing pods). [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
4. **Docker (Optional):** Required if deploying locally or testing with Docker Compose. [Install Docker](https://docs.docker.com/get-docker/)

---

## Deployment Steps

### 1. Authenticate with Google Cloud
```sh
gcloud auth login
```
```sh
gcloud auth list
```

### 2. Create a New Google Cloud Project
```sh
gcloud projects create <your-project-id> --name="<your-project-name>"
```
```sh
gcloud projects create microcks123 --name="cncf-microcks"
```
```sh
gcloud projects list
```

### 3. Set the Active Project
```sh
gcloud config set project <your-project-id>
```
```sh
gcloud config set project microcks123
```

### 4. Enable Billing for Your Project
#### Check if billing is enabled:
```sh
gcloud beta billing projects describe microcks123
```

#### List available billing accounts:
```sh
gcloud beta billing accounts list
```

#### Link the project to a billing account:
```sh
gcloud beta billing projects link microcks123 --billing-account=<your-billing-account-id>
```

#### Verify billing status:
```sh
gcloud beta billing projects describe microcks123
```

### 5. Create a Service Account for Deployment
```sh
gcloud iam service-accounts create microcks-deployer \
  --display-name "Microcks Deployer" \
  --project microcks123
```

### 6. Assign Roles to the Service Account
```sh
gcloud projects add-iam-policy-binding microcks123 \
  --member="serviceAccount:microcks-deployer@microcks123.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```
```sh
gcloud projects add-iam-policy-binding microcks123 \
  --member="serviceAccount:microcks-deployer@microcks123.iam.gserviceaccount.com" \
  --role="roles/cloudsql.admin"
```
```sh
gcloud projects add-iam-policy-binding microcks123 \
  --member="serviceAccount:microcks-deployer@microcks123.iam.gserviceaccount.com" \
  --role="roles/serviceusage.serviceUsageAdmin"
```
```sh
gcloud projects add-iam-policy-binding microcks123 \
  --member="serviceAccount:microcks-deployer@microcks123.iam.gserviceaccount.com" \
  --role="roles/datastore.user"
```
```sh
gcloud projects add-iam-policy-binding microcks123 \
  --member="serviceAccount:microcks-deployer@microcks123.iam.gserviceaccount.com" \
  --role="roles/container.developer"
```
```sh
gcloud projects add-iam-policy-binding microcks123 \
  --member="serviceAccount:microcks-deployer@microcks123.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```

### 7. Create a Service Account Key
```sh
gcloud iam service-accounts keys create ~/microcks-deployer-key.json \
  --iam-account=microcks-deployer@microcks123.iam.gserviceaccount.com
```

### 8. Enable Kubernetes Engine API
```sh
gcloud services enable container.googleapis.com --project=microcks123
```

### 9. Create a GKE Cluster
```sh
gcloud container clusters create microcks-cluster \
  --zone us-central1-a \
  --num-nodes=3 \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=6 \
  --enable-ip-alias \
  --enable-network-policy
```

### 10. Authenticate with the Service Account
```sh
gcloud auth activate-service-account --key-file=/home/alihere/microcks-deployer-key.json
```

### 11. Set the Project
```sh
gcloud config set project microcks123
```

### 12. Get Kubernetes Credentials and Verify Access
```sh
gcloud container clusters get-credentials microcks-cluster --zone us-central1-a
```
```sh
kubectl get nodes
```

### 13. Add the Microcks Helm Chart Repository
```sh
helm repo add microcks https://microcks.io/helm
```
```sh
helm repo update
```

### 14. Create a Cloud SQL Instance
```sh
gcloud sql instances create microcks-cloudsql \
  --database-version=POSTGRES_14 \
  --tier=db-f1-micro \
  --region=us-central1 \
  --root-password=microcks123 \
  --project=microcks123
```

### 15. Create a Database and Get Cloud SQL Connection Name
```sh
gcloud sql databases create microcks --instance=microcks-cloudsql --project=microcks123
```
```sh
gcloud sql instances describe microcks-cloudsql --format="value(connectionName)"
```

### 16. Enable Firestore API and Create Firestore Database
```sh
gcloud services enable firestore.googleapis.com --project=microcks123
```
```sh
gcloud firestore databases create --location=us-central1 --project=microcks123
