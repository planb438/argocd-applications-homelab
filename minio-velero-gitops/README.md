[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)

# ☁️ MinIO + Velero GitOps - Cloud-Native Backup Solution

## 📋 Overview

#### A **GitOps-managed** backup infrastructure for Kubernetes using **MinIO** (S3-compatible object storage) and **Velero** (Kubernetes backup/restore). This setup provides automated backup of persistent volumes, cluster resources, and application configurations.

### 🎯 Purpose
- **Disaster Recovery** for all homelab applications
- **Automated Backups** of PostgreSQL, WordPress, Nextcloud, and more
- **S3-Compatible Storage** without vendor lock-in
- **Point-in-Time Restore** capability
- **GitOps Management** via Argo CD

---

## 🚀 Deploy via Argo CD

### Step 1: Apply the Application

#### bash
kubectl apply -f argocd-application.yaml

#### Step 2: Sync
#### bash
argocd app sync minio-velero --force

#### Step 3: Verify
#### bash
kubectl get pods,svc,pvc -n velero

#### Expected Output:

#### text
NAME                         READY   STATUS    RESTARTS   AGE
pod/minio-xxxxx              1/1     Running   0          2m
pod/velero-xxxxx             1/1     Running   0          2m

NAME                 TYPE       CLUSTER-IP     PORT(S)
service/minio        NodePort   10.xxx.xxx.x   9000:30900/TCP,9001:30901/TCP
service/velero       ClusterIP  10.xxx.xxx.x   8085/TCP

NAME                           STATUS   VOLUME   CAPACITY
persistentvolumeclaim/minio-pvc   Bound    pvc-xxx   50Gi

# 🔧 Component Details
#### 1. MinIO - S3 Storage Backend
#### Setting	Value
#### Namespace	velero
#### Internal Access	minio.velero.svc.cluster.local:9000
#### External Access	NodePort 30900
#### Console Access	NodePort 30901
#### Default Credentials (⚠️ CHANGE IMMEDIATELY):

#### text
Username: minioadmin
Password: minioadmin123

#### 2. Velero - Backup Controller
Setting	Value
Namespace	velero
Storage Location	S3 (MinIO)
Bucket	velero
Backup Schedule	Configured via CLI
🔄 Backup Operations
Create a Backup
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

# Backup everything
velero backup create full-cluster-$(date +%Y%m%d) \
  --include-namespaces "*" \
  --exclude-namespaces velero,argocd,kube-system \
  --ttl 168h
Schedule Backups
bash
# Daily backup
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces default,postgres,wordpress,nextcloud \
  --ttl 72h

# Weekly full backup
velero schedule create weekly-backup \
  --schedule="0 3 * * 0" \
  --include-namespaces "*" \
  --exclude-namespaces velero,argocd,kube-system \
  --ttl 168h
List Backups
bash
velero backup get
🔄 Restore Operations
Restore from Backup
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
Disaster Recovery - Full Cluster Restore
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
🛠️ Management Commands
MinIO Management
bash
# Access MinIO Console
kubectl port-forward -n velero svc/minio 9001:9001
# Open http://localhost:9001

# List buckets
kubectl exec -n velero deployment/minio -- mc ls local/

# Create bucket
kubectl exec -n velero deployment/minio -- mc mb local/velero

# Set bucket policy
kubectl exec -n velero deployment/minio -- mc policy set public local/velero
Velero Management
bash
# Install Velero CLI (if not installed)
wget https://github.com/vmware-tanzu/velero/releases/download/v1.13.0/velero-v1.13.0-linux-amd64.tar.gz
tar -xzf velero-v1.13.0-linux-amd64.tar.gz
sudo mv velero-v1.13.0-linux-amd64/velero /usr/local/bin/

# Check Velero status
velero describe backupstoragelocation default

# View backup logs
velero backup logs daily-backup

