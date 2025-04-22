# Microcks Production Deployment Guidelines for Managed Kubernetes

**Objective:** This document provides general guidelines for deploying Microcks in a production-ready configuration on managed Kubernetes services offered by various cloud providers (e.g., AWS EKS, Google GKE, Azure AKS, OVH Managed Kubernetes, etc.).

**Community Driven:** This is a living document. We encourage the Microcks community to contribute provider-specific guides (e.g., `DEPLOYING_ON_AWS_EKS.md`, `DEPLOYING_ON_AZURE_AKS.md`) that follow these general principles, detailing the specific commands and configurations for their chosen environment.

## Core Deployment Strategy

The recommended production setup involves:

1.  **Managed Kubernetes Cluster:** Use the provider's managed Kubernetes offering (EKS, GKE, AKS, etc.).
2.  **Managed PostgreSQL:** Utilize the provider's managed PostgreSQL service (AWS RDS, Google Cloud SQL, Azure Database for PostgreSQL, etc.) for Keycloak's database.
3.  **Managed/External MongoDB:**
    * **Preferred:** Use the provider's managed MongoDB-compatible service (AWS DocumentDB, Azure Cosmos DB for MongoDB API, Atlas on the respective cloud).
    * **Alternative:** If a suitable managed MongoDB service is unavailable or not preferred, deploy a highly-available MongoDB cluster within Kubernetes (e.g., using the Bitnami MongoDB Helm chart with persistence).
