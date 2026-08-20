[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)

# 📦 Dolibarr GitOps Deployment
#### 🎯 Overview

#### Dolibarr is MUCH lighter than ERPNext/Odoo and works great on small AWS free-tier clusters (even 1–2 GB RAM).
#### Below is a ready-to-deploy ArgoCD + Kubernetes setup that works out-of-the-box using NodePort.
#### This repository contains a production-ready GitOps deployment for Dolibarr ERP/CRM on Kubernetes using ArgoCD. The setup includes:

#### ✅ Dolibarr (latest version) — lightweight ERP/CRM

#### ✅ MariaDB (10.11) — reliable, compatible database (replaces MySQL 8)

#### ✅ NodePort service for easy access

#### ✅ Persistent storage (PVCs) for data persistence

#### ✅ Resource optimized (< 500MB total RAM usage)

#### ✅ ArgoCD GitOps — fully automated sync

#### 🏗️ Architecture
#### text
#### ┌─────────────────────────────────────────────────────────────┐
#### │                     KUBERNETES CLUSTER                      │
#### ├─────────────────────────────────────────────────────────────┤
#### │                                                              │
#### │  ┌───────────────────┐        ┌────────────────────────┐   │
#### │  │    DOLIBARR       │        │       MARIADB           │   │
#### │  │  (tuxgasy/dolibarr│        │    (mariadb:10.11)      │   │
#### │  │    :latest)       │        │                        │   │
#### │  │   Port: 80        │───────▶│   Port: 3306           │   │
#### │  │   RAM: ~250MB     │        │   RAM: ~200MB          │   │
#### │  └───────────────────┘        └────────────────────────┘   │
#### │           │                              │                   │
#### │           ▼                              ▼                   │
#### │  ┌───────────────────┐        ┌────────────────────────┐   │
#### │  │  dolibarr-pvc     │        │     mysql-pvc          │   │
#### │  │  (10Gi)           │        │     (20Gi)             │   │
#### │  └───────────────────┘        └────────────────────────┘   │
#### │                                                              │
#### │  ┌─────────────────────────────────────────────────────┐    │
#### │  │              NodePort: 30081                        │    │
#### │  └─────────────────────────────────────────────────────┘    │
#### └─────────────────────────────────────────────────────────────┘
#### 📁 Repository Structure
#### text
#### dolibarr-argocd/
#### ├── namespace.yaml          # Namespace definition
#### ├── secret.yaml             # Database credentials
#### ├── mysql.yaml              # MariaDB + PVC + Service
#### ├── dolibarr.yaml           # Dolibarr + PVC
#### ├── service.yaml            # NodePort service
#### ├── kustomization.yaml      # Kustomize configuration
#### └── README.md               # This file
#### 📄 File Contents
#### 1. namespace.yaml
#### yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: dolibarr
#### 2. secret.yaml
#### yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: mysql-secret
      namespace: dolibarr
    type: Opaque
    stringData:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: dolibarr
      MYSQL_USER: dolibarr
      MYSQL_PASSWORD: dolipass
