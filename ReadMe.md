Deploying the NVIDIA GPU Operator on Kubernetes with Proxmox PCI Passthrough for A100 and A10 GPUs: A Comprehensive Technical Guide

Overview
Deploying the NVIDIA GPU Operator on a self-hosted Kubernetes cluster that leverages PCI passthrough under Proxmox for NVIDIA A100 and A10 GPUs requires careful orchestration of several components. This guide walks you through compatibility verification, OS and kernel configuration, Kubernetes cluster preparation, GPU Operator installation (with Helm), container runtime setup, Node Feature Discovery enablement, node labeling, resource management, MIG configuration, monitoring, and troubleshooting—all tailored for Ubuntu 22.04, Proxmox VE, and your specific hardware scenario.
We integrate the latest product compatibility notes and operational best practices as of October 2025, focusing especially on issues and configurations relevant to the coexistence of A100 (40GB) and A10 (20GB) cards in a Proxmox virtualized environment using PCI passthrough.

1. System, Hardware, and Compatibility Preparation
   1.1 Verifying Hardware and Proxmox PCI Passthrough Compatibility
   PCI passthrough under Proxmox requires compatible CPU and motherboard features, along with specific BIOS and kernel settings. Both Intel and AMD platforms are supported, with VT-d (Intel) or AMD-Vi (AMD) and IOMMU support being mandatory.
   Essential BIOS Settings Checklist

- Enable Virtualization Extensions:
- Intel: VT-d
- AMD: AMD-Vi/SVM
- Enable IOMMU:
- Found under North Bridge/Advanced settings.
- Enable Above 4G Decoding:
- Required for high-end/multi-GPU systems.
- Enable ACS (Access Control Services):
- For precise IOMMU grouping, especially with multi-GPU setups.
- UEFI Boot Mode and Secure Boot:
- Use UEFI (OMVF) for VMs with GPU PCI passthrough.
- Disable Secure Boot: Required for loading NVIDIA drivers.
  Proxmox Host Preparation
- Keep BIOS and Proxmox up-to-date:
  apt-get update
  apt-get dist-upgrade
- Reboot after updates and BIOS configuration.
- Verify IOMMU is enabled on the host:
  dmesg | grep -e DMAR -e IOMMU

- Expect to see lines like DMAR: IOMMU enabled.
  PCI Passthrough Configuration Steps- Edit Grub or systemd-boot kernel parameters:
- For Intel:
  GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

- For AMD:
  GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"

- For some motherboards and multi-GPU setups, append:
  pci_acs_override=downstream,multifunction

- Update bootloader and reboot:
  update-grub # for GRUB

# or

proxmox-boot-tool refresh # for systemd-boot
reboot

- Confirm that dmesg | grep 'remapping' shows interrupt remapping enabled.
  Note: All parameters must be on a single line. Incorrect or split-line configurations will be ignored.IOMMU Group and GPU Isolation- List all groups:
  find /sys/kernel/iommu_groups/ -type l | sort -V

- Check individual device assignments:
  lspci -nn

- Each GPU must reside in its own IOMMU group, ideally not grouped with critical system devices (USB, storage controllers, etc.).
- If the group is shared, move the GPU to a different slot or use ACS override.
  Kernel Module and Driver Blacklisting- Prevent the Proxmox host from grabbing the GPU:
- Edit /etc/modprobe.d/blacklist.conf:
  blacklist nouveau
  blacklist nvidia
  blacklist nvidiafb
  blacklist radeon
  blacklist i915

- Edit /etc/modules to enable VFIO modules:
  vfio
  vfio_iommu_type1
  vfio_pci
  vfio_virqfd

- Optionally, pin the PCI device to vfio-pci by adding its device/vendor IDs to /etc/modprobe.d/vfio.conf:
  options vfio-pci ids=10de:20f2,10de:1aef disable_vga=1

- Update initramfs and reboot:
  update-initramfs -u -k all
  reboot

