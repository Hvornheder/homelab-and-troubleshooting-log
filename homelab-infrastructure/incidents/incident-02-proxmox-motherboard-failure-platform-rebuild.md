# Incident 02 — Proxmox Hypervisor Motherboard Failure & Platform Rebuild

**Date:** July 31 – August 23, 2026 (symptom onset approx. 2–3 weeks prior)

**System:** Proxmox VE Hypervisor: Dell Precision 3650 Tower (production home lab host)

**Severity:** Critical — hypervisor offline; all hosted network services down (pfSense router/DHCP, TrueNAS storage)

**Resolution:** Motherboard failure isolated via systematic component elimination; host migrated to a standard retail platform with full reuse of CPU, RAM, storage, and NIC; Proxmox boot restored through UEFI reconfiguration

**Time to Resolution:** ~4 weeks, 6 working sessions (diagnosis, procurement, rebuild)

---

## Environment

### Original Platform
| Component | Details |
|---|---|
| Host | Dell Precision 3650 Tower |
| CPU | Intel LGA 1200 <!-- fill in exact model from UEFI main page or `lscpu` --> |
| RAM | 2x 16GB DDR4 UDIMM (32GB, non-ECC) |
| Boot Drive | 1TB Kingston NVMe SSD (Proxmox VE) |
| Data Drives | 2x 4TB HDD, RAID 1 mirror, passed through to TrueNAS VM |
| Network | PCIe NIC add-in card; pfSense VM serving as LAN router/DHCP |
| Video | Integrated graphics only (no discrete GPU) |
| BIOS | Updated to 1.48.0 (July 2026 release) during diagnosis |

### Replacement Platform
| Component | Details |
|---|---|
| Motherboard | ASUS Prime B560M-A (Intel B560, LGA 1200, Micro-ATX) |
| PSU | MSI MAG A550BN — 550W, 80+ Bronze, standard ATX |
| CPU Cooler | Cooler Master Hyper 212 Black (159mm single tower) |
| Case | Micro-ATX, 2x internal 3.5" bays <!-- fill in case model --> |
| Carried Over | CPU, both DDR4 DIMMs, 1TB NVMe, both 4TB HDDs, PCIe NIC |

---

## Incident Summary

A Proxmox VE host that had run stable for roughly four weeks began crashing intermittently, escalating over two to three weeks from occasional kernel panics to a complete no-POST condition and finally a power on/off loop. Because the host runs pfSense as a VM, the failure also took down routing and DHCP for the entire LAN.

Initial kernel panic analysis pointed to RAM or storage. Vendor diagnostics (Dell ePSA), a BIOS update, a fresh RTC battery, and a full PSU replacement all failed to resolve the fault. Systematic add-back testing then produced the decisive finding: a known-good minimal configuration that had just booted cleanly failed minutes later with zero changes. That baseline regression, combined with inconsistent CMOS-clear behavior, settings that failed to persist; and a "BIOS recovery failed to complete" event log entry, isolated the fault to the motherboard.

Rather than replace the proprietary OEM board, the host was rebuilt on a standard retail platform for approximately the same cost, restoring component-level serviceability. Network service for the household was maintained throughout via an interim router reconfiguration.

---

## Root Cause Analysis

**Final diagnosis: motherboard failure (POST/video path), ~80% confidence, reached by elimination.**

Supporting evidence:

1. **Baseline regression.** The strongest single finding. A minimal configuration (board + CPU + one DIMM + SSD) booted cleanly, then failed to produce video minutes later with no hardware changes. A fault that appears and disappears against an unchanged configuration excludes the swappable components and implicates the board itself.

2. **Vendor diagnostics passed while the fault persisted.** Dell ePSA thorough mode returned a clean pass (error code 2000-0000, validation 125478) across memory and drives; however, the failure continued. Passing diagnostics cleared the peripherals, not the board.

3. **Firmware-layer symptoms.** BIOS settings intermittently failed to persist, video output sometimes required a CMOS clear to restore, and the BIOS event log contained a "BIOS recovery failed to complete" entry which all pointing at the board's firmware/POST subsystem rather than any attached component.

4. **Every replaceable component was individually cleared:** drives (ePSA pass; fault persisted with drives disconnected), RAM (both sticks tested individually; zero-RAM test below), PSU (fault identical on a new unit), NIC (removed during minimal-config testing).

**Contributing factor — proprietary OEM platform.** The Precision 3650 uses a Dell-proprietary PSU form factor and pinout, chassis standoff layout, and front-panel header, plus a locked OEM BIOS. This constrained both diagnosis (limited firmware visibility) and repair (no drop-in retail board replacement), and directly shaped the rebuild decision.

---

## Troubleshooting Methodology

### Phase 1 — Symptom Progression & Kernel Panic Analysis

Captured and decoded the recurring panic before the system stopped POSTing:

