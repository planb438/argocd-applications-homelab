[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)

#### Here's a step-by-step guide to install Argo CD in your cluster, with Git documentation best practices. I'll include commands, verification steps, and how to structure your Git repo.

---

#### Step 1: Prerequisites
#### Working Kubernetes Cluster (You already have this)

#### kubectl Configured (Confirmed via kubectl get nodes)

#### Git Repository (Initialize one if you haven’t)

#### bash
    mkdir argocd-setup && cd argocd-setup
    git init

#### ---

#### Step 2: Install Argo CD
#### A. Create Namespace
#### bash
    kubectl create ns argocd

#### B. Apply Official Manifest
#### bash
    kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
#### C. Verify Installation
#### bash
    kubectl get pods -n argocd --watch
#### Wait for all pods to show Running (2-3 minutes). Key pods:

#### argocd-server (UI/API)

#### argocd-applicationset-controller (GitOps engine)

#### argocd-repo-server (Git sync)

#### ---

#### Step 3: Access Argo CD
#### A. Expose Argo CD UI
#### Option 1: NodePort (Quick Start)

#### bash
    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
#### Get the NodePort:

#### bash
    kubectl get svc -n argocd argocd-server
#### Access: http://<worker-node-ip>:<NodePort>

#### ---

#### Option 2: Ingress (Production)

bash
kubectl apply -n argocd -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
EOF

---

#### B. Retrieve Admin Password

---

#### bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
#### Save this password securely!

---

#### Step 4: Configure CLI (Optional but Recommended)
#### bash
    curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
    sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
    rm argocd-linux-amd64
#### Login via CLI:

#### bash
    argocd login <argocd-server-address> --username admin --password <password-from-step-3b>

#### ---

#### Step 5: Document in Git
#### A. Structure Your Repo
#### text
    argocd-setup/
    ├── manifests/
    │   ├── 00-namespace.yaml
    │   ├── 01-install.yaml
    │   └── 02-ingress.yaml (if using Ingress)
    ├── README.md
    └── apps/
        └── my-app.yaml (future apps)
#### B. Save Manifests
#### Save the namespace creation:

#### bash
    kubectl create namespace argocd --dry-run=client -o yaml > manifests/00-namespace.yaml
#### Download the install manifest:

#### bash
    curl -sSL https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -o manifests/01-install.yaml
#### C. Commit to Git
#### bash
    git add .
    git commit -m "Initial Argo CD installation manifests"
    git remote add origin <your-git-repo-url>
    git push -u origin main

#### ---

#### Step 6: Deploy Your First App via Argo CD
#### A. Create an Application Manifest
#### bash
cat > apps/my-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-repo/your-app.git
    targetRevision: HEAD
    path: k8s-manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
EOF
#### **B. Apply via CLI or UI
#### CLI:

#### bash
    kubectl apply -f apps/my-app.yaml
#### UI: Go to Argo CD → "+ New App" → Paste YAML.

#### ---

#### Step 7: Verify Sync
#### bash
    argocd app get my-app
#### Look for STATUS: Synced.

#### Troubleshooting
#### Issue	Solution
    argocd-server CrashLoopBackOff	Check logs: kubectl logs -n argocd deploy/argocd-server
#### Sync hangs at Progressing	Run argocd app sync my-app --force
     Ingress 404	Verify Ingress controller is running: kubectl get pods -n ingress-nginx
#### Next Steps
#### Secure Argo CD:

#### Enable SSO (e.g., Dex, OIDC)

#### Rotate the admin password:

#### bash
    argocd account update-password --account admin --current-password <old> --new-password <new>
#### Automate Everything:
#### Use Argo CD to manage itself (!):

#### bash
    kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd
#### Git Commit Example
#### bash
    git add .
    git commit -m "Add Argo CD app deployment manifest"
    git push
#### Now you have:

#### Fully documented Argo CD setup

#### Reproducible installation via Git

#### A foundation for GitOps workflows

#### Want to try deploying a sample app (e.g., NGINX) next?