- After reboot, confirm with:
  lspci -nnk | grep 'NVIDIA'
- Ensure Kernel driver in use: vfio-pci.
  Assign GPU to Virtual Machine- In Proxmox VM configuration file (e.g., /etc/pve/qemu-server/101.conf):
  hostpci0: 01:00,x-vga=on,pcie=1
- Always use OVMF (UEFI) BIOS for the VM and set VM machine type to q35.
- Assign all necessary GPU/Audio PCI functions.
- Set CPU type to host in the VM configuration for maximum compatibility with CUDA and AVX/AVX2 vector extensions.

2. Ubuntu 22.04 Guest and Container Runtime Preparation2.1 OS Installation and Base ConfigurationAfter PCI passthrough, install Ubuntu 22.04 on the Kubernetes worker (VM) nodes. Use the minimal server ISO for stability and minimal interference.Validate GPU Device Visibility- Inside the guest, verify with:
   lspci | grep NVIDIA
   NVIDIA Driver Installation- Identify the correct driver version for both A100 and A10:
   As of Q4 2025, driver versions from the R570 and R580 branches (e.g., 570.86.15, 580.82.07) are officially supported and recommended by the GPU Operator for both cards. Use the same driver branch/version on all nodes for consistency.

- Install kernel headers and update:
  sudo apt-get install linux-headers-$(uname -r)
  sudo apt update
  sudo apt install nvidia-driver-580
- or use the Ubuntu PPA for graphics drivers, or ubuntu-drivers autoinstall to let Ubuntu choose the optimal version.
- Hold NVIDIA packages to prevent accidental upgrades:
  sudo apt-mark hold nvidia-driver-580 nvidia-utils-580
- Validate driver with:
  nvidia-smi
- You should see both GPUs (A100 40GB and A10 20GB) listed with correct driver and CUDA version information.
  Container Runtime: containerd with NVIDIA Runtime- Install containerd (preferred):
  sudo apt-get install containerd
- Install NVIDIA Container Toolkit:
  curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
  curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
   sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
   sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
  sudo apt update
  sudo apt install -y nvidia-container-toolkit
  - Configure containerd with NVIDIA runtime:
  sudo nvidia-ctk runtime configure --runtime=containerd
  sudo systemctl restart containerd
  sudo systemctl status containerd
  cat /etc/containerd/config.toml | grep "containerd.runtimes.nvidia"
- Updates config.toml to use:
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  privileged_without_host_devices = false
  runtime_engine = ""
  runtime_root = ""
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
- Test direct container GPU access:
  sudo nerdctl run -it --rm --gpus all nvidia/cuda:12.3.1-base-ubuntu20.04 nvidia-smi
  - If using Docker as a fallback (not recommended for production K8s):
- Install and configure nvidia-docker2 and set "default-runtime": "nvidia" in /etc/docker/daemon.json.

3. Kubernetes Cluster Setup and GPU Operator Deployment3.1 Kubernetes Preparation- Networking: Ensure worker VMs have static or DHCP-assigned IPs, with full pod-to-pod and service networking.

- Version Matrix: Use a compatible Kubernetes version (1.29–1.33 recommended for Ubuntu 22.04 and GPU Operator v25.3.x).
- Install kubectl, helm, etc.
  3.2 Node Feature Discovery (NFD) for GPU Labeling- NFD is required for the GPU Operator to auto-label GPU nodes appropriately. It will be installed automatically by the GPU Operator Helm chart unless explicitly disabled.
