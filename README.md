# Air-Gapped OpenShift 4.18 Installation on VMware vSphere

[![OpenShift](https://img.shields.io/badge/OpenShift-4.18-EE0000?style=for-the-badge&logo=redhatopenshift&logoColor=white)](https://www.redhat.com/en/technologies/cloud-computing/openshift)
[![VMware](https://img.shields.io/badge/VMware-vSphere-607078?style=for-the-badge&logo=vmware&logoColor=white)](https://www.vmware.com/products/vsphere.html)
[![Quay](https://img.shields.io/badge/Quay-Mirror%20Registry-40B4E5?style=for-the-badge&logo=redhat&logoColor=white)](https://www.redhat.com/en/technologies/cloud-computing/quay)

> **Enterprise-grade disconnected OpenShift deployment with comprehensive operator support for AI/ML workloads**

---

## 📋 Executive Summary

This repository documents a complete **air-gapped (disconnected) OpenShift Container Platform 4.18** installation on **VMware vSphere** infrastructure. The deployment includes a fully configured mirror registry, mirrored operator catalogs, and support for advanced workloads including **OpenShift AI**, **GPU computing**, and **software-defined storage**.

### Key Achievements

| Component | Details |
|-----------|---------|
| **Platform** | OpenShift 4.18 (IPI on vSphere) |
| **Mirror Registry** | Red Hat Quay (Mirror Registry for Red Hat OpenShift) |
| **Storage Backend** | OpenShift Data Foundation (ODF) |
| **AI/ML Stack** | Red Hat OpenShift AI + NVIDIA GPU Operator |
| **Network Mode** | Fully Disconnected / Air-Gapped |

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           VMware vSphere Infrastructure                          │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                        Gym Member Resource Pool                            │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                                                                     │  │  │
│  │  │   ┌──────────────────┐          ┌──────────────────────────────┐   │  │  │
│  │  │   │   Bastion Host   │          │      OpenShift Cluster       │   │  │  │
│  │  │   │  192.168.252.2   │          │     (IPI Provisioned)        │   │  │  │
│  │  │   │                  │          │                              │   │  │  │
│  │  │   │  • Quay Registry │ ◄──────► │  • 3x Control Plane Nodes    │   │  │  │
│  │  │   │    (port 8443)   │          │  • 3x Worker Nodes           │   │  │  │
│  │  │   │  • HTTP Server   │          │  • Router VM                 │   │  │  │
│  │  │   │    (RHCOS OVA)   │          │                              │   │  │  │
│  │  │   │  • oc-mirror     │          │  Domain: ocpinstall.gym.lan  │   │  │  │
│  │  │   │  • Install Tools │          │                              │   │  │  │
│  │  │   └──────────────────┘          └──────────────────────────────┘   │  │  │
│  │  │                                                                     │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Deployment Workflow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   PHASE 1       │     │   PHASE 2       │     │   PHASE 3       │     │   PHASE 4       │
│   Prepare       │────►│   Mirror        │────►│   Install       │────►│   Configure     │
│   Bastion       │     │   Content       │     │   OpenShift     │     │   Operators     │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │                       │
        ▼                       ▼                       ▼                       ▼
  • Add Storage           • Mirror OCP            • IPI Install           • Apply ICSP
  • Install Quay            Platform              • Configure             • Import Catalog
  • Setup HTTP            • Mirror Operators        kubeconfig            • Install ODF
  • Download Tools        • Push to Registry                              • Install AI Stack
```

---

## 📦 Mirrored Operators

The following operators have been mirrored and are available in the disconnected environment:

### Red Hat Operator Catalog (`registry.redhat.io/redhat/redhat-operator-index:v4.18`)

| Operator | Channel | Purpose |
|----------|---------|---------|
| `odf-operator` | stable-4.18 | OpenShift Data Foundation orchestration |
| `ocs-operator` | stable-4.18 | OpenShift Container Storage |
| `odf-csi-addons-operator` | stable-4.18 | CSI add-on functionality |
| `mcg-operator` | stable-4.18 | Multi-Cloud Gateway for object storage |
| `openshift-cert-manager-operator` | stable-v1 | Certificate management |
| `local-storage-operator` | stable | Local storage provisioning |
| `nfd` | stable | Node Feature Discovery |
| `rhods-operator` | stable | Red Hat OpenShift AI |

### Certified Operator Catalog (`registry.redhat.io/redhat/certified-operator-index:v4.18`)

| Operator | Channel | Purpose |
|----------|---------|---------|
| `gpu-operator-certified` | v25.10 | NVIDIA GPU support |

---

## 🖥️ Infrastructure Details

### Network Configuration

| Host | IP Address | DNS Name | Role |
|------|------------|----------|------|
| Bastion | 192.168.252.2 | registry.gym.lan | Mirror Registry, HTTP Server |
| API VIP | 192.168.252.3 | api.ocpinstall.gym.lan | Kubernetes API |
| Ingress VIP | - | *.apps.ocpinstall.gym.lan | Application Ingress |

### Storage Allocation

| Volume | Initial Size | Final Size | Purpose |
|--------|-------------|------------|---------|
| `/dev/sda` | 50 GB | 50 GB | OS & Boot |
| `/dev/sdb` | - | 1.5 TB | Mirror Registry Data |
| **LVM Root** | 43.4 GB | **1.51 TB** | Expanded Root Filesystem |

---

## 📁 Repository Structure

```
├── README.md                    # This file - Executive summary
├── TECHNICAL_WORKFLOW.md        # Detailed step-by-step guide
├── SCRIPTS_CODEBOOK.md          # All scripts and commands
├── configs/
│   └── isc-platform-4.18.yaml   # ImageSetConfiguration for mirroring
└── images/                      # Screenshots and diagrams
    ├── vsphere-resource-pool.png
    ├── storage-expansion.png
    ├── quay-installation.png
    ├── quay-success.png
    ├── rhcos-download.png
    ├── pull-secret.png
    └── operatorhub.png
```

---

## 🚀 Quick Start

### Prerequisites

- VMware vSphere 7.x or later with sufficient resources
- RHEL 8/9 bastion host with internet access (for initial mirroring)
- Red Hat account with pull secret access
- DNS configured for cluster domain

### Deployment Steps

1. **Prepare Bastion Host**
   ```bash
   # Expand storage (after adding disk in vSphere)
   sudo vgextend rhel_bastion /dev/sdb
   sudo lvextend -l +100%FREE /dev/rhel_bastion/root
   sudo xfs_growfs /
   ```

2. **Install Mirror Registry**
   ```bash
   ./mirror-registry install --quayHostname 192.168.252.2 \
       --initUser admin --initPassword <password> \
       --sslKey /home/admin/quay.key --sslCert /home/admin/quay.crt
   ```

3. **Mirror Content**
   ```bash
   oc-mirror --config=${HOME}/isc-platform-4.18.yaml file://ocp-4.18 --v1
   oc-mirror --from=ocp-4.18/mirror_seq1_000000.tar docker://192.168.252.2:8443 --v1 --dest-skip-tls
   ```

4. **Install OpenShift**
   ```bash
   openshift-install create cluster --dir ocpinstall --log-level debug
   ```

> 📖 **For detailed instructions, see [TECHNICAL_WORKFLOW.md](TECHNICAL_WORKFLOW.md)**
> 📜 **For all scripts, see [SCRIPTS_CODEBOOK.md](SCRIPTS_CODEBOOK.md)**

---

## ✅ Verification

After installation, verify the deployment:

```bash
# Set kubeconfig
export KUBECONFIG=${HOME}/ocpinstall/auth/kubeconfig

# Check nodes
oc get nodes

# Verify OperatorHub (should show mirrored operators only)
oc get catalogsource -n openshift-marketplace

# Check mirrored operators are available
oc get packagemanifests -n openshift-marketplace
```

---

## 🔑 Key Learnings

1. **Storage Planning**: Mirroring OCP + operators requires 100+ GB; plan storage accordingly
2. **Certificate Management**: Self-signed certificates require proper SAN configuration
3. **ImageSetConfiguration**: Avoid `minVersion` for initial mirrors to prevent graph errors
4. **Operator Dependencies**: ODF requires multiple sub-operators (ocs, mcg, csi-addons)
5. **oc-mirror v1 vs v2**: Use `--v1` flag explicitly for production stability

---

## 📚 References

- [OpenShift Disconnected Installation Documentation](https://docs.openshift.com/container-platform/4.18/installing/disconnected_install/index.html)
- [Mirror Registry for Red Hat OpenShift](https://docs.openshift.com/container-platform/4.18/installing/disconnected_install/installing-mirroring-installation-images.html)
- [oc-mirror Plugin Documentation](https://docs.openshift.com/container-platform/4.18/installing/disconnected_install/installing-mirroring-disconnected.html)

---

## 📄 License

This documentation is provided as-is for educational and reference purposes.

---

## 👤 Author

**Barış Sözen**

- Deployment Date: January 3-4, 2026
- OpenShift Version: 4.18
- Environment: IBM TechZone vSphere Lab

---

*Built with ❤️ for the OpenShift community*
