# Deploying Microcks on OVH Managed Kubernetes Service

## Overview

This guide provides a comprehensive, step-by-step process for deploying Microcks on OVHcloud Managed Kubernetes Service. It leverages an external PostgreSQL database (hosted on OVH or elsewhere), a Helm-deployed MongoDB for Microcks storage, and Keycloak for authentication. The goal is to establish a secure, scalable, and production-ready Microcks environment on OVH.

## Prerequisites

### Ensure the following tools are installed and configured:
- OVHcloud CLI (ovh): For managing OVH services.
- kubectl: For managing Kubernetes resources.
- Helm: To install Microcks and MongoDB via Helm charts.
- Docker (optional): For local development and testing.


## 1. Provision OVH Managed Kubernetes Cluster

### a. Log in to OVHcloud CLI

```sh
$ ovh login
```

### b. Create Kubernetes Cluster

Create a Kubernetes cluster via the OVHcloud dashboard or using the CLI (recommended using the dashboard for region and node selection).
Once provisioned, download the kubeconfig file and configure access:

```sh
$ export KUBECONFIG=/path/to/ovh-kubeconfig.yaml
$ kubectl config use-context <context-name>
```

## 2. Deploy External PostgreSQL (on OVH or Other Cloud)

Use OVHcloud’s managed PostgreSQL service or any external PostgreSQL provider.

Ensure you create a user, database, and configure remote access.

Note down host, port, username, password, and DB name.

## 3. Deploy MongoDB via Helm

### a. Add Bitnami Helm Repository

```sh
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo update
```

### b. Install MongoDB

```sh
$ helm install mongodb bitnami/mongodb -n microcks --create-namespace \
  --set architecture=standalone \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set resources.requests.cpu=500m \
  --set resources.requests.memory=1Gi \
  --set resources.limits.cpu=1 \
  --set resources.limits.memory=2Gi \
  --set auth.enabled=true \
  --set auth.rootPassword=<ROOT_PASSWORD> \
  --set auth.username=<USERNAME> \
  --set auth.password=<PASSWORD> \
  --set auth.database=microcks
```

### c. Create MongoDB Secret

```sh
$ kubectl create secret generic microcks-mongodb-connection -n microcks \
  --from-literal=username=<USERNAME> \
  --from-literal=password=<PASSWORD>
```

## 4. Deploy Keycloak (Optional or External)

You can deploy Keycloak manually on the OVH Kubernetes cluster or use an existing Keycloak instance.

Ensure it connects to the PostgreSQL DB created earlier.

Set up realm, client, and user for Microcks.

## 5. Configure Helm Values for Microcks

## a. Add Microcks Helm Repository

```sh
$ helm repo add microcks https://microcks.io/helm
$ helm repo update
```

### b. Create Configuration File

```ah
$ cat << microcks.yaml
appName: microcks
ingresses: true

microcks:
  url: microcks.<YOUR-DOMAIN>.ovh
  ingressClassName: nginx
  ingressAnnotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

  grpcEnableTLS: true
  grpcIngressClassName: nginx
  grpcIngressAnnotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"

  env:
    - name: SPRING_DATA_MONGODB_URI
      value: "mongodb://<USERNAME>:<PASSWORD>@mongodb.microcks.svc.cluster.local:27017/microcks?authSource=microcks"
    - name: CORS_REST_ALLOWED_ORIGINS
      value: "https://keycloak.<YOUR-DOMAIN>.ovh"
    - name: CORS_REST_ALLOW_CREDENTIALS
      value: "true"

keycloak:
  enabled: true
  install: false
  url: keycloak.<YOUR-DOMAIN>.ovh
  privateUrl: https://keycloak.<YOUR-DOMAIN>.ovh
  realm: microcks
  client:
    id: <YOUR-CLIENT-ID>
    secret: <CLIENT-SECRET>

mongodb:
  install: false
  database: microcks
  secretRef:
    secret: microcks-mongodb-connection
    usernameKey: username
    passwordKey: password

ingress:
  enabled: true
  tls: true
EOF
```

## 6. Deploy Microcks

```sh
$ helm install microcks microcks/microcks -n microcks -f microcks.yaml
```

## 7. Verify Deployment

```sh
$ kubectl get pods -n microcks
$ kubectl get ingress -n microcks
```

Example output:
NAME                        READY   STATUS    RESTARTS   AGE
microcks-xxx                1/1     Running   0          2m
mongodb-xxx                 1/1     Running   0          2m

Access Microcks at:
https://microcks.<YOUR-DOMAIN>.ovh

