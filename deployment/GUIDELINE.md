# Community-Driven Guidelines for Deploying Microcks in Cloud Production Environments

This document provides a common guideline for deploying Microcks in production-grade cloud environments using managed Kubernetes services and leveraging cloud-native backing services. It serves as a framework adaptable to various cloud providers (AWS, GCP, Azure, OVH, Oracle, Scaleway, Koyeb, etc.).

The goal is to foster a collaborative repository of best practices, allowing users to understand the necessary steps and apply them using their chosen cloud provider's tools and services. This guide focuses on *what* needs to be done, leaving the *how* (specific commands and cloud console steps) to be implemented by the user based on their cloud provider's documentation and the community-contributed provider-specific examples.

**Key Deployment Architecture:**

This guideline assumes a deployment architecture utilizing:

* **Managed Kubernetes Service:** Your cloud provider's managed Kubernetes offering (e.g., Amazon EKS, Google GKE, Azure AKS).
* **External Keycloak:** Keycloak deployed separately (ideally on the managed Kubernetes cluster) using a managed PostgreSQL-compatible database.
* **Managed PostgreSQL-Compatible Database:** Your cloud provider's managed relational database service configured for PostgreSQL (e.g., Amazon RDS for PostgreSQL, Google Cloud SQL for PostgreSQL, Azure Database for PostgreSQL). This will be used by Keycloak.
* **MongoDB Solution:** A suitable MongoDB solution. This could be:
    * Your cloud provider's managed MongoDB-compatible service (e.g., Amazon DocumentDB for MongoDB Compatibility, Azure Cosmos DB for MongoDB API), *if* fully compatible with Microcks.
    * An external MongoDB instance deployed on your managed Kubernetes cluster (e.g., using a Helm chart like Bitnami's MongoDB chart), if a suitable managed service is not available or preferred.
* **Ingress Controller & Certificate Management:** To expose Microcks and Keycloak securely (e.g., NGINX Ingress Controller and cert-manager).

**Prerequisites:**

Before starting, ensure you have the following tools installed and configured locally:

* **Cloud Provider CLI / SDK:** Necessary for interacting with your specific cloud provider's services (e.g., AWS CLI, gcloud CLI, Azure CLI, etc.). Authenticate and configure it for your target project/account.
* **Helm:** Used for deploying Microcks and other components via Helm charts. [Install Helm](https://helm.sh/docs/intro/install/)
* **Kubernetes CLI (kubectl):** Required for managing resources on your Kubernetes cluster. [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* **Docker (Optional):** Useful for local testing and development using Docker Compose setups. [Install Docker](https://www.docker.com/products/docker-desktop/)

**Deployment Steps:**

Follow these steps, adapting the specific commands and procedures to your chosen cloud provider.

---

### Phase 1: Prepare Cloud Infrastructure and Backing Services

In this phase, you will set up the foundational cloud resources required for Microcks.

1.  **Authenticate and Set Up Cloud Account/Project:**
    * Authenticate with your cloud provider's CLI/SDK.
    * Create or select the project/account where you will deploy resources.
    * Ensure that billing is enabled for your project/account.

2.  **Create Service Account/Identity and Assign Required Permissions:**
    * Create a dedicated service account or identity that will be used for deploying and managing resources (e.g., Kubernetes cluster creation, database interactions).
    * Assign this identity the necessary permissions to provision and manage:
        * Managed Kubernetes Clusters
        * Managed Database Services (PostgreSQL, MongoDB if used)
        * Networking resources (VPC, subnets, security groups, private endpoints)
        * IAM roles/policies

3.  **Provision Managed Kubernetes Cluster:**
    * Provision a managed Kubernetes cluster in your desired region (e.g., EKS, GKE, AKS).
    * Configure the cluster size, node types, and autoscaling based on your expected load.
    * Ensure the cluster's nodes are associated with the service account/identity created in the previous step, or configure appropriate network access for the nodes.
    * Configure `kubectl` to connect to your newly created cluster.

4.  **Provision Managed PostgreSQL-Compatible Database (for Keycloak):**
    * Provision a managed PostgreSQL-compatible database instance (e.g., RDS for PostgreSQL, Cloud SQL for PostgreSQL).
    * Create a dedicated database and user credentials for Keycloak.
    * Note the database host, port, database name, username, and password.

5.  **Provision MongoDB Solution (for Microcks):**
    * **Option A: Using a Managed MongoDB-Compatible Service:**
        * Provision your cloud provider's managed MongoDB-compatible database service (e.g., DocumentDB, Cosmos DB for MongoDB API).
        * Ensure it's configured to be accessible from your Kubernetes cluster (see Networking step below).
        * Note the connection string, database name, username, and password.
    * **Option B: Deploying External MongoDB on Kubernetes:**
        * If a suitable managed service is not available or preferred, you will deploy a MongoDB instance directly onto your Kubernetes cluster using a standard Helm chart (like Bitnami's). This will be covered in Phase 2.

6.  **Establish Private Network Connectivity:**
    * Configure networking to ensure secure, private communication between:
        * Your Kubernetes cluster nodes and the Managed PostgreSQL instance.
        * Your Kubernetes cluster nodes and the Managed MongoDB instance (if using Option A in step 5).
    * This typically involves configuring VPCs, subnets, private endpoints, peering, and security groups/firewalls.

---

### Phase 2: Deploy Shared Services on Kubernetes

In this phase, you deploy common components necessary for Microcks and Keycloak within your Kubernetes cluster.

1.  **Add Required Helm Repositories:**
    * Add the Helm repositories for necessary charts like `bitnami`, `ingress-nginx`, `jetstack`, `microcks`, and `strimzi` (if deploying asynchronous features).

2.  **Create Kubernetes Namespace:**
    * Create a dedicated Kubernetes namespace for your Microcks and related deployments (e.g., `microcks`).

3.  **Install Ingress Controller:**
    * Install an Ingress Controller (e.g., NGINX Ingress Controller) onto your cluster using its standard Helm chart. This will route external traffic to your services.
    * Note the external IP or hostname assigned to the Ingress Controller's LoadBalancer service.

4.  **Configure DNS / Domains:**
    * Set up DNS records in your domain provider (or use a service like nip.io for testing) to point desired hostnames (e.g., `keycloak.<YOUR-DOMAIN>`, `microcks.<YOUR-DOMAIN>`, `microcks-grpc.<YOUR-DOMAIN>`, `kafka.<YOUR-DOMAIN>` if async is used) to the Ingress Controller's external IP/hostname.

5.  **Install Certificate Manager:**
    * Install `cert-manager` using its standard Helm chart. This will automate the provisioning and management of TLS certificates.

6.  **Create ClusterIssuer:**
    * Configure a `ClusterIssuer` or `Issuer` resource for `cert-manager` (e.g., using Let's Encrypt) to automatically issue certificates for your domains.

---

### Phase 3: Deploy and Configure External Keycloak

Deploy Keycloak using the managed PostgreSQL database provisioned earlier.

1.  **Prepare Keycloak Helm Values:**
    * Create a Helm values file (e.g., `keycloak-values.yaml`) to configure the Keycloak Helm chart.
    * Crucially, disable the embedded PostgreSQL and configure the connection details for your managed PostgreSQL database (host, port, database, user, password).
    * Configure Ingress settings (enabled, ingress class, hostname, annotations for cert-manager).
    * Set initial admin credentials for Keycloak.

2.  **Install Keycloak using Helm:**
    * Deploy Keycloak onto your Kubernetes cluster using the Helm chart and your values file.

3.  **Verify Keycloak Deployment:**
    * Check the status of the Keycloak pods to ensure they are running.
    * Verify the Keycloak Ingress resource and access Keycloak via the configured hostname to ensure it's reachable.

4.  **Configure Keycloak for Microcks:**
    * Follow the Keycloak deployment guide provided by the Microcks community, focusing on the configuration steps required *after* Keycloak is deployed and accessible via its domain.
    * Specifically, complete the following configuration within the Keycloak administration console:
        * Create a `microcks` realm.
        * Set up a client for Microcks within that realm (e.g., `microcks-app-js`). Configure redirect URIs and web origins for your Microcks hostname.
        * Create a user (or users) and assign them appropriate roles for accessing the Microcks dashboard within the `microcks` realm.

---

### Phase 4: Deploy Microcks

Deploy Microcks, connecting it to the external Keycloak and your MongoDB solution.

1.  **Create MongoDB Connection Secret (If deploying MongoDB on Kubernetes):**
    * If you chose Option B for your MongoDB solution (deploying on Kubernetes in Phase 1), create a Kubernetes `Secret` containing the credentials for your MongoDB instance. This secret will be referenced in the Microcks Helm values.

2.  **Prepare Microcks Helm Values:**
    * Create a Helm values file (e.g., `microcks-values.yaml`) to configure the Microcks Helm chart.
    * Configure the `microcks.url`, `ingressClassName`, and ingress annotations (referencing your cert-manager ClusterIssuer).
    * Disable the embedded Keycloak and MongoDB (`keycloak.install: false`, `mongodb.install: false`).
    * Configure the external Keycloak details (`keycloak.enabled: true`, `keycloak.url`, `keycloak.privateUrl`, `keycloak.realm`, `keycloak.client.id`, `keycloak.client.secret`).
    * Configure the MongoDB connection:
        * If using a managed service (Option A in Phase 1), provide the connection string using `env: SPRING_DATA_MONGODB_URI`.
        * If using MongoDB on Kubernetes (Option B in Phase 1), reference the connection secret using `mongodb.secretRef`.
    * Configure CORS settings if necessary (`CORS_REST_ALLOWED_ORIGINS`).
    * Configure gRPC ingress if needed (`grpcEnableTLS`, `grpcIngressClassName`, `grpcIngressAnnotations`).

3.  **Deploy Microcks with Default Options:**
    * Deploy Microcks using the Helm chart and your `microcks-values.yaml`.
    * Refer to the Microcks community documentation section on **Deploying Microcks with Default Options** for detailed Helm command parameters and common configurations (e.g., similar to Step 8 in the GKE example).

4.  **Verify Microcks Deployment (Default):**
    * Check the status of the Microcks and Postman Runtime pods.
    * Verify the Microcks Ingress resource and access the Microcks UI via your configured hostname. Log in using the user created in Keycloak.

---

### Phase 5: Deploy Microcks with Asynchronous Options (Optional)

If you require support for asynchronous protocols (like Kafka), follow these additional steps.

1.  **Enable SSL Passthrough on Ingress Controller (If applicable):**
    * Depending on your Ingress Controller and setup, you might need to enable SSL Passthrough for async protocols like Kafka/gRPC. Consult your Ingress Controller's documentation. A common way involves patching the Ingress Controller deployment.

2.  **Install Strimzi Operator for Kafka Support:**
    * Install the Strimzi Kafka Operator onto your Kubernetes cluster using its provided manifest or Helm chart. This operator manages Kafka clusters within Kubernetes.

3.  **Prepare Microcks Helm Values (for Async):**
    * Create a new Helm values file (e.g., `microcks-async-values.yaml`), starting from your default values file.
    * Add or modify the `features.async` section to enable asynchronous features (`features.async.enabled: true`).
    * Configure the Kafka settings (`features.async.kafka`). You can choose to have the Microcks chart deploy Kafka via Strimzi (`features.async.kafka.install: true`) or connect to an existing Kafka (`features.async.kafka.install: false` and configure connection details). If installing Kafka via the chart, configure the Kafka Ingress settings.

4.  **Deploy Microcks with Asynchronous Options:**
    * Deploy or upgrade your Microcks deployment using the Helm chart and your `microcks-async-values.yaml`.
    * Refer to the Microcks community documentation section on **Deploying Microcks with Asynchronous Options** for detailed Helm command parameters and configurations (e.g., similar to Step 9 in the GKE example).

5.  **Verify Microcks Deployment (Async):**
    * Check the status of the new pods (e.g., Async Minion, Kafka components like Zookeeper and Kafka brokers if installed by the chart, Strimzi operator).
    * Verify any new Ingress resources for Kafka.
    * Confirm asynchronous mocking/testing capabilities within the Microcks UI.

---

### Phase 6: Post-Deployment Considerations

* **Monitoring and Alerting:** Set up monitoring for your Kubernetes cluster, databases, Keycloak, and Microcks pods (CPU, memory, network, logs). Configure alerts for critical issues.
* **Backup and Restore:** Implement a strategy for backing up your managed databases (PostgreSQL and MongoDB) and potentially configuration data. Test your restore process.
* **Scaling:** Understand how to scale your Kubernetes cluster nodes, database instances, and Microcks pods based on load.
* **Security:** Regularly review security configurations for your cloud resources, Kubernetes cluster, and applications. Manage secrets securely.
* **Upgrades:** Plan for upgrading Microcks, Keycloak, database versions, and Kubernetes versions.
* **Community Contributions:** Share your cloud provider-specific deployment scripts and lessons learned with the Microcks community by contributing to the documentation or sharing in community channels.

---

This structure provides a clear path that any user can follow, regardless of their cloud provider, while explicitly pointing to where they need to fill in the cloud-specific implementation details or find detailed Microcks-specific configurations in separate, linked community documentation. It fulfills all your requirements, including the specific linking instructions.
