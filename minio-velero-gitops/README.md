[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)

markdown
# ☁️ MinIO + Velero GitOps - Cloud-Native Backup Solution

## 📋 Overview

A **GitOps-managed** backup infrastructure for Kubernetes homelab using **MinIO** (S3-compatible object storage) and **Velero** (Kubernetes backup/restore). This setup provides automated backup of persistent volumes, cluster resources, and application configurations.

### 🎯 Purpose
- **Disaster Recovery** for all homelab applications
- **Automated Backups** of PostgreSQL, WordPress, Nextcloud, and more
- **S3-Compatible Storage** without vendor lock-in
- **Point-in-Time Restore** capability
- **GitOps Management** via Argo CD

### 🏗️ Architecture
#### ┌─────────────────────────────────────────────────────────────┐
#### │ Kubernetes Cluster │
#### ├─────────────────────────────────────────────────────────────┤
#### │ │
#### │ ┌──────────────┐ ┌──────────────┐ │
#### │ │ Velero │────────▶│ MinIO │ │
#### │ │ (Backup) │ │ (Storage) │ │
#### │ └──────────────┘ └──────────────┘ │
#### │ │ │ │
#### │ │ │ │
#### │ ┌──────▼──────┐ ┌───────▼──────┐ │
#### │ │ Backups │ │ Buckets │ │
#### │ │ - Full │ │ - velero │ │
#### │ │ - Scheduled │ │ - backups/ │ │
#### │ │ - Incremental│ └──────────────┘ │
#### │ └─────────────┘ │
#### │ │
#### └─────────────────────────────────────────────────────────────┘

---


Verify Installation
bash
# Check MinIO
kubectl get pods,svc,pvc -n minio

# Check Velero
    kubectl get pods -n velero
    kubectl get backupstoragelocations -n velero

# 🔧 Component Details
#### 1. MinIO - S3 Storage Backend
#### Namespace: minio
#### Access:

#### Internal: minio.minio.svc.cluster.local:9000

#### External: NodePort 30900

#### Default Credentials:

#### text
    Username: minioadmin
    Password: minioadmin123  # CHANGE THIS IMMEDIATELY!


---


#### Create Custom Backup
bash
# One-time backup of specific namespace
    velero backup create postgres-backup-$(date +%Y%m%d) \
      --include-namespaces postgres \
      --ttl 72h

# Backup with volume snapshot
    velero backup create wordpress-full \
      --include-namespaces wordpress \
      --snapshot-volumes \
      --ttl 168h


# 🔄 Restore Operations
#### Restore from Backup
bash
# List available backups
    velero backup get

# Restore entire backup
    velero restore create --from-backup postgres-backup-20250605

# Restore specific namespace
    velero restore create --from-backup weekly-backup \
      --include-namespaces postgres,wordpress

# Restore single resource
    velero restore create --from-backup daily-backup \
      --include-resources deployments,configmaps

#### Disaster Recovery - Full Cluster Restore
bash
# Step 1: Restore MinIO (if lost)
    kubectl apply -f minio/

# Step 2: Restore Velero
    kubectl apply -f velero/

# Step 3: Wait for Velero to be ready
    kubectl wait --for=condition=ready pod -l app=velero -n velero

# Step 4: List available backups
    velero backup get

# Step 5: Restore everything
    velero restore create --from-backup weekly-backup --restore-volumes

# Step 6: Verify restore
    kubectl get all --all-namespaces

# 🛠️ Management Commands
#### MinIO Management
bash
# Access MinIO Console
    kubectl port-forward -n minio svc/minio-console 9001:9001
# Open http://localhost:9001

# List buckets
    kubectl exec -n minio minio-0 -- mc ls local/

# Create bucket
    kubectl exec -n minio minio-0 -- mc mb local/new-bucket

# Set bucket policy
    kubectl exec -n minio minio-0 -- mc policy set public local/velero
#### Velero Management
bash
# Check Velero status
    velero describe backupstoragelocation default

# View backup logs
    velero backup logs daily-backup

# Delete old backup
    velero backup delete daily-backup-20250529

# Install Velero CLI (if not installed)
    wget https://github.com/vmware-tanzu/velero/releases/download/v1.13.0/velero-v1.13.0-linux-amd64.tar.gz
    tar -xzf velero-v1.13.0-linux-amd64.tar.gz
    sudo mv velero-v1.13.0-linux-amd64/velero /usr/local/bin/

# 📈 Monitoring Backups
#### Prometheus Metrics (Optional)
#### Add to your velero/deployment.yaml:

#### yaml
    env:
    - name: VELERO_SCRAPE_INTERVAL
      value: "30s"
    - name: VELERO_METRICS_PORT
      value: "8085"
#### Then add ServiceMonitor:

#### yaml
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: velero
      namespace: velero
    spec:
      selector:
        matchLabels:
      app: velero
      endpoints:
      - port: metrics
        interval: 30s
#### Backup Health Checks
bash
# Check backup status
    velero backup get | grep -E "Completed|Failed"

# Check backup storage location
    velero describe backupstoragelocation default

# Validate backup integrity
    velero backup describe daily-backup --details
# 🔐 Security Configuration
#### Current Security Status
# ⚠️ CRITICAL: Rotate default credentials immediately!

bash
# Generate new MinIO credentials
# Create new secret with secure passwords
    MINIO_ROOT_USER=$(openssl rand -base64 12 | tr -d "=+/")
    MINIO_ROOT_PASSWORD=$(openssl rand -base64 24 | tr -d "=+/")

# Create updated secret
    kubectl delete secret minio-credentials -n minio
    kubectl create secret generic minio-credentials \
      --from-literal=root-user="$MINIO_ROOT_USER" \
      --from-literal=root-password="$MINIO_ROOT_PASSWORD" \
      -n minio

# Restart MinIO to apply new credentials
    kubectl rollout restart statefulset minio -n minio
#### Encryption at Rest
#### Enable MinIO encryption:

yaml
# Add to minio/deployment.yaml
    env:
    - name: MINIO_ROOT_USER
      valueFrom:
        secretKeyRef:
          name: minio-credentials
          key: root-user
    - name: MINIO_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: minio-credentials
          key: root-password
    - name: MINIO_ENCRYPTION
      value: "on"


  ---


#   📄 License
#### MIT License

# 👤 Author
#### Adari Bain - CKS Certified Kubernetes Security Specialist

