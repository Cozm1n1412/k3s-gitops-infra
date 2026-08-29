# Production-Grade GitOps Infrastructure on k3s

An automated, declarative Kubernetes infrastructure engineered from scratch on lightweight **k3s**, managed entirely via **ArgoCD (App-of-Apps pattern)** and secured with **SealedSecrets** and **Traefik automated TLS**.

---

## 🏛 Architecture Overview

The cluster follows a strict separation of concerns across dedicated namespaces:

```text
k3s-gitops-infra/
├── argocd-apps/             # Root ArgoCD App-of-Apps Declarations
│   ├── root-apps.yaml       # Master Application controller
│   ├── backend-api.yaml     # Application layer spec
│   └── infra-database.yaml  # State & persistence spec
├── apps/                    # Workload Layer (Namespace: app)
│   └── backend/             # FastAPI REST Service (2 Replicas, HPA-ready)
│       ├── deployment.yaml  # Health Probes (liveness/readiness), Resource Limits
│       ├── service.yaml     # ClusterIP Service abstraction
│       └── ingress.yaml     # Traefik L7 Ingress with automated TLS
└── infra/                   # Infrastructure Layer (Namespace: database, monitoring)
    └── database/
        ├── postgres.yaml    # PostgreSQL StatefulSet + PVC (Local Path Provisioner)
        ├── redis.yaml       # Redis In-Memory Cache (Deployment + Service)
        └── postgres-sealed-secret.yaml # Asymmetric encrypted at-rest credentials
Core Engineering Pillars
1. Declarative Control & Self-Healing (ArgoCD)

    App-of-Apps Pattern: A single root-apps controller orchestrates the deployment and synchronization of microservices and infrastructure.

    Server-Side Apply (SSA): Field management is delegated to the Kubernetes API Server, preventing CRD update locks and resourceVersion: 0 conflicts.

    Continuous Reconciliation & Anti-Drift: Automatic pruning and self-healing enforce Git as the absolute single source of truth. Manual kubectl mutations are immediately overwritten.
2. Zero-Trust Secrets Management (Bitnami SealedSecrets)

    Asymmetric encryption at rest: Public key encrypts credentials inside Git commits; private key decrypts secrets in-memory inside the cluster.

    Eliminates hardcoded environment variables and secrets leakage across the version control lifecycle.
3. Edge Routing & Ingress Automation

    Integrated Traefik Ingress Controller terminating TLS.

    Declarative routing directly to upstream backend endpoints with automated SSL lifecycle management.
4. Stateful Reliability & Observability Ready

    Dedicated StatefulSet for PostgreSQL ensuring deterministic network identity and persistent volume binding via local storage provisioner.

    In-memory Redis caching tier decoupled from the application runtime.

    Standardized labels across all workloads for integration with Prometheus Operator / Grafana dashboards.
Bootstrap & Disaster Recovery

Rebuilding the entire cluster infrastructure from scratch requires only two commands:
# 1. Install SealedSecrets controller
kubectl apply -f [https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.3/controller.yaml](https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.3/controller.yaml)

# 2. Bootstrap the entire stack via Root Application
kubectl apply -f argocd-apps/root-apps.yaml
