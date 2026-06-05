[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)


markdown
# 🚦 GitOps Homelab - Argo CD Application Repository

## 📋 Overview

A **GitOps-driven** repository containing Kubernetes application manifests for my homelab environment. All applications are deployed via Argo CD using a combination of Helm charts, Kustomize, and raw YAML manifests.

### 🎯 Purpose
- **Single source of truth** for all homelab applications
- **Declarative application management** with Argo CD
- **Version-controlled configurations** for disaster recovery
- **Consistent deployment patterns** across PostgreSQL, WordPress, Odoo, Nextcloud, and more

### 📊 Applications Managed

| Application | Type | Deployment Method | Sync Status |
|-------------|------|-------------------|--------------|
| PostgreSQL | Database | Helm + Kustomize | ✅ |
| WordPress | CMS | Helm + Kustomize | ✅ |
| Odoo | ERP | Kustomize | ✅ |
| Nextcloud | Cloud Storage | Helm + Kustomize | ✅ |
| Snipe-IT | Asset Management | Kustomize | ✅ |
| Dolibarr | ERP/CRM | Kustomize | ✅ |
| Prometheus + Grafana | Monitoring | Helm + Kustomize | ✅ |
| MinIO + Velero | Backup | Kustomize | ✅ |
| ERPNext | ERP System | Kustomize | ✅ |

## 📁 Repository Structure
gitops-homelab/
├── postgres/ # PostgreSQL database
│ ├── kustomization.yaml # Kustomize configuration
│ ├── deployment.yaml # StatefulSet deployment
│ ├── service.yaml # ClusterIP service
│ ├── secret.yaml # Encrypted secrets
│ └── pvc.yaml # Persistent volume claim
│
├── word-press/ # WordPress CMS
│ ├── kustomization.yaml
│ ├── wordpress-app.yaml # Argo CD Application
│ └── values.yaml # Helm values override
│
├── odoo-argocd/ # Odoo ERP
│ ├── kustomization.yaml
│ └── deployment.yaml
│
├── nextcloud-gitops/ # Nextcloud storage
│ ├── kustomization.yaml
│ └── values.yaml
│
├── snipeit-argocd/ # Snipe-IT asset management
│ ├── kustomization.yaml
│ └── deployment.yaml
│
├── dolibarr/ # Dolibarr ERP/CRM
│ ├── kustomization.yaml
│ └── README.md
│
├── erp-next/ # ERPNext
│ ├── kustomization.yaml
│ └── deployment.yaml
│
├── prometheus-grafana-gitops/ # Monitoring stack
│ ├── kustomization.yaml
│ └── values.yaml
│
├── minio-velero-gitops/ # Backup infrastructure
│ ├── kustomization.yaml
│ └── deployment.yaml
│
├── argocd-applications.yaml # Root Argo CD Application
└── README.md # This file

text

## 🚀 Quick Start

### Prerequisites

- Kubernetes cluster (1.28+)
- Argo CD installed in `argocd` namespace
- kubectl configured with cluster access

### Deploy All Applications

