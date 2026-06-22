[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)

# 🎯 GitOps Approach for Ingress + Cert Manager

#### 📁 Step 1: Create Git Repo Structure
#### bash
    cd /tmp/Argo-CD
    mkdir -p apps/ingress-nginx apps/cert-manager apps/cluster-issuers
#### 📦 Step 2: Ingress Controller (Helm via Argo CD)
#### Create apps/ingress-nginx/00-application.yaml:

#### bash
    cat > apps/ingress-nginx/00-application.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: ingress-nginx
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        repoURL: https://kubernetes.github.io/ingress-nginx
        targetRevision: latest
        chart: ingress-nginx
        helm:
          values: |
            controller:
              hostNetwork: true
              service:
                type: ClusterIP
              admissionWebhooks:
                enabled: false
              publishService:
                enabled: true
              replicaCount: 2
              nodeSelector:
                kubernetes.io/os: linux
            defaultBackend:
              nodeSelector:
                kubernetes.io/os: linux
      destination:
        server: https://kubernetes.default.svc
        namespace: ingress-nginx
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
    EOF
#### 🔐 Step 3: Cert Manager (Helm via Argo CD)
#### Option A: Using Helm Chart (Simpler)
#### bash
    cat > apps/cert-manager/00-application.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: cert-manager
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
     source:
    repoURL: https://charts.jetstack.io
    targetRevision: latest
    chart: cert-manager
    helm:
      values: |
        installCRDs: true
        nodeSelector:
          kubernetes.io/os: linux
     destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
     syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
    EOF
#### Option B: Using Manifests (If Helm Times Out)
#### bash
    cat > apps/cert-manager/00-manifests.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: cert-manager
      namespace: argocd
     finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
     source:
    repoURL: https://github.com/planb438/Argo-CD.git
    targetRevision: HEAD
    path: apps/cert-manager/manifests
     destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
     syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
    EOF

# Download manifests
    mkdir -p apps/cert-manager/manifests
    cd apps/cert-manager/manifests

# Download CRDs
    curl -sL https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.crds.yaml -o 00-crds.yaml

# Download controller
    curl -sL https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml -o 01-controller.yaml
#### 🏗️ Step 4: ClusterIssuers
#### Create apps/cluster-issuers/00-cluster-issuers.yaml:

#### bash
    cat > apps/cluster-issuers/00-cluster-issuers.yaml << 'EOF'
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: selfsigned-issuer
    spec:
     selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
     metadata:
      name: selfsigned-ca
      namespace: cert-manager
    spec:
      isCA: true
      commonName: selfsigned-ca
      secretName: selfsigned-ca-secret
     issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    ---
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: ca-issuer
    spec:
     ca:
    secretName: selfsigned-ca-secret
    EOF
#### Create apps/cluster-issuers/00-application.yaml:

#### bash
    cat > apps/cluster-issuers/00-application.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Application
     metadata:
      name: cluster-issuers
      namespace: argocd
      finalizers:
         - resources-finalizer.argocd.argoproj.io
     spec:
       project: default
      source:
    repoURL: https://github.com/planb438/Argo-CD.git
    targetRevision: HEAD
    path: apps/cluster-issuers
     destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
     syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
     EOF
#### 🧪 Step 5: Test App (whoami) via GitOps
#### bash
    mkdir -p apps/whoami

     cat > apps/whoami/00-deployment.yaml << 'EOF'
     apiVersion: apps/v1
     kind: Deployment
    metadata:
      name: whoami
      namespace: default
    spec:
      replicas: 1
      selector:
    matchLabels:
      app: whoami
     template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: traefik/whoami:latest
        ports:
        - containerPort: 80
    EOF

    cat > apps/whoami/01-service.yaml << 'EOF'
    apiVersion: v1
    kind: Service
    metadata:
     name: whoami
     namespace: default
    spec:
     selector:
    app: whoami
     ports:
     - port: 80
    targetPort: 80
    EOF

    cat > apps/whoami/02-ingress.yaml << 'EOF'
     apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
     name: whoami-selfsigned
     namespace: default
     annotations:
    cert-manager.io/cluster-issuer: ca-issuer
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    spec:
      ingressClassName: nginx
     tls:
     - hosts:
    - whoami.local
    secretName: whoami-local-tls
     rules:
     - host: whoami.local
     http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: whoami
            port:
              number: 80
    EOF
#### Create apps/whoami/00-application.yaml:

#### bash
    cat > apps/whoami/00-application.yaml << 'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: Application
     metadata:
      name: whoami
     namespace: argocd
     finalizers:
    - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
     source:
    repoURL: https://github.com/planb438/Argo-CD.git
    targetRevision: HEAD
    path: apps/whoami
     destination:
    server: https://kubernetes.default.svc
    namespace: default
     syncPolicy:
    automated:
      selfHeal: true
      prune: true
    EOF
#### 🚀 Step 6: Commit and Deploy
#### bash
    cd /tmp/Argo-CD

####  Check structure
#### tree apps/
#### apps/
####  ├── cert-manager/
####  │   └── 00-application.yaml
####  ├── cluster-issuers/
####  │   ├── 00-application.yaml
####  │   └── 00-cluster-issuers.yaml
####  ├── ingress-nginx/
####  │   └── 00-application.yaml
####  └── whoami/
####      ├── 00-application.yaml
####      ├── 00-deployment.yaml
####      ├── 01-service.yaml
####      └── 02-ingress.yaml

# Commit and push
    git add .
    git commit -m "Add Ingress + Cert Manager + ClusterIssuers + whoami via GitOps"
    git push

# Deploy each app
    kubectl apply -f apps/ingress-nginx/00-application.yaml
    kubectl apply -f apps/cert-manager/00-application.yaml
    kubectl apply -f apps/cluster-issuers/00-application.yaml
    kubectl apply -f apps/whoami/00-application.yaml

# Check sync status
    argocd app list
    argocd app get ingress-nginx
    argocd app get cert-manager
    argocd app get cluster-issuers
    argocd app get whoami

# Force sync if needed
     argocd app sync ingress-nginx --force
    argocd app sync cert-manager --force
    argocd app sync cluster-issuers --force
    argocd app sync whoami --force
#### 🧹 Step 7: Clean Up Manual Installs (Optional)
#### bash
# Once GitOps is working, remove manual installations
    helm uninstall ingress-nginx -n ingress-nginx 2>/dev/null
    helm uninstall cert-manager -n cert-manager 2>/dev/null

# Let Argo CD recreate them from Git
    argocd app sync ingress-nginx --force
    argocd app sync cert-manager --force
#### 📊 Summary: GitOps vs Manual
#### Component	Before (Manual)	Now (GitOps)
#### Ingress Controller	helm install	Argo CD App
#### Cert Manager	helm install	Argo CD App
#### ClusterIssuers	kubectl apply	Argo CD App
#### Test App	kubectl apply	Argo CD App
#### Updates	Manual	Git commit + push
#### 🏆 What You've Achieved
#### Feature	Status
#### GitOps for Ingress	✅
#### GitOps for Cert Manager	✅
#### GitOps for ClusterIssuers	✅
#### GitOps for Applications	✅
#### Full Argo CD Management	✅
#### Infrastructure as Code	✅
#### Entire cluster is now managed via GitOps! 🚀

#### Any changes to Ingress, Cert Manager, or applications are now done by updating Git and letting Argo CD sync automatically.
