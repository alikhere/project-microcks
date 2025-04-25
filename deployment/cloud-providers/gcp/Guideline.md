# Microcks Production Deployment Guidelines for Managed Kubernetes

**Objective:** This document provides general guidelines for deploying Microcks in a production-ready configuration on managed Kubernetes services offered by various cloud providers (e.g., AWS EKS, Google GKE, Azure AKS, OVH Managed Kubernetes, etc.).

**Philosophy:** We leverage cloud-native services for dependencies like databases (PostgreSQL, MongoDB) and externalize authentication (Keycloak) for better scalability, manageability, and security, aligning with standard cloud best practices. These guidelines focus on the *common patterns*, while specific implementation details will vary per cloud provider.

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
2.  **Managed Kubernetes Cluster:** A running managed Kubernetes cluster.
3.  **`kubectl`:** Configured to interact with your Kubernetes cluster.
4.  **`helm`:** Helm v3+ installed locally.
5.  **Domain Name:** A registered domain name is highly recommended for user-friendly access and proper TLS certificate management.
6.  **Basic Knowledge:** Familiarity with Kubernetes concepts (Pods, Services, Ingress, Secrets, Persistent Volumes), Helm charts, and your chosen cloud provider's services (IAM, networking, databases).

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
      # Optional: Use an existing secret for DB credentials
      # existingSecret: keycloak-db-secret
      # existingSecretPasswordKey: password

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

* **Log in:** Access the Keycloak Admin Console using the admin credentials defined in `keycloak-values.yaml`.
* **Create Realm:** Create a new realm named `microcks`.
* **Create Client:**
    * Within the `microcks` realm, navigate to `Clients` and click `Create client`.
    * **Client ID:** `microcks-app-js` (or your preferred ID).
    * **Client authentication:** ON
    * Click `Next`.
    * **Valid redirect URIs:** `https://microcks.<YOUR-DOMAIN.COM>/*` (Replace with your Microcks domain). Add others if needed (e.g., for local development `http://localhost:8080/*`).
    * **Web origins:** `+ https://microcks.<YOUR-DOMAIN.COM>` (Allows JavaScript from the Microcks UI). Add others if needed.
    * Save the client.
    * Go to the `Credentials` tab for the client and note the **Client secret**. You will need this for the Microcks configuration.
* **Create Users/Roles (Optional):** Create users within the `microcks` realm who should be able to log in to Microcks. Assign roles as needed (Microcks typically uses `user` and `admin` roles).

### 7. Deploy Microcks

* **Add Microcks Helm Repo:**
    ```bash
    helm repo add microcks [https://microcks.io/helm](https://microcks.io/helm)
    helm repo update
    ```
* **Prepare `microcks-values.yaml`:** Create a values file to configure the Microcks deployment.

    ```yaml
    # microcks-values.yaml

    # Disable bundled components - we use external ones
    keycloak:
      enabled: true
      install: false # IMPORTANT: Do not install bundled Keycloak
      # URL accessible from user browsers AND Microcks backend pods
      url: https://keycloak.<YOUR-DOMAIN.COM>
      # Usually the same as 'url' unless specific internal routing exists
      privateUrl: https://keycloak.<YOUR-DOMAIN.COM>
      realm: microcks
      client:
        id: microcks-app-js # The Client ID created in Keycloak
        # Use the secret generated in Keycloak Client Credentials tab
        secret: "<YOUR-KEYCLOAK-CLIENT-SECRET>"
        # Optional: Use an existing secret
        # existingSecret: microcks-keycloak-client-secret
        # existingSecretSecretKey: client-secret

    mongodb:
      install: false # IMPORTANT: Do not install bundled MongoDB
      database: microcks # The MongoDB database name
      # Use the secret created earlier for MongoDB credentials
      secretRef:
        secret: microcks-mongodb-credentials # Name of the K8s secret
        usernameKey: username              # Key within the secret for username
        passwordKey: password              # Key within the secret for password

    # Configure Microcks Ingress
    microcks:
      url: microcks.<YOUR-DOMAIN.COM> # Replace with your Microcks subdomain
      ingressClassName: nginx        # Match your Ingress Controller class
      ingressAnnotations:
        cert-manager.io/cluster-issuer: "letsencrypt-prod" # Match your ClusterIssuer
        nginx.ingress.kubernetes.io/proxy-body-size: "50m" # Increase if importing large specs
        # Add other annotations as needed

      # Configure connection to MongoDB
      # Option 1: Using SPRING_DATA_MONGODB_URI (if secretRef is not sufficient/complex URI)
      # Needs username/password directly or fetched from secret via envFrom
      # env:
      # - name: SPRING_DATA_MONGODB_URI
      #   value: "mongodb://<USER>:<PASSWORD>@<MONGO_HOST>:<MONGO_PORT>/microcks?authSource=admin&tls=true..." # Construct your full URI

      # Option 2: Standard way using secretRef (preferred if URI is simple)
      # Ensure the mongodb.secretRef section above is configured. Microcks Helm chart
      # will typically construct the standard mongodb:// URI from secretRef details.

      # Add Keycloak URL to allowed CORS origins for UI interaction
      extraEnv: |
        - name: CORS_REST_ALLOWED_ORIGINS
          value: "https://keycloak.<YOUR-DOMAIN.COM>"
        - name: CORS_REST_ALLOW_CREDENTIALS
          value: "true"

    # Enable Ingress and TLS
    ingress:
      enabled: true
      tls: true

    # Postman runtime (usually fine with defaults)
    postman:
      enabled: true

    # Async features (Optional - see below)
    # features:
    #   async:
    #     enabled: false

    # Resource limits (Adjust based on load)
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
    postman:
      resources:
        requests:
          cpu: "200m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
    # asyncMinion: # If async enabled
    #   resources:
    #     requests: ...
    #     limits: ...
    ```