# Delete old backup
velero backup delete daily-backup-20250529
🔐 Security Configuration
⚠️ CRITICAL: Rotate Default Credentials Immediately!
bash
# Generate new credentials
MINIO_ROOT_USER=$(openssl rand -base64 12 | tr -d "=+/")
MINIO_ROOT_PASSWORD=$(openssl rand -base64 24 | tr -d "=+/")

# Create secret
kubectl delete secret minio-credentials -n velero 2>/dev/null
kubectl create secret generic minio-credentials -n velero \
  --from-literal=root-user="$MINIO_ROOT_USER" \
  --from-literal=root-password="$MINIO_ROOT_PASSWORD"

# Update deployment
kubectl set env deployment/minio -n velero \
  MINIO_ROOT_USER=$MINIO_ROOT_USER \
  MINIO_ROOT_PASSWORD=$MINIO_ROOT_PASSWORD

# Restart MinIO
kubectl rollout restart deployment/minio -n velero
Enable MinIO Encryption
Add to minio/deployment.yaml:

yaml
env:
- name: MINIO_ENCRYPTION
  value: "on"
📈 Monitoring Backups
Prometheus Metrics
bash
# Add to velero/deployment.yaml
env:
- name: VELERO_SCRAPE_INTERVAL
  value: "30s"
- name: VELERO_METRICS_PORT
  value: "8085"
Backup Health Checks
bash
# Check backup status
velero backup get | grep -E "Completed|Failed"

# Check backup storage location
velero describe backupstoragelocation default

# Validate backup integrity
velero backup describe daily-backup --details
🔧 Troubleshooting
Backup Failing
bash
# Check Velero logs
kubectl logs -n velero -l app=velero

# Check MinIO status
kubectl get pods -n velero -l app=minio

# Check PVC
kubectl get pvc -n velero

# Check MinIO logs
kubectl logs -n velero -l app=minio
MinIO Not Accessible
bash
# Check service
kubectl get svc -n velero

# Check NodePort (should be 30900)
kubectl get svc -n velero minio -o jsonpath='{.spec.ports[?(@.name=="api")].nodePort}'
Velero Can't Connect to MinIO
bash
# Check MinIO service
kubectl get svc -n velero minio

# Verify credentials
kubectl get secret -n velero minio-credentials

# Test connection from Velero pod
kubectl exec -n velero deployment/velero -- \
  curl -s http://minio.velero.svc.cluster.local:9000
🧹 Cleanup
bash
# Delete via Argo CD
argocd app delete minio-velero

# Or
kubectl delete -f argocd-application.yaml

# Delete namespace
kubectl delete namespace velero
📚 Additional Resources
MinIO Documentation

Velero Documentation

AWS S3 Backup Storage

👤 Author
Adari Bain - CKS Certified Kubernetes Security Specialist

GitHub

LinkedIn

🏆 Why This Backup Solution Matters
Benefit	Description
Disaster Recovery	Full cluster restore capability
GitOps Management	Infrastructure as Code
Automated Backups	Scheduled backups for all apps
S3-Compatible	No vendor lock-in
Persistent Volumes	Snapshot and restore
Tested Restores	Verified restore procedures
Built with ☁️ for Production Kubernetes Backup

text

---

## 🚀 Quick Deploy

```bash
# 1. Apply the Application
kubectl apply -f argocd-application.yaml

# 2. Sync
argocd app sync minio-velero --force

# 3. Check
kubectl get pods -n velero

# 4. Access MinIO Console
kubectl port-forward -n velero svc/minio 9001:9001
# Open http://localhost:9001
# Username: minioadmin
# Password: minioadmin123

# 5. Create first backup
velero backup create first-backup --include-namespaces default --ttl 72h
📁 Complete File Structure
text
minio-velero-gitops/
├── README.md                          # Full documentation
├── argocd-application.yaml            # Argo CD Application
├── minio/
│   ├── kustomization.yaml
│   └── deployment.yaml                # MinIO deployment
└── velero/
    ├── kustomization.yaml
    └── deployment.yaml                # Velero deployment
Your backup solution is now fully GitOps-managed! 🚀

