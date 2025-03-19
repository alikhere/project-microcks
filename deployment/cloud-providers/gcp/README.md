🚀 Step 1: Prerequisites
Before deploying Microcks on GKE, ensure the following tools are installed on your local system:

✅ Install Required CLI Tools
1️⃣ Install Google Cloud SDK (if not already installed)
```sh
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```
2️⃣ Install Kubectl
```sh
gcloud components install kubectl
```
3️⃣ Install Helm
```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
4️⃣ Install Docker
```sh
sudo apt update && sudo apt install -y docker.io
```
5️⃣ Install gcloud CLI authentication plugin for Kubernetes
```sh
gcloud components install gke-gcloud-auth-plugin
```

🎯 Step 2: Create and Configure a Service Account
Create a service account for Microcks:
```sh
gcloud iam service-accounts create microcks-user \
    --display-name "Microcks Service Account"
```
Grant IAM roles to the service account:
```sh
gcloud projects add-iam-policy-binding [PROJECT_ID] \
    --member="serviceAccount:microcks-user@[PROJECT_ID].iam.gserviceaccount.com" \
    --role="roles/container.admin"

gcloud projects add-iam-policy-binding [PROJECT_ID] \
    --member="serviceAccount:microcks-user@[PROJECT_ID].iam.gserviceaccount.com" \
    --role="roles/cloudsql.admin"

gcloud projects add-iam-policy-binding [PROJECT_ID] \
    --member="serviceAccount:microcks-user@[PROJECT_ID].iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountAdmin"

gcloud projects add-iam-policy-binding [PROJECT_ID] \
    --member="serviceAccount:microcks-user@[PROJECT_ID].iam.gserviceaccount.com" \
    --role="roles/storage.admin"
```
 Replace `[PROJECT_ID]` with your Google Cloud project ID.
 Enable Required APIs
```sh
gcloud services enable container.googleapis.com
```
```sh
gcloud services enable sqladmin.googleapis.com
```
```sh
gcloud services enable iam.googleapis.com
```

Step 3: Generate a Service Account Key
```sh
gcloud iam service-accounts keys create ~/microcks-key.json \
    --iam-account=microcks-user@[PROJECT_ID].iam.gserviceaccount.com
```
Move the key to a secure location:
```sh
mv ~/microcks-key.json ~/.microcks/
```

Step 4: Authenticate with gcloud Using the Service Account
```sh
gcloud auth activate-service-account \
    microcks-user@[PROJECT_ID].iam.gserviceaccount.com \
    --key-file ~/.microcks/microcks-key.json
```
Verify authentication:
```sh
gcloud auth list
```

🎯 Step 5: Create a Kubernetes (GKE) Cluster
```sh
gcloud container clusters create microcks-cluster \
    --zone us-central1-a \
    --num-nodes=3 \
    --machine-type=e2-standard-4 \
    --enable-ip-alias \
    --enable-autorepair \
    --enable-autoupgrade
```
Check the cluster status:
```sh
gcloud container clusters list
```

Step 6: Authenticate kubectl with GKE
```sh
gcloud container clusters get-credentials microcks-cluster --zone us-central1-a
```
Verify by running:
```sh
kubectl get nodes
```

🎯 Step 7: Set up Cloud SQL for Microcks
Create a Cloud SQL Instance
```sh
gcloud sql instances create microcks-sql \
    --database-version=POSTGRES_14 \
    --tier=db-f1-micro \
    --region=us-central1
```
Create a Database for Microcks
```sh
gcloud sql databases create microcks \
    --instance=microcks-sql
```
Create a User for Microcks
```sh
gcloud sql users create microcks-user \
    --instance=microcks-sql \
    --password=your-secure-password
```
🔹 Replace `your-secure-password` with a strong password.

🎯 Step 8: Deploy Microcks on GKE using Helm
Add the Microcks Helm repository
```sh
helm repo add microcks https://microcks.io/helm
helm repo update
```
Install Microcks
```sh
helm install microcks microcks/microcks \
    --set mongodb.enabled=false \
    --set mongodb.externalHost="cluster0.svr3g.mongodb.net" \
    --set mongodb.externalPort=27017 \
    --set mongodb.externalUser="alikhere" \
    --set mongodb.externalPassword="<your-mongodb-password>" \
    --set keycloak.enabled=true \
    --set keycloak.url="http://34.120.5.12:8080" \
    --set keycloak.realm="Microcks"

```
🔹 Replace `your-secure-password` with the actual password you set earlier.

Check if Microcks is Running
```sh
kubectl get pods
```

Next Steps
- Expose Microcks using an Ingress (so you can access it via a domain).
- Set up monitoring & logging.