* **Install Microcks:**
    ```bash
    helm install microcks microcks/microcks \
      -f microcks-values.yaml \
      --namespace microcks
    ```
* **Verify:** Check pod status (`kubectl get pods -n microcks -w`) and ingress (`kubectl get ingress -n microcks`). Wait for Microcks pods (main, postman) to be ready and the TLS certificate to be issued.

### 8. (Optional) Configure Advanced Features (Async)

If you need asynchronous API mocking/testing:

* **Prepare Broker:** Set up a Kafka broker (e.g., using Strimzi operator on Kubernetes, or a managed Kafka service like AWS MSK, Confluent Cloud, etc.). Ensure network connectivity from Microcks pods.
* **Update `microcks-values.yaml`:**
    * Set `features.async.enabled: true`.
    * Configure the Kafka connection details under `features.async.kafka`:
        * Set `install: false` if using an external Kafka.
        * Provide `bootstrap.servers`.
        * Configure authentication (SASL/TLS) and potentially schema registry (`schemaRegistry.url`) if used.
        * If using the bundled Kafka (`install: true`), configure its ingress, TLS (`features.async.kafka.tls.enabled: true`), and potentially persistence. Note that deploying Kafka requires enabling SSL Passthrough on the NGINX Ingress Controller and setting up appropriate TLS certs.
* **Upgrade Microcks:**
    ```bash
    helm upgrade microcks microcks/microcks \
      -f microcks-values.yaml \
      --namespace microcks
    ```
* **Verify:** Check the `microcks-async-minion` pod starts correctly and can connect to Kafka.

### 9. Verification

* Access the Microcks UI at `https://microcks.<YOUR-DOMAIN.COM>`.
* You should be redirected to the Keycloak login page (`https://keycloak.<YOUR-DOMAIN.COM>`).
* Log in using a user created in the `microcks` realm in Keycloak.
* You should be redirected back to the Microcks UI, logged in.
* Try importing an API specification or sample mocks to test functionality.

## Contributing Provider-Specific Guides

Please follow this general structure when creating guides for specific cloud providers (e.g., `DEPLOYING_ON_AWS_EKS.md`). Focus on the provider-specific commands and configurations for:

1.  **Prerequisites:** Any provider-specific tools (`awscli`, `az`, `gcloud`) or setup needed.
2.  **Step 1: Prepare Cloud Environment:** Specific commands/console steps for cluster creation, IAM roles/policies, and VPC/network setup (Security Groups, Subnets, VPC Peering, PrivateLink, etc.).
3.  **Step 2: Set up Managed PostgreSQL:** Specific commands/console steps for provisioning RDS/Cloud SQL/Azure DB, configuring security groups/firewalls, and getting connection details.
4.  **Step 3: Set up Managed/External MongoDB:** Specific commands/console steps for DocumentDB/CosmosDB/Atlas or detailed Helm values for deploying MongoDB on K8s within that provider's environment (including `storageClass`).
5.  **Step 4-9:** Reference the common steps, but highlight any provider-specific nuances (e.g., specific Ingress annotations, Load Balancer types, StorageClass names).

Submit contributions via Pull Requests to the Microcks community repository.
