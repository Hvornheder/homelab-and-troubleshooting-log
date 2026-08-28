# Hardware

Full hardware specifications and selection rationale for the home lab build.

> **Platform change, August 2026:** The original Dell Precision T3650 host suffered a motherboard failure and was rebuilt on a standard retail platform in August 2026. All CPU, memory, storage, and NIC components carried over unchanged. See [incident-02](../incidents/incident-02-proxmox-motherboard-failure-platform-rebuild.md) for the full diagnostic process. Sections below reflect current hardware, with the original platform retained for context.

---

## Platform Selection: Original Rationale and What Changed

### Why Dell (original reasoning)

Dell hardware was highly praised during a shadowing engagement at MetroTech by a lead infrastructure technician. The reasoning: Dell's service tag system at dell.com/support maps directly to the exact hardware installed in your machine, providing targeted BIOS updates, driver downloads, and warranty lookup without guesswork. This mirrors enterprise-grade hardware lifecycle management in a way that consumer or white-label hardware cannot match.

Every firmware update on the original build was performed through Dell's service tag system; no manual driver searching, no version mismatches.

### What the platform failure revealed

That reasoning held up on the firmware side; the service tag system genuinely made updates precise. What it did not account for was **serviceability at the component level**.

When the motherboard failed, the OEM design became the constraint:

* The power supply uses a Dell proprietary form factor and pinout, so it cannot be reused on any standard board
* The chassis uses non-standard standoff positions and a non-standard front panel header, so a retail board cannot be mounted or wired into it
* The BIOS is vendor-locked, limiting firmware-level diagnostics during the failure
* Replacement OEM boards were priced at $200 and up, close to the cost of an entire retail platform

The rebuild reused every functional component and returned the system to standard parts. Firmware updates now come from the board vendor rather than a service tag lookup, which is a small loss; component-level serviceability is a much larger gain on a machine expected to run for years.

**Takeaway for future builds:** OEM workstations are excellent for firmware lifecycle management and poor for repairability. For an always-on host where any single component may need replacing, standard form factors matter more than vendor tooling.

---

## Primary Server (current)

Custom build, August 2026

| Spec | Detail |
|---|---|
| CPU | Intel Core i7-11700; 8 cores / 16 threads, 2.5GHz base / 4.9GHz boost (carried over) |
| Motherboard | ASUS Prime B560M-A; Intel B560 chipset, LGA 1200, Micro ATX |
| RAM | 32GB DDR4 (2 × 16GB UDIMM, non-ECC) (carried over) |
| Boot drive | Kingston SNV3S 1TB NVMe PCIe Gen 3 (carried over) |
| PSU | MSI MAG A550BN; 550W, 80 Plus Bronze, standard ATX |
| CPU cooler | Cooler Master Hyper 212 Black; 159mm single tower, PWM |
| Case | Micro ATX; 2 × internal 3.5" bays *(model TBD)* |
| Form factor | Micro ATX; multiple PCIe slots, 2 × 3.5" drive bays |
| Onboard NIC | Realtek gigabit (integrated) |

### Component selection rationale

**ASUS Prime B560M-A:** B560 is the consumer chipset paired with LGA 1200. Verified compatible with the existing i7-11700 and non-ECC DDR4 before purchase, since B560 supports neither Xeon W-1300 series processors nor ECC memory. Micro ATX was chosen deliberately over ATX: the larger board's extras (overclocking, Thunderbolt, a third M.2 slot, RGB) provide nothing to a headless hypervisor while forcing a larger case.

**MSI MAG A550BN, 550W:** Measured system draw is roughly 80 to 120W. 550W provides ample headroom for two spinning drives and an add-in NIC while leaving room for future expansion, without paying for capacity that will never be used.

**Cooler Master Hyper 212 Black:** Socket support for LGA 1200 verified against the manufacturer's installation manual rather than the retail listing. A single tower design was chosen over a dual tower specifically to avoid overhanging the RAM slots or the PCIe slot occupied by the add-in NIC. Height of 159mm set the minimum CPU clearance requirement for case selection.

**Case requirements:** Two or more internal 3.5" bays was the binding constraint, since the RAID 1 mirror requires both drives mounted internally. Most current cases ship with only one. Additional requirements: Micro ATX support, 160mm or greater CPU cooler clearance, standard bottom mounted ATX PSU, and at least two free expansion slots for the NIC.

---

## Original Primary Server (retired August 2026)

