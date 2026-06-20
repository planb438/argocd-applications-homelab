[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)


# 📦 Argo CD Application for ERPNext

#### 🚀 Deploy ERPNext
bash
# Apply the Argo CD application
    kubectl apply -f erpnext-app.yaml -n argocd

# Monitor the deployment
    argocd app get erpnext -w

# Check pods (will take 5-10 minutes to initialize)
    kubectl get pods -n erpnext -w
#### 🔧 What Gets Deployed
#### The Helm chart creates multiple components :

#### Gunicorn - Python web server (main ERPNext app)

#### Nginx - Static file serving

#### Workers - Background job processing (default, short, long queues)

#### Scheduler - Automated tasks

#### SocketIO - Real-time communications

#### Create-site job - Initial site setup

#### 📊 Access ERPNext
bash
# Get the NodePort
    kubectl get svc -n erpnext erpnext-nginx -o jsonpath='{.spec.ports[0].nodePort}'

# Access via any worker node
    http://<your-worker-node-ip>:32000
#### Default login:

#### Username: Administrator

#### Password: Generated during site creation (check logs)

bash
# Get admin password from logs
    kubectl logs -n erpnext -l job-name=erpnext-create-site --tail=50 | grep "Administrator"
#### 🛠️ Custom Apps (HR, Manufacturing, etc.)
#### If you need additional apps like HRMS, you'll need a custom Docker image :

yaml
# Modified values section
    values: |
      image:
        repository: your-dockerhub/erpnext-custom  # Your custom image
        tag: with-hrms
  
      createSite:
        enabled: true
        siteName: "erpnext.local"
        installApps:
          - hrms  # Install HR app after site creation
#### Build custom image:

bash
# Create apps.json
    cat > apps.json <<EOF
    [
      {"url": "https://github.com/frappe/erpnext", "branch": "version-15"},
      {"url": "https://github.com/frappe/hrms", "branch": "version-15"}
    ]
    EOF

    export APPS_JSON_BASE64=$(base64 -w 0 apps.json)

    docker build \
      --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
      --build-arg=FRAPPE_BRANCH=version-15 \
      --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
      --tag=yourdockerhub/erpnext:custom \
      -f Dockerfile .
#### ⚠️ Important Notes for Your Setup
#### RWX Storage: ERPNext needs ReadWriteMany for the sites volume . Your local-path provisioner only supports ReadWriteOnce. Options:

#### Quick fix: Deploy NFS provisioner (as shown in the helm docs) 

#### Alternative: Use single node for now (RWO works if pods schedule on same node)

#### Memory Requirements: ERPNext needs at least 4GB total . Your t3.medium (4GB) is borderline but should work for testing.

#### Persistent Volumes: The chart creates multiple PVCs :

#### Main sites volume (10GB)

#### Logs volume (2GB)

#### Database: By default, the chart deploys MariaDB internally. If you want to use an external database (recommended for production), set #### mariadbHost .

#### 🏁 Troubleshooting
bash
# Check site creation job
    kubectl logs -n erpnext -l job-name=erpnext-create-site

# Check worker logs
    kubectl logs -n erpnext -l app.kubernetes.io/component=worker-d

# If pods are pending due to storage
    kubectl describe pvc -n erpnext

# Fix storage by deploying NFS provisioner [citation:2]
    kubectl create namespace nfs
    helm repo add nfs-ganesha-server-and-external-provisioner https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner
    helm upgrade --install -n nfs in-cluster nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner \
      --set 'storageClass.mountOptions={vers=4.1}' \
      --set persistence.enabled=true \
      --set persistence.size=8Gi
#### This gets you a fully GitOps-managed ERPNext deployment. Once running, you can configure the parking module just like we discussed for #### Nextcloud, but with full ERP capabilities for the other businesses.

#### 🔧 Use the Correct Raw URL
bash
# CORRECT - Use raw.githubusercontent.com
    kubectl apply -f https://raw.githubusercontent.com/planb438/ERPNext/main/erpnext-app.yaml

#### Option 2: Apply Locally (Quickest)
bash
# Save the fixed YAML above to a local file
    nano erpnext-app-fixed.yaml
# Paste the corrected content

# Apply locally
    kubectl apply -f erpnext-app-fixed.yaml -n argocd
#### Option 3: Use Curl to Download Raw
#### bash
     curl -LO https://raw.githubusercontent.com/planb438/ERPNext/main/erpnext-app.yaml
# Edit to fix indentation
    kubectl apply -f erpnext-app.yaml -n argocd
#### ✅ After Successful Apply
bash
# Check application status
     argocd app get erpnext

# Watch pods come up
     kubectl get pods -n erpnext -w


---

# 🔥 CoreDNS is Blocking Argo CD! Classic Calico Issue
#### The error lookup argocd-repo-server on 10.96.0.10:53: i/o timeout means exactly what we've seen before - Calico is blocking DNS traffic from the Argo CD repo-server to CoreDNS.

