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