# Typesense Kubernetes Deployment

This repository contains **Kubernetes manifests and a GitHub Actions workflow**
for deploying **Typesense** into a Kubernetes cluster.

The setup is designed to be:
- ✅ **Production-ready**
- ✅ **Safe for redeploys**
- ✅ **Compatible with Ceph RBD (ReadWriteOnce)**
- ✅ **Fully automated via GitHub Actions**
- ✅ **TLS-enabled using cert-manager**

---

## 📦 What is deployed

- Typesense server (`typesense/typesense` Docker image)
- PersistentVolumeClaim backed by **Ceph RBD**
- Kubernetes Deployment (single replica, `Recreate` strategy)
- ClusterIP Service
- NGINX Ingress with TLS
- Kubernetes Secret containing the **ADMIN API key**

---

## 📁 Repository structure

```text
.
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions deployment workflow
├── k8s/
│   ├── 00-namespace.yaml       # Namespace (typesense)
│   ├── 01-pvc-typesense.yaml   # PVC (Ceph RBD)
│   ├── 02-secret-typesense.yaml # Secret (TYPESENSE_API_KEY)
│   ├── 03-deployment-typesense.yaml
│   ├── 04-service-typesense.yaml
│   └── 05-ingress-typesense.yaml
└── README.md
```


---

## 🧠 Design decisions (IMPORTANT)

### Why **Ceph RBD** and not CephFS?
- Typesense **does not support shared data directories**
- RWX (CephFS) causes file locking errors such as:

```
/data/db/LOCK: Resource temporarily unavailable
```

- Ceph RBD (RWO) guarantees a **single writer**, which is required by Typesense

### Why `replicas: 1` and `strategy: Recreate`?
- Typesense uses filesystem locking
- RollingUpdate could start two pods using the same PVC
- `Recreate` guarantees that **only one Typesense process exists at any time**

---

## 🔐 Secrets and variables

### GitHub Secrets (required)

| Name | Description |
|----|----|
| `C2_KUBE_CONFIG` | Base64-encoded kubeconfig |

### GitHub Variables (optional)

| Name | Default |
|----|----|
| `KUBE_NAMESPACE` | `typesense` |

---

## 🚀 Deployment process

### Automatic deployment (recommended)

Deployment is triggered by:
- ✅ every push to the `main` branch
- ✅ manual execution via **GitHub Actions → Run workflow**

The workflow:
1. loads the kubeconfig
2. applies Kubernetes manifests (03–05)
3. waits for the Deployment rollout to complete

---

### Manual deployment (for initial setup or debugging)

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-pvc-typesense.yaml
kubectl apply -f k8s/02-secret-typesense.yaml
```

# (perform manual data migration here if needed)

```bash
kubectl apply -f k8s/03-deployment-typesense.yaml
kubectl apply -f k8s/04-service-typesense.yaml
kubectl apply -f k8s/05-ingress-typesense.yaml
```

## 🌐 Ingress & TLS

- Host: `https://typesense.*`
- TLS: `cert-manager + letsencrypt-prod`
- TLS secret: `typesense-*-tls`

The Ingress is tuned for:

- long-running search requests
- bulk indexing
- large responses


## ❤️ Health checks
The deployment uses:

- startupProbe → tolerant of long startup times (large datasets)
- readinessProbe → /health
- livenessProbe → /health

This prevents:

- restart loops during cold start
- routing traffic before Typesense is ready

## 🔑 API keys – CRITICAL NOTE
Typesense supports two types of API keys:

✅ ADMIN API key
Used for:

- creating and deleting collections
- importing documents
- schema changes

⚠️ Must never be exposed to the frontend

✅ SCOPED API key

- Generated from the admin key
- Search-only
- Intended for browser / frontend usage

❌ Using a scoped key for write operations will result in:

```
Malformed scoped API key
Scoped API keys can only be used for searches
```

## 🛠️ Operations & maintenance
Upgrade Typesense version

1. Update the image tag in 03-deployment-typesense.yaml
2. Push to main
3. GitHub Actions performs the redeploy

Restart without data loss

```bash
kubectl rollout restart deploy/typesense -n typesense
```

## ⚙️ Resource Management

Typesense is **memory-intensive** and relies heavily on available RAM to load and query indexes efficiently.

### 🧠 Memory

- Typesense keeps indexes in memory (via mmap and caching).
- Required memory typically scales with dataset size:
  - ~1.2–1.5× index size
- Insufficient memory will lead to degraded performance or restarts.

**Example configuration:**

```yaml
resources:
  requests:
    memory: "10Gi"
  limits:
    memory: "12Gi"
```


Check status

```bash
kubectl get pods -n typesense
kubectl logs deploy/typesense -n typesense
curl https://domain/health
```

✅ Status

✅ production ready
✅ safe for redeploys
✅ no data loss
✅ compatible with Ceph RBD

📌 References

- https://typesense.org/docs/
- https://kubernetes.io/docs/
- https://cert-manager.io/
