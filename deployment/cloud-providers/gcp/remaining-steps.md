## Step 5: Create the `values.yaml` File

The `values.yaml` file contains configuration details for your Microcks deployment. This file will include the connection details for Cloud SQL, Firestore, and external Keycloak.

Here’s an example `values.yaml`:

```yaml
microcks:
  db:
    external: true
    uri: "jdbc:postgresql://<CLOUD_SQL_CONNECTION_NAME>:5432/microcks?user=microcks-user&password=strongpassword"
    
  mongodb:
    external: true
    uri: "firestore://your-project-id/microcks"

  keycloak:
    external: true
    url: "https://your-external-keycloak-server.com"
    realm: "microcks-realm"
    clientId: "microcks-client-id"
    clientSecret: "your-client-secret"

replicaCount: 3
```

## Step 6: Deploy Microcks Using Helm

Deploy Microcks to your GKE cluster using Helm:

```bash
helm install microcks microcks/microcks -f values.yaml --namespace microcks-prod --create-namespace
```

---

## Step 7: Expose Microcks with Ingress

To expose Microcks externally, create and apply an Ingress resource.

### Example `microcks-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microcks-ingress
  namespace: microcks-prod
spec:
  rules:
    - host: microcks.yourdomain.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: microcks-service
              port:
                number: 8080
```

### Apply the Ingress Resource

```bash
kubectl apply -f microcks-ingress.yaml
```

---

## Step 8: Monitor the Deployment

### Check the Status of the Deployment

```bash
kubectl get pods -n microcks-prod
```


### Scale the Deployment if Needed

```bash
kubectl scale deployment microcks --replicas=5 -n microcks-prod

```

## Step 9: Accessing the Microcks Application

Once the deployment is successful and the Ingress resource is applied, you can access the Microcks application using the configured domain name.

### Retrieve the External IP Address
Run the following command to get the external IP of your Ingress controller:

```bash
kubectl get ingress microcks-ingress -n microcks-prod
```

This will return output similar to:

```plaintext
NAME                CLASS    HOSTS                     ADDRESS        PORTS   AGE
microcks-ingress    nginx    microcks.yourdomain.com   35.123.45.67   80      10m
```

If your ingress does not have an external IP, check if your ingress controller is properly set up.

### Access via Browser
Open a web browser and go to:

```
http://microcks.yourdomain.com
```