- Check existing NFD deployment:
  kubectl get nodes -o json | jq '.items[].metadata.labels | keys | any(startswith("feature.node.kubernetes.io"))'
  - If true, NFD is already present; otherwise it will be installed anew.
  3.3 Installing the NVIDIA GPU OperatorAdd NVIDIA Helm repositoryhelm repo add nvidia https://helm.ngc.nvidia.com/nvidia
  helm repo update
  Default installation (Driver and Toolkit managed by Operator)kubectl create namespace gpu-operator
  kubectl label --overwrite ns gpu-operator pod-security.kubernetes.io/enforce=privileged
  helm install gpu-operator nvidia/gpu-operator \
   -n gpu-operator \
   --create-namespace \
   --version=v25.3.4
  Using Pre-installed Drivers and Container ToolkitIf you've already installed the correct NVIDIA drivers and container toolkit on the host (recommended with custom PCI passthrough/Proxmox setups), disable these in Operator:helm install gpu-operator nvidia/gpu-operator \
   -n gpu-operator \
   --create-namespace \
   --version=v25.3.4 \
   --set driver.enabled=false \
   --set toolkit.enabled=false
  Custom containerd/NFD/MIG/Toolkit ParametersExample with custom containerd runtime settings:helm install gpu-operator nvidia/gpu-operator \
   -n gpu-operator \
   --create-namespace \
   --version=v25.3.4 \
   --set toolkit.env[0].name=CONTAINERD_CONFIG \
   --set toolkit.env[0].value=/etc/containerd/config.toml \
   --set toolkit.env[1].name=CONTAINERD_SOCKET \
   --set toolkit.env[1].value=/run/containerd/containerd.sock \
   --set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS \
   --set toolkit.env[2].value=nvidia \
   --set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT \
   --set-string toolkit.env[3].value=true
  Key custom values for A100/A10 and MIG in YAML (values.yaml) format:driver:
  version: "580.82.07"
  repository: nvcr.io/nvidia
  mig:
  strategy: mixed
  migManager:
  enabled: true
  dcgmExporter:
  enabled: true
  toolkit:
  enabled: false
  You can supply this to helm with -f custom-values.yaml.Check installation progress:kubectl get pods -n gpu-operator
  All relevant pods should be in Running or Completed state (e.g., nvidia-driver-daemonset, nvidia-device-plugin-daemonset, nvidia-mig-manager, nvidia-dcgm-exporter, etc.).Sample pod list after successful installation:| | |
  | | |
  | | |
  | | |
  | | |
  | | |
  | | |
  | | |
  | | |

4. Advanced Node Feature Discovery and Label Customization4.1 Node Labels and Taints- Node Taints for GPU Exclusivity:
   kubectl taint nodes <gpu-node> nvidia.com/gpu:NoSchedule

- This prevents pods from being scheduled unless they tolerate the taint.
- Pod Tolerations for GPU Workloads:
  spec:
  tolerations: - key: nvidia.com/gpu
  operator: Exists
  effect: NoSchedule
- Custom Node Labels for Workload Management:
- For fine-grained scheduling or MIG resource mapping, label nodes accordingly:
  kubectl label node <node-name> accelerator=nvidia-a100
  kubectl label node <node-name> nvidia.com/mig.config=all-1g.5gb

# For device plugin configuration, e.g. time-slicing:

kubectl label node <node-name> nvidia.com/device-plugin.config=a100-tesla

- Use nvidia.com/gpu.workload.config=container for GPU Operator container workloads.
  4.2 Verifying Discovery- List all allocatable GPU resources:
  kubectl get node <node-name> -o json | jq '.status.allocatable | with_entries(select(.key | startswith("nvidia.com/"))) | with_entries(select(.value != "0"))'
  - Inspect node labels for nvidia.com/gpu.product, nvidia.com/gpu.memory, etc.

5. Multi-Instance GPU (MIG) Configuration for A1005.1 About MIG- The A100 supports partitioning into up to 7 instances, each isolated and independently addressable via Kubernetes.

- MIG enables fine-grained allocation of GPU resources for multi-tenant and mixed-workload clusters.
  5.2 MIG Manager Strategies- Single Strategy: All GPUs on the node have identical partitioning profiles. Simple and compatible, but less flexible.
- Mixed Strategy: Different GPUs on the node can have different MIG geometries. Maximizes resource utilization, suitable for heterogeneous clusters.
  5.3 Enabling and Configuring MIG- Enable MIG mode on device:
  sudo nvidia-smi -mig 1
