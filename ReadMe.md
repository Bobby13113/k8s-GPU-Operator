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

Assign each GPU to a VM using Proxmox’s hardware settings → Add → PCI Device → Enable “Primary GPU” and “All Functions”. 2. Validate GPU Visibility in Ubuntu VM
lspci | grep -i nvidia
nvidia-smi

If nvidia-smi fails, install the NVIDIA driver manually:
sudo apt update
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall

3. Install NVIDIA Container Toolkit
   distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
   curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
   curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo systemctl restart containerd

 4. Label GPU Nodes in Kubernetes
kubectl label node <gpu-node-name> nvidia.com/gpu.present=true

5. Deploy NVIDIA GPU Operator via Helm
   helm repo add nvidia https://nvidia.github.io/gpu-operator
   helm repo update

helm install gpu-operator nvidia/gpu-operator \
 --namespace gpu-operator \
 --create-namespace

6. Verify Operator Deployment
   kubectl get pods -n gpu-operator
   kubectl describe node <gpu-node-name> | grep -i allocatable

7. Test GPU Access in a Pod
   apiVersion: v1
   kind: Pod
   metadata:
   name: gpu-test
   spec:
   containers:

- name: cuda-container
  image: nvcr.io/nvidia/cuda:12.2.0-base-ubuntu22.04
  resources:
  limits:
  nvidia.com/gpu: 1
  command: ["nvidia-smi"]

kubectl apply -f gpu-test.yaml
kubectl logs gpu-test

🛠 Troubleshooting
| | |
| nvidia-smi | |
| | kubectl logs |
| | |

📌 Notes

- The A100 and A10 GPUs require different driver versions—ensure compatibility across nodes.
- Use nvidia-smi topo -m to inspect GPU topology if deploying multi-GPU workloads.
- For advanced scheduling, consider using the NVIDIA device plugin directly.
