[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Ubuntu%2022.04%2B-lightgrey)](#)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-MicroK8s%20%7C%20kubeadm-blue)](#)
[![YouTube](https://img.shields.io/badge/YouTube-TechShorts-red)](https://www.youtube.com/@adaribain)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Adari%20Bain-blue)](https://www.linkedin.com/in/adari-bain-298924152/)

# 🛡️ Falco GitOps - Runtime Security

## 📋 Overview

#### Falco is a runtime security tool that monitors syscalls from the Linux kernel (via eBPF) and alerts when suspicious behavior occurs in your containers or host. This repository provides a **GitOps-managed** Falco deployment using **Argo CD**.

## What Falco Detects

#### | Rule | Description | Priority |
#### |------|-------------|----------|
#### | **Terminal shell in container** | Someone enters a shell inside a pod | WARNING |
#### | **Write below binary dir** | Modifying configs in /usr/bin, /etc | ERROR |
#### | **Launch Suspicious Network Tool** | nmap, netcat, tcpdump running in container | WARNING |
#### | **Crypto Miner Detection** | Cryptocurrency mining activity detected | CRITICAL |
#### | **Read Sensitive File** | Accessing /etc/shadow, /proc/kcore | ERROR |

---

## 🚀 Deploy via Argo CD

### Step 1: Apply the Application

#### bash
    kubectl apply -f 01-falco-application.yaml
#### Step 2: Sync
#### bash
    argocd app sync falco --force
#### Step 3: Verify
#### bash
    kubectl get pods -n falco
#### Expected Output:

#### text
    NAME          READY   STATUS    RESTARTS   AGE
    falco-xxxxx   2/2     Running   0          1m
    falco-yyyyy   2/2     Running   0          1m
    falco-zzzzz   2/2     Running   0          1m

# 🧪 Test Falco
#### Trigger a Shell Alert
#### bash
# Run a test pod with a shell
    kubectl run test-shell --rm -it --image=alpine -- sh

# Exit the shell (Ctrl+D)
#### Check the Alert
#### bash
    kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco | grep -i shell
#### Expected Output:

#### text
#### Notice A shell was spawned in a container with an attached terminal

# 📂 Repository Structure
#### text
     falco-gitops/
     ├── README.md                          # This file
     └── 01-falco-application.yaml          # Argo CD Application

# 🔧 Troubleshooting
#### Pods Stuck in Init/CrashLoopBackOff
#### bash
# Check driver logs
    kubectl logs -n falco -l app.kubernetes.io/name=falco -c falco-driver-loader

# If driver fails, try modern_ebpf
#### Edit 01-falco-application.yaml:
     driver:
       kind: modern_ebpf
#### No Alerts Being Generated
#### bash

# Check Falco logs
    kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco --tail=50

# Verify rules are loaded
    kubectl exec -n falco daemonset/falco -- cat /etc/falco/falco_rules.yaml

#### Pods Not Starting Due to Resources
#### bash

# Reduce resource limits
#### Edit 01-falco-application.yaml:
     resources:
      requests:
         memory: "128Mi"
         cpu: "50m"
       limits:
         memory: "256Mi"
         cpu: "100m"

# 🔌 Integrations
#### Slack Alerts
#### Add to 01-falco-application.yaml:

#### yaml
    falco:
      webhook:
        enabled: true
        url: "https://hooks.slack.com/services/your/webhook"
#### Prometheus Metrics
#### Add to 01-falco-application.yaml:

#### yaml
    falco-exporter:
      enabled: true
# 🧹 Cleanup
#### bash
# Delete from Argo CD
    argocd app delete falco

# Or
    kubectl delete -f 01-falco-application.yaml

# Delete namespace
    kubectl delete namespace falco
# 📚 Additional Resources
#### Falco Documentation

#### Falco Rules

#### eBPF Documentation

# 👤 Author
#### Adari Bain - CKS Certified Kubernetes Security Specialist

#### GitHub

#### LinkedIn

# 🏆 CKS Alignment
#### CKS Domain	How Falco Helps
#### Runtime Security	Monitors syscalls and detects attacks in real-time
#### System Hardening	Detects privilege escalation attempts
#### Supply Chain Security	Detects crypto miners and malicious processes
#### Monitoring & Logging	Provides audit trail of suspicious activity
#### Built with ☁️ for Production Kubernetes Security

#### text

#### ---

## 📁 File Structure
#### falco-gitops/
#### ├── README.md # Full documentation
#### └── 01-falco-application.yaml # Working Argo CD Application

#### text

---

## 🚀 Quick Commands

#### bash
# Deploy
    kubectl apply -f 01-falco-application.yaml

# Sync
    argocd app sync falco --force

# Check
    kubectl get pods -n falco

# Test
    kubectl run test-shell --rm -it --image=alpine -- sh

# View alerts
    kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco | grep -i shell
####  Falco is fully GitOps-managed! 🚀