- Create desired profiles/partitions:
- E.g., to partition into 7x 1g.5gb slices.
- Configure GPU Operator to manage MIG:
- During Helm install, specify:
  --set mig.strategy=mixed
  --set migManager.enabled=true
- Optionally, create a ConfigMap describing custom MIG configurations, and patch the ClusterPolicy CRD accordingly.
- Label node with chosen MIG profile:
  kubectl label node <node-name> nvidia.com/mig.config=all-1g.5gb --overwrite
- Check with:
  kubectl get node <node-name> -o json | jq '.metadata.labels'
  kubectl exec -it -n gpu-operator ds/nvidia-driver-daemonset -- nvidia-smi -L
- Look for distinct MIG device UUIDs and MIG device types.
  5.4 MIG Usage Notes- Pod requests for MIG devices must use resource names matching exposed profiles:
  resources:
  limits:
  nvidia.com/mig-1g.5gb: 1
- Driver and Device Plugin Considerations:
- Certain driver versions (e.g., some R570 releases) have known bugs in mixed full-GPU + MIG setups. For maximum stability, use R570.86.15 or 580.82.07.
- Upgrading MIG configuration or switching strategies may require draining/rebooting nodes.

6. A10 GPU ConsiderationsThe A10, although Ampere-based, does not support NVIDIA's MIG technology (unlike the A100) but otherwise follows similar Operator and driver procedures.- Resource exposure: Each A10 will appear as one allocatable resource (e.g., nvidia.com/GA102GL_A10: 1).

- For scheduling, request the A10 GPU resource by its product label.
- vGPU is supported for A10 (not commonly used in PCI passthrough scenarios).
- The A10 is apt for inference workloads and lower TDP servers, providing cost-efficient access.

7. Scheduling, Resource Quotas, and Pod Best Practices7.1 Pod Resource Requests- Standard Pod Spec for Using a GPU:
   spec:
   runtimeClassName: nvidia
   containers:

- name: my-gpu-job
  image: nvidia/cuda:12.3.1-base-ubuntu20.04
  resources:
  limits:
  nvidia.com/gpu: 1
  nodeSelector:
  kubernetes.io/hostname: <gpu-node>
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
- For a specific MIG instance:
  resources:
  limits:
  nvidia.com/mig-1g.5gb: 1
  7.2 Quota Management- Namespace GPU resource quota:
  apiVersion: v1
  kind: ResourceQuota
  metadata:
  name: compute-resources
  spec:
  hard:
  requests.nvidia.com/gpu: 4
- Only the requests prefix is supported for extended resources like GPUs in K8s quotas.

8. Monitoring: DCGM Exporter and Observability8.1 DCGM ExporterThe Data Center GPU Manager (DCGM) Exporter integrates with the Operator and exposes GPU telemetry (utilization, memory, power, thermals, errors, MIG state, etc.) via Prometheus metrics.Helm Installation (if not deployed with Operator):helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
   helm repo update
   helm install dcgm-exporter nvidia/dcgm-exporter
   Example Custom Metrics ConfigurationProvide a ConfigMap (in gpu-operator namespace):apiVersion: v1
   kind: ConfigMap
   metadata:
   name: dcgm-exporter-metrics
   data:
   dcp-metrics-included.csv: |
   DCGM_FI_DEV_SM_CLOCK, gauge, SM clock frequency (MHz)
   DCGM_FI_DEV_GPU_UTIL, gauge, GPU utilization (%)
   DCGM_FI_DEV_FB_FREE, gauge, Framebuffer memory free (MiB)
   DCGM_FI_DEV_FB_USED, gauge, Framebuffer memory used (MiB)
   Attach to your DCGM exporter configuration using Helm:helm install dcgm-exporter nvidia/dcgm-exporter \
    --set customMetricsConfigMap.name=dcgm-exporter-metrics
   Expose Prometheus and Grafana dashboards as needed to visualize GPU data.9. Troubleshooting GPU Operator, PCI Passthrough, and Device Plugin Issues9.1 Validating PCI and Proxmox Passthrough- Confirm in guest:
   lspci | grep NVIDIA
   nvidia-smi