Dell Precision T3650 Workstation (Refurbished)

| Spec | Detail |
|---|---|
| CPU | Intel Core i7-11700 |
| RAM | 32GB DDR4 |
| Boot drive | Kingston 1TB NVMe PCIe Gen 3 |
| Form factor | Mid tower |
| Chassis NIC | Intel I219-LM (onboard, e1000e driver) |
| BIOS | Updated to 1.48.0 (July 2026) during diagnosis |
| Status | Motherboard failure; host migrated to retail platform |

Original selection rationale: the T3650 supports up to 128GB RAM, has multiple PCIe expansion slots, multiple internal drive bays, and Dell's full firmware ecosystem. A consumer desktop would have required replacement as the project expanded. Buying a machine with headroom upfront avoided that cost.

That reasoning was sound on capacity and proved correct in practice; every component except the board itself carried directly into the replacement platform.

---

## Add-in NIC

Intel 82576 Dual-Port Gigabit PCIe

| Spec | Detail |
|---|---|
| Ports | 2 × RJ45 gigabit |
| Driver | igb (Intel gigabit) |
| Interface | PCIe x1 |
| Bracket | Full-height and low-profile included |

**Why a dual-port NIC:** pfSense requires at minimum two separate network interfaces; one for WAN (internet in) and one for LAN (internal network). A single onboard NIC cannot fulfill both roles simultaneously. The 82576 provides:

* `nic1` (igb) → pfSense LAN port → connects to switch
* `nic2` (igb) → future DMZ port → isolated from personal LAN

The onboard NIC handles Proxmox management and will become the pfSense WAN port when the gateway is placed into bridge mode.

> **Post-migration note:** Linux derives predictable interface names from PCI topology, so the board change renamed every interface. The onboard NIC also changed vendor, from Intel I219-LM (e1000e) on the Dell board to Realtek on the ASUS board. Bridge mappings in `/etc/network/interfaces` and pfSense interface assignments require remapping to the new names; this is expected migration work, not a fault.

---

## Storage

### NAS Drives

Seagate IronWolf 4TB × 2 (ST4000VNZ06/006)

| Spec | Detail |
|---|---|
| Capacity | 4TB each |
| Interface | SATA 6Gb/s |
| Recording | CMR (conventional magnetic recording) |
| RPM | 5400 |
| Cache | 64MB |
| Rating | NAS; designed for 24/7 multi-drive operation |

**Why IronWolf specifically:**

* **NAS-rated:** Built for vibration tolerance and continuous operation. Desktop drives are not rated for always-on NAS use and have higher failure rates in that environment.
* **CMR not SMR:** ZFS (the filesystem used by TrueNAS) performs poorly with SMR (shingled magnetic recording) drives under sustained writes. CMR was a deliberate technical requirement. The IronWolf (non-Pro) explicitly uses CMR on this model.
* **Identical drives × 2:** RAID 1 mirrors require two identical drives. Matching model numbers eliminates compatibility variables.

The mirror carried through the platform failure intact. Data on these drives was never at risk, which was confirmed as the first step of incident triage.

### Boot Drive

Kingston SNV3S 1TB NVMe (PCIe Gen 3)

Dedicated to Proxmox OS and VM disk images. Kept entirely separate from the NAS storage drives. Proxmox, all VM operating systems, and VM disk images live here. The IronWolf drives are passed directly to TrueNAS and never touched by Proxmox itself.

Noted during incident triage: this is the only unmirrored component in the storage layout, making it the single point of failure for the host OS. Mirroring or imaging the boot drive is under consideration.

### Backup Drive

WD Elements External USB drive (5TB)

On-site backup of photo library. Part of the 3-2-1 backup strategy; see planned-phases.md for the full backup architecture.

---

## Networking

### Switch

TP-Link TL-SG108E; 8-port Gigabit Managed

| Spec | Detail |
|---|---|
| Ports | 8 × RJ45 gigabit |
| Management | Web-managed (Easy Smart) |
| VLAN support | 802.1Q; up to 32 VLANs |
| Additional | QoS, IGMP snooping, LAG |

**Why managed over unmanaged:** An unmanaged switch would have required replacement when VLAN segmentation becomes necessary for DMZ isolation. The TL-SG108E supports 802.1Q VLANs natively; therefore, buying it now means the infrastructure is already in place for future network segmentation without additional hardware cost.

### Gateway

Sercomm DG4244-S (WOW Internet provided)

