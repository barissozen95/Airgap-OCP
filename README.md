# 🔒 OpenShift Air-Gapped Installation Guide

[![OpenShift](https://img.shields.io/badge/OpenShift-4.18-EE0000?style=for-the-badge&logo=red-hat-open-shift&logoColor=white)](https://www.redhat.com/en/technologies/cloud-computing/openshift)
[![vSphere](https://img.shields.io/badge/VMware-vSphere-607078?style=for-the-badge&logo=vmware&logoColor=white)](https://www.vmware.com/products/vsphere.html)
[![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=for-the-badge)](http://makeapullrequest.com)

> **A comprehensive, production-ready guide for deploying Red Hat OpenShift Container Platform 4.18 in completely isolated (air-gapped) environments using IPI on VMware vSphere.**

In highly regulated industries like finance, government, healthcare, and defense, organizations must operate critical infrastructure in air-gapped networks—environments with zero internet connectivity. This repository provides a complete, battle-tested guide for successfully deploying OpenShift in these challenging scenarios.

---

## 📋 Table of Contents

- [Overview](#-overview)
- [What You'll Build](#-what-youll-build)
- [Prerequisites](#-prerequisites)
- [Architecture](#-architecture)
- [Quick Start](#-quick-start)
- [Detailed Guide](#-detailed-guide)
- [Pros & Cons](#-pros--cons)
- [Alternative Approach](#-alternative-approach)
- [Troubleshooting](#-troubleshooting)
- [Contributing](#-contributing)
- [Resources](#-resources)
- [License](#-license)

---

## 🎯 Overview

This guide walks you through the **complete end-to-end process** of installing a production-ready OpenShift cluster in an air-gapped environment, including:

- ✅ Mirror registry setup and configuration
- ✅ Content mirroring (platform images, operators, catalogs)
- ✅ Cluster installation with IPI (Installer Provisioned Infrastructure)
- ✅ Infrastructure node configuration
- ✅ OpenShift Data Foundation (ODF) deployment
- ✅ Post-installation hardening and optimization

**Time Required:** 4-6 hours (mostly automated, mirroring is the longest phase)  
**Difficulty Level:** Advanced  
**Target Audience:** Platform engineers, DevOps teams, infrastructure architects

---

## 🏗️ What You'll Build

By following this guide, you'll deploy a **9-node production cluster** with the following specifications:

### Cluster Architecture

| Node Type | Count | vCPU | Memory | Disk | Purpose |
|-----------|-------|------|--------|------|---------|
| **Control Plane** | 3 | 4 | 16 GB | 120 GB | Cluster management, etcd, API servers |
| **Compute** | 3 | 4 | 16 GB | 120 GB | Application workloads |
| **Infrastructure** | 3 | 16 | 64 GB | 120 GB | Registry, monitoring, logging, routing |

**Total Resources:** 96 vCPUs, 288 GB RAM, 1.08 TB storage

### Key Features

- 🔐 **Complete network isolation** - Zero internet connectivity
- 📦 **Local mirror registry** - Quay-based container registry
- 🎯 **IPI deployment** - Fully automated infrastructure provisioning
- 💾 **Persistent storage** - OpenShift Data Foundation (Ceph)
- 📊 **Production-ready** - Separated infrastructure workloads
- 🔄 **Operator support** - Pre-mirrored operator catalogs

---

## ✅ Prerequisites

### Infrastructure Requirements

- **VMware vSphere:** Version 7.0 or later
- **Bastion Host:** RHEL 8/9 with initial internet access
- **Storage:** Minimum 1.5 TB for mirror content
- **Network:** Configured DNS, DHCP, and network segmentation capability
- **Access:** vCenter admin credentials

### Required Knowledge

- Linux system administration
- VMware vSphere operations
- Container and Kubernetes concepts
- Networking fundamentals
- OpenShift architecture basics

### Software Versions

- OpenShift: `4.18.0` (stable)
- RHCOS: `4.18`
- oc-mirror: `latest`
- Mirror Registry: `latest`

---

## 🏛️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   AIR-GAP BARRIER                            │
│                   (No Internet)                              │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │
                ┌───────────┴───────────┐
                │                       │
                │   Bastion Host        │
                │   ┌─────────────────┐ │
                │   │ Mirror Registry │ │
                │   │   (Quay)        │ │
                │   │ :8443          │ │
                │   └─────────────────┘ │
                │   ┌─────────────────┐ │
                │   │ HTTP Server     │ │
                │   │ RHCOS OVA       │ │
                │   │ :80            │ │
                │   └─────────────────┘ │
                └───────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌──────▼──────┐   ┌───────▼────────┐
│ Control Plane  │  │   Compute   │   │ Infrastructure │
│   Nodes (3)    │  │  Nodes (3)  │   │   Nodes (3)    │
│                │  │             │   │                │
│  • etcd        │  │ • Apps      │   │ • Registry     │
│  • API         │  │ • Workloads │   │ • Monitoring   │
│  • Controllers │  │             │   │ • Logging      │
└────────────────┘  └─────────────┘   └────────────────┘
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
                    ┌───────▼────────┐
                    │  ODF Storage   │
                    │  (Ceph Cluster)│
                    │                │
                    │ • RBD (Block)  │
                    │ • CephFS (RWX) │
                    │ • NooBaa (S3)  │
                    └────────────────┘
```

---

## 🚀 Quick Start

### 1. Clone This Repository

```bash
git clone https://github.com/yourusername/openshift-airgapped-guide.git
cd openshift-airgapped-guide
```

### 2. Prepare Bastion Host

```bash
# Add storage (1.5TB) via vSphere GUI
# Then extend the filesystem
sudo vgextend rhel_bastion /dev/sdb
sudo lvextend -l +100%FREE /dev/rhel_bastion/root
sudo xfs_growfs /
```

### 3. Install Mirror Registry

```bash
# Generate TLS certificate
openssl req -newkey rsa:4096 -nodes -sha256 -keyout ~/quay.key \
    -x509 -out ~/quay.crt -days 3650 \
    -subj "/O=gym,CN=registry.gym.lan" \
    -addext "subjectAltName = DNS:registry.gym.lan,IP:192.168.252.2"

# Download and install
MIRROR_DIR=$(mktemp -d)
curl -Lo ${MIRROR_DIR}/mirror-registry.tar.gz \
    https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz
cd ${MIRROR_DIR} && tar xf mirror-registry.tar.gz
./mirror-registry install --quayHostname 192.168.252.2 \
    --initUser admin --initPassword QuayForAll! \
    --sslKey ~/quay.key --sslCert ~/quay.crt
```

### 4. Mirror Content

```bash
# Download oc-mirror
OCP_VERSION=stable-4.18
curl -Lo oc-mirror.tar.gz \
    https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/${OCP_VERSION}/oc-mirror.tar.gz
tar xzf oc-mirror.tar.gz && chmod +x oc-mirror && sudo mv oc-mirror /usr/local/bin/

# Mirror OpenShift content (2-4 hours)
oc-mirror --config=imageset-config.yaml file://ocp-4.18
oc-mirror --from=file://ocp-4.18 docker://192.168.252.2:8443
```

### 5. Install Cluster

```bash
# Configure air-gap (block internet)
# Create install-config.yaml (see docs/)
openshift-install create cluster --dir ~/ocp-install --log-level=info
```

📚 **For detailed step-by-step instructions, see the [Complete Guide](docs/COMPLETE_GUIDE.md)**

---

## 📖 Detailed Guide

This repository includes comprehensive documentation:

- **[Complete Installation Guide](docs/COMPLETE_GUIDE.md)** - Full walkthrough with screenshots
- **[Mirror Registry Setup](docs/01-mirror-registry.md)** - Detailed registry configuration
- **[Content Mirroring](docs/02-content-mirroring.md)** - Image and operator mirroring
- **[Cluster Installation](docs/03-cluster-install.md)** - IPI deployment steps
- **[Post-Installation](docs/04-post-install.md)** - Infrastructure nodes and ODF
- **[Troubleshooting Guide](docs/TROUBLESHOOTING.md)** - Common issues and solutions
- **[Configuration Examples](examples/)** - Sample YAML files and configs

---

## ⚖️ Pros & Cons

### ✅ Advantages

| Benefit | Description |
|---------|-------------|
| **🔐 Enhanced Security** | Complete isolation from external threats, zero-day exploits, and internet-based attacks |
| **📋 Compliance** | Meets strict regulatory requirements (FISMA, PCI-DSS, HIPAA, DoD) |
| **🌍 Data Sovereignty** | Complete control over data location and movement with no external dependencies |
| **🔒 IP Protection** | Prevents data exfiltration and protects proprietary algorithms and data |
| **⚡ Predictable Environment** | No unexpected updates or changes from external sources |

### ⚠️ Disadvantages

| Challenge | Impact |
|-----------|--------|
| **🔄 Complex Upgrades** | Every cluster upgrade requires downloading, mirroring, and testing all images offline |
| **📦 Limited Operators** | Not all operators are easily available in disconnected environments |
| **❌ Missing Community Content** | Community operators and some ISV operators may be unavailable |
| **💾 Storage Requirements** | Maintaining full mirror requires 150-300GB+ that grows with each version |
| **⏱️ Update Lag** | Security patches and bug fixes require manual intervention, creating exposure windows |
| **🔧 Maintenance Overhead** | Requires dedicated resources for managing mirrors, testing updates |

---

## 💡 Alternative Approach

### Restricted Network with Whitelisting

For organizations that don't require **complete** air-gapping, a **restricted network with domain whitelisting** offers an excellent middle ground:

#### Benefits Over Full Air-Gap

- ✅ Automatic operator updates and cluster upgrades
- ✅ Full access to Red Hat, certified, and community operator catalogs
- ✅ Immediate security patches and bug fixes
- ✅ OpenShift Insights for proactive issue detection
- ✅ Significantly reduced operational overhead
- ✅ Still maintains strong security boundaries

#### Recommended Whitelisted Domains

```bash
# Allow HTTPS (443) to these domains only:
registry.redhat.io           # Red Hat container registry
quay.io                      # OpenShift images and operators
registry.connect.redhat.com  # Certified partner content
icr.io                       # IBM Cloud Registry
api.openshift.com            # Telemetry and cluster management
mirror.openshift.com         # Installation assets and updates
```

#### Implementation

```bash
# Example firewall rules (iptables)
iptables -A OUTPUT -p tcp --dport 443 -d registry.redhat.io -j ACCEPT
iptables -A OUTPUT -p tcp --dport 443 -d quay.io -j ACCEPT
iptables -A OUTPUT -p tcp --dport 443 -d registry.connect.redhat.com -j ACCEPT
iptables -A OUTPUT -p tcp --dport 443 -d icr.io -j ACCEPT
iptables -A OUTPUT -p tcp --dport 443 -d api.openshift.com -j ACCEPT
iptables -A OUTPUT -p tcp --dport 443 -d mirror.openshift.com -j ACCEPT
iptables -A OUTPUT -j REJECT
```

**💡 Recommendation:** Unless compliance mandates complete isolation, this approach offers the best balance of security and operational efficiency.

---

## 🔍 Troubleshooting

### Common Issues

<details>
<summary><b>Mirror registry installation fails</b></summary>

**Issue:** Mirror registry installer errors out

**Solution:**
- Ensure ports 8443, 8080, 5432 are available
- Check firewall settings: `sudo firewall-cmd --list-all`
- Verify certificates are valid: `openssl x509 -in quay.crt -text -noout`
- Check available disk space: `df -h`
</details>

<details>
<summary><b>oc-mirror takes too long or fails</b></summary>

**Issue:** Content mirroring is extremely slow or times out

**Solution:**
- Increase timeout values in imageset-config.yaml
- Mirror in smaller batches (fewer operators)
- Check network bandwidth: `speedtest-cli`
- Use `--continue-on-error` flag
- Monitor resources: `top`, `iostat`
</details>

<details>
<summary><b>Cluster installation hangs</b></summary>

**Issue:** openshift-install hangs during bootstrap

**Solution:**
- Check bootstrap VM console in vSphere
- Verify DNS resolution: `nslookup api.cluster.domain`
- Ensure DHCP is assigning IPs correctly
- Check install logs: `tail -f .openshift_install.log`
- Verify mirror registry is accessible from nodes
</details>

<details>
<summary><b>Operators fail to install</b></summary>

**Issue:** Operators show ImagePullBackOff errors

**Solution:**
- Verify imageContentSources in install-config.yaml
- Check that operator was mirrored: `oc-mirror list operators`
- Ensure pull secret includes mirror registry
- Verify CA trust bundle is correct
- Check operator pod logs: `oc logs -n namespace pod-name`
</details>

**📚 For more troubleshooting tips, see [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)**

---

## 🤝 Contributing

Contributions are welcome! Whether it's:

- 🐛 Bug fixes
- 📝 Documentation improvements
- ✨ New features or sections
- 💡 Suggestions and ideas

### How to Contribute

1. Fork this repository
2. Create a feature branch: `git checkout -b feature/amazing-feature`
3. Commit your changes: `git commit -m 'Add amazing feature'`
4. Push to the branch: `git push origin feature/amazing-feature`
5. Open a Pull Request

### Code of Conduct

Please read [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) before contributing.

---

## 📚 Resources

### Official Documentation

- [OpenShift Documentation](https://docs.openshift.com/)
- [Installing in Restricted Networks](https://docs.openshift.com/container-platform/4.18/installing/installing-restricted-networks-preparations.html)
- [Mirror Registry for Red Hat OpenShift](https://docs.openshift.com/container-platform/4.18/installing/disconnected_install/installing-mirroring-installation-images.html)
- [oc-mirror Plugin Documentation](https://docs.openshift.com/container-platform/4.18/installing/disconnected_install/installing-mirroring-disconnected.html)

### Related Projects

- [openshift/oc-mirror](https://github.com/openshift/oc-mirror) - Official oc-mirror plugin
- [quay/mirror-registry](https://github.com/quay/mirror-registry) - Mirror registry installer
- [Red Hat Demos](https://github.com/redhat-cop/openshift-disconnected-operators) - Disconnected operators guide

### Community

- [OpenShift Community](https://www.openshift.com/community)
- [Red Hat Customer Portal](https://access.redhat.com/)
- [OpenShift Blog](https://www.openshift.com/blog)

---

## 📊 Project Statistics

![GitHub stars](https://img.shields.io/github/stars/yourusername/openshift-airgapped-guide?style=social)
![GitHub forks](https://img.shields.io/github/forks/yourusername/openshift-airgapped-guide?style=social)
![GitHub watchers](https://img.shields.io/github/watchers/yourusername/openshift-airgapped-guide?style=social)

---

## 📜 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgments

- Red Hat OpenShift team for excellent documentation
- OpenShift community for shared knowledge and best practices
- Contributors who have helped improve this guide

---

## 📧 Contact & Support

- **Issues:** [GitHub Issues](https://github.com/yourusername/openshift-airgapped-guide/issues)
- **Discussions:** [GitHub Discussions](https://github.com/yourusername/openshift-airgapped-guide/discussions)
- **Email:** your.email@example.com

---

<div align="center">

**⭐ If this guide helped you, please consider giving it a star! ⭐**

Made with ❤️ by the OpenShift community

[⬆ Back to Top](#-openshift-air-gapped-installation-guide)

</div>
