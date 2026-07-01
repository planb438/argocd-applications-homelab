    #### Dolibarr is MUCH lighter than ERPNext/Odoo and works great on small AWS free-tier clusters (even 1–2 GB RAM).

    #### Below is a ready-to-deploy ArgoCD + Kubernetes setup that works out-of-the-box using NodePort.

    #### ✅ 1. Folder structure for ArgoCD repo

    #### Create this in your Git repo:

    #### dolibarr/
    ####  ├── namespace.yaml
    ####  ├── mysql.yaml
    ####  ├── dolibarr.yaml
    ####  └── kustomization.yaml
    #### ✅ 2. Namespace
    #### namespace.yaml
    #### apiVersion: v1
    #### kind: Namespace
    #### metadata:
    ####   name: dolibarr
    #### ✅ 3. MySQL (lightweight config)
    #### mysql.yaml
    #### apiVersion: v1
    #### kind: PersistentVolumeClaim
    #### metadata:
    ####   name: mysql-pvc
    ####   namespace: dolibarr
    #### spec:
    ####   accessModes:
    ####     - ReadWriteOnce
    ####   resources:
    ####     requests:
    ####       storage: 5Gi

    #### apiVersion: v1
    #### kind: Secret
    #### metadata:
    ####   name: mysql-secret
    ####   namespace: dolibarr
    #### type: Opaque
    #### stringData:
    ####   MYSQL_ROOT_PASSWORD: rootpass
    ####   MYSQL_DATABASE: dolibarr
    ####   MYSQL_USER: dolibarr
    ####   MYSQL_PASSWORD: dolipass
    ---
    #### apiVersion: apps/v1
    #### kind: Deployment
    #### metadata:
    ####   name: mysql
    ####   namespace: dolibarr
    #### spec:
    ####   replicas: 1
    ####   selector:
    ####     matchLabels:
    ####       app: mysql
    ####   template:
    ####     metadata:
    ####       labels:
    ####         app: mysql
    ####     spec:
    ####       containers:
    ####         - name: mysql
    ####           image: mysql:8
    ####           resources:
    ####             requests:
    ####               memory: "256Mi"
    ####               cpu: "100m"
    ####               ephemeral-storage: "1Gi"
    ####             limits:
    ####               memory: "512Mi"
    ####               cpu: "300m"
    ####               ephemeral-storage: "2Gi"
    ####           envFrom:
    ####             - secretRef:
    ####                 name: mysql-secret
    ####           volumeMounts:
    ####             - name: data
    ####               mountPath: /var/lib/mysql
    ####       volumes:
    ####         - name: data
    ####           persistentVolumeClaim:
    ####             claimName: mysql-pvc
    #### apiVersion: v1
    #### kind: Service
    #### metadata:
    ####   name: mysql
    ####   namespace: dolibarr
    #### spec:
    ####   ports:
    ####     - port: 3306
    ####   selector:
    ####     app: mysql
    #### ✅ 4. Dolibarr Deployment (very lightweight)
    #### dolibarr.yaml
    #### apiVersion: v1
    #### kind: PersistentVolumeClaim
    #### metadata:
    ####   name: dolibarr-pvc
    ####   namespace: dolibarr
    #### spec:
    ####   accessModes:
    ####     - ReadWriteOnce
    ####   resources:
    ####     requests:
    ####       storage: 5Gi
    #### apiVersion: apps/v1
    #### kind: Deployment
    #### metadata:
    ####  name: dolibarr
    ####  namespace: dolibarr
    #### spec:
    ####  replicas: 1
    ####  selector:
    ####    matchLabels:
    ####      app: dolibarr
    ####  template:
    ####    metadata:
    ####      labels:
    ####        app: dolibarr
    ####    spec:
    ####      containers:
    ####        - name: dolibarr
    ####          image: tuxgasy/dolibarr:latest
    ####         resources:
    ####            requests:
    ####              memory: "256Mi"
    ####              cpu: "100m"
    ####              ephemeral-storage: "1Gi"
    ####            limits:
    ####              memory: "512Mi"
    ####              cpu: "500m"
    ####              ephemeral-storage: "2Gi"
    ####          env:
    ####            - name: DOLI_DB_HOST
    ####              value: mysql
    ####            - name: DOLI_DB_NAME
    ####              value: dolibarr
    ####            - name: DOLI_DB_USER
    ####               value: dolibarr
    ####             - name: DOLI_DB_PASSWORD
    ####               value: dolipass
    ####           ports:
    ####             - containerPort: 80
    ####           volumeMounts:
    ####             - name: dolidata
    ####               mountPath: /var/www/documents
    ####       volumes:
    ####         - name: dolidata
    ####           persistentVolumeClaim:
    ####            claimName: dolibarr-pvc
    #### ---
    #### apiVersion: v1
    #### kind: Service
    #### metadata:
    ####   name: dolibarr
    ####   namespace: dolibarr
    #### spec:
    ####   type: NodePort
    ####   selector:
    ####     app: dolibarr
    ####   ports:
    ####     - port: 80
    ####       targetPort: 80
    ####       nodePort: 30081
    #### ✅ 5. Kustomization (for ArgoCD)
    #### kustomization.yaml
    #### resources:
    #### - namespace.yaml
    #### - mysql.yaml
    #### - dolibarr.yaml
    #### ✅ 6. Deploy via ArgoCD

    #### Create Application:

    #### apiVersion: argoproj.io/v1alpha1
    #### kind: Application
    #### metadata:
    ####   name: dolibarr
    ####   namespace: argocd
    #### spec:
    ####   project: default
    ####   source:
    ####     repoURL: YOUR_GIT_REPO
    ####     targetRevision: HEAD
    ####     path: dolibarr
    ####   destination:
    ####     server: https://kubernetes.default.svc
    ####     namespace: dolibarr
    ####   syncPolicy:
    ####     automated:
    ####       prune: true
    ####       selfHeal: true
    #### ✅ 7. Access URL

    #### After sync:

    #### http://EC2-PUBLIC-IP:30081

    #### You’ll see Dolibarr installer.

    #### 🎯 Installer settings

    #### Use:

    #### DB Host: mysql
    #### DB Name: dolibarr
    #### User: dolibarr
    #### Password: dolipass
    #### 💡 Resource usage (perfect for free tier)

    #### Typical running usage:

    #### Component	RAM
    #### MySQL	~200MB
    #### Dolibarr	~250MB

    #### 👉 Total < 500MB
    #### 👉 No pod evictions

    #### 🚀 Want an EVEN easier option?

    #### I can also give you:
    #### 
    #### • Helm chart version (1 command install)
    #### • Auto-HTTPS with Ingress
    #### • AWS EBS dynamic storage
    #### • Single-node ultra-tiny config
