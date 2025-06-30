# Kubernetes Deployment and Secret Management Workflow

## 1. Build Docker Image and Push to Google Container Registry (GCR)

```bash
# Build Docker image locally
docker build -t gcr.io/YOUR_PROJECT_ID/example-image:v1 .

# Authenticate Docker to GCR
gcloud auth configure-docker

# Push image to GCR
docker push gcr.io/YOUR_PROJECT_ID/example-image:v1
```

> Replace `YOUR_PROJECT_ID` with your actual Google Cloud project ID.

## 2. Configure kubeconfig to Access GKE Cluster

```bash
gcloud container clusters get-credentials <CLUSTER_NAME> --zone <ZONE> --project <PROJECT_ID>
```

Example:

```bash
gcloud container clusters get-credentials example-cluster --zone us-central1-a --project my-gcp-project
```


## 3. Create `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
  labels:
    app: example
spec:
  replicas: 3
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
      - name: example-container
        image: gcr.io/YOUR_PROJECT_ID/example-image:v1
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets"
          readOnly: true
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "gcp-secrets"
```


## 4. Create `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
spec:
  selector:
    app: example
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
```


## 5. Create `SecretProviderClass` to Mount Google Secret Manager Secrets

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: gcp-secrets
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/YOUR_PROJECT_ID/secrets/secret-one"
        fileName: "secret-one"
      - resourceName: "projects/YOUR_PROJECT_ID/secrets/secret-two"
        fileName: "secret-two"
```

> Replace `YOUR_PROJECT_ID` and secret names accordingly.

## 6. Apply Manifests in Correct Order

```bash
# Apply SecretProviderClass first
kubectl apply -f secretproviderclass.yaml

# Apply Service next
kubectl apply -f service.yaml

# Apply Deployment last
kubectl apply -f deployment.yaml
```


## 7. Verify Pods and Service

```bash
kubectl get pods
kubectl get service example-service
```

- Wait for external IP to be assigned to the LoadBalancer service.
- Access your app via the external IP on port 80.


## 8. Debug Inside a Running Pod

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

- Replace `<pod-name>` with the actual pod name from `kubectl get pods`.
- This opens an interactive shell inside the container for troubleshooting.


## 9. Removing Secrets from Deployment and Cluster

### Step 1: Remove volumes and volumeMounts referencing secrets from `deployment.yaml`

Remove these sections from your `deployment.yaml`:

```yaml
volumeMounts:
- name: secrets-store-inline
  mountPath: "/mnt/secrets"
  readOnly: true
```

and

```yaml
volumes:
- name: secrets-store-inline
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: "gcp-secrets"
```


### Step 2: Apply updated Deployment

```bash
kubectl apply -f deployment.yaml
```


### Step 3: Delete Kubernetes Secret objects (if synced) or delete secrets from Google Secret Manager

```bash
kubectl delete secret secret-one
kubectl delete secret secret-two
```

> If secrets reside only in Google Secret Manager, delete them there via the GCP Console or `gcloud` CLI.

# Notes

- The `volumeMounts.mountPath` controls **where inside the container** the secrets appear (you can customize this path).
- The `SecretProviderClass` controls **which secrets and filenames** are fetched from the external secret store.
- One `SecretProviderClass` can fetch multiple secrets, all mounted together in one volume.
- Pods are managed by Deployments; do **not** create Pods manually in production.
- Always apply the Service before the Deployment for smooth startup and service discovery.
- Use `kubectl exec` to debug containers interactively.
- Removing secrets from Pods requires both detaching volumes and deleting the Secret objects for full cleanup.
