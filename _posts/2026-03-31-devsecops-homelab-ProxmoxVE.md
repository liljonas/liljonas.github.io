---
layout: post
title: "Home Lab PROJECT (DevSecOps) - ProxmoxVE"
date: 2026-03-31 00:00:00 +0000
categories: [DevSecOps, Homelab]
tags: [ProxmoxVE, OPNsense, Tailscale, Authentik, OpenBao, Internal PKI]
---
# Overview:

| **Phase**                                 | **Horizon** | **What you get**                                                                                                     |
| ----------------------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------- |
| **1. Network foundations**                | Immediate   | OPNsense, clean LAN, remote access, base DNS                                                                         |
| **2. Trusted admin node**                 | Short term  | **[Proxmox](https://www.proxmox.com/en/)** admin, authentik, OpenBao                                                 |
| **3. Application cluster**                | Medium term | Talos, GitOps, basic observability                                                                                   |
| **4. Industrialization and supply chain** | Progressive | Hardened images, signatures, SBOM, policies                                                                          |

![Figure 1: Target Architecture Overview](/assets/img/homelab/Proxmox.png)
_Homelab Target Architecture_

_**Trusted layer (Proxmox admin node)**_

A **Proxmox** node **dedicated** to hosting the trusted services:

| **Service**      | **Role**                            | **Why out of cluster?**                                          |
| ---------------- | ----------------------------------- | ---------------------------------------------------------------- |
| **authentik**    | Identity Provider (SSO OIDC)        | The cluster cannot validate its own tokens                       |
| **Openbao**      | Secrets management (fork **Vault**) | Cluster secrets cannot be stored in the cluster                  |
| **Internal PKI** | Certificates TLS internal           | The certification authority must be external to the signed scope |

## **Phase 1: Foundations**

**Objectif :** disposer d’une base réseau propre, administrable à distance, avant d’introduire la segmentation avancée.

_**Ce que vous installez**_

| **Component**                              | **Role**                           | **Guide**                   |
| ------------------------------------------ | ---------------------------------- | --------------------------- |
| **OPNsense**                               | Firewall, router, DNS              | **Installation**            |
| **Tailscale**                              | Remote access without port exposed | **OPNsense + Tailscale**    |
| **Managed switch**                         | Prepare future VLANs               | **OPNsense administration** |
| [**Proxmox VE**](https://www.proxmox.com/) | Hypervisor for the admin node      | **Installation**            |

---

# Installing Proxmox VE (Full Beginner-Friendly Homelab Guide)

>If you're building a homelab, installing a proper hypervisor is a huge step. In this guide, I’ll walk you through installing **Proxmox VE 9** on a physical server from scratch.
>Even if you’ve never installed a server OS before, you’ll end up with a fully working system ready to run virtual machines and containers.
>⏱️ **Estimated time:** 30–45 minutes
{: .prompt-info }
---

## ⚡ Quick Install (If You Already Know the Basics)

>If you’re experienced, here’s the short version:
>1. Download ISO from [https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)
>2. Flash it to USB (use **DD mode**, not ISO mode) 
>3. Boot → **Install Proxmox VE (Graphical)** 
>4. Choose filesystem:
>    - `ext4` → simple   
>    - `ZFS` → advanced / RAID 
>5. Set:
>    - Static IP  
>    - FQDN hostname 
>6. After install:
>    - Enable **no-subscription repo**    
>7. Access:
>    ```
>    https://YOUR-IP:8006
>    ```
{: .prompt-tip }

---

## What You’ll Need

Before starting:

|Item|Why|
|---|---|
|**Dedicated machine**|Proxmox installs directly on hardware|
|**USB drive (≥ 2 GB)**|Installation media|
|**Keyboard + monitor**|Initial setup|
|**Ethernet connection**|Required for networking|
|**Another PC**|To download ISO|

>**Important:**  
>This installation wipes the entire disk. Don’t run this on a machine with important data.
{: .prompt-danger}

---

## 🤔 Why Use the Official ISO?

There are two ways to install Proxmox:

|Method|Difficulty|Who it’s for|
|---|---|---|
|**Official ISO**|⭐ Easy|Everyone|
|Install on Debian|⭐⭐⭐ Hard|Advanced Linux users|

👉 The ISO is the best option because:

- Guided installer
    
- Automatic partitioning
    
- Everything preconfigured
    
- Web UI ready immediately
    

---

## What’s Inside Proxmox VE 9?

The ISO includes:

- Debian 13 (Trixie)
    
- Linux kernel 6.17
    
- QEMU/KVM (VM engine)
    
- LXC (containers)
    
- Web interface
    

---

## Step 1: Check Hardware Compatibility

### Virtualization Support (Required)

Your CPU must support virtualization:

|Vendor|Feature|
|---|---|
|Intel|VT-x|
|AMD|AMD-V / SVM|

Enable it in BIOS/UEFI if needed.

### Check from Linux

```bash
grep -E '(vmx|svm)' /proc/cpuinfo
```

- `vmx` → Intel OK
    
- `svm` → AMD OK
    

---

## Hardware Guidelines

### Minimal (Homelab Starter)

|Component|Minimum|
|---|---|
|CPU|64-bit with virtualization|
|RAM|4 GB|
|Disk|32 GB|
|Network|1 NIC|

### Recommended

|Component|Recommended|
|---|---|
|CPU|4+ cores|
|RAM|16–32 GB|
|OS Disk|SSD (64 GB+)|
|VM Storage|SSD/HDD (256 GB+)|
|Network|2 NICs|

### RAM Rule

Each VM needs its own memory.

👉 Simple rule:

```
Total RAM = (VM RAM) + 2 GB (for Proxmox)
```

---

## Step 2: Download Proxmox ISO

Go to:  
[https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)

Download the latest version (e.g. `proxmox-ve_9.1.iso`)

Optional verification:

```bash
sha256sum proxmox-ve_9.1.iso
```

---

## Step 3: Create Bootable USB

### Recommended: Rufus (Windows)

- Select ISO
    
- Choose **DD mode** (very important)
    

⚠️ If you use ISO mode → it may not boot.

---

### Alternative: Balena Etcher

- Select ISO
    
- Select USB
    
- Flash
    

---

## Step 4: Boot From USB

1. Insert USB
    
2. Boot machine
    
3. Open boot menu (F12 / ESC / F11)
    
4. Select USB
    

Choose:

```
Install Proxmox VE (Graphical)
```

---

### Common Boot Issues

- Boots into Windows → fix boot order
    
- Black screen → use `nomodeset`
    
- USB not detected → recreate in DD mode
    
- Secure Boot → disable it
    

---

## Step 5: Installation Walkthrough

### 1. License

Click **I Agree**

---

### 2. Disk Selection (Important)

Choose your installation disk.

👉 Best practice:

- Use SSD for OS
    
- Keep larger disk for VM storage
    

---

### Filesystem Choice

|Option|Use case|
|---|---|
|**ext4**|✅ Best for beginners|
|**ZFS**|Advanced (RAID, snapshots)|
|XFS|Large storage|
|BTRFS|❌ Avoid|

👉 Recommendation:

- Start with **ext4**
    
- Move to ZFS later if needed
    

---

### 3. Location

Set:

- Country
    
- Timezone
    
- Keyboard layout
    

---

### 4. Password & Email

- Strong root password
    
- Real email (alerts & notifications)
    

---

### 5. Network Configuration (Critical)

|Field|Example|
|---|---|
|Interface|`eth0` / `eno1`|
|Hostname|`pve1.home.lab`|
|IP|`192.168.1.100`|
|Gateway|`192.168.1.1`|
|DNS|`8.8.8.8`|

---

### Key Points

- Use a **static IP**
    
- Use a proper **FQDN** (`pve1.home.lab`)
    
- Avoid DHCP range conflicts
    

---

### 6. Install

- Review settings
    
- Click **Install**
    
- Wait 5–15 minutes
    

Remove USB after install.

---

## Step 6: Access Web Interface

From another machine:

```
https://YOUR-IP:8006
```

Example:

```
https://192.168.1.100:8006
```

Login:

- User: `root`
    
- Password: (your password)
    

⚠️ Ignore certificate warning (self-signed)

---

## Enable Free Updates (Important)

By default, Proxmox uses the paid repo.

Switch to free repo:

1. Node → Updates → Repositories
    
2. Disable `pve-enterprise`
    
3. Add `no-subscription`
    
4. Refresh + Upgrade
    

---

## Verify Installation

Run:

```bash
pveversion
```

```bash
systemctl status pve-cluster pvedaemon pveproxy
```

```bash
df -h
```

Everything should be running normally.

---

## Troubleshooting

|Problem|Fix|
|---|---|
|Can't boot|Fix BIOS boot order|
|No web UI|Check IP (`ip a`)|
|Connection refused|Restart `pveproxy`|
|Update errors|Disable enterprise repo|

---

## Basic Security (Do This Early)

|Action|Why|
|---|---|
|Create non-root user|Safer daily usage|
|Enable firewall|Limit access|
|Install fail2ban|Block brute force|
|Update system|Security patches|

---

## Key Takeaways

- Proxmox installs directly on hardware (not inside a VM)
    
- Use **ext4** if you’re new, **ZFS** later
    
- Always configure:
    
    - Static IP
        
    - Proper hostname
        
- Web UI:
    
    ```
    https://IP:8006
    ```
    
- Switch to **no-subscription repo**
    
- Keep system updated
    


---

## Problems you may run into:
-  _Couldn't boot into Proxmox VE?_

> 🔍 **Most common causes**
>
> **1. 🔌 USB still plugged in**  
> Your system might just be booting back into the installer.  
> **Fix:**
> - Remove the USB drive  
> - Reboot again  
>
> **2. 🥇 Boot order issue**  
> Your system may not be booting from the disk where Proxmox was installed.  
> **Fix:**
> - Enter BIOS/UEFI  
> - Set your SSD/HDD as **first boot device**  
> - Disable USB boot (temporarily)  
>
> **3. UEFI vs Legacy mismatch (This one worked for me)**  
> If you install using UEFI but BIOS is set to Legacy (or vice versa), boot can fail silently.  
> **Fix:**
> - Check BIOS mode  
> - Try switching between **UEFI ↔ Legacy/CSM**  
> - Ideally, use **UEFI mode**
{: .prompt-warning }

---

# What’s next → **Discover the Proxmox VE interface: get to the point**


>**Express glossary: 10 terms to know:**
>_Before we begin, here are the words you'll encounter everywhere_
> - **Datacenter** : level “cluster” - anything that applies to all your servers 
> - **Node (Node)** : a physical Proxmox server 
> - **Guest (Guest)** : a VM or container hosted on a node 
> - **VM (QEMU)** : complete virtual machine with its own kernel 
> - **CT (LXC)** : container Linux, lighter than a VM 
> - **Storage** : storage space (disks, ISO, backups) 
> - **Task** : background operation (VM creation, backup, migration...) 
> - **Pool** : resource group to simplify permissions 
> - **Tag** : label to organize your machines 
> - **Snapshot** : snapshot of a VM/CT at any given time
{: .prompt-info}

## **The 3-level rule: where to configure what?**

Something to understand so you never get lost in Proxmox. The interface is organized into 3 hierarchical levels, and each level has its own responsibility.

### _**The mental model**_

Imagine a business:

- **Datacenter** = head office → decisions that concern all subsidiaries
- **Node** = a subsidiary → local management (building, maintenance)
- **Guest** = an employee → his workstation, his tools

**In Proxmox, it's the same:**
![Figure 1: The 3-level rule:where to configure  what?](/assets/img/homelab/Proxmox2.png)
_The 3-level rule_

### _**The 3 golden rules:**_



>**RULE 1: I CONFIGURE THE CLUSTER AT THE DATACENTER**
>
>Everything that should apply to **all servers** is configured at the **Datacenter** level:
>
>- Create a user? → Datacenter → Permissions
>- Add storage NFS shared? → Datacenter → Storage
>- Schedule automatic backups? → Datacenter → Backup
>- Configure the SSL certificate? → Datacenter → ACME
{: .prompt-tip}


>**RULE 2: I KEEP THE OS AT NODE**
>
>Everything related to **the physical server** is configured at **Node** level:
>
>- Update Proxmox? → Node→ Updates
>- Configure the network (bridge, VLAN)? → Node → System → Network
>- See physical disks? → Node → Disks
>- Access the shell root? → Node → Shell 
{: .prompt-tip}


>**RULE 3: I CONFIGURE THE WORKLOAD AT THE GUEST**
>
>Everything related to **a VM or container** is configured at **Guest** level:
>
>- Add RAM? → VM → Hardware
>- Change boot options? → VM → Options
>- Create a snapshot? → VM → Snapshots
>- Open a console? → VM → Console
{: .prompt-tip}

### _**In practice: where to go for...**_

Rather than an exhaustive table, here are the **10 most common actions** and where to do them:

|**I want...**|**Level**|**Path**|
|---|---|---|
|Create a VM|Header|Button **Create VM** (top)|
|Create a container|Header|Button **Create CT** (top)|
|Add a user|Datacenter|Permissions → Users → Add|
|Configure an auto backup|Datacenter|Backup → Add|
|Update Proxmox|Node|Updates → Refresh→ Upgrade|
|Configure the network|Node|System → Network|
|Add RAM to a VM|VM|Hardware → Memory → Edit|
|Open a console|VM/CT|Button **Console** or Console menu|
|See why it failed|Anywhere|Double-click the task at the bottom|
|Change my password|Header|User menu (your name) → Password|