```bash
# Apply the root Argo CD Application
kubectl apply -f argocd-applications.yaml

# Or sync individual apps via Argo CD UI/CLI
argocd app sync postgres
argocd app sync word-press
argocd app sync nextcloud-gitops
Deploy Single Application
bash
# Example: Deploy PostgreSQL
kubectl apply -f postgres/kustomization.yaml

# Or using Argo CD CLI
argocd app create postgres \
  --repo https://github.com/planb438/git-ops \
  --path postgres \
  --dest-namespace postgres \
  --dest-server https://kubernetes.default.svc \
  --sync-policy automated
📝 Application Configuration Guide
1. PostgreSQL Database
Namespace: postgres
Deployment Method: Kustomize + Raw YAML

yaml
# PostgreSQL configuration
Storage: 10Gi (PVC)
Port: 5432
Credentials: Managed via Kubernetes Secrets
Manual entry features:

Auto-generated report numbers

Helper tables for incident types

Validation constraints

Audit logging

2. WordPress CMS
Namespace: wordpress
Deployment Method: Helm + Kustomize

bash
# Customize values
cd word-press
kubectl edit kustomization.yaml
# Update repoURL, targetRevision as needed
3. Odoo ERP
Namespace: odoo
Deployment Method: Kustomize

4. Nextcloud Storage
Namespace: nextcloud
Deployment Method: Helm + Kustomize

5. Snipe-IT Asset Management
Namespace: snipeit
Deployment Method: Kustomize

6. Dolibarr ERP/CRM
Namespace: dolibarr
Deployment Method: Kustomize

7. Prometheus + Grafana Monitoring
Namespace: monitoring
Deployment Method: Helm + Kustomize

8. MinIO + Velero Backup
Namespace: velero
Deployment Method: Kustomize

9. ERPNext
Namespace: erpnext
Deployment Method: Kustomize

🔧 Argo CD Configuration
Root Application (argocd-applications.yaml)
yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-homelab
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/planb438/git-ops
    targetRevision: main
    path: ./
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
Sync Options
Policy	Setting	Effect
Automated Sync	prune: true	Delete resources not in Git
Self Heal	selfHeal: true	Auto-correct drift
Create Namespace	CreateNamespace: true	Auto-create app namespaces
🛠️ Common Operations
Update an Application
bash
# 1. Edit the application configuration
cd postgres
vi deployment.yaml

# 2. Commit and push changes
git add .
git commit -m "Update PostgreSQL storage to 20Gi"
git push origin main

# 3. Argo CD auto-syncs (if automated sync enabled)
# Or manually sync:
argocd app sync postgres
Rollback to Previous Version
bash
# Using Argo CD UI
# Applications → postgres → HISTORY AND ROLLBACK

# Using CLI
argocd app rollback postgres <revision-id>
Add a New Application
bash
# 1. Create application directory
mkdir new-app

# 2. Create kustomization.yaml
cat > new-app/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - secret.yaml
EOF

# 3. Create Argo CD Application
cat > new-app/application.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: new-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/planb438/git-ops
    path: new-app
  destination:
    server: https://kubernetes.default.svc
    namespace: new-app
EOF

# 4. Commit and push
git add new-app/
git commit -m "Add new-app"
git push
🔐 Security Best Practices
Secrets Management
⚠️ Never commit raw secrets to Git! Use:

Sealed Secrets (Bitnami)

yaml
# Instead of Secret, use SealedSecret
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: postgres-secret
spec:
  encryptedData:
    password: AgA...
External Secrets Operator (with AWS Secrets Manager)

yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  secretStoreRef:
    name: aws-secretsmanager
  target:
    name: postgres-secret
  data:
    - secretKey: password
      remoteRef:
        key: postgres-password
Current Secret Status
⚠️ Warning: Your repository contains plaintext secrets in:

postgres/secret.yaml

word-press/values.yaml

nextcloud-gitops/values.yaml

Fix immediately:

bash
# Remove secrets from Git history
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch **/secret.yaml **/values.yaml" \
  --prune-empty --tag-name-filter cat -- --all

# Add to .gitignore
echo "**/secret.yaml" >> .gitignore
echo "**/secrets.yaml" >> .gitignore
echo "**/*-secret.yaml" >> .gitignore

# Use Sealed Secrets instead
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml
kubectl create secret generic mysecret --dry-run=client -o yaml | kubeseal > sealed-secret.yaml
📈 Monitoring Applications
Check Application Status
bash
# Argo CD CLI
argocd app list
argocd app get postgres

# Kubernetes
kubectl get applications -n argocd
kubectl get pods -n postgres
kubectl get pods -n wordpress
View Logs
bash
# PostgreSQL logs
kubectl logs -n postgres -l app=postgres --tail=100

# Argo CD logs
kubectl logs -n argocd deployment/argocd-server --tail=50
🐛 Troubleshooting
Application OutOfSync
bash
# Force sync
argocd app sync postgres --force

# Check differences
argocd app diff postgres

# Hard reset (destructive)
kubectl patch application postgres -n argocd -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl delete application postgres -n argocd
ImagePullBackOff
bash
# Check if nodes can pull images
kubectl describe pod -n postgres <pod-name>

# Common fixes:
# 1. Add imagePullSecrets
# 2. Use private registry mirror
# 3. Increase disk space on nodes
Storage Issues
bash
# Check PVC status
kubectl get pvc -n postgres

# Expand PVC (if storage class allows)
kubectl patch pvc postgres-pvc -n postgres -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
📝 Git Commit Convention
text
<type>(<scope>): <subject>

Types:
- feat: New application or feature
- fix: Bug fix
- docs: Documentation only
- style: Code style changes
- refactor: Code refactoring
- perf: Performance improvement
- test: Test updates
- chore: Maintenance tasks

Example:
feat(postgres): Add auto-report number generation
fix(wordpress): Update database connection string
docs(readme): Add troubleshooting section
🔄 CI/CD Pipeline (Optional)
For automated sync, add GitHub Actions:

yaml
# .github/workflows/argocd-sync.yaml
name: Sync Argo CD Apps

on:
  push:
    branches: [ main ]

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Sync Argo CD
        run: |
          argocd app sync postgres --grpc-web
          argocd app sync word-press --grpc-web
📚 Additional Resources
Argo CD Documentation

Kustomize Documentation

Sealed Secrets Guide

📄 License
MIT License

👤 Author
Adari Bain - CKS Certified Kubernetes Security Specialist

Last Updated: June 2025
Argo CD Version: 2.10+
Kubernetes: 1.28+

text

## **Suggested Repository Name Changes**

| Current | Suggestion | Why |
|---------|------------|-----|
| `git-ops` | `gitops-homelab` | More descriptive |
| or | `argocd-apps` | Clear purpose |
| or | `homelab-gitops` | Standard naming |

## **Critical Security Action Required**

⚠️ **Your repository contains exposed secrets!** 

```bash
# Immediately rotate these passwords:
# - PostgreSQL credentials
# - WordPress database passwords
# - Nextcloud admin passwords
# - Odoo database credentials
Rotate all secrets now:

bash
# Generate new passwords
openssl rand -base64 32

# Update in your cluster
kubectl edit secret postgres-secret -n postgres

# Update in sealed secrets (recommended)
kubectl create secret generic postgres-secret \
  --from-literal=password=NEW_PASSWORD \
  --dry-run=client -o yaml | kubeseal > postgres/sealed-secret.yaml