#### ---

    argocd app get my-app
    {"level":"fatal","msg":"Argo CD server address unspecified ,argocd app sync my-app --force
    {"level":"fatal","msg":"Argo CD server address unspecified","time":"2025-08-19T03:33:03Z"}
#### This error occurs because the argocd CLI doesn’t know how to connect to your Argo CD server. Let’s fix this step by step.

#### Step 1: Configure Argo CD CLI Access
#### A. Get the Argo CD Server Address
#### If you used NodePort:

#### bash
    ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.clusterIP}')
    ARGOCD_PORT=$(kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
    echo "Argo CD URL: http://$ARGOCD_SERVER:$ARGOCD_PORT"

#### If you used Ingress:

#### bash
    ARGOCD_SERVER="argocd.your-domain.com"  # Replace with your Ingress host
#### B. Log in via CLI
#### bash
    argocd login $ARGOCD_SERVER --username admin --password <your-password> --insecure
    --insecure skips TLS verification (ok for home labs).

#### For Ingress with HTTPS, omit --insecure and ensure your domain has valid TLS.

#### Step 2: Retry Your Commands
#### Now the CLI knows where to connect:

bash
argocd app get my-app
argocd app sync my-app --force
Step 3: Persist the Server Address (Optional)
To avoid specifying --server repeatedly:

#### bash
    argocd context set-default $ARGOCD_SERVER

#### Troubleshooting
#### If Login Fails
#### Verify the Argo CD server is running:

#### bash
    kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server

#### Check service exposure:

#### bash
    kubectl get svc -n argocd argocd-server

#### For NodePort, ensure your firewall allows traffic to the port (e.g., 30007).

#### For Ingress, check ingress controller logs:

#### bash
    kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

#### Reset admin password (if lost):

#### bash
    kubectl -n argocd delete secret argocd-initial-admin-secret
    kubectl -n argocd rollout restart deploy/argocd-server

#### Example: Full Workflow
#### bash
# 1. Set server address
    ARGOCD_SERVER="localhost:8080"  # Use your actual IP/domain

# 2. Login
    argocd login $ARGOCD_SERVER --username admin --password <password> --insecure

# 3. Sync an app
    argocd app sync my-app --force

# 4. Verify
    argocd app get my-app

#### Next Steps
#### Secure your setup:

#### Enable TLS for production.

#### Set up SSO (e.g., GitHub OAuth).

#### Explore GitOps patterns:

#### bash
    argocd app create my-gitops-app \
      --repo https://github.com/your/repo.git \
      --path manifests \
      --dest-namespace default

     ---

    argocd app get my-app
    Name:               argocd/my-app
    Project:            default
    Server:             https://kubernetes.default.svc
    Namespace:          default
    URL:                https://10.108.118.60/applications/my-app
    Source:
    - Repo:             https://github.com/your-repo/your-app.git
      Target:           HEAD
      Path:             k8s-manifests
    SyncWindow:         Sync Allowed
    Sync Policy:        Automated (Prune)
    Sync Status:        Unknown
    Health Status:      Healthy

    CONDITION        MESSAGE                                                                                                                                                                           LAST TRANSITION

#### ComparisonError  Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = failed to list refs: authentication required: #### Repository not found.  2025-08-19 03:31:48 +0000 UTC
#### The error indicates that Argo CD cannot access your Git repository (https://github.com/your-repo/your-app.git) due to authentication issues. Let's fix this step by step.

#### Step 1: Verify the Git Repo URL
#### Ensure the repository:

#### Exists and is accessible (visit the URL in your browser).

#### Contains a valid k8s-manifests directory with Kubernetes YAMLs.

#### Step 2: Add Git Repository Credentials to Argo CD
#### A. For Public Repos (No Auth)
#### Skip this step if your repo is public.

#### B. For Private Repos (SSH or HTTPS)
#### SSH Key (Recommended):

#### bash
# 1. Generate SSH key (if you don't have one)
    ssh-keygen -t ed25519 -C "argocd@your-cluster"

# 2. Add the public key to GitHub/GitLab:
    cat ~/.ssh/id_ed25519.pub
# Paste this into your Git provider's SSH keys settings

# 3. Add repo to Argo CD via CLI:
    argocd repo add git@github.com:your-repo/your-app.git \
      --ssh-private-key-path ~/.ssh/id_ed25519 \
      --insecure-ignore-host-key

#### HTTPS (Username/Password or Token):

bash
# For GitHub, use a Personal Access Token (PAT):
    argocd repo add https://github.com/your-repo/your-app.git \
      --username your-github-username \
      --password your-github-token

#### Step 3: Update the Application
#### If you already created the app, update its source:

#### bash
    argocd app set my-app \
      --repo https://github.com/your-repo/your-app.git \
      --path k8s-manifests \
      --revision HEAD

#### Step 4: Force a Sync
#### bash
    argocd app sync my-app --force

#### Step 5: Verify
#### bash
    argocd app get my-app

#### Look for:

#### Sync Status: Synced

#### Health Status: Healthy

#### Troubleshooting
#### Issue	Solution
#### Repository not found	Double-check repo URL and credentials
#### Permission denied (SSH)	Ensure the SSH key is added to your Git account
#### Invalid token (HTTPS)	Regenerate the GitHub PAT with repo scope
#### Example: Working Setup
#### Public Repo:

#### bash
argocd repo add https://github.com/argoproj/argocd-example-apps.git
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
Private Repo (SSH):

#### bash
argocd repo add git@github.com:your-repo/private-app.git \
  --ssh-private-key-path ~/.ssh/id_ed25519
#### Next Steps
#### Automate Syncs:

#### bash
argocd app set my-app --sync-policy automated
#### Enable Pruning (auto-delete removed resources):

#### bash
argocd app set my-app --auto-prune
#### Let me know if you need help with Git authentication!

---

argocd repo add https://github.com/planb438/Argo-CD.git \
>  --username plab438 \
>    --password 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
Repository 'https://github.com/planb438/Argo-CD.git' added
master-node-001@master-node-001:~/argocd-setup/apps$ 
master-node-001@master-node-001:~/argocd-setup/apps$ 
master-node-001@master-node-001:~/argocd-setup/apps$ argocd app set my-app \
--repo https://github.com/planb438/Argo-CD.git \
>   --path k8s-manifests \
  --revision HEAD
{"level":"fatal","msg":"rpc error: code = InvalidArgument desc = application spec for my-app is invalid: InvalidSpecError: Unable to generate manifests in k8s-manifests: rpc error: code = Unknown desc = k8s-manifests: app path does not exist","time":"2025-08-19T03:58:00Z"}
master-node-001@master-node-001:~/argocd-setup/apps$ 
master-node-001@master-node-001:~/argocd-setup/apps$ argocd app set my-app --repo https://github.com/planb438/Argo-CD.git --path k8s-manifests --revi
sion HEAD
{"level":"fatal","msg":"rpc error: code = InvalidArgument desc = application spec for my-app is invalid: InvalidSpecError: Unable to generate manifests in k8s-manifests: rpc error: code = Unknown desc = k8s-manifests: app path does not exist","time":"2025-08-19T03:58:29Z"}
master-node-001@master-node-001:~/argocd-setup/apps$ 
master-node-001@master-node-001:~/argocd-setup/apps$ 
master-node-001@master-node-001:~/argocd-setup/apps$ argocd app sync my-app --force
{"level":"fatal","msg":"rpc error: code = FailedPrecondition desc = error resolving repo revision: rpc error: code = Unknown desc = failed to list refs: authentication required: Repository not found.","time":"2025-08-19T03:59:10Z"}
master-node-001@master-node-001:~/argocd-setup/apps$ 
master-node-001@master-node-001:~/argocd-setup/apps$ 
master-node-001@master-node-001:~/argocd-setup/apps$ 
master-node-001@master-node-001:~/argocd-setup/apps$ argocd app get my-app
Name:               argocd/my-app
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://10.108.118.60/applications/my-app
Source:
- Repo:             https://github.com/your-repo/your-app.git
  Target:           HEAD
  Path:             k8s-manifests
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Unknown
Health Status:      Healthy

CONDITION        MESSAGE                                                                                                                                                                           LAST TRANSITION
#### ComparisonError  Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = failed to list refs: authentication required: Repository not found.  2025-08-19 03:31:48 +0000 UTC

master-node-001@master-node-001:~/argocd-setup/apps$

---

It looks like you're encountering two key issues with your Argo CD setup:

Authentication Problems (Repository not found)

Path Validation Issues (k8s-manifests directory doesn't exist)

Let me help you fix this systematically:

Step 1: Verify Your Git Repository Structure
First, confirm your repository has the correct structure:

#### bash
    git clone https://github.com/planb438/Argo-CD.git
    cd Argo-CD
    ls -la
#### You should see a k8s-manifests/ directory containing Kubernetes YAML files. If not:

bash
mkdir -p k8s-manifests
echo "apiVersion: v1
kind: Namespace
metadata:
  name: demo" > k8s-manifests/00-namespace.yaml
git add . && git commit -m "Add example manifests" && git push
Step 2: Properly Configure the Repository in Argo CD
A. Delete and Re-add the Repository
bash
argocd repo rm https://github.com/planb438/Argo-CD.git
argocd repo add https://github.com/planb438/Argo-CD.git \
  --username plab438 \
  --password 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
B. Verify Repository Access
bash
argocd repo get https://github.com/planb438/Argo-CD.git
Should show Connection Status: Successful.

Step 3: Recreate the Application Correctly
A. Delete the Broken App
bash
argocd app delete my-app
B. Create a New App with Valid Path
bash
argocd app create my-app \
  --repo https://github.com/planb438/Argo-CD.git \
  --path k8s-manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune
Step 4: Verify and Sync
#### bash
    argocd app get my-app
    argocd app sync my-app
#### Key Troubleshooting Tips
#### For Authentication Issues:

#### GitHub tokens need repo scope

#### Tokens expire - generate a new one if needed

#### bash
# Test repo access manually
    curl -H "Authorization: token ghp_yourtoken" \
      https://api.github.com/repos/planb438/Argo-CD/contents/k8s-manifests
#### For Path Errors:

#### The path must be relative to repo root

#### Case-sensitive - confirm exact directory name

#### Common Fixes:

#### bash
# Update existing app's path
    argocd app set my-app --path correct/path

# Check app differences
    argocd app diff my-app
Example Working Setup
Here's a minimal valid structure for your repo:

#### text
    Argo-CD/
    └── k8s-manifests/
        ├── 00-namespace.yaml
        └── 01-deployment.yaml
With 01-deployment.yaml containing:

yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
#### Next Steps
#### Commit real applications to your repo

#### Set up webhooks for automatic syncs:

#### bash
    argocd app set my-app --sync-policy automated
#### Secure your setup:

#### bash
    argocd account update-password --account admin