#### Quick Fix: Restart CoreDNS and Argo CD
bash
# 1. Restart CoreDNS
    kubectl delete pods -n kube-system -l k8s-app=kube-dns

# 2. Wait for CoreDNS to be ready
    kubectl wait --for=condition=ready pod -l k8s-app=kube-dns -n kube-system --timeout=60s

# 3. Restart Argo CD repo-server
    kubectl delete pods -n argocd -l app.kubernetes.io/name=argocd-repo-server

# 4. Wait for restart
    sleep 20

# 5. Check if it's working
    kubectl get pods -n argocd

# 6. Try syncing again
    argocd app sync my-app
#### If Still Failing: Add Network Policy for DNS
bash
# Allow DNS traffic from Argo CD namespace to CoreDNS
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: argocd
spec:
  podSelector: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  policyTypes:
  - Egress
EOF

#### Alternative: Use Host Network for Argo CD
#### This bypasses Calico entirely:

bash
# Patch Argo CD repo-server to use host network
    kubectl patch deployment argocd-repo-server -n argocd \
      -p '{"spec":{"template":{"spec":{"hostNetwork":true}}}}'

# Restart
    kubectl delete pods -n argocd -l app.kubernetes.io/name=argocd-repo-server

# Wait
    sleep 20

# Test
    argocd app sync my-app
#### Complete Fix Script
#### Save as fix-argocd-dns.sh:

bash
#!/bin/bash
# Fix Argo CD DNS Resolution Issues

#### set -e

#### GREEN='\033[0;32m'
#### YELLOW='\033[1;33m'
#### BLUE='\033[0;34m'
#### NC='\033[0m'

#### log_info() { echo -e "${GREEN}[INFO]${NC} $1"; }
#### log_warn() { echo -e "${YELLOW}[WARN]${NC} $1"; }
#### log_step() { echo -e "${BLUE}[STEP]${NC} $1"; }

#### echo "========================================="
#### echo "🔧 Fixing Argo CD DNS Resolution"
#### echo "========================================="

# 1. Restart CoreDNS
#### log_step "Restarting CoreDNS..."
#### kubectl delete pods -n kube-system -l k8s-app=kube-dns
#### sleep 10
#### kubectl wait --for=condition=ready pod -l k8s-app=kube-dns -n kube-system --timeout=60s
#### log_info "✅ CoreDNS restarted"

# 2. Add network policy
#### log_step "Adding DNS network policy..."
#### cat <<EOF | kubectl apply -f -
#### apiVersion: networking.k8s.io/v1
#### kind: NetworkPolicy
#### metadata:
####   name: allow-dns
####   namespace: argocd
#### spec:
####   podSelector: {}
####   egress:
####   - to:
####     - namespaceSelector:
####         matchLabels:
####           kubernetes.io/metadata.name: kube-system
####       podSelector:
####         matchLabels:
####           k8s-app: kube-dns
####     ports:
####     - port: 53
####      protocol: UDP
####     - port: 53
####       protocol: TCP
####   policyTypes:
####   - Egress
#### EOF
#### log_info "✅ DNS network policy added"

# 3. Restart Argo CD pods
#### log_step "Restarting Argo CD pods..."
#### kubectl delete pods -n argocd --all
#### sleep 30
#### kubectl get pods -n argocd
#### log_info "✅ Argo CD pods restarted"

# 4. Test DNS from repo-server
#### log_step "Testing DNS resolution..."
#### REPO_POD=$(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-repo-server -o jsonpath='{.items[0].metadata.name}')

#### if [ -n "$REPO_POD" ]; then
####     echo "Testing DNS from repo-server pod..."
####     kubectl exec -n argocd $REPO_POD -- nslookup kubernetes.default.svc.cluster.local 2>/dev/null || echo "DNS test failed"
#### fi

# 5. Check if app sync works
#### log_step "Syncing application..."
#### argocd app sync my-app --force 2>/dev/null || log_warn "Sync failed - try manually: argocd app sync my-app --force"

#### echo ""
#### echo "========================================="
#### log_step "DNS Fix Applied!"
#### echo "========================================="
#### echo ""
#### echo "📝 If still failing, try:"
#### echo "   argocd app sync my-app --force"
#### echo "   kubectl logs -n argocd deployment/argocd-repo-server --tail=50"
#### echo ""
#### Quick One-Liner to Fix Everything
#### bash
#### kubectl delete pods -n kube-system -l k8s-app=kube-dns && \
#### kubectl delete pods -n argocd --all && \
#### sleep 30 && \
#### argocd app sync my-app --force
#### Run This Now
#### bash
#### chmod +x fix-argocd-dns.sh
#### ./fix-argocd-dns.sh
#### Summary
#### Issue	Fix
#### DNS timeout from Argo CD	Restart CoreDNS
#### Calico blocking DNS	Add network policy
#### Repo-server can't resolve	Use hostNetwork or restart pods