```
BUG: kernel NULL pointer dereference
  in: find_lock_entries
  trace: truncate_inode_pages_range
```

Both functions live in the filesystem/page-cache layer, initially pointing to RAM or the boot SSD rather than networking; correctly ruling out the pfSense/WAN layer as a cause despite the network being the most visible symptom.

Data exposure was assessed immediately: VM data resides on a RAID 1 mirror (protected); the single boot SSD was identified as the only unprotected point of failure.

### Phase 2 — Firmware & Vendor Diagnostics

- Updated Dell BIOS to 1.48.0 (July 2026) via the F12 pre-boot flash utility from a FAT32 USB drive. The system booted fully into Proxmox post-flash, but the fault returned.
- Ran Dell ePSA thorough diagnostics: **full pass** (2000-0000, validation 125478), clearing memory and drives by vendor tooling.
- Replaced the CR2032 RTC battery to rule out CMOS corruption — power-loop behavior persisted.

### Phase 3 — PSU Replacement

Installed a Dell-compatible 460W replacement PSU. A stripped-down configuration (board, CPU, one DIMM, SSD) cold-booted cleanly into Proxmox and held — temporarily implicating the original PSU. Continued testing disproved this. The PSU was later returned for refund once the board diagnosis was confirmed.

### Phase 4 — Systematic Component Isolation

All testing performed with AC disconnected and flea power drained between steps.

| Step | Result | Conclusion |
|---|---|---|
| Add second DIMM to working baseline | No video | Ambiguous |
| **Revert to known-good single-DIMM baseline** | **No video — nothing changed** | **Decisive: fault is not the RAM** |
| Zero-RAM POST test | Clean repeating **2 amber / 3 white** LED code (Dell memory-fault code) | POST logic alive and correctly reporting |
| Single DIMM reinstalled | Solid white power LED, no fault code, no video | Board completes power-on but fails video/POST handoff |
| Repeated CMOS clears | Video restored inconsistently | Firmware/board-level instability |

The zero-RAM test is worth noting as a technique: with no memory installed, a healthy board must emit its memory-fault beep/LED pattern. Getting the correct fault code proved the board could still execute POST logic, making its *inconsistent* behavior with known-good RAM installed the signature of a board-level fault rather than a dead board or bad component.

### Phase 5 — Repair Path Evaluation & Procurement

Three paths were costed:

| Option | Cost | Decision |
|---|---|---|
| Used OEM replacement board (Dell IPRKL-RDT) | $200+ | Rejected — near-retail cost to stay on a proprietary, locked platform |
| Refurbished Precision 3650 | $560–$930 | Rejected — replaces working components already owned |
| Retail platform rebuild (board, PSU, cooler, case; reuse everything else) | ~$230 + case | **Selected** |

Key findings during procurement:

- **Proprietary constraints identified before purchase:** the Dell PSU form factor/pinout, chassis standoffs, and front-panel header are all non-standard, meaning a retail board requires a new ATX PSU and case. The recently purchased Dell-format PSU was returned to recover its cost.
- **Compatibility verification against manufacturer documentation, not marketplace listings.** Amazon spec fields proved unreliable across multiple listings (one motherboard listing claimed 1 GB memory support). Caught a near-miss between an AMD "B550M-A" and the Intel "B560M-A" — visually similar model names, physically incompatible sockets. Cooler socket support (LGA 1200) was verified from Cooler Master's installation manual rather than the retail listing.
- **Case requirements derived from the build:** Micro-ATX support, two or more internal 3.5" bays for the mirror, ≥160mm CPU cooler clearance, standard ATX PSU mount, and two free expansion slots for the NIC.

### Phase 6 — Service Continuity Workaround

With the hypervisor down, the pfSense VM could no longer provide routing or DHCP for the LAN. Rather than leave the household offline for the multi-week diagnosis and parts cycle:

- Reconfigured the downstream TP-Link Deco mesh unit from access-point mode to full router mode (factory reset, manual re-provisioning, dynamic WAN IP from the bridged ISP gateway).
- LAN connectivity restored the same evening and maintained for the full duration of the incident.

### Phase 7 — Rebuild & First Power-On Diagnosis

Migrated all carried-over components to the new platform with fresh thermal paste. First power-on produced no response — no fans, no LEDs.

Diagnosed by elimination against the ASUS board manual's header pinout map: the case's power-switch lead had been placed on the **COM port header (header 13)** — physically identical 2-pin fit — instead of the **system panel header (header 19, PWR_BTN pins)**. Relocated the lead; system POSTed immediately.

First POST enumerated every component correctly: CPU recognized, all 32GB RAM detected, both 4TB drives and the NVMe SSD visible.

### Phase 8 — UEFI Configuration for Linux Boot

