# Proxmox Setup

Step-by-step documentation of the Proxmox VE installation and initial configuration on the Dell Precision T3650.

---

## Overview

Proxmox VE is a Type-1 bare-metal hypervisor, it installs directly onto the hardware and runs everything else (pfSense, TrueNAS, web servers, monitoring) as isolated virtual machines. This means the physical server never runs just one operating system; instead, Proxmox manages multiple independent VMs simultaneously, each with their own CPU, RAM, storage, and network allocations.

After installation, Proxmox is managed entirely through a web browser from any machine on the network.

---

## Pre-Installation: Firmware Updates

Before installing any OS, all firmware was updated to current versions using Dell's service tag system at `dell.com/support`. Updates were downloaded and run directly on the machine while Windows was still present, then Windows was wiped when Proxmox installation began.

| Update | Category | Date |
|--------|----------|------|
| Dell Precision T3650 BIOS | Critical | Jun 2026 |
| Intel Management Engine Components | Chipset | Dec 2025 |
| Intel PCIe Ethernet Controller Driver | Network | Oct 2025 |
| Intel Chipset Device Software | Chipset | Dec 2024 |
| Intel UHD Graphics Driver | Video | Apr 2026 |
| Intel Rapid Storage Technology | Storage | Feb 2023 |
| Intel Serial IO Driver | Chipset | Aug 2021 |

**BIOS recovery incident:** The initial BIOS flash caused a display initialization failure on reboot. The system was powering on but outputting no video signal. This is a known behavior when a BIOS update resets display output priority. Recovery was performed using Dell's built-in USB BIOS recovery mode:

1. Formatted a USB drive as FAT32
2. Downloaded the BIOS `.exe` from Dell support and renamed it `BIOS_IMG.rcv`
3. Placed file at the root of the USB drive
4. With USB inserted, held `Ctrl+Esc` while pressing the power button
5. System detected the recovery file, power-cycled several times, and successfully restored the BIOS

Full incident details are in [troubleshooting.md](troubleshooting.md).

---

## Installation

**Media preparation (on a separate PC):**

1. Downloaded Proxmox VE 9.2.2 ISO from `proxmox.com/en/downloads`
2. Flashed to USB drive using Balena Etcher

**Boot and install on T3650:**

1. Inserted USB, powered on, pressed `F12` for one-time boot menu
2. Selected USB drive
3. Chose **Install Proxmox VE (Graphical)**

**Installer configuration:**

| Setting | Value | Reason |
|---------|-------|--------|
| Target disk | `/dev/nvme0n1` (Kingston 1TB NVMe) | Separate from NAS drives |
| Filesystem | ext4 | Standard for Proxmox boot volume |
| Hostname | `pve` | |
| IP address | `192.168.0.9/24` | Static — ensures consistent reachability |
| Gateway | `192.168.0.1` | Sercomm gateway LAN address |
| DNS | `75.76.160.3` | WOW ISP DNS |
| Management NIC | `nic0` (e1000e — onboard Intel) | Onboard NIC for management; 82576 ports reserved for pfSense |

Installation completed and system rebooted automatically. USB removed after reboot.

---

## Initial Configuration

### Accessing the Web UI

From any machine on the same network, navigated to:

```
https://192.168.0.9:8006
```

Logged in with username `root`, realm **Linux PAM standard authentication**.

---

### Verifying Virtualization Support

Proxmox requires Intel VT-x to be enabled for VM hardware acceleration. Verified before creating any VMs:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

**Output: 32** — confirms all 16 threads (8 cores × 2 with hyperthreading) are reporting VMX virtualization support. Full hardware virtualization acceleration is available.

---

### Confirming Storage Devices

Verified that Proxmox can see all storage devices and that the IronWolf drives are unconfigured (available for TrueNAS passthrough):

```bash
lsblk
```

```
NAME        MAJ:MIN  SIZE  TYPE  MOUNTPOINT
nvme0n1     259:0    931G  disk
├─nvme0n1p1           1G  part  /boot/efi
├─nvme0n1p2           1G  part  /boot
└─nvme0n1p3         929G  part  (Proxmox LVM)
sda           8:0    3.6T  disk              ← IronWolf 1 — unconfigured
sdb           8:16   3.6T  disk              ← IronWolf 2 — unconfigured
```

---

### NIC Identification

Confirmed all three network interfaces and their drivers:

| Proxmox name | Driver | MAC prefix | Role |
|---|---|---|---|
| nic0 | e1000e | 30:d0:42 | Onboard Intel I219-LM — Proxmox management (current), pfSense WAN (future) |
| nic1 | igb | 98:b7:85 | 82576 port 1 — pfSense LAN (future) |
| nic2 | igb | 98:b7:85 | 82576 port 2 — DMZ (future) |

nic1 and nic2 share a MAC prefix because they are two ports on the same physical card.

---

## Current Proxmox Resource Utilization

| Resource | Installed | In use (host only) |
|----------|-----------|-------------------|
| CPU | i7-11700 · 16 threads | ~0% |
| RAM | 32GB DDR4 | ~5.8% |
| NVMe | 1TB Kingston | ~4.4% |
| NAS drives | 2 × 4TB IronWolf | Unconfigured |

The low host utilization is intentional as Proxmox itself consumes minimal resources so the headroom is available for VMs. With 32GB RAM and 16 threads, the machine can comfortably run 5-6 VMs simultaneously without resource contention.

---

## Web UI Access

The Proxmox dashboard is accessible from any machine on the local network at:

```
https://192.168.0.9:8006
```

From this dashboard, all VM creation, storage management, network configuration, and monitoring is handled without ever needing a monitor or keyboard attached to the server.