| Spec | Detail |
|---|---|
| Standard | DOCSIS 3.1 |
| WiFi | WiFi 6 (802.11ax) |
| LAN ports | 3 × gigabit + 1 × 2.5G |
| Bridge mode | Enabled |

Bridge mode is active. The gateway operates as a modem only, passing the public IP through to the downstream router.

### Router (interim)

TP-Link Deco mesh unit, router mode

Originally deployed as a downstream access point with pfSense handling routing and DHCP. When the hypervisor failed, the pfSense VM went offline and took LAN routing and DHCP with it. The Deco was reconfigured from access point mode to full router mode, restoring household connectivity within the same evening and maintaining it for the duration of the rebuild.

This remains the active configuration until pfSense interface assignments are remapped to the new board. It also serves as a documented fallback: an always-on hypervisor is a single point of failure for the entire network, and having a tested path back to independent routing is worth keeping.

### Cabling

| Item | Detail |
|---|---|
| Backbone run | 100ft flat Cat6 (open issue; see below) |
| Mounting hardware | 6-32 UNC screw kit |

**Known issue:** the 100ft flat Cat6 run negotiates at 100 Mbps rather than 1 Gbps, capping throughput at roughly 90 Mbps on that segment while other segments reach 900 Mbps. Isolated to the cable itself by testing a client directly at the far end. Flat cable abandons the twisted pair geometry that cancels crosstalk and typically uses thinner conductors (roughly 30 AWG against 23 or 24 AWG), which is harmless at desk length but loses gigabit margin over 100ft. Planned resolution: replace with round Cat5e or Cat6, tested loose before being secured.

---

## Power Protection

CyberPower CP1500PFCLCD (1500VA / 1000W)

| Spec | Detail |
|---|---|
| Capacity | 1500VA / 1000W |
| Output waveform | PFC Sinewave |
| Battery outlets | 6 |
| Surge-only outlets | 6 |
| Runtime monitoring | USB (Type B) |
| AVR | Yes; automatic voltage regulation |

**Why PFC Sinewave specifically:** Both the original Dell 80 Plus Gold supply and the current MSI 80 Plus Bronze supply use Active PFC. Cheaper stepped sine wave UPS units can cause instability, erratic behavior, or damage to Active PFC power supplies during power transfer events. PFC Sinewave output matches utility power quality exactly, making it safe for sensitive electronics.

**What connects to battery outlets:** Server and switch; everything that needs to stay up during a power event and needs a graceful shutdown.

**USB monitoring:** The UPS connects to the server via USB (Type A to Type B cable). This allows Proxmox and TrueNAS to detect a power event programmatically and initiate a graceful shutdown before the battery depletes, protecting ZFS from mid-write corruption.

**Note:** The RJ45 ports on the back of this UPS are surge-protection passthroughs for ethernet cables, not network management ports. Network management requires an optional RMCARD accessory not used in this build.

**Surge protection gap identified:** A nearby lightning strike in August 2026 disrupted WAN service while UPS-protected equipment stayed online. Surges frequently enter through data cabling rather than mains power, and the UPS ethernet passthrough covers only cables actually routed through it. Routing the ISP-side ethernet run through surge protection is under review.

---

## Full Hardware Summary

| Component | Model |
|---|---|
| Motherboard | ASUS Prime B560M-A |
| CPU | Intel Core i7-11700 |
| RAM | 32GB DDR4 (2 × 16GB, non-ECC) |
| PSU | MSI MAG A550BN 550W 80 Plus Bronze |
| CPU cooler | Cooler Master Hyper 212 Black |
| Case | Micro ATX, 2 × 3.5" bays *(model TBD)* |
| Boot drive | Kingston SNV3S 1TB NVMe |
| Add-in NIC | Intel 82576 dual-port PCIe |
| NAS drives | Seagate IronWolf 4TB × 2 |
| Switch | TP-Link TL-SG108E |
| Router (interim) | TP-Link Deco, router mode |
| Gateway | Sercomm DG4244-S (bridge mode) |
| UPS | CyberPower CP1500PFCLCD |
| Backup | WD Elements 5TB external USB |
| Ethernet | 100ft flat Cat6 (replacement planned) |
| Mounting hardware | 6-32 UNC screw kit |
| Retired | Dell Precision T3650 (motherboard failure, August 2026) |

All software in this build is free and open source: Proxmox, TrueNAS SCALE, pfSense, WireGuard, Nginx, ntopng. Zero licensing costs.
