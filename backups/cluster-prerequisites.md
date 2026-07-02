# Data Cluster Prerequisites & Setup Guide
# Backed up from: do-data-prod (DigitalOcean) on 2026-03-24

## Helm Releases Installed

### 1. ingress-nginx
- Chart: ingress-nginx-4.15.1
- App Version: 1.15.1
- Namespace: ingress-nginx
- Install:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --version 4.15.1
```

### 2. cert-manager
- Chart: cert-manager-v1.20.0
- App Version: v1.20.0
- Namespace: cert-manager
- Install:
```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.20.0 \
  --set crds.enabled=true
```

### 3. ClusterIssuer (letsencrypt-prod)
Apply after cert-manager is ready:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: devops@fitness22.com
    privateKeySecretRef:
      name: letsencrypt-prod
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: nginx
```

### 4. Airbyte
- Chart: airbyte-2.0.19
- App Version: 2.0.1
- Namespace: airbyte-v2
- Install:
```bash
kubectl create namespace airbyte-v2

# Recreate secrets first (from backed up YAML files):
kubectl apply -f airbyte-auth-secrets.yaml
kubectl apply -f airbyte-secrets.yaml

# Install Airbyte
helm repo add airbyte https://airbytehq.github.io/helm-charts
helm install airbyte airbyte/airbyte \
  --namespace airbyte-v2 \
  --version 2.0.19 \
  -f ../values.yaml
```

### 5. Restore Database (optional - restores connections/sync history)
```bash
# Get the new airbyte-db pod name
kubectl cp airbyte-db-backup.sql airbyte-v2/<airbyte-db-pod>:/tmp/backup.sql
kubectl exec -n airbyte-v2 <airbyte-db-pod> -- psql -U airbyte -f /tmp/backup.sql
```

## DNS
- `airbyte.fitness22services.com` -> Update to point to new cluster's ingress external IP

## Namespaces
- airbyte-v2
- cert-manager
- ingress-nginx
