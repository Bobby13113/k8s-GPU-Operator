# NVIDIA GPU Operator Setup for Proxmox-Based Kubernetes Cluster

This guide walks through deploying the NVIDIA GPU Operator in a self-hosted Kubernetes cluster running on Ubuntu 22.04, with PCI passthrough-enabled NVIDIA A100 (40 GB) and A10 (20 GB) GPUs. The cluster is virtualized via Proxmox (Type 1 hypervisor) and consists of one controller and two worker nodes.

## 🧠 Architecture Overview

| Component        | Details                          |
| ---------------- | -------------------------------- |
| Hypervisor       | Proxmox VE (Type 1)              |
| Guest OS         | Ubuntu 22.04                     |
| Kubernetes Nodes | 1 Controller, 2 Workers          |
| GPUs             | NVIDIA A100 (40 GB), A10 (20 GB) |
| GPU Access       | PCI Passthrough to K8s Nodes     |

## 🚀 Prerequisites

- Proxmox VE with PCI passthrough configured for both GPUs
- Ubuntu 22.04 installed on all K8s nodes
- Kubernetes cluster initialized and operational
- Helm installed on the controller node
- Internet access for pulling container images

## 🔧 Step-by-Step Setup

### 1. Enable PCI Passthrough in Proxmox

Ensure IOMMU is enabled and passthrough is configured:

```bash
# Add to /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# Update GRUB
update-grub

# Load vfio modules
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules
```
