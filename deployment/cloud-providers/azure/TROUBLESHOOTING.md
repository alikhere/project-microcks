## 1. **Project Creation Error**

### Issue: 
gcloud projects create microcks-demo ERROR: (gcloud.projects.create) Project creation failed. The project ID you specified is already in use by another project. Please try an alternative ID.

### Solution:
The error occurs because the project ID should be globally unique so ensure you are using a unique project ID.

#### Create a New Project:
```sh
$ gcloud projects create <unique-project-id>
```

## 2. Billing Account Not Found

### Issue:
gcloud services enable container.googleapis.com --project=microcks123
ERROR: (gcloud.services.enable) FAILED_PRECONDITION: Billing account for project '952413075526' is not found.

### Solution:
Billing must be enabled for the project to use Google Cloud services. Follow the steps below:
```sh
# Step 1: Check current billing status
$ gcloud beta billing projects describe <your-project-id>

# Step 2: List billing accounts
$ gcloud beta billing accounts list

# Step 3: Link billing account
$ gcloud beta billing projects link <your-project-id> --billing-account=<your-billing-account-id>

# Verify
$ gcloud beta billing projects describe <your-project-id>
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
`