The board halted each boot at a "New CPU installed — press F1" screen rather than loading the OS. Forced boot selection reached the Proxmox bootloader successfully, proving the install was intact and pointing to boot policy rather than hardware. Three settings resolved it:

1. **Secure Boot OS Type: "Windows UEFI Mode" → "Other OS."** The default only trusts Microsoft-signed bootloaders and silently rejects Proxmox's GRUB. (CSM was confirmed disabled — the install is pure UEFI.)
2. **Boot priority set to the "proxmox" UEFI entry**, with the raw-drive entry retained below it as a fallback path.
3. **Fast Boot and "Wait For F1 If Error" disabled** — full device initialization for a host with two spinning drives and an add-in NIC, and no keypress-gated halts on a headless machine.

Verified by full cold-boot test: system now boots unattended into Proxmox VE.

### Phase 9 — Post-Migration Follow-Up (scheduled)

Linux derives predictable network interface names from PCI topology, so the new board changed every NIC name. Remaining work, performed at the local console:

- Enumerate new interface names with `ip a`
- Update the bridge mapping in `/etc/network/interfaces`
- Reassign pfSense VM interfaces to the renamed devices
- Verify VM autostart and web UI reachability

This is expected, routine post-migration work — not a fault condition.

---

## Skills Demonstrated

| Skill | Application |
|---|---|
| Kernel panic analysis | Decoding NULL pointer dereference and call trace to subsystem level |
| Hardware fault isolation | Minimal-config testing, add-back methodology, baseline regression, flea-power discipline |
| POST diagnostics | Dell LED blink-code interpretation, zero-RAM fault-code verification |
| Vendor diagnostic tooling | Dell ePSA, F12 pre-boot BIOS flash, BIOS event log review |
| Platform migration | OEM-to-retail rebuild with full component reuse |
| Compatibility verification | Socket/chipset validation, cooler clearance, case constraints — from manufacturer documentation |
| UEFI administration | Secure Boot policy, CSM, boot entry management, unattended-boot hardening |
| Front-panel/header wiring | Pinout diagnosis from board manual documentation |
| Network continuity | Emergency AP-to-router conversion to maintain LAN service during outage |
| Cost engineering | Three-path repair costing; OEM part return to recover sunk cost |
| Risk management | Immediate data-exposure assessment; RAID 1 mirror confirmed protected throughout |

---

## Tools Used

- Dell ePSA pre-boot diagnostics
- Dell F12 BIOS flash utility (FAT32 USB)
- Dell diagnostic LED blink codes
- ASUS UEFI (Secure Boot, CSM, boot priority, Fast Boot configuration)
- ASUS motherboard manual (front-panel header pinout)
- Manufacturer spec sheets and installation manuals (compatibility verification)
- Proxmox VE, GRUB
- TP-Link Deco management app (router-mode conversion)
- `ip a` (interface enumeration, post-migration)

---

## Key Takeaways

- **Baseline regression is the decisive test for intermittent faults.** When a known-good configuration fails with zero changes, every component you were swapping is exonerated and the constant — the board — is implicated.
- **Passing vendor diagnostics does not clear the motherboard.** ePSA validated memory and drives while the board itself was failing; diagnostics test what the board can see, not the board's own stability.
- **The zero-RAM POST test is free and conclusive.** A correct memory-fault LED code with no RAM installed proves POST logic is alive, which reframes inconsistent behavior as board-level instability.
- **Proprietary OEM platforms change repair economics.** Non-standard PSU pinout, standoffs, front-panel headers, and a locked BIOS meant a $150 retail board plus supporting parts beat a $200+ used OEM board — and permanently restored standard-parts serviceability.
- **Marketplace listing specs are not documentation.** Multiple listings carried wrong or impossible specifications; manufacturer spec pages and installation manuals are the source of truth for compatibility.
- **Secure Boot's Windows-only default silently blocks Linux.** "Boots when forced, halts unattended" points at boot policy, not hardware. OS Type "Other OS" is the fix for unsigned Linux bootloaders on ASUS boards.
- **Identical connectors are not identical headers.** A 2-pin front-panel lead fits the COM header just as well as the power-switch pins; the manual's pinout map, not physical fit, is the authority.
- **Interface names are board-derived by design.** Any Linux platform migration will rename NICs and break bridge/VM configs; plan the remap as a standard migration step.

---

## What I Would Do Differently

- Maintain off-host backups of Proxmox configuration (`/etc/network/interfaces`, VM definitions) and the pfSense config export, so a platform migration is a restore rather than a rebuild
- Add basic health monitoring (SMART reporting, temperature alerts, remote syslog) to catch degradation before hard failure
- Keep a known-good spare PSU on hand for any always-on production host
- Formally verify CPU/chipset support before ordering rather than resolving it at first POST
- Run a burn-in window (memory test plus sustained load) after any rebuild before returning the host to production duty
