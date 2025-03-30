## 14.A Create Database

```sh
gcloud sql databases create keycloak_db --instance=microcks-cloudsql --project=microcks123

# Create user with correct password
gcloud sql users create keycloak_user \
  --instance=microcks-cloudsql \
  --password=microcks123 \
  --project=microcks123

# Enable Service Networking API
gcloud services enable servicenetworking.googleapis.com --project=microcks123

# Allocate IP range
gcloud compute addresses create google-managed-services-default \
  --global \
  --purpose=VPC_PEERING \
  --prefix-length=16 \
  --network=default \
  --project=microcks123

# Create VPC peering
gcloud services vpc-peerings connect \
  --service=servicenetworking.googleapis.com \
  --ranges=google-managed-services-default \
  --network=default \
  --project=microcks123

# Assign private IP to Cloud SQL
gcloud sql instances patch microcks-cloudsql \
  --network=default \
  --assign-ip \
  --project=microcks123

# Get private IP
gcloud sql instances describe microcks-cloudsql \
  --format="value(ipAddresses)" \
  --project=microcks123
```

## 14.B Create `values-keycloak.yaml`

```yaml
auth:
  adminUser: admin
  adminPassword: microcks321  # for login keycloak as admin

postgresql:
  enabled: false  # Disable embedded DB

externalDatabase:
  host: "$CLOUD_SQL_PRIVATE_IP"  # e.g., 10.23.0.3
  port: 5432
  user: "keycloak_user"
  password: "microcks123"  # Matches Cloud SQL user
  database: "keycloak_db"

service:
  type: LoadBalancer
```

## Deploy Keycloak

```sh
helm install keycloak bitnami/keycloak -f values-keycloak.yaml -n microcks --create-namespace

# Check pod status (wait for 1/1 READY)
kubectl get pods -n microcks -l app.kubernetes.io/name=keycloak

# Get LoadBalancer external IP of Keycloak and visit the URL to login
kubectl get svc -n microcks keycloak -o wide
