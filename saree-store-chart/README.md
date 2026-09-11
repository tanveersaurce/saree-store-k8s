# Saree Store — End-to-End DevOps CI/CD & GitOps Pipeline

A production-style DevOps pipeline built for a full-stack e-commerce application (React frontend, Node.js/Express backend, MongoDB), covering the complete journey from code commit to a self-healing, monitored Kubernetes deployment.

---

## Project Overview

This project implements a **fully automated CI/CD + GitOps pipeline** for a multi-service e-commerce application, deployed on Kubernetes and managed through Infrastructure-as-Code (Helm) and continuous delivery (ArgoCD). It includes security scanning, code quality gates, automated rollback capability, and real-time observability.

**Tech Stack:** Kubernetes (Minikube) · Helm · ArgoCD · Jenkins · SonarQube · Trivy · Docker · Prometheus · Grafana · MongoDB · Node.js · React · Nginx

---

## Architecture

```
Developer Push (GitHub - App Repo)
        │
        ▼
   Jenkins Pipeline
   ├── SonarQube (code quality analysis)
   ├── Docker Build (backend + frontend images)
   ├── Trivy (container vulnerability scanning)
   ├── Docker Hub (image registry push)
   └── Auto-update Helm values.yaml → Push to GitOps Repo
        │
        ▼
   ArgoCD (continuous monitoring of Git state)
   ├── Detects change automatically (Auto-Sync)
   ├── Applies Helm chart to Kubernetes cluster
   └── Self-Heals any manual drift
        │
        ▼
   Kubernetes Cluster (Minikube)
   ├── Frontend (Nginx + React) — 2 replicas
   ├── Backend (Node.js/Express) — 2 replicas
   ├── MongoDB (StatefulSet + persistent storage)
   ├── Ingress-NGINX (routing /api → backend, / → frontend)
   └── Prometheus + Grafana (real-time monitoring)
```

---

## Key Features Implemented

### 1. Containerization & Orchestration
- Dockerized frontend, backend, and database services
- Custom **Helm chart** with reusable templates for Deployments, Services, StatefulSets, Ingress, Secrets, and ConfigMaps
- Environment-specific configuration via `values.yaml`

### 2. Continuous Integration (Jenkins)
- Automated pipeline triggered on code push
- **SonarQube** integration for static code quality analysis
- **Trivy** vulnerability scanning on all container images before publishing
- Automated Docker image build, tag, and push to Docker Hub

### 3. GitOps Continuous Delivery (ArgoCD)
- Jenkins pipeline automatically updates Helm chart `values.yaml` with new image tags and pushes to a dedicated GitOps repository
- ArgoCD continuously watches the repo and auto-syncs the cluster to match the desired state
- **Self-Heal** enabled — any manual, out-of-band cluster change is automatically reverted to match Git
- Tested rollback capability via both Git revert and ArgoCD's built-in rollback

### 4. Production-Grade Kubernetes Practices
- Proper **label/selector conventions** (`app.kubernetes.io/name`, `app.kubernetes.io/instance`) to avoid cross-release traffic collisions
- **Liveness & Readiness probes** tuned for each service (HTTP and TCP-based, with correct timeouts)
- **Resource requests/limits** defined for all containers
- **Secrets** for sensitive configuration (DB credentials, JWT secrets, API keys)
- **StatefulSet with volumeClaimTemplates** for MongoDB persistent, per-replica storage
- **RollingUpdate** deployment strategy for zero-downtime releases

### 5. Observability
- **Prometheus** for cluster-wide metrics collection (CPU, memory, pod health)
- **Grafana** dashboards for real-time visualization of resource usage per namespace/pod
- Enables proactive detection of crashes, resource exhaustion, and scaling needs

---

## Real-World Problems Debugged

This project wasn't built from a tutorial — every issue below was diagnosed and fixed through hands-on root-cause analysis:

| Issue | Root Cause | Fix |
|---|---|---|
| Wrong backend/frontend pods receiving traffic | Missing `release`-specific labels caused Services to match pods across multiple Helm releases | Added `app.kubernetes.io/instance` to all selectors |
| MongoDB pod stuck in `CrashLoopBackOff` | Liveness probe (`mongosh` exec) timing out under resource constraints | Switched to lightweight TCP socket probe |
| `CreateContainerConfigError` on all pods | `runAsNonRoot: true` set on images that only support root | Adjusted security context to match image capabilities |
| ArgoCD sync failing with namespace errors | Application's Destination Namespace left blank, so `.Release.Namespace` resolved empty | Explicitly set namespace in ArgoCD Application spec |
| CI/CD pipeline silently corrupting `values.yaml` | Greedy regex in the automated tag-update script matched to end-of-file | Rewrote as a block-aware line parser that respects YAML section boundaries |
| Frontend returning `502`/nginx `host not found` | Hardcoded Kubernetes Service name in `nginx.conf` didn't match the Helm-release-prefixed Service | Corrected `proxy_pass` target to the actual Service name |
| Deployment/StatefulSet `selector` immutable error | Kubernetes disallows changing `selector` post-creation | Deleted and recreated the resources under the corrected spec |

---

## What This Project Demonstrates

- End-to-end ownership of a CI/CD pipeline, from commit to production-style deployment
- Practical, production-relevant Kubernetes debugging (not just "it works" — understanding *why* it broke)
- GitOps principles: Git as the single source of truth, declarative infrastructure, automated reconciliation
- Security-conscious practices: vulnerability scanning, secrets management, least-privilege container execution
- Observability-first mindset: monitoring built in from the start, not bolted on afterward

---

## Repository Structure

```
saree-store-k8s/                  # GitOps repo (Helm chart)
└── saree-store-chart/
    ├── templates/
    │   ├── backend/               # Deployment, Service, ConfigMap, Secret
    │   ├── frontend/               # Deployment, Service
    │   ├── mongodb/                # StatefulSet, Service, Secret
    │   └── ingress.yaml
    └── values.yaml                 # Auto-updated by CI pipeline

saree-store-main/                 # Application repo
├── backend/                       # Node.js/Express API
├── frontend/                      # React application + Nginx config
└── Jenkinsfile                    # CI pipeline definition
```
