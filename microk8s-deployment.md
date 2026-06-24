# Deploying ODIN Data Catalog on MicroK8s

This guide walks through installing and running the ODIN Data Catalog Helm chart on a local MicroK8s cluster.

## Prerequisites

| Requirement | Minimum | Notes |
|-------------|---------|-------|
| OS | Ubuntu 22.04 LTS | MicroK8s snap available on other distros too |
| CPU | 8 cores | OpenSearch + Kafka are resource-heavy |
| RAM | 16 GB | ~10 GB used at steady state |
| Disk | 200 GB free | PVCs total ~150 GB |
| Helm | 3.14+ | Or use `microk8s helm3` (bundled) |

> **Apple Silicon / Windows**: Run MicroK8s inside Multipass (`multipass launch --cpus 8 --memory 16G --disk 200G`).

---

## 1. Install MicroK8s

```bash
sudo snap install microk8s --classic --channel=1.30/stable
sudo usermod -aG microk8s $USER
newgrp microk8s          # apply group without logout
microk8s status --wait-ready
```

### Enable required add-ons

```bash
microk8s enable dns storage ingress registry
```

| Add-on | Why |
|--------|-----|
| `dns` | In-cluster service discovery |
| `storage` | `hostpath` StorageClass for PVCs |
| `ingress` | NGINX Ingress Controller |
| `registry` | Local Docker registry at `localhost:32000` |

Wait for all add-ons to become ready:

```bash
microk8s status --wait-ready
```

### Optional: set up shell aliases

```bash
echo 'alias kubectl="microk8s kubectl"' >> ~/.bashrc
echo 'alias helm="microk8s helm3"'      >> ~/.bashrc
source ~/.bashrc
```

All `kubectl` and `helm` commands below assume these aliases are in place. If you are using system-installed tools, replace `microk8s kubectl` / `microk8s helm3` accordingly.

---

## 2. Allow privileged init containers (OpenSearch requirement)

OpenSearch requires `vm.max_map_count=262144`. The chart sets this via a privileged `busybox` init container. MicroK8s allows privileged containers by default, but confirm the setting is not blocked:

```bash
microk8s kubectl get psp 2>/dev/null || echo "PSP not enabled — privileged containers allowed"
```

If Pod Security Admission is enforcing `restricted`, label the namespace to allow it (done automatically in step 4).

---

## 3. Build and push service images

The chart references images under `odin/` (e.g. `odin/inventory-service:latest`). You must build these locally and push them to the MicroK8s built-in registry before installing.

### Build all services

From the repo root:

```bash
# Backend services (requires Java 21 + Gradle)
./gradlew :services:inventory-service:bootBuildImage   --imageName=localhost:32000/odin/inventory-service:latest
./gradlew :services:harvest-service:bootBuildImage   --imageName=localhost:32000/odin/harvest-service:latest
./gradlew :services:lineage-service:bootBuildImage   --imageName=localhost:32000/odin/lineage-service:latest
./gradlew :services:search-service:bootBuildImage    --imageName=localhost:32000/odin/search-service:latest
./gradlew :services:ai-service:bootBuildImage        --imageName=localhost:32000/odin/ai-service:latest
./gradlew :services:identity-service:bootBuildImage  --imageName=localhost:32000/odin/identity-service:latest

# Frontend apps (requires Node 20+)
cd frontend && npm ci
npm run build --workspace=consumer
npm run build --workspace=producer
cd ..

# Build frontend Docker images
docker build -t localhost:32000/odin/frontend-consumer:latest -f frontend/consumer/Dockerfile frontend
docker build -t localhost:32000/odin/frontend-producer:latest -f frontend/producer/Dockerfile frontend
```

### Push to the local registry

```bash
docker push localhost:32000/odin/inventory-service:latest
docker push localhost:32000/odin/harvest-service:latest
docker push localhost:32000/odin/lineage-service:latest
docker push localhost:32000/odin/search-service:latest
docker push localhost:32000/odin/ai-service:latest
docker push localhost:32000/odin/identity-service:latest
docker push localhost:32000/odin/frontend-consumer:latest
docker push localhost:32000/odin/frontend-producer:latest
```

> The MicroK8s registry listens on `localhost:32000` and is already trusted by the cluster — no `imagePullSecret` needed.

---

## 4. Create the namespace