#### 3. mysql.yaml (MariaDB)
#### yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: mysql
      namespace: dolibarr
      labels:
        app: mariadb
    spec:
      ports:
        - port: 3306
          targetPort: 3306
      selector:
        app: mariadb
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mariadb
      namespace: dolibarr
      labels:
        app: mariadb
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mariadb
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: mariadb
        spec:
          containers:
            - name: mariadb
              image: mariadb:10.11
              ports:
                - containerPort: 3306
              envFrom:
                - secretRef:
                    name: mysql-secret
              volumeMounts:
                - name: data
                  mountPath: /var/lib/mysql
              resources:
                requests:
                  cpu: "100m"
                 memory: "256Mi"
                  ephemeral-storage: "1Gi"
                limits:
                  cpu: "300m"
                  memory: "512Mi"
                  ephemeral-storage: "2Gi"
              livenessProbe:
                exec:
                  command:
                    - mysqladmin
                    - ping
                    - -h
                    - localhost
                    - -u
                    - root
                    - -p$MYSQL_ROOT_PASSWORD
                initialDelaySeconds: 30
                periodSeconds: 10
              readinessProbe:
                exec:
                  command:
                    - mysqladmin
                    - ping
                    - -h
                    - localhost
                    - -u
                    - root
                    - -p$MYSQL_ROOT_PASSWORD
                initialDelaySeconds: 10
                periodSeconds: 5
          volumes:
            - name: data
              persistentVolumeClaim:
                claimName: mysql-pvc
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: mysql-pvc
      namespace: dolibarr
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi
#### 4. dolibarr.yaml
#### yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: dolibarr
      namespace: dolibarr
      labels:
        app: dolibarr
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: dolibarr
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: dolibarr
        spec:
          containers:
            - name: dolibarr
             image: tuxgasy/dolibarr:latest
              ports:
                - containerPort: 80
              env:
                - name: DOLI_DB_HOST
                  value: "mysql"
                - name: DOLI_DB_NAME
                  value: "dolibarr"
                - name: DOLI_DB_USER
                  value: "dolibarr"
                - name: DOLI_DB_PASSWORD
                  value: "dolipass"
              volumeMounts:
                - name: dolibarr-data
                  mountPath: /var/www/documents
              resources:
                requests:
                  cpu: "100m"
                  memory: "256Mi"
                  ephemeral-storage: "1Gi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
                  ephemeral-storage: "2Gi"
              livenessProbe:
                httpGet:
                  path: /
                  port: 80
                initialDelaySeconds: 60
                periodSeconds: 10
              readinessProbe:
                httpGet:
                  path: /
                  port: 80
                initialDelaySeconds: 30
                periodSeconds: 5
          volumes:
            - name: dolibarr-data
              persistentVolumeClaim:
                claimName: dolibarr-pvc
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: dolibarr-pvc
      namespace: dolibarr
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
#### 5. service.yaml
#### yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: dolibarr
      namespace: dolibarr
      labels:
       app: dolibarr
    spec:
      type: NodePort
      selector:
        app: dolibarr
      ports:
        - port: 80
          targetPort: 80
          nodePort: 30081
          name: dolibarr-http
#### 6. kustomization.yaml
#### yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    namespace: dolibarr

    resources:
      - namespace.yaml
      - secret.yaml
      - mysql.yaml
      - dolibarr.yaml
      - service.yaml

# Resource optimization labels
    labels:
      - pairs:
          app.kubernetes.io/managed-by: argocd
          app.kubernetes.io/version: "latest"
        includeSelectors: false
        includeTemplates: false
#### 🚀 Deployment Instructions
#### Prerequisites
#### ✅ Kubernetes cluster (AWS EKS, minikube, or any)

#### ✅ ArgoCD installed and configured

#### ✅ Git repository with these files

#### Option 1: Deploy via ArgoCD UI
#### Login to ArgoCD UI

#### Click "New App"

#### Fill in the details:

#### Field	Value
#### Application Name	dolibarr
#### Project	default
#### Repo URL	https://github.com/YOUR_REPO/dolibarr-argocd.git
#### Path	.
#### Target Revision	HEAD
#### Destination Server	https://kubernetes.default.svc
#### Namespace	dolibarr
#### Click "Create"

#### Click "Sync"

#### Option 2: Deploy via CLI
#### bash
# 1. Add the ArgoCD application
    kubectl apply -f argocd-app.yaml

# 2. Sync the application
    argocd app sync dolibarr

# 3. Check status
    argocd app get dolibarr
    argocd-app.yaml
#### yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: dolibarr
      namespace: argocd
      labels:
        app: dolibarr
    spec:
      project: default
      source:
        repoURL: https://github.com/YOUR_REPO/dolibarr-argocd.git
        targetRevision: HEAD
        path: .
      destination:
        server: https://kubernetes.default.svc
        namespace: dolibarr
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
#### 🌐 Access Dolibarr
#### Via NodePort
#### bash
# Get the node IP
    kubectl get nodes -o wide

