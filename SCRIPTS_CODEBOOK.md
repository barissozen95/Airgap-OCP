# Scripts & Codebook: Air-Gapped OpenShift 4.18 Installation

> **Complete collection of scripts and commands for disconnected OpenShift deployment**

---

## Table of Contents

1. [Environment Variables](#1-environment-variables)
2. [Storage Expansion Scripts](#2-storage-expansion-scripts)
3. [TLS Certificate Generation](#3-tls-certificate-generation)
4. [Mirror Registry Installation](#4-mirror-registry-installation)
5. [HTTP Server Setup](#5-http-server-setup)
6. [OpenShift Tools Installation](#6-openshift-tools-installation)
7. [ImageSetConfiguration](#7-imagesetconfiguration)
8. [Mirroring Commands](#8-mirroring-commands)
9. [vSphere Preparation](#9-vsphere-preparation)
10. [OpenShift Installation](#10-openshift-installation)
11. [Post-Installation Commands](#11-post-installation-commands)
12. [Utility Scripts](#12-utility-scripts)

---

## 1. Environment Variables

### Core Variables

```bash
#!/bin/bash
# environment.sh - Core environment variables

# OpenShift Version
export OCP_VERSION="stable-4.18"
export RHCOS_VERSION="4.18"

# Network Configuration
export BASTION_IP="192.168.252.2"
export API_VIP="192.168.252.3"
export REGISTRY_HOSTNAME="registry.gym.lan"
export REGISTRY_PORT="8443"
export REGISTRY_URL="${BASTION_IP}:${REGISTRY_PORT}"

# vCenter Configuration
export VCENTER_HOSTNAME="ocpgym-vc.techzone.ibm.local"

# Cluster Configuration
export CLUSTER_NAME="ocpinstall"
export BASE_DOMAIN="gym.lan"

# Paths
export KUBECONFIG="${HOME}/ocpinstall/auth/kubeconfig"
export MIRROR_DIR="${HOME}/ocp-4.18"
export ISC_FILE="${HOME}/isc-platform-4.18.yaml"

# Registry Credentials
export REGISTRY_USER="admin"
export REGISTRY_PASS="QuayForAll!"
```

### Load Environment

```bash
source ~/environment.sh
```

---

## 2. Storage Expansion Scripts

### Check Current Storage

```bash
#!/bin/bash
# check-storage.sh - Display current storage status

echo "=== Block Devices ==="
lsblk

echo -e "\n=== Volume Groups ==="
sudo vgs

echo -e "\n=== Logical Volumes ==="
sudo lvs

echo -e "\n=== Filesystem Usage ==="
df -h /
```

### Expand Storage (LVM)

```bash
#!/bin/bash
# expand-storage.sh - Expand root filesystem with new disk
# Usage: ./expand-storage.sh /dev/sdb

NEW_DISK="${1:-/dev/sdb}"
VG_NAME="rhel_bastion"
LV_PATH="/dev/${VG_NAME}/root"

echo "Expanding storage with ${NEW_DISK}..."

# Create physical volume and extend VG
sudo vgextend ${VG_NAME} ${NEW_DISK}

# Extend logical volume to use all free space
sudo lvextend -l +100%FREE ${LV_PATH}

# Grow XFS filesystem online
sudo xfs_growfs /

# Verify
echo -e "\n=== New Filesystem Size ==="
df -h /
```

---

## 3. TLS Certificate Generation

### Generate Self-Signed Certificate

```bash
#!/bin/bash
# generate-cert.sh - Generate TLS certificate for mirror registry

CERT_DIR="${HOME}"
KEY_FILE="${CERT_DIR}/quay.key"
CERT_FILE="${CERT_DIR}/quay.crt"
REGISTRY_IP="192.168.252.2"
REGISTRY_DNS="registry.gym.lan"
VALIDITY_DAYS=3650

echo "Generating TLS certificate..."

openssl req -newkey rsa:4096 -nodes -sha256 \
    -keyout ${KEY_FILE} \
    -x509 -out ${CERT_FILE} \
    -days ${VALIDITY_DAYS} \
    -subj "/O=gym,CN=${REGISTRY_DNS}" \
    -addext "subjectAltName = DNS:${REGISTRY_DNS},IP:${REGISTRY_IP}"

echo "Certificate generated:"
echo "  Key:  ${KEY_FILE}"
echo "  Cert: ${CERT_FILE}"

# Display certificate details
echo -e "\n=== Certificate Details ==="
openssl x509 -in ${CERT_FILE} -text -noout | grep -A2 "Subject:"
openssl x509 -in ${CERT_FILE} -text -noout | grep -A1 "Subject Alternative Name"
```

### Verify Certificate

```bash
#!/bin/bash
# verify-cert.sh - Verify certificate configuration

CERT_FILE="${HOME}/quay.crt"

echo "=== Certificate Subject ==="
openssl x509 -in ${CERT_FILE} -subject -noout

echo -e "\n=== Certificate Validity ==="
openssl x509 -in ${CERT_FILE} -dates -noout

echo -e "\n=== Subject Alternative Names ==="
openssl x509 -in ${CERT_FILE} -text -noout | grep -A1 "Subject Alternative Name"

echo -e "\n=== SHA256 Fingerprint ==="
openssl x509 -in ${CERT_FILE} -fingerprint -sha256 -noout
```

---

## 4. Mirror Registry Installation

### Download and Install Mirror Registry

```bash
#!/bin/bash
# install-mirror-registry.sh - Install Quay mirror registry

REGISTRY_IP="192.168.252.2"
REGISTRY_USER="admin"
REGISTRY_PASS="QuayForAll!"
CERT_DIR="${HOME}"

# Create temp directory
MIRROR_DIR=$(mktemp -d)
echo "Working directory: ${MIRROR_DIR}"

# Download mirror-registry
echo "Downloading mirror-registry..."
curl -Lo ${MIRROR_DIR}/mirror-registry.tar.gz \
    https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz

# Extract
cd ${MIRROR_DIR}
tar xf mirror-registry.tar.gz

# Install
echo "Installing mirror registry..."
./mirror-registry install \
    --quayHostname ${REGISTRY_IP} \
    --initUser ${REGISTRY_USER} \
    --initPassword ${REGISTRY_PASS} \
    --sslKey ${CERT_DIR}/quay.key \
    --sslCert ${CERT_DIR}/quay.crt

# Return to home
cd ${HOME}

echo "Installation complete!"
```

### Configure Firewall for Registry

```bash
#!/bin/bash
# configure-firewall.sh - Open required ports

echo "Configuring firewall..."

# Quay registry port
sudo firewall-cmd --add-port 8443/tcp --permanent

# HTTP server port  
sudo firewall-cmd --permanent --add-service http

# Reload firewall
sudo firewall-cmd --reload

# Verify
echo -e "\n=== Open Ports ==="
sudo firewall-cmd --list-all
```

### Verify Registry Health

```bash
#!/bin/bash
# check-registry.sh - Verify mirror registry health

REGISTRY_URL="192.168.252.2:8443"

echo "Checking registry health..."
curl -sk https://${REGISTRY_URL}/health/instance | jq

echo -e "\n=== Registry Pods ==="
podman ps --filter name=quay
```

---

## 5. HTTP Server Setup

### Install and Configure Apache

```bash
#!/bin/bash
# setup-httpd.sh - Install and configure Apache web server

echo "Installing Apache..."
sudo dnf install -y httpd

echo "Enabling and starting httpd..."
sudo systemctl enable httpd --now

echo "Configuring firewall..."
sudo firewall-cmd --permanent --add-service http
sudo firewall-cmd --reload

echo "Apache status:"
sudo systemctl status httpd --no-pager
```

### Download RHCOS OVA

```bash
#!/bin/bash
# download-rhcos.sh - Download RHCOS OVA for VMware

RHCOS_VERSION="${RHCOS_VERSION:-4.18}"
WWW_DIR="/var/www/html"

echo "Downloading RHCOS ${RHCOS_VERSION} OVA..."
curl -Lo rhcos-vmware.x86_64.ova \
    https://mirror.openshift.com/pub/openshift-v4/amd64/dependencies/rhcos/${RHCOS_VERSION}/latest/rhcos-vmware.x86_64.ova

echo "Moving to web server directory..."
sudo mv rhcos-vmware.x86_64.ova ${WWW_DIR}/

echo "Setting SELinux context..."
sudo restorecon -Rv ${WWW_DIR}

echo -e "\n=== RHCOS OVA Checksum ==="
sha256sum ${WWW_DIR}/rhcos-vmware.x86_64.ova

echo -e "\n=== OVA URL ==="
echo "http://$(hostname -I | awk '{print $1}')/rhcos-vmware.x86_64.ova"
```

---

## 6. OpenShift Tools Installation

### Install OpenShift CLI (oc)

```bash
#!/bin/bash
# install-oc.sh - Install OpenShift client

OCP_VERSION="${OCP_VERSION:-stable-4.18}"

echo "Downloading OpenShift client ${OCP_VERSION}..."
curl -Lo openshift-client-linux.tar.gz \
    https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux-amd64-rhel8.tar.gz

echo "Extracting..."
tar xf openshift-client-linux.tar.gz oc

echo "Installing to /usr/local/bin..."
sudo install oc /usr/local/bin

echo "Cleaning up..."
rm -f openshift-client-linux.tar.gz oc

echo "Verification:"
oc version
```

### Install oc-mirror Plugin

```bash
#!/bin/bash
# install-oc-mirror.sh - Install oc-mirror plugin

OCP_VERSION="${OCP_VERSION:-stable-4.18}"

echo "Downloading oc-mirror ${OCP_VERSION}..."
curl -Lo oc-mirror.tar.gz \
    https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/${OCP_VERSION}/oc-mirror.tar.gz

echo "Extracting..."
tar xf oc-mirror.tar.gz oc-mirror

echo "Installing..."
chmod +x oc-mirror
sudo install oc-mirror /usr/local/bin

echo "Cleaning up..."
rm -f oc-mirror.tar.gz oc-mirror

echo "Verification:"
oc-mirror version
```

### Install OpenShift Installer

```bash
#!/bin/bash
# install-openshift-install.sh - Install OpenShift installer

OCP_VERSION="${OCP_VERSION:-stable-4.18}"

echo "Downloading OpenShift installer ${OCP_VERSION}..."
curl -Lo openshift-install-linux.tar.gz \
    https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-install-linux.tar.gz

echo "Extracting..."
tar xf openshift-install-linux.tar.gz openshift-install

echo "Installing..."
sudo install openshift-install /usr/local/bin

echo "Cleaning up..."
rm -f openshift-install-linux.tar.gz openshift-install

echo "Verification:"
openshift-install version
```

### Install All Tools

```bash
#!/bin/bash
# install-all-tools.sh - Install all OpenShift tools

OCP_VERSION="${OCP_VERSION:-stable-4.18}"
RHCOS_VERSION="${RHCOS_VERSION:-4.18}"

echo "=== Installing OpenShift Tools ==="

# oc client
echo -e "\n[1/3] Installing oc client..."
curl -sLo openshift-client-linux.tar.gz \
    https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux-amd64-rhel8.tar.gz
tar xf openshift-client-linux.tar.gz oc
sudo install oc /usr/local/bin
rm -f openshift-client-linux.tar.gz oc

# oc-mirror
echo -e "\n[2/3] Installing oc-mirror..."
curl -sLo oc-mirror.tar.gz \
    https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/${OCP_VERSION}/oc-mirror.tar.gz
tar xf oc-mirror.tar.gz oc-mirror
chmod +x oc-mirror
sudo install oc-mirror /usr/local/bin
rm -f oc-mirror.tar.gz oc-mirror

# openshift-install
echo -e "\n[3/3] Installing openshift-install..."
curl -sLo openshift-install-linux.tar.gz \
    https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-install-linux.tar.gz
tar xf openshift-install-linux.tar.gz openshift-install
sudo install openshift-install /usr/local/bin
rm -f openshift-install-linux.tar.gz openshift-install

echo -e "\n=== Installed Versions ==="
oc version --client
oc-mirror version
openshift-install version
```

---

## 7. ImageSetConfiguration

### Full Configuration (Platform + All Operators)

```yaml
# isc-platform-4.18.yaml
# Complete ImageSetConfiguration for OCP 4.18 with AI/ML operators

kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: ./
mirror:
  platform:
    channels:
    - name: stable-4.18
      type: ocp
    graph: true
  operators:
  # -------------------------------------------------------------------------
  # CATALOG 1: RED HAT OPERATORS (v4.18)
  # -------------------------------------------------------------------------
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.18
    packages:
    # --- OpenShift Data Foundation (ODF) ---
    - name: odf-operator
      channels:
      - name: stable-4.18
    - name: ocs-operator
      channels:
      - name: stable-4.18
    - name: odf-csi-addons-operator
      channels:
      - name: stable-4.18
    - name: mcg-operator
      channels:
      - name: stable-4.18
    # --- Red Hat Cert Manager ---
    - name: openshift-cert-manager-operator
      channels:
      - name: stable-v1
    # --- Local Storage Operator ---
    - name: local-storage-operator
      channels:
      - name: stable
    # --- Node Feature Discovery (NFD) ---
    - name: nfd
      channels:
      - name: stable
    # --- OpenShift AI (formerly RHODS) ---
    - name: rhods-operator
      channels:
      - name: stable
  # -------------------------------------------------------------------------
  # CATALOG 2: CERTIFIED OPERATORS (v4.18)
  # -------------------------------------------------------------------------
  - catalog: registry.redhat.io/redhat/certified-operator-index:v4.18
    packages:
    # --- NVIDIA GPU ---
    - name: gpu-operator-certified
      channels:
      - name: v25.10
```

### Platform Only Configuration

```yaml
# isc-platform-only.yaml
# Platform images only (no operators)

kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: ./
mirror:
  platform:
    channels:
    - name: stable-4.18
      type: ocp
    graph: true
```

### Minimal ODF Configuration

```yaml
# isc-odf-minimal.yaml
# Minimal ODF + dependencies only

kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: ./
mirror:
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.18
    packages:
    - name: odf-operator
      channels:
      - name: stable-4.18
    - name: ocs-operator
      channels:
      - name: stable-4.18
    - name: odf-csi-addons-operator
      channels:
      - name: stable-4.18
    - name: mcg-operator
      channels:
      - name: stable-4.18
    - name: local-storage-operator
      channels:
      - name: stable
```

---

## 8. Mirroring Commands

### Mirror to Local Disk

```bash
#!/bin/bash
# mirror-to-disk.sh - Mirror content to local disk

ISC_FILE="${HOME}/isc-platform-4.18.yaml"
MIRROR_DIR="ocp-4.18"

echo "Starting mirror to disk..."
echo "Config: ${ISC_FILE}"
echo "Output: ${MIRROR_DIR}"

oc-mirror --config=${ISC_FILE} file://${MIRROR_DIR} --v1

echo "Mirror complete. Contents:"
ls -lh ${MIRROR_DIR}/
```

### Push to Mirror Registry

```bash
#!/bin/bash
# push-to-registry.sh - Push mirrored content to registry
# Usage: ./push-to-registry.sh [sequence_number]

REGISTRY_URL="192.168.252.2:8443"
MIRROR_DIR="ocp-4.18"
SEQ_NUM="${1:-1}"

# Login to registry
echo "Logging into registry..."
podman login ${REGISTRY_URL} --tls-verify=false

# Find and push archive
ARCHIVE="${MIRROR_DIR}/mirror_seq${SEQ_NUM}_000000.tar"

if [ -f "${ARCHIVE}" ]; then
    echo "Pushing ${ARCHIVE}..."
    oc-mirror --from=${ARCHIVE} docker://${REGISTRY_URL} --v1 --dest-skip-tls
else
    echo "Archive not found: ${ARCHIVE}"
    echo "Available archives:"
    ls -1 ${MIRROR_DIR}/*.tar 2>/dev/null || echo "No tar files found"
    exit 1
fi

echo "Push complete!"
```

### List Available Content

```bash
#!/bin/bash
# list-content.sh - List available releases and operators

echo "=== Available OpenShift Releases ==="
oc-mirror list releases --channel stable-4.18

echo -e "\n=== Available Operators (Red Hat Catalog) ==="
oc-mirror list operators \
  --catalog registry.redhat.io/redhat/redhat-operator-index:v4.18 2>/dev/null

echo -e "\n=== Available Operators (Certified Catalog) ==="
oc-mirror list operators \
  --catalog registry.redhat.io/redhat/certified-operator-index:v4.18 2>/dev/null
```

---

## 9. vSphere Preparation

### Import vCenter CA Certificates

```bash
#!/bin/bash
# import-vcenter-certs.sh - Import vCenter CA certificates

VCENTER_HOSTNAME="${VCENTER_HOSTNAME:-ocpgym-vc.techzone.ibm.local}"

echo "Downloading certificates from ${VCENTER_HOSTNAME}..."
curl -kL https://${VCENTER_HOSTNAME}/certs/download.zip -o download.zip

echo "Extracting certificates..."
unzip -q download.zip

echo "Installing to trust store..."
sudo cp certs/lin/* /etc/pki/ca-trust/source/anchors

echo "Updating CA trust..."
sudo update-ca-trust extract

echo "Cleaning up..."
rm -rf download.zip certs/

echo "Done! vCenter certificate installed."
```

### Test vCenter Connection

```bash
#!/bin/bash
# test-vcenter.sh - Test vCenter API connectivity

VCENTER_HOSTNAME="${VCENTER_HOSTNAME:-ocpgym-vc.techzone.ibm.local}"

echo "Testing connection to ${VCENTER_HOSTNAME}..."

if curl -s --connect-timeout 5 https://${VCENTER_HOSTNAME}/sdk > /dev/null; then
    echo "✓ vCenter API is accessible"
else
    echo "✗ Cannot connect to vCenter API"
    exit 1
fi
```

---

## 10. OpenShift Installation

### Generate Install Config

```bash
#!/bin/bash
# create-install-config.sh - Generate install-config.yaml interactively

echo "Creating install configuration..."
echo "You will be prompted for:"
echo "  - SSH public key"
echo "  - vSphere credentials"
echo "  - Network selection"
echo "  - API/Ingress VIPs"
echo "  - Base domain and cluster name"
echo "  - Pull secret"
echo ""

openshift-install create install-config

echo -e "\nInstall config created: install-config.yaml"
```

### Prepare Installation Directory

```bash
#!/bin/bash
# prepare-install-dir.sh - Prepare installation directory

CLUSTER_NAME="${CLUSTER_NAME:-ocpinstall}"

echo "Creating installation directory: ${CLUSTER_NAME}"
mkdir -p ${CLUSTER_NAME}

if [ -f "install-config.yaml" ]; then
    echo "Copying install-config.yaml..."
    cp install-config.yaml ${CLUSTER_NAME}/
else
    echo "ERROR: install-config.yaml not found"
    exit 1
fi

echo "Directory prepared: ${CLUSTER_NAME}/"
ls -la ${CLUSTER_NAME}/
```

### Run Installation

```bash
#!/bin/bash
# run-installation.sh - Execute OpenShift installation

CLUSTER_DIR="${CLUSTER_NAME:-ocpinstall}"

echo "Starting OpenShift installation..."
echo "Cluster directory: ${CLUSTER_DIR}"
echo "This will take approximately 40-50 minutes."
echo ""

openshift-install create cluster --dir ${CLUSTER_DIR} --log-level debug

echo -e "\n=== Installation Complete ==="
echo "Kubeconfig: ${CLUSTER_DIR}/auth/kubeconfig"
echo "Kubeadmin password: ${CLUSTER_DIR}/auth/kubeadmin-password"
```

---

## 11. Post-Installation Commands

### Set Kubeconfig

```bash
#!/bin/bash
# set-kubeconfig.sh - Export kubeconfig for cluster access

CLUSTER_DIR="${CLUSTER_NAME:-ocpinstall}"

export KUBECONFIG="${HOME}/${CLUSTER_DIR}/auth/kubeconfig"
echo "KUBECONFIG set to: ${KUBECONFIG}"

# Verify access
oc whoami
oc get nodes
```

### Disable Default OperatorHub Sources

```bash
#!/bin/bash
# disable-default-sources.sh - Disable default OperatorHub sources

echo "Disabling default OperatorHub sources..."
oc patch OperatorHub cluster --type json \
    -p '[{"op": "add", "path": "/spec/disableAllDefaultSources","value": true}]'

echo "Verifying..."
oc get OperatorHub cluster -o jsonpath='{.spec.disableAllDefaultSources}'
echo ""
```

### Apply Mirror Configuration

```bash
#!/bin/bash
# apply-mirror-config.sh - Apply ICSP and CatalogSources

RESULTS_DIR="${HOME}/oc-mirror-workspace"
LATEST_RESULTS=$(ls -td ${RESULTS_DIR}/results-* 2>/dev/null | head -1)

if [ -z "${LATEST_RESULTS}" ]; then
    echo "ERROR: No results directory found in ${RESULTS_DIR}"
    exit 1
fi

echo "Using results from: ${LATEST_RESULTS}"

# Apply ImageContentSourcePolicy
echo -e "\n[1/4] Applying ImageContentSourcePolicy..."
oc apply -f ${LATEST_RESULTS}/imageContentSourcePolicy.yaml

# Apply Red Hat Operator CatalogSource
echo -e "\n[2/4] Applying Red Hat Operator CatalogSource..."
oc apply -f ${LATEST_RESULTS}/catalogSource-cs-redhat-operator-index.yaml

# Apply Certified Operator CatalogSource
echo -e "\n[3/4] Applying Certified Operator CatalogSource..."
oc apply -f ${LATEST_RESULTS}/catalogSource-cs-certified-operator-index.yaml

# Apply release signatures
echo -e "\n[4/4] Applying release signatures..."
oc apply -f ${LATEST_RESULTS}/release-signatures/

echo -e "\n=== Configuration Applied ==="
echo "Waiting for catalog sources to be ready..."
sleep 30

oc -n openshift-marketplace get catalogsources
oc -n openshift-marketplace get pods
```

### Verify Cluster Status

```bash
#!/bin/bash
# verify-cluster.sh - Comprehensive cluster health check

echo "=== Node Status ==="
oc get nodes

echo -e "\n=== Cluster Operators ==="
oc get clusteroperators

echo -e "\n=== Machine Config Pools ==="
oc get mcp

echo -e "\n=== Catalog Sources ==="
oc -n openshift-marketplace get catalogsources

echo -e "\n=== Marketplace Pods ==="
oc -n openshift-marketplace get pods

echo -e "\n=== Available Operators ==="
oc get packagemanifests -n openshift-marketplace --no-headers | wc -l
echo "operators available in OperatorHub"
```

---

## 12. Utility Scripts

### Monitor oc-mirror Progress

```bash
#!/bin/bash
# monitor-mirror.sh - Monitor oc-mirror process

echo "=== oc-mirror Process ==="
top -b -n1 -p $(pgrep -d',' oc-mirror) 2>/dev/null || echo "oc-mirror not running"

echo -e "\n=== Mirror Directory Size ==="
du -sh ocp-4.18/ 2>/dev/null || echo "Mirror directory not found"

echo -e "\n=== Recent Log Entries ==="
tail -20 .oc-mirror.log 2>/dev/null || echo "Log file not found"
```

### Generate Registry Credentials JSON

```bash
#!/bin/bash
# gen-registry-creds.sh - Generate registry credentials for pull secret

REGISTRY_URL="${1:-192.168.252.2:8443}"
REGISTRY_USER="${2:-admin}"
REGISTRY_PASS="${3:-QuayForAll!}"

REG_CREDS=$(echo -n "${REGISTRY_USER}:${REGISTRY_PASS}" | base64 -w0)

cat <<EOF
Add this to your pull secret:

{
  "auths": {
    "${REGISTRY_URL}": {
      "auth": "${REG_CREDS}",
      "email": "${REGISTRY_USER}@quay.io"
    }
  }
}
EOF
```

### Restart Catalog Pods

```bash
#!/bin/bash
# restart-catalog-pods.sh - Restart CatalogSource pods

echo "Restarting Red Hat Operator catalog pod..."
oc delete pod -l olm.catalogSource=cs-redhat-operator-index -n openshift-marketplace

echo "Restarting Certified Operator catalog pod..."
oc delete pod -l olm.catalogSource=cs-certified-operator-index -n openshift-marketplace

echo "Waiting for pods to restart..."
sleep 10

oc -n openshift-marketplace get pods
```

### View Image Configuration

```bash
#!/bin/bash
# view-image-config.sh - View cluster image configuration

echo "=== Image Config ==="
oc get image.config.openshift.io/cluster -o yaml

echo -e "\n=== ImageContentSourcePolicies ==="
oc get imagecontentsourcepolicies

echo -e "\n=== ICSP Details ==="
oc get imagecontentsourcepolicies -o yaml | grep -A5 "repositoryDigestMirrors:"
```

---

## Quick Reference Card

### Essential Commands

| Task | Command |
|------|---------|
| Set kubeconfig | `export KUBECONFIG=${HOME}/ocpinstall/auth/kubeconfig` |
| Login to registry | `podman login 192.168.252.2:8443 --tls-verify=false` |
| Check registry health | `curl -sk https://192.168.252.2:8443/health/instance \| jq` |
| Mirror to disk | `oc-mirror --config=isc.yaml file://ocp-4.18 --v1` |
| Push to registry | `oc-mirror --from=archive.tar docker://192.168.252.2:8443 --v1 --dest-skip-tls` |
| Check nodes | `oc get nodes` |
| Check operators | `oc get clusteroperators` |
| Check catalog pods | `oc -n openshift-marketplace get pods` |

---

*Last Updated: January 2026 | OpenShift 4.18*
