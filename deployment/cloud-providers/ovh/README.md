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

