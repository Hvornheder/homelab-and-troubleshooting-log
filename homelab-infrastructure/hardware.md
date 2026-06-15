# Hardware

Full hardware specifications and selection rationale for the home lab build.

---

## Why Dell

Dell hardware was highly praised during a shadowing engagement at MetroTech by a lead infrastructure technician. The reasoning: Dell's service tag system at `dell.com/support` maps directly to the exact hardware installed in your machine, providing targeted BIOS updates, driver downloads, and warranty lookup without guesswork. This mirrors enterprise-grade hardware lifecycle management in a way that consumer or white-label hardware cannot match.

Every firmware update in this build was performed through Dell's service tag system; no manual driver searching, no version mismatches.

---

## Primary Server

**Dell Precision T3650 Workstation (Refurbished)**

| Spec | Detail |
|------|--------|
| CPU | Intel Core i7-11700 — 8 cores / 16 threads, 2.5GHz base / 4.9GHz boost |
| RAM | 32GB DDR4 |
| Boot drive | Kingston 1TB NVMe PCIe Gen 3 |
| Form factor | Mid tower — multiple PCIe slots, multiple drive bays |
| Chassis NIC | Intel I219-LM (onboard, e1000e driver) |

**Why the T3650 over a cheaper option:**
The T3650 supports up to 128GB RAM, has multiple PCIe expansion slots, multiple internal drive bays, and Dell's full firmware ecosystem. A consumer desktop would have required replacement as the project expanded. Buying a machine with headroom upfront avoided that cost.

---

## Add-in NIC

**Intel 82576 Dual-Port Gigabit PCIe**

| Spec | Detail |
|------|--------|
| Ports | 2 × RJ45 gigabit |
| Driver | igb (Intel gigabit) |
| Interface | PCIe x1 |
| Bracket | Full-height and low-profile included |

**Why a dual-port NIC:**
pfSense requires at minimum two separate network interfaces — one for WAN (internet in) and one for LAN (internal network). The onboard single NIC cannot fulfill both roles simultaneously. The 82576 provides:

- `nic1` (igb) → pfSense LAN port → connects to switch
- `nic2` (igb) → future DMZ port → isolated from personal LAN

The onboard `nic0` (e1000e) handles Proxmox management and will become the pfSense WAN port when the gateway is placed into bridge mode.

---

## Storage

### NAS Drives

**Seagate IronWolf 4TB × 2 (ST4000VNZ06/006)**

| Spec | Detail |
|------|--------|
| Capacity | 4TB each |
| Interface | SATA 6Gb/s |
| Recording | CMR (conventional magnetic recording) |
| RPM | 5400 |
| Cache | 64MB |
| Rating | NAS — designed for 24/7 multi-drive operation |

**Why IronWolf specifically:**
- **NAS-rated:** Built for vibration tolerance and continuous operation. Desktop drives are not rated for always-on NAS use and have higher failure rates in that environment.
- **CMR not SMR:** ZFS (the filesystem used by TrueNAS) performs poorly with SMR (shingled magnetic recording) drives under sustained writes. CMR was a deliberate technical requirement. The IronWolf (non-Pro) explicitly uses CMR on this model.
- **Identical drives × 2:** RAID-1 mirrors require two identical drives. Matching model numbers eliminates compatibility variables.

### Boot Drive

**Kingston SNV3S 1TB NVMe (PCIe Gen 3)**

Dedicated to Proxmox OS and VM disk images. Kept entirely separate from the NAS storage drives. Proxmox, all VM operating systems, and VM disk images live here. The IronWolf drives are passed directly to TrueNAS and never touched by Proxmox itself.

### Backup Drive

**WD Elements External USB drive (5TB)**

On-site backup of photo library. Part of the 3-2-1 backup strategy — see [planned-phases.md](planned-phases.md) for the full backup architecture.

---

## Networking

### Switch

**TP-Link TL-SG108E — 8-port Gigabit Managed**

| Spec | Detail |
|------|--------|
| Ports | 8 × RJ45 gigabit |
| Management | Web-managed (Easy Smart) |
| VLAN support | 802.1Q — up to 32 VLANs |
| Additional | QoS, IGMP snooping, LAG |

**Why managed over unmanaged:**
An unmanaged switch would have required replacement when VLAN segmentation becomes necessary for DMZ isolation. The TL-SG108E supports 802.1Q VLANs natively; therefore, buying it now means the infrastructure is already in place for future network segmentation without additional hardware cost.

### Gateway

**Sercomm DG4244-S (WOW Internet provided)**

| Spec | Detail |
|------|--------|
| Standard | DOCSIS 3.1 |
| WiFi | WiFi 6 (802.11ax) |
| LAN ports | 3 × gigabit + 1 × 2.5G |
| Bridge mode | Available on main dashboard |

Bridge mode is confirmed available and will be enabled when pfSense is ready to take over routing. In bridge mode the gateway acts as a dumb modem only, passing the public IP directly to pfSense.

---

## Power Protection

**CyberPower CP1500PFCLCD (1500VA / 1000W)**

| Spec | Detail |
|------|--------|
| Capacity | 1500VA / 1000W |
| Output waveform | PFC Sinewave |
| Battery outlets | 6 |
| Surge-only outlets | 6 |
| Runtime monitoring | USB (Type B) |
| AVR | Yes — automatic voltage regulation |

**Why PFC Sinewave specifically:**
The Dell T3650 uses an 80Plus Gold certified Active PFC power supply. Cheaper stepped-sine wave UPS units can cause instability, erratic behavior, or damage to Active PFC power supplies during power transfer events. PFC Sinewave output matches utility power quality exactly, making it safe for sensitive electronics.

**What connects to battery outlets:** Server and switch, everything that needs to stay up during a power event and needs a graceful shutdown.

**USB monitoring:** The UPS connects to the server via USB (Type A to Type B cable). This allows Proxmox and TrueNAS to detect a power event programmatically and initiate a graceful shutdown before the battery depletes, protecting ZFS from mid-write corruption.

> Note: The RJ45 ports on the back of this UPS are surge-protection passthroughs for ethernet cables, not network management ports. Network management requires an optional RMCARD accessory not used in this build.

---

## Full Hardware Summary

| Component | Model |
|-----------|-------|
| Server | Dell Precision T3650 (refurb) |
| Add-in NIC | Intel 82576 dual-port PCIe |
| NAS drives | Seagate IronWolf 4TB × 2 |
| Switch | TP-Link TL-SG108E |
| UPS | CyberPower CP1500PFCLCD |
| Ethernet | 100ft flat Cat6 |
| Mounting hardware | 6-32 UNC screw kit |

> All software in this build is free and open source — Proxmox, TrueNAS SCALE, pfSense, WireGuard, Nginx, ntopng. Zero licensing costs.
