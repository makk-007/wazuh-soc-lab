# Wazuh SOC Lab: Environment Setup

**Completed:** June 18, 2026
**Host OS:** Ubuntu 24.04 LTS
**Hardware:** ASUS ROG Zephyrus G15 (AMD Ryzen 6900HS, 16 GB RAM, Nvidia RTX 3060)
**Stack Version:** Wazuh 4.14.5 (manager, indexer, dashboard, Linux agent, Windows agent)

---

## Overview

This document covers the full environment setup for the Wazuh SOC Lab, including installation of Docker, VirtualBox, a Windows 11 Pro VM, and the complete Wazuh stack. Six deviations from the standard setup path were encountered and resolved; each is documented in the [Troubleshooting](#troubleshooting) section below.

---

## Architecture

```
Host Machine (Ubuntu 24.04 LTS)
├── Docker
│   ├── wazuh.manager      (Wazuh Manager)
│   ├── wazuh.indexer      (OpenSearch)
│   └── wazuh.dashboard    (Kibana-based UI)
├── Wazuh Agent (Linux)    (installed on host, reports to manager)
└── VirtualBox
    └── Windows 11 Pro VM
        └── Wazuh Agent (Windows)  (reports to manager via local network)
```

---

## Step 1: Docker Engine and Docker Compose

Installed from Docker's official repository rather than Ubuntu's default package list, which ships an older version.

```bash
# Remove conflicting packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove $pkg
done

# Add Docker's GPG key and repository
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

# Install Docker Engine and Compose plugin
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
docker run hello-world
```

---

## Step 2: VirtualBox and Windows 11 Pro VM

### VirtualBox Installation

```bash
sudo apt update
sudo apt install -y virtualbox virtualbox-ext-pack
sudo usermod -aG vboxusers $USER
newgrp vboxusers
```

> **Note:** The ext-pack installation prompts for acceptance of Oracle's license terms. Accept to enable USB 2.0/3.0 and RDP support.

> **Troubleshooting:** Secure Boot blocked unsigned kernel modules during this step. See [Issue 1](#issue-1-secure-boot-blocking-virtualbox-kernel-modules) and [Issue 2](#issue-2-kvm-amd-v-conflict-with-virtualbox).

### KVM Module Unload (required before every VirtualBox session)

The host's KVM modules conflict with VirtualBox's AMD-V usage. Unload them before launching VirtualBox:

```bash
sudo modprobe -r kvm_amd && sudo modprobe -r kvm
```

This must be repeated after every reboot since KVM reloads automatically on boot.

### Windows 11 Pro VM Configuration

The ISO was downloaded from Microsoft's official page:
`https://www.microsoft.com/software-download/windows11`

VM settings used in VirtualBox:

| Setting | Value |
|---------|-------|
| Name | `wazuh-windows-agent` |
| Type | Microsoft Windows |
| Version | Windows 11 (64-bit) |
| RAM | 4096 MB |
| CPU cores | 2 |
| Disk | 55 GB (dynamically allocated VDI) |
| TPM | Virtual TPM 2.0 enabled (Settings > System > TPM) |
| EFI | Enabled (required for Windows 11 and Secure Boot) |
| Network | Bridged Adapter |

> **Troubleshooting:** The initial disk size of 50 GB was rejected by the Windows 11 installer. See [Issue 3](#issue-3-windows-11-minimum-disk-requirement).

No product key is required during installation. Skip the key entry screen and select **Windows 11 Pro** when prompted for an edition.

---

## Step 3: Wazuh Stack via Docker

```bash
# Clone the official Wazuh Docker repository at the target version
cd ~
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.5
cd wazuh-docker/single-node

# Generate SSL certificates
docker compose -f generate-indexer-certs.yml run --rm generator

# Increase virtual memory (required by the Wazuh indexer)
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Start the stack
docker compose up -d

# Verify all three containers are running
docker compose ps
```

Expected output shows `wazuh.manager`, `wazuh.indexer`, and `wazuh.dashboard` all with `Up` status.

> **Troubleshooting:** The guide was initially written for v4.12.0. The stack was updated to v4.14.5 before agent installation. See [Issue 4](#issue-4-wazuh-stack-updated-from-412-to-4145).

### Dashboard Access

Navigate to `https://localhost` in a browser. Accept the self-signed certificate warning.

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `SecretPassword` |

Change the password immediately after first login.

---

## Step 4: Wazuh Linux Agent

```bash
# Add Wazuh repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update

# Install agent
sudo apt install -y wazuh-agent

# Configure manager address
sudo nvim /var/ossec/etc/ossec.conf
# Set <address>127.0.0.1</address> under the <server> block

# Enable and start the agent
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

> **Troubleshooting:** The Wazuh repository was not present in the sources list during an attempted upgrade, blocking the version update. See [Issue 5](#issue-5-wazuh-apt-repository-not-configured-for-agent-update).

The Linux host should appear as an active agent in the dashboard within one minute under **Agents > Agent List**.

---

## Step 5: Wazuh Windows Agent

The Windows agent MSI installer was downloaded from:
`https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi`

During installation:
- Set the **Manager IP** to the Linux host's local IP address (find it with `ip addr` on the host).
- Leave the **Authentication Key** field empty. Wazuh handles agent registration automatically.

After installation, start the Wazuh service:

```powershell
NET START WazuhSvc
```

The Windows VM should appear as a second active agent in the dashboard.

> **Troubleshooting:** See [Issue 6](#issue-6-windows-agent-authentication-key) regarding the authentication key field.

---

## Troubleshooting

### Issue 1: Secure Boot Blocking VirtualBox Kernel Modules

**Symptom:** After VirtualBox installation, `vboxdrv` failed to load. Running `sudo modprobe vboxdrv` returned an error indicating the module was blocked due to Secure Boot being enabled.

**Cause:** Ubuntu's Secure Boot enforcement blocks unsigned third-party kernel modules, which includes VirtualBox's `vboxdrv`.

**Fix:** During installation, the system prompted to set a password for a Machine Owner Key (MOK). On the next reboot, the UEFI MOK Manager appeared. The steps followed were:

1. Select **Enroll MOK**
2. Select **Continue**
3. Select **Yes** to enroll the key
4. Enter the MOK password set during installation
5. Select **Reboot**

After rebooting, `vboxdrv` loaded successfully and VirtualBox launched without errors.

---

### Issue 2: KVM/AMD-V Conflict with VirtualBox

**Symptom:** VirtualBox could not enable AMD-V hardware virtualization. The error indicated that AMD-V was already in use by another hypervisor.

**Cause:** The Linux kernel's KVM modules (`kvm_amd` and `kvm`) were loaded at boot and hold exclusive access to AMD-V, preventing VirtualBox from using it.

**Fix:** Unload the KVM modules before launching VirtualBox:

```bash
sudo modprobe -r kvm_amd && sudo modprobe -r kvm
```

**Persistence note:** KVM modules reload automatically on every reboot. This command must be run each time before starting VirtualBox. It does not affect Docker or any other running services.

---

### Issue 3: Windows 11 Minimum Disk Requirement

**Symptom:** The Windows 11 installer rejected the VM and refused to proceed, citing insufficient disk space.

**Cause:** Windows 11 requires a minimum of 52 GB of disk space. The initial VM was created with a 50 GB dynamically allocated disk.

**Fix:** The VM was deleted and recreated with a 55 GB dynamically allocated disk, giving a small buffer above the minimum.

---

### Issue 4: Wazuh Stack Updated from 4.12.0 to 4.14.5

**Symptom:** The setup guide referenced v4.12.0, but v4.14.5 was the latest stable release at the time of setup.

**Cause:** The guide was written before the newer version was released.

**Fix:** The Docker repository was cloned at the `v4.14.5` tag:

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.5
```

The Linux agent and Windows agent were also installed at 4.14.5 to ensure version consistency across all components.

---

### Issue 5: Wazuh apt Repository Not Configured for Agent Update

**Symptom:** Running `sudo apt upgrade wazuh-agent` returned an error indicating the package could not be found, even though the agent was already installed.

**Cause:** The Wazuh apt repository was not present in the system's sources list, so `apt` had no reference for the package.

**Fix:** The repository was added manually before upgrading:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo apt install -y wazuh-agent
```

---

### Issue 6: Windows Agent Authentication Key

**Symptom:** The MSI installer displayed an **Authentication Key** field, which was unclear whether it was mandatory.

**Cause:** Older Wazuh documentation referenced manual key-based agent registration. Current versions support automatic registration via the manager's API.

**Fix:** The field was left empty. Wazuh completed agent registration automatically and the Windows VM appeared as an active agent in the dashboard without a manual key.

---

## Outcome

Both agents are active and reporting to the Wazuh manager:

| Agent | OS | Status |
|-------|----|--------|
| `ubuntu-host` | Ubuntu 24.04 LTS | Active |
| `wazuh-windows-agent` | Windows 11 Pro | Active |

The lab environment is fully operational and ready for detection use case simulations.