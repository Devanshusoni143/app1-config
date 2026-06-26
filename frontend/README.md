# ArgoCD Sync Waves Demo — Minikube

A complete working example to observe ArgoCD sync wave behavior locally.

## Directory Structure

```
argocd-sync-waves/
├── argocd-application.yaml       # ArgoCD Application CR — apply this to argocd namespace
├── wave-minus1/
│   ├── namespace.yaml            # wave: -1  — Namespace
│   └── rbac.yaml                 # wave: -1  — ServiceAccount, Role, RoleBinding
├── wave-0/
│   ├── secret.yaml               # wave:  0  — App secrets
│   └── configmap.yaml            # wave:  0  — App config
├── wave-1/
│   ├── postgres-statefulset.yaml # wave:  1  — PostgreSQL StatefulSet (readinessProbe gates wave 2)
│   ├── postgres-service.yaml     # wave:  1  — Headless Service for StatefulSet
│   ├── redis-deployment.yaml     # wave:  1  — Redis cache
│   └── redis-service.yaml        # wave:  1  — Redis Service
├── wave-2/
│   └── db-migration-job.yaml     # wave:  2  — Migration Job (must complete before wave 3)
├── wave-3/
│   ├── backend-deployment.yaml   # wave:  3  — Backend API (httpbin as placeholder)
│   └── backend-service.yaml      # wave:  3  — Backend Service
├── wave-4/
│   ├── frontend-deployment.yaml  # wave:  4  — Frontend (nginx as placeholder)
│   └── frontend-service.yaml     # wave:  4  — Frontend Service
└── wave-6/
    └── ingress.yaml              # wave:  6  — Ingress (nginx, no TLS)
```

> Wave 5 (ServiceMonitor) is intentionally skipped — requires Prometheus Operator CRD.

---

## One-Time Minikube Setup

```bash
# 1. Start minikube with enough resources
minikube start --cpus=4 --memory=4096

# 2. Enable ingress addon
minikube addons enable ingress

# 3. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 4. Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=180s

# 5. Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo

# 6. Open ArgoCD UI (run in a separate terminal, keep it open)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 7. Add hosts entry so ingress hostnames resolve
echo "$(minikube ip) api.myapp.local app.myapp.local" | sudo tee -a /etc/hosts
```

---

## Deploy the Demo

```bash
# 1. Push this folder to a GitHub repo, then update repoURL in argocd-application.yaml

# 2. Apply the ArgoCD Application
kubectl apply -f argocd-application.yaml

# 3. Watch the sync happen wave by wave
argocd login localhost:8080 --username admin --password <your-password> --insecure
argocd app get sync-wave-demo --refresh
argocd app resources sync-wave-demo
```

Or just watch it in the ArgoCD UI at https://localhost:8080

---

## Test Wave Gating (The Important Part)

### Test 1 — Break the migration job, verify backend never deploys

Edit `wave-2/db-migration-job.yaml`, change the command to:
```yaml
command: ["sh", "-c", "echo 'Migration failed'; exit 1"]
```
Also bump the job name to `db-migration-v1-0-1` (job names must be unique).

Push and sync. ArgoCD will stop at wave 2. Backend stays undeployed. That's wave gating working correctly.

### Test 2 — Watch live in terminal

```bash
# Terminal 1 — watch all resources
kubectl get all -n production -w

# Terminal 2 — watch ArgoCD sync status
watch -n 2 'argocd app get sync-wave-demo'
```

### Test 3 — Verify ingress is last

After a full successful sync:
```bash
curl http://api.myapp.local/get      # should return httpbin JSON response
curl http://app.myapp.local/         # should return nginx default page
```

---

## Wave Summary

| Wave | Resources | ArgoCD waits for |
|------|-----------|-----------------|
| -1   | Namespace, ServiceAccount, Role, RoleBinding | Resources to exist |
|  0   | Secret, ConfigMap | Resources to exist |
|  1   | Postgres StatefulSet + Service, Redis Deployment + Service | readinessProbe to pass on all pods |
|  2   | DB Migration Job | Job to complete with exit 0 |
|  3   | Backend Deployment + Service | Deployment to reach availableReplicas |
|  4   | Frontend Deployment + Service | Deployment to reach availableReplicas |
|  6   | Ingress | Resource to exist |

---

## Switching to Real Images

When you're ready to use real app images, change these two files:

**wave-3/backend-deployment.yaml**
```yaml
image: kennethreitz/httpbin  ->  image: your-registry/your-backend:tag
readinessProbe path: /get    ->  readinessProbe path: /health
```

**wave-4/frontend-deployment.yaml**
```yaml
image: nginx:alpine          ->  image: your-registry/your-frontend:tag
```

**wave-2/db-migration-job.yaml**
```yaml
# Replace busybox dummy with your real migration command
command: ["node", "scripts/migrate.js"]       # Node.js
command: ["python", "manage.py", "migrate"]   # Django
command: ["flyway", "migrate"]                # Flyway
```