# Access at:
# http://<NODE_IP>:30081
#### Via Port Forward (Local Testing)
#### bash
    kubectl port-forward -n dolibarr service/dolibarr 8080:80

# Then open: http://localhost:8080
#### 🎯 First-Time Setup
#### When you first access Dolibarr, you'll see the installation wizard:

#### Language: Select your language

#### Database Configuration (auto-detected):

#### DB Host: mysql

#### DB Name: dolibarr

#### DB User: dolibarr

#### DB Password: dolipass

#### Create Admin User:

#### Username: admin

#### Password: (set your own)

#### Email: (your email)

#### Click "Install"

#### Default credentials after install:

#### Username: admin

#### Password: admin (change immediately!)

#### 📊 Resource Usage
#### Component	Requests	Limits	Typical Usage
#### MariaDB	100m CPU, 256Mi RAM	300m CPU, 512Mi RAM	~200MB RAM
#### Dolibarr	100m CPU, 256Mi RAM	500m CPU, 512Mi RAM	~250MB RAM
#### Total	200m CPU, 512Mi RAM	800m CPU, 1Gi RAM	~450MB RAM
#### ✅ Works perfectly on AWS free tier / t2.micro instances!

#### 🔧 Troubleshooting
#### Check Pod Status
#### bash
    kubectl get pods -n dolibarr
    kubectl describe pod -n dolibarr -l app=dolibarr
    kubectl describe pod -n dolibarr -l app=mariadb
#### Check Logs
#### bash
# Dolibarr logs
    kubectl logs -n dolibarr -l app=dolibarr --tail=50

# MariaDB logs
    kubectl logs -n dolibarr -l app=mariadb --tail=50
#### Check Database Connection
#### bash
    kubectl exec -n dolibarr deploy/mariadb -- mysql -u dolibarr -pdolipass -e "SELECT 1"
#### Restart Dolibarr
#### bash
    kubectl rollout restart deployment dolibarr -n dolibarr
#### Access Database
#### bash
    kubectl exec -n dolibarr deploy/mariadb -it -- mysql -u root -prootpass
#### 🔄 GitOps Workflow
#### bash
# 1. Make changes to YAML files
    vim dolibarr.yaml

# 2. Commit and push to Git
    git add .
    git commit -m "Update Dolibarr configuration"
    git push

# 3. ArgoCD automatically syncs (with automated sync enabled)
# Or manually sync:
    argocd app sync dolibarr
#### 🛡️ Security Notes
#### 🔑 Change default passwords immediately after installation

#### 🌐 Use Ingress + TLS for production (cert-manager)

#### 🔒 Restrict NodePort access with network policies

#### 📦 Enable backup for PVCs (e.g., Velero)

#### 🚀 Next Steps (Production Ready)
#### □ Add Ingress with TLS (cert-manager)
#### □ Enable HTTPS
#### □ Set up automatic backups
#### □ Implement monitoring (Prometheus/Grafana)
#### □ Add resource quotas
#### □ Enable multi-company
#### □ Install additional Dolibarr modules
#### 📝 Why MariaDB instead of MySQL 8?
#### Issue	MySQL 8	MariaDB 10.11
#### Authentication	caching_sha2_password	mysql_native_password
#### Dolibarr Compatibility	❌ Plugin missing	✅ Works out of the box
#### Resource Usage	~300MB	~200MB
#### Community Support	✓	✓
#### MariaDB is a drop-in replacement for MySQL and works perfectly with Dolibarr.

#### 📚 Additional Resources
#### Dolibarr Documentation

#### MariaDB Documentation

#### ArgoCD Documentation

#### Kubernetes Documentation

#### 🤝 Contributing
#### Feel free to fork this repo and customize for your needs!

