[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)

# 🛡️ Falco GitOps - Runtime Security

## 📋 Overview

#### Falco is a runtime security tool that monitors syscalls from the Linux kernel (via eBPF) and alerts when suspicious behavior occurs in your containers or host.

#### This repository provides a **GitOps-managed** Falco deployment using Argo CD, including:

#### - ✅ Falco DaemonSet with eBPF driver (EC2-optimized)
#### - ✅ Custom security rules
#### - ✅ Event forwarding via Falcosidekick
#### - ✅ Prometheus metrics integration
#### - ✅ Network Policy for event forwarding

---

## 🚀 Quick Start

### Prerequisites

#### - Kubernetes cluster (1.28+)
#### - Argo CD installed and configured
#### - kubectl configured

### Deploy Falco via Argo CD

#### bash
# 1. Clone this repository
    git clone https://github.com/CloudKanata/falco-gitops.git
    cd falco-gitops

# 2. Apply the Argo CD Application
    kubectl apply -f 01-falco-helm-application.yaml

# 3. Sync via Argo CD
    argocd app sync falco --force

#### Verify Installation
#### bash
# Check Falco pods are running
    kubectl get pods -n falco-system

# Check Falco logs
    kubectl logs -n falco-system -l app.kubernetes.io/name=falco --tail=20

# Check event forwarder
    kubectl get pods -n falco-monitoring
# 🧪 Test Falco
#### Trigger a Shell Alert
#### bash
# Run a test pod with a shell
    kubectl run test-shell --rm -it --image=alpine -- sh

# Once inside, exit (Ctrl+D)
#### Check the Alert
#### bash
# View Falco logs
    kubectl logs -n falco-system -l app.kubernetes.io/name=falco --tail=50

# You should see:
# "Notice A shell was spawned in a container with an attached terminal"
#### Test Custom Rules
#### bash
# Simulate a crypto miner (should trigger CRITICAL alert)
    kubectl run test-miner --rm -it --image=alpine -- sh -c "echo 'stratum+tcp://xmr.pool.minergate.com:45560' > /tmp/miner-config.json"

# Check Falco logs
    kubectl logs -n falco-system -l app.kubernetes.io/name=falco --tail=50 | grep -i crypto

# 📊 What Falco Catches
#### Rule	Description	Priority
#### Terminal shell in container	Someone enters a shell inside a pod	WARNING
#### Write below binary dir	Modifying configs in /usr/bin, /etc	ERROR
#### Launch Suspicious Network Tool	nmap, netcat, tcpdump running in container	WARNING
#### Crypto Miner Detection	Cryptocurrency mining activity detected	CRITICAL
#### Read Sensitive File Untrusted	Accessing /etc/shadow, /proc/kcore	ERROR
#### Inbound Connections to High Ports	Suspicious network access	WARNING

# 🔧 Custom Rules
#### Custom rules are defined in 03-falco-rules-config.yaml. To add or modify rules:

#### Edit 03-falco-rules-config.yaml in Git repository

#### Commit and push changes

#### Argo CD will auto-sync the ConfigMap

#### Restart Falco pods to pick up changes:

#### bash
    kubectl delete pods -n falco-system -l app.kubernetes.io/name=falco

# 🔌 Event Forwarding
#### Falco events are forwarded to falco-event-forwarder (Falcosidekick) which can send alerts to:

#### Slack

#### Microsoft Teams

#### Datadog

#### Webhook endpoints

#### And more...

#### Configure Slack Integration
#### Edit 04-falco-event-forwarder.yaml:

#### yaml
    env:
    - name: SLACK_WEBHOOKURL
      value: "https://hooks.slack.com/services/your/webhook/url"

# 📈 Monitoring
#### Falco Exporter exposes Prometheus metrics at :9376.

#### Prometheus Integration
#### Add this to your Prometheus configuration:

#### yaml
    scrape_configs:
    - job_name: 'falco-exporter'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - falco-system
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
        action: keep
        regex: falco-exporter
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2

#### Grafana Dashboard
#### Import Falco dashboard from Grafana Labs:

#### Dashboard ID: 12345 (Falco dashboard)

#### Select Prometheus data source

#### Import

# 🧹 Cleanup
#### bash
# Delete Falco from Argo CD
    kubectl delete -f 01-falco-helm-application.yaml

# Or use Argo CD CLI
    argocd app delete falco

# Clean up namespaces (if needed)
    kubectl delete namespace falco-system
    kubectl delete namespace falco-monitoring

# 📚 Additional Resources
#### Falco Documentation

#### Falco Rules

#### Falcosidekick

#### CKS Runtime Security

# 📄 License
#### MIT License - Use freely for learning and production.

# 👤 Author
#### Adari Bain - CKS Certified Kubernetes Security Specialist

#### GitHub

#### LinkedIn

# 🔄 What's Next
#### After Falco is deployed, consider:

#### Component	Purpose
#### Slack/Teams Integration	Real-time alert notifications
#### Prometheus/Grafana	Falco metrics visualization
#### Kyverno Policies	Block malicious deployments before runtime
#### Network Policies	Restrict traffic to/from Falco components
#### Sealed Secrets	Secure webhook URLs in Git
#### Built with ☁️ for Production Kubernetes Security

text

---

## 🚀 Deployment Commands Summary

#### bash
# Clone the repo
    git clone https://github.com/CloudKanata/falco-gitops.git
    cd falco-gitops

# Deploy via Argo CD
    kubectl apply -f 01-falco-helm-application.yaml

# Force sync
    argocd app sync falco --force

# Verify
    kubectl get pods -n falco-system
    kubectl get pods -n falco-monitoring

# Test
    kubectl run test-shell --rm -it --image=alpine -- sh
    kubectl logs -n falco-system -l app.kubernetes.io/name=falco --tail=20 | grep -i shell


# 📁 Repository Structure

    falco-gitops/
    ├── README.md                          # This file
    ├── 00-namespace.yaml                  # Falco namespace
    ├── 01-falco-helm-application.yaml     # Argo CD Application (Helm)
    ├── 02-falco-helm-values.yaml          # Helm values (reference)
    ├── 03-falco-rules-config.yaml         # Custom Falco rules
    ├── 04-falco-event-forwarder.yaml      # Falcosidekick + NetworkPolicy
    └── kustomization.yaml                 # Kustomize resources

 #### Falco is now fully GitOps-managed via Argo CD! 🚀