- Check for kernel errors:
  dmesg | grep NVRM
  dmesg | grep Xid
  9.2 Operator, Plugin, or Device IssuesPods Stuck in Init or CrashLoopBackOff- Logs:
- kubectl logs -n gpu-operator <pod>
- Typical issues:
- Driver module errors: Check logs and dmesg.
- Incompatible kernel or secure boot blocking driver loading.
- Missing runtime class: "no runtime for 'nvidia' is configured" — check containerd configuration.
- Hardware marked as unhealthy (XidCriticalError in logs).
  Node Not Advertising Expected GPU- Node allocatable GPUs don't match expectation? Look for Xid errors, possible hardware faults, or configuration mismatches.
  DCGM, Operator, or Plugin Not Starting- Check for node taints blocking DaemonSets (nvidia.com/gpu:NoSchedule without proper tolerations).
- Confirm network/firewall settings for pod communication.
  Pods in Pending State- Known bug with specific R570 drivers and mixed MIG/full-GPU resource exposure (use R570.86.15 or newer R580).
  Secure Boot Issues- Disable EFI Secure Boot. NVIDIA's kernel modules are not signed for Secure Boot.
  General Debugging- Use the must-gather script from NVIDIA's GPU Operator repo for a complete cluster dump:
  curl -o must-gather.sh -L https://raw.githubusercontent.com/NVIDIA/gpu-operator/main/hack/must-gather.sh
  chmod +x must-gather.sh
  ./must-gather.sh
  Proxmox-Specific Issues- Double-check kernel parameters and module assignments in both host and guest.
- Use Proxmox forums and wikis for host/PCI enumeration troubles and version-specific gotchas.

10. Best Practices and Recommendations10.1 Use Consistent Driver and CUDA VersionsKeep all worker VMs on matched driver/CUDA versions to avoid plugin incompatibility.10.2 Secure Host Management- Always ensure alternate console access to Proxmox—once PCI devices are passed through, the host may lose display output.

- Never pass through the only GPU controlling Proxmox console unless remote, serial, or secondary GPU access is already arranged.
  10.3 Node Labeling/Tainting Hygiene- Use taints and node selectors to control workloads and Operator component scheduling.
- Do not manually "split" a single GPU between multiple VMs or containers unless using MIG (for A100) or time-slicing with the device plugin.
  10.4 Compatibility Watch- Review NVIDIA Operator documentation for supported K8s/driver/containerd versions before major upgrades.
- Pin/tune Proxmox or Linux kernel versions if new releases cause breakage in passthrough support.

11. Key Configuration Summary Table| | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |
    | | | | |

ConclusionSuccessfully deploying the NVIDIA GPU Operator in a Proxmox-based, PCI passthrough-enabled environment, managing both the A100 (40GB) and A10 (20GB) GPUs, hinges on meticulously aligning host/VM configuration, OS compatibility, driver/container runtime management, and Kubernetes ecosystem integration. Paying close attention to BIOS and kernel settings, blacklisting competing drivers, matching driver versions, enabling and validating IOMMU and VFIO, and ensuring appropriate node labeling and scheduling policies are all critical for seamless, robust operation.With this guide, you should be able to orchestrate highly efficient, multi-GPU Kubernetes clusters—optimally managing heterogeneous A100 and A10 GPU resources from within a secure, scalable, and monitorable infrastructure.For further scenario-specific optimization (e.g., advanced time-slicing, dynamic MIG reconfiguration, or specialized KubeVirt/vGPU integration), consult the official documentation and community knowledge bases linked above. Keeping abreast of GPU Operator, Proxmox, and container toolkit release notes is essential for maintaining continued compatibility and performance stability.