```bash
kubectl create namespace odin-catalog

# Allow privileged init containers (needed for OpenSearch sysctl)
kubectl label namespace odin-catalog \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged
```

---

## 5. Install the Helm chart

```bash
helm install odin infra/helm/charts/odin-catalog \
  --namespace odin-catalog \
  --set global.imageRegistry=localhost:32000 \
  --set ingress.className=nginx \
  --timeout 10m \
  --wait
```

`--wait` blocks until all Deployments and StatefulSets report `Ready`. The first install takes several minutes while PVCs are provisioned and images are pulled.

### What gets deployed

| Component | Kind | Count |
|-----------|------|-------|
| Backend services | Deployment | 6 |
| Frontend apps | Deployment | 2 |
| PostgreSQL (per service) | StatefulSet | 5 |
| Kafka (KRaft) | StatefulSet | 1 |
| OpenSearch | StatefulSet | 1 |
| MinIO | StatefulSet | 1 |
| Redis | StatefulSet | 1 |
| Keycloak | Deployment | 1 |
| Ingress rules | Ingress | 3 |
| PersistentVolumeClaims | PVC | 9 |

---

## 6. Configure local DNS

The three ingress hosts must resolve to the MicroK8s node IP. Get the IP:

```bash
microk8s kubectl get nodes -o wide
# Note the INTERNAL-IP column, e.g. 192.168.64.5
```

Add entries to `/etc/hosts` (replace `<NODE-IP>`):

```bash
sudo tee -a /etc/hosts <<EOF
<NODE-IP>  catalog.local
<NODE-IP>  manage.catalog.local
<NODE-IP>  api.catalog.local
EOF
```

If running MicroK8s inside Multipass, use the Multipass VM IP from `multipass info <vm-name>`.

---

## 7. Verify the deployment

### Check pod status

```bash
kubectl -n odin-catalog get pods -o wide
```

All pods should reach `Running` or `Completed` (for init Jobs). Expected output:

```
NAME                                        READY   STATUS      ...
odin-inventory-service-...                  1/1     Running
odin-harvest-service-...                    1/1     Running
odin-lineage-service-...                    1/1     Running
odin-search-service-...                     1/1     Running
odin-ai-service-...                         1/1     Running
odin-identity-service-...                   1/1     Running
odin-frontend-consumer-...                  1/1     Running
odin-frontend-producer-...                  1/1     Running
odin-postgres-inventory-0                   1/1     Running
odin-postgres-harvest-0                     1/1     Running
odin-postgres-lineage-0                     1/1     Running
odin-postgres-identity-0                    1/1     Running
odin-postgres-ai-0                          1/1     Running
odin-kafka-0                                1/1     Running
odin-opensearch-0                           1/1     Running
odin-minio-0                                1/1     Running
odin-redis-0                                1/1     Running
odin-keycloak-...                           1/1     Running
odin-kafka-init-...                         0/1     Completed
odin-opensearch-init-...                    0/1     Completed
```

### Check ingress

```bash
kubectl -n odin-catalog get ingress
```

### Smoke test the endpoints

```bash
# Consumer frontend
curl -s -o /dev/null -w "%{http_code}" http://catalog.local/

# Management frontend
curl -s -o /dev/null -w "%{http_code}" http://manage.catalog.local/

# API health (via ingress path prefix)
curl -s http://api.catalog.local/inventory/actuator/health
curl -s http://api.catalog.local/search/actuator/health
```

---

## 8. Access the UIs

| URL | Interface |
|-----|-----------|
| `http://catalog.local` | Consumer / discovery app |
| `http://manage.catalog.local` | Producer / management app |
| `http://api.catalog.local/inventory/swagger-ui.html` | Inventory service Swagger UI |
| `http://api.catalog.local/search/swagger-ui.html` | Search service Swagger UI |

Keycloak admin console — via the ingress host (add `keycloak.catalog.local` to
`/etc/hosts` as printed by `deploy.sh`):

```
http://keycloak.catalog.local/admin   —  user: admin / pass: admin
```

Or directly via port-forward:

```bash
kubectl -n odin-catalog port-forward svc/odin-catalog-keycloak 8180:8180
# Open http://localhost:8180  —  user: admin / pass: admin
```

### Adding a user manually

New users need two custom attributes set so the backend can authorize them:
`tenant_id` and `permissions` (e.g. `catalog:read`, `catalog:write`,
`catalog:admin`). Set them via the **Keycloak Admin Console**:

1. Open `http://keycloak.catalog.local/admin` → select the **datacatalog** realm.
2. **Users → Add user** (or select an existing one). Set username, email, and on
   the **Credentials** tab set a password.
3. Go to the **Attributes** tab and add:
   - `tenant_id` → the tenant UUID (e.g. `00000000-0000-0000-0000-000000000001`)
   - `permissions` → one row per grant: `catalog:read`, `catalog:write`,
     `catalog:admin`

The realm sets `unmanagedAttributePolicy: ENABLED`, so these unmanaged attributes
always save correctly. This policy is part of the realm import, so it applies
automatically to any fresh deployment. Attributes can equivalently be set through
the Keycloak Admin REST API.

---

## 9. Upgrade the chart

After changing values or rebuilding images:

```bash
# Rebuild and push a specific image, then:
helm upgrade odin infra/helm/charts/odin-catalog \
  --namespace odin-catalog \
  --set global.imageRegistry=localhost:32000 \
  --timeout 10m \
  --wait
```

Force a rolling restart to pull updated images:

```bash
kubectl -n odin-catalog rollout restart deployment/odin-inventory-service
```

---

## 10. Reduce resource requests for constrained machines

If you have less than 16 GB RAM, override resource requests:

```bash
helm upgrade odin infra/helm/charts/odin-catalog \
  --namespace odin-catalog \
  --set global.imageRegistry=localhost:32000 \
  --set resources.backend.requests.memory=256Mi \
  --set resources.backend.limits.memory=512Mi \
  --set resources.opensearch.requests.memory=512Mi \
  --set resources.opensearch.limits.memory=1Gi \
  --set opensearch.javaOpts="-Xms256m -Xmx512m" \
  --set kafka.heapOpts="-Xmx256m -Xms256m" \
  --timeout 10m \
  --wait
```

---

## 11. Uninstall

```bash
helm uninstall odin --namespace odin-catalog

# Remove PVCs (data is lost)
kubectl delete namespace odin-catalog
```

---

## Troubleshooting

### Pod stuck in `Pending`

```bash
kubectl -n odin-catalog describe pod <pod-name>
```

Common causes:

| Symptom | Fix |
|---------|-----|
| `0/1 nodes are available: ... Insufficient memory` | Reduce resource requests (see step 10) |
| `no persistent volumes available` | Ensure `storage` add-on is enabled: `microk8s enable storage` |
| `Unschedulable: ... disk pressure` | Free disk space; MicroK8s evicts when disk > 85% |

### OpenSearch pod in `Init:Error`

The `sysctl` init container requires `privileged: true`. If it fails:

```bash
kubectl -n odin-catalog logs <opensearch-pod> -c sysctl
```

Confirm the namespace is labelled for privileged pods (step 4). Alternatively, set `vm.max_map_count` on the host and disable the init container:

```bash
# On the host (persists until reboot)
sudo sysctl -w vm.max_map_count=262144

# Permanent (survives reboot)
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.d/99-opensearch.conf
sudo sysctl -p /etc/sysctl.d/99-opensearch.conf
```

### Kafka topics not created

The `kafka-init` Job runs after Kafka is ready. Check its logs:

```bash
kubectl -n odin-catalog logs job/odin-kafka-init
```

If the Job completed but topics are missing, re-run it:

```bash
kubectl -n odin-catalog delete job odin-kafka-init
helm upgrade odin infra/helm/charts/odin-catalog --namespace odin-catalog --reuse-values
```

### Image pull errors (`ImagePullBackOff`)

Images must be pushed to `localhost:32000` before install. Verify an image is accessible to the cluster:

```bash
microk8s ctr images list | grep odin
```

If the image is missing, push it again (see step 3) and then restart the affected deployment:

```bash
kubectl -n odin-catalog rollout restart deployment/odin-inventory-service
```

### Services cannot reach each other

Confirm CoreDNS is running:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

If pods are crashing, restart CoreDNS:

```bash
kubectl -n kube-system rollout restart deployment/coredns
```

### View logs for a service

```bash
# Live log tail
kubectl -n odin-catalog logs -f deployment/odin-inventory-service

# Previous (crashed) container
kubectl -n odin-catalog logs deployment/odin-inventory-service --previous
```