4.  **External Keycloak:** Deploy Keycloak separately (using its Helm chart) within the Kubernetes cluster, configured to use the managed PostgreSQL database.
5.  **Microcks Deployment:** Deploy Microcks (using its Helm chart), configured to connect to the external Keycloak and the external/managed MongoDB.
6.  **Ingress & TLS:** Use an Ingress controller (e.g., NGINX Ingress) and Cert-Manager for managing external access and automating TLS certificates (e.g., via Let's Encrypt).

## Prerequisites (Generic)

Before starting, ensure you have:

1.  **Cloud Provider Account:** Access to your chosen cloud provider (AWS, GCP, Azure, etc.) with appropriate permissions.
2. **Helm:** Used for deploying Microcks via Helm charts.  [Install Helm](https://helm.sh/docs/intro/install/)
3. **Kubernetes CLI (kubectl):** Required for managing Kubernetes resources on GKE.  [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
4. **Cloud Provider CLI:** The command-line interface specific to your chosen cloud provider (e.g., gcloud for Google Cloud Platform, aws for Amazon Web Services, az for Microsoft Azure).  
5.  **Domain Name:** A registered domain name is highly recommended for user-friendly access and proper TLS certificate management.


## General Deployment Steps

These steps outline the logical flow. Specific commands and console actions will differ based on your cloud provider.

### 1. Prepare Cloud Environment (Provider-Specific)

* **Provision Kubernetes Cluster:** Create your managed Kubernetes cluster if you haven't already. Ensure node pools have adequate resources (CPU/Memory/Disk).
* **Configure IAM/Permissions:** Set up necessary service accounts or user roles with permissions to manage Kubernetes resources and interact with managed database services.
* **Network Configuration:** Ensure your Kubernetes cluster's network (VPC/VNet) can securely communicate with the managed database services (e.g., via VPC Peering, Private Endpoints, firewall rules). This is critical and highly provider-dependent.

### 2. Set up Managed PostgreSQL (Provider-Specific)

* **Provision Instance:** Create a managed PostgreSQL instance (e.g., AWS RDS, Cloud SQL, Azure DB for PostgreSQL). Choose an appropriate instance size and version (check Keycloak compatibility).
* **Configure Networking:** Ensure the instance is accessible *privately* from your Kubernetes cluster's network. Note the private IP address or endpoint.
* **Create Database:** Create a dedicated database for Keycloak (e.g., `keycloak_db`).
* **Create User:** Create a dedicated database user for Keycloak with appropriate permissions on the database (e.g., `keycloak_user`). Note the username and password securely.

### 3. Set up Managed/External MongoDB (Provider/Choice-Specific)

* **Option A: Managed MongoDB Service (Preferred):**
    * **Provision Service:** Create an instance of a managed MongoDB-compatible service (e.g., DocumentDB, Cosmos DB MongoDB API, Atlas).
    * **Configure Networking:** Ensure private accessibility from your Kubernetes cluster.
    * **Create Database/User:** Create a database (e.g., `microcks`) and a user with read/write permissions. Note the connection string, username, and password securely.
* **Option B: Self-Hosted MongoDB on Kubernetes:**
    * **Install MongoDB:** Deploy MongoDB within your Kubernetes cluster, typically using a Helm chart (e.g., `bitnami/mongodb`).
        * **Enable Authentication:** Ensure authentication is enabled (`auth.enabled=true`). Set a root password and create a dedicated user/database for Microcks during installation.
        * **Enable Persistence:** Ensure persistence is enabled (`persistence.enabled=true`) using an appropriate `storageClass` available in your cluster.
        * **Configure Resources:** Set appropriate CPU/Memory requests and limits.
        * **(Optional) High Availability:** For production, consider deploying a replica set architecture instead of standalone.
    * **Note Connection Details:** The internal Kubernetes service name (e.g., `mongodb.microcks.svc.cluster.local`), database name, username, and password.

* **Create Kubernetes Secret:** Regardless of the option chosen, create a Kubernetes secret in the `microcks` namespace to store the MongoDB username and password. This allows Microcks to connect securely without hardcoding credentials.

    ```bash
    kubectl create namespace microcks # If not already created
    kubectl create secret generic microcks-mongodb-credentials -n microcks \
      --from-literal=username='<YOUR-MONGODB-USERNAME>' \
      --from-literal=password='<YOUR-MONGODB-PASSWORD>'
    ```

### 4. Prepare Kubernetes Cluster (Common Patterns)

* **Create Namespace:**
    ```bash
    kubectl create namespace microcks
    ```
* **Install Ingress Controller:** An Ingress controller is needed to expose Keycloak and Microcks. NGINX Ingress is a popular choice.
    ```bash
    # Add NGINX Helm repo
    helm repo add ingress-nginx [https://kubernetes.github.io/ingress-nginx](https://kubernetes.github.io/ingress-nginx)
    helm repo update

    # Install (example for LoadBalancer service type, adjust as needed)
    helm install ingress-nginx ingress-nginx/ingress-nginx \
      --namespace ingress-nginx \
      --create-namespace \
      --set controller.service.type=LoadBalancer \
      --set controller.config."proxy-buffer-size"="128k" # Adjust buffer if needed
    ```
    *Wait for the external IP address to be assigned to the `ingress-nginx-controller` service.*
* **Configure DNS:** Point subdomains (e.g., `keycloak.your-domain.com` and `microcks.your-domain.com`) to the External IP address of the Ingress controller service using A records in your DNS provider. (Or use nip.io for testing: `keycloak.<INGRESS_IP>.nip.io`).
* **Install Cert-Manager:** For automatic TLS certificate provisioning (e.g., Let's Encrypt).
    ```bash
    # Add Jetstack Helm repo
    helm repo add jetstack [https://charts.jetstack.io](https://charts.jetstack.io)
    helm repo update

    # Install Cert-Manager (CRDs are required)
    helm install cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --set installCRDs=true
    ```
* **Create ClusterIssuer:** Define how TLS certificates will be obtained (e.g., Let's Encrypt HTTP-01 challenge).
    ```yaml
    # cluster-issuer.yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod # Or letsencrypt-staging for testing
    spec:
      acme:
        server: [https://acme-v02.api.letsencrypt.org/directory](https://acme-v02.api.letsencrypt.org/directory) # Or staging server
        email: <YOUR-EMAIL-ADDRESS> # Replace with your email
        privateKeySecretRef:
          name: letsencrypt-prod-private-key # Or staging key name
        solvers:
        - http01:
            ingress:
              class: nginx # Match your ingress controller class
    ```
    ```bash
    kubectl apply -f cluster-issuer.yaml
    ```

### 5. Deploy External Keycloak

* **Add Keycloak Helm Repo:**
    ```bash
    helm repo add bitnami [https://charts.bitnami.com/bitnami](https://charts.bitnami.com/bitnami)
    helm repo update
    ```
* **Prepare `keycloak-values.yaml`:** Create a values file to configure the Keycloak deployment.

    ```yaml
    # keycloak-values.yaml
    auth:
      adminUser: admin
      # !! CHANGE THIS !! Use a strong password or manage via secrets
      adminPassword: "<YOUR-KEYCLOAK-ADMIN-PASSWORD>"
      # Optional: Existing secret for admin credentials
      # existingSecret: keycloak-admin-secret

    # Disable the bundled PostgreSQL
    postgresql:
      enabled: false

    # Configure connection to your Managed PostgreSQL
    externalDatabase:
      host: "<YOUR-CLOUDSQL/RDS-PRIVATE-IP-OR-ENDPOINT>" # e.g., 10.x.x.x or instance.xyz.rds.amazonaws.com
      port: 5432
      database: "keycloak_db" # The database you created
      user: "keycloak_user"   # The user you created
      password: "<YOUR-POSTGRES-USER-PASSWORD>" # The password for the user


    # Configure Service and Ingress for external access
    service:
      type: ClusterIP # Expose via Ingress, not LoadBalancer directly

    ingress:
      enabled: true
      ingressClassName: nginx # Match your Ingress Controller class
      hostname: keycloak.<YOUR-DOMAIN.COM> # Replace with your Keycloak subdomain
      annotations:
        # Use Cert-Manager for TLS
        cert-manager.io/cluster-issuer: "letsencrypt-prod" # Match your ClusterIssuer name
        # Specific annotations might be needed depending on ingress controller/provider
        nginx.ingress.kubernetes.io/backend-protocol: "HTTP" # Keycloak service runs HTTP internally
      tls: true # Enable TLS termination at the Ingress

    # Adjust resources based on expected load
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "2Gi"

    # Consider enabling persistence for Keycloak themes/providers if needed
    # persistence:
    #   enabled: true
    #   storageClass: "your-storage-class" # e.g., gp2, standard-rwo
    #   size: 5Gi
    ```
* **Install Keycloak:**
    ```bash
    helm install keycloak bitnami/keycloak \
      -f keycloak-values.yaml \
      --namespace microcks
    ```
* **Verify:** Check pod status (`kubectl get pods -n microcks`) and ingress (`kubectl get ingress -n microcks`). Wait for Keycloak to be ready and the TLS certificate to be issued. Access the Keycloak admin console via `https://keycloak.<YOUR-DOMAIN.COM>`.

### 6. Configure Keycloak
