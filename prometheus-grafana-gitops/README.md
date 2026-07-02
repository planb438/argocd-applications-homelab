# 📊 Prometheus + Grafana GitOps

## 📋 Overview

#### Complete monitoring stack deployed via GitOps using Argo CD. Includes:

| Component | Description |
|-----------|-------------|
| **Prometheus** | Metrics collection and storage |
| **Grafana** | Dashboards and visualization |
| **Alertmanager** | Alert routing and notifications |
| **Node Exporter** | Node metrics (CPU, memory, disk) |
| **Kube State Metrics** | Kubernetes API object metrics |

---

## 🚀 Deploy via Argo CD

### Step 1: Apply the Application

#### bash
    kubectl apply -f 01-prometheus-application.yaml

#### Step 2: Sync
#### bash
    argocd app sync prometheus-grafana --force

#### Step 3: Verify
#### bash
    kubectl get pods -n monitoring

#### Expected Output:

#### text
    NAME                                                            READY   STATUS    RESTARTS   AGE
    alertmanager-prometheus-grafana-gitops-alertmanager-0           2/2     Running   0          2m
    prometheus-grafana-gitops-849468495-8vn2x                       3/3     Running   0          2m
    prometheus-grafana-gitops-kube-state-metrics-7bd776b6f6-xwv7c   1/1     Running   0          2m
    prometheus-grafana-gitops-operator-d5fc58ffd-5vc2n              1/1     Running   0          2m
    prometheus-grafana-gitops-prometheus-node-exporter-44j2l        1/1     Running   0          2m
    prometheus-prometheus-grafana-gitops-prometheus-0               2/2     Running   0          2m

# 🌐 Access the Dashboards
#### Grafana
#### bash
# Get the NodePort
    kubectl get svc -n monitoring | grep grafana

# Access in browser
http://<node-ip>:32001
#### Default Credentials:

     Username: admin

     Password: prom-operator

#### Prometheus
#### bash
# Get the NodePort
    kubectl get svc -n monitoring | grep prometheus

# Access in browser
    http://<node-ip>:32000

# 📊 Pre-Installed Dashboards
#### Dashboard	Description
#### Kubernetes / Compute Resources / Cluster	Cluster resource usage
#### Kubernetes / Compute Resources / Namespace	Namespace-level resource usage
#### Kubernetes / Compute Resources / Pod	Pod-level resource usage
#### Kubernetes / Networking / Namespace	Network traffic per namespace
#### Node Exporter / Nodes	Node-level metrics

# 🔧 Configuration
#### Customize Values
#### Edit values.yaml:

#### yaml
    kube-prometheus-stack:
      prometheus:
        service:
          nodePort: 32000  # Change Prometheus port
      grafana:
        service:
          nodePort: 32001  # Change Grafana port
        adminPassword: "your-secure-password"  # Change default password

#### Add Custom Dashboards
#### Add to values.yaml:

#### yaml
    kube-prometheus-stack:
      grafana:
        sidecar:
          dashboards:
            enabled: true
            searchNamespace: ALL
        additionalDataSources:
          - name: Prometheus
            type: prometheus
            url: http://prometheus-operated.monitoring.svc:9090
            access: proxy
            isDefault: true

# 🧪 Test the Stack
#### Check Prometheus Targets
#### bash
# Port-forward to Prometheus
    kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090

# Open http://localhost:9090/targets
#### Check Grafana
#### bash
# Port-forward to Grafana
    kubectl port-forward -n monitoring svc/grafana-operated 3000:3000

# Open http://localhost:3000
#### Run a Test Query in Prometheus
#### text
#### container_cpu_usage_seconds_total

# 📈 Adding Custom Alerts
#### 1. Create Alert Rule
#### Add to values.yaml:

#### yaml
    kube-prometheus-stack:
      prometheus:
        prometheusSpec:
          additionalPrometheusRules:
            - name: cpu-threshold
              groups:
                - name: cpu-alerts
                  rules:
                    - alert: HighCPUUsage
                      expr: sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod) > 0.8
                      for: 5m
                      labels:
                        severity: warning
                      annotations:
                        summary: "High CPU usage detected"
                        description: "Pod {{ $labels.pod }} is using >80% CPU"

#### 2. Configure Alertmanager
#### yaml
    kube-prometheus-stack:
      alertmanager:
        config:
          route:
            receiver: 'slack'
            group_by: ['alertname']
            group_wait: 30s
            group_interval: 5m
            repeat_interval: 4h
          receivers:
            - name: 'slack'
              slack_configs:
                - api_url: 'https://hooks.slack.com/services/your/webhook'
                  channel: '#alerts'
                  title: 'Kubernetes Alert'
                  text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}'

# 🔧 Troubleshooting
#### Pods Not Starting
#### bash
# Check pod status
    kubectl get pods -n monitoring

# Check logs
    kubectl logs -n monitoring <pod-name>

# Check events
    kubectl describe pod -n monitoring <pod-name>
#### Can't Access Grafana
bash
# Check service
    kubectl get svc -n monitoring

# Check NodePort
    kubectl get svc -n monitoring grafana-operated -o jsonpath='{.spec.ports[0].nodePort}'

# Ensure security group allows port 32001
#### Reset Grafana Password
#### bash
# Get current password
    kubectl get secret -n monitoring grafana-admin-credentials -o jsonpath='{.data.password}' | base64 -d

# Or reset
    kubectl delete secret -n monitoring grafana-admin-credentials
# Then restart Grafana pod
    kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana

# 🧹 Cleanup
#### bash
# Delete via Argo CD
    argocd app delete prometheus-grafana

# Or
    kubectl delete -f 01-prometheus-application.yaml

# Delete namespace
    kubectl delete namespace monitoring

# 📊 Resource Usage
#### Component	CPU	Memory
#### Prometheus	~200m	~500Mi
#### Grafana	~100m	~200Mi
#### Alertmanager	~50m	~100Mi
#### Node Exporter	~20m	~50Mi
#### Kube State Metrics	~30m	~80Mi
#### Operator	~50m	~100Mi
#### Total	~450m	~1Gi

# 📚 Additional Resources
#### Prometheus Documentation

#### Grafana Documentation

#### kube-prometheus-stack

# 👤 Author
#### Adari Bain - CKS Certified Kubernetes Security Specialist

#### GitHub

#### LinkedIn

# 🏆 CKS Alignment
#### CKS Domain	How Prometheus+Grafana Helps
#### Monitoring & Logging	Metrics collection and visualization
#### Runtime Security	Detect anomalies via metrics
#### Cluster Hardening	Monitor resource usage and security posture
#### Built with ☁️ for Production Kubernetes Monitoring

#### text

---

# 📁 Complete File Structure
    prometheus-grafana-gitops/
    ├── README.md # Full documentation
    ├── 00-namespace.yaml # monitoring namespace
    ├── 01-prometheus-application.yaml # Argo CD Application
    ├── Chart.yaml # Helm chart definition
    └── values.yaml # Helm values

#### text

---

# 🚀 Deploy

#### bash
# 1. Apply the Application
    kubectl apply -f 01-prometheus-application.yaml

# 2. Sync
    argocd app sync prometheus-grafana --force

# 3. Check
    kubectl get pods -n monitoring

# 4. Access Grafana
    kubectl get svc -n monitoring
# Open http://<node-ip>:32001

#### monitoring stack is now fully GitOps-managed! 🚀
