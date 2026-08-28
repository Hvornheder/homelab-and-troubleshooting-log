# Incident 01 — Windows Boot Failure & RAID 0 Recovery

**Date:** May 29–31, 2026
**System:** Custom Desktop — ROG Crosshair VII Hero (WiFi), AMD X470 Chipset
**Severity:** Critical — System completely non-bootable
**Resolution:** Full Windows 11 Pro clean install with AMD RAID driver injection
**Time to Resolution:** ~12 hours (multi-session)

---

## Environment

| Component | Details |
|---|---|
| Motherboard | ASUS ROG Crosshair VII Hero (WiFi) |
| Chipset | AMD X470 |
| BIOS Version (start) | 0804 (July 2018) |
| BIOS Version (end) | 5901 (October 2025) |
| Storage | 2x SATA HDD (Toshiba + WDC) in RAID 0 array |
| Storage Controller | AMD X470 RAID bus controller |
| OS | Windows 11 Pro (fresh install) |
| Boot Mode | UEFI / GPT |

---

## Incident Summary

The system failed to boot following a Windows Update cycle. Error code `0xc000000f` indicated corrupted or missing Boot Configuration Data (BCD). Initial repair attempts using a Windows 11 recovery USB failed due to the AMD RAID controller requiring drivers that are not included in the default WinPE environment. This cascaded into a multi-stage recovery process involving BIOS firmware updates, Linux-based hardware diagnostics, manual RAID driver injection into the Windows installer, and manual EFI bootloader reconstruction.

---

## Root Cause Analysis

Three compounding factors caused this incident:

**1. Outdated BIOS Firmware**
The system was running BIOS version 0804 from July 2018 — seven years without a firmware update. Known bugs in AMD RAID handling and Windows 11 compatibility fixes had accumulated across dozens of subsequent releases. A Windows Update likely triggered a bootloader write operation that the outdated firmware mishandled, corrupting the BCD store.

**2. AMD RAID 0 Controller Dependency**
The two HDDs were configured in a RAID 0 (striped) array managed by the AMD X470 onboard controller. RAID 0 splits data across both drives simultaneously for performance, meaning neither drive is independently readable. The Windows PE recovery environment ships without AMD RAID drivers, rendering the array completely invisible to any standard recovery tool.

**3. Deferred Maintenance**
No driver updates, BIOS updates, or storage health checks had been performed in several years. On aging hardware running a demanding RAID configuration, this created a brittle environment where a routine update could cause catastrophic boot failure.

---

## Troubleshooting Methodology

### Phase 1 — Initial Diagnosis

Observed error: `Windows failed to start — Status: 0xc000000f`

Attempted standard recovery steps:
- Booted Windows 11 repair USB → repair environment crashed immediately
- Tested multiple USB drives including a previously verified working drive → same result
- Ruled out faulty USB media as the cause

**Key finding:** The repair environment crashed at the moment it attempted to scan storage — consistent with a missing storage driver rather than OS corruption.

Checked BIOS SATA configuration:
```
SATA Mode: RAID
NVMe RAID Mode: Enabled
```
Confirmed AMD RAID as the likely cause of WinPE crashes.

---

### Phase 2 — BIOS Firmware Update

Identified board as ROG Crosshair VII Hero **(WiFi variant)** — critical distinction as ASUS maintains separate firmware for WiFi and non-WiFi models. Initial flash attempts failed because the non-WiFi BIOS file was downloaded.

**Method used: USB BIOS Flashback**
- Downloaded correct BIOS (v5901) from ASUS support page for WiFi variant
- Ran BIOSRenamer utility to rename file to `C7H.CAP` (required checksum embedded by tool)
- Copied to FAT32-formatted USB 2.0 drive (USB 3.0 drives cause compatibility issues with Flashback)
- Located dedicated FLBK port on rear IO panel
- Held BIOS Flashback button 3 seconds until LED blinked — flash completed without powering on system

**Result:** BIOS updated from 0804 → 5901 successfully.

---

### Phase 3 — Linux-Based Hardware Diagnostics

Switched SATA mode to AHCI in BIOS in preparation for clean install. Created Ubuntu 26.04 LTS Live USB using Rufus (GPT / UEFI mode) to investigate why drives remained invisible even in AHCI mode.

**Commands run:**

```bash
lsblk              # Listed block devices — only USB drives visible, no HDDs
sudo fdisk -l      # Confirmed no additional disks detected
sudo dmesg | grep -i ata   # Output: "SATA link down (SStatus 0 SControl 300)"
lspci              # Key output below
lsmod | grep ahci  # Confirmed AHCI driver loaded correctly
```

**Critical lspci finding:**
```
RAID bus controller: Advanced Micro Devices, Inc. [AMD] device 43bd (rev 01)
RAID bus controller: Advanced Micro Devices, Inc. [AMD] device 7916 (rev 51)
```

The AMD X470 controller identified itself as a **RAID bus controller** regardless of BIOS SATA mode setting. This means the controller fundamentally requires AMD RAID drivers to communicate with any OS — AHCI mode in BIOS is insufficient to make standard AHCI drivers work with this chipset.

**Conclusion:** Standard AHCI driver path was a dead end. RAID mode and AMD-specific drivers were required for the installation.

---

### Phase 4 — AMD RAID Driver Acquisition

Downloaded AMD RAIDXpert2 driver package (v9.3.3.267.0) — a separate package from the general AMD chipset software.

Located driver files at path:
```
RAIDXpert\16299\Drivers\RAID\Win11\x64\NVMe_DID\
├── rcbottom\
├── rccfg\
└── rcraid\
```

Copied `x64` folder to secondary USB drive for use during Windows installation.

Switched BIOS back to RAID mode and moved both HDD SATA cables to SATA6G_1 and SATA6G_2 (primary ports guaranteed active on this board).

---

### Phase 5 — Windows 11 Installation with Driver Injection

Booted Windows 11 installer USB. Navigated to drive selection screen:

`Install Now → No Product Key → Windows 11 Pro → Accept License → Custom Install`

At **"Select location to install Windows"** screen — drives not visible (expected, RAID driver not yet loaded).

Clicked **Load Driver** → Browse → navigated to RAIDXpert USB.

**Driver loading sequence (order matters):**
1. `rcbottom` — loaded first (base layer driver)
2. `rccfg` — loaded second (configuration layer)
3. `rcraid` — loaded last (AMD-RAID Controller Storport)

After loading all three in sequence, RAID array appeared:
```
Disk 2 — 1.8TB (combined RAID 0 array)
```

Used `diskpart` via Shift+F10 to wipe existing partitions and RAID metadata:
```cmd
select disk 2
clean
convert gpt
exit
```

Windows installation proceeded and completed file copy phase successfully.

---

### Phase 6 — Bootloader Reconstruction

Windows installation completed file copy but failed to write bootloader correctly to the EFI partition — system did not appear in BIOS boot options post-install.

**Diagnosis:** RAID drivers present during install but not injected into the installed Windows image, preventing Windows from seeing its own drives on first boot attempt.

**Fix 1 — Inject RAID drivers into installed Windows image:**
```cmd
dism /image:E:\ /add-driver /driver:D:\x64\NVMe_DID\rcraid /recurse
```
This permanently embedded the AMD RAID drivers into the Windows installation so they load on every subsequent boot.

**Fix 2 — Manually reconstruct EFI bootloader:**

Identified volumes via diskpart:
```
Volume 2 (E:) — 1861GB  — Windows installation (Primary)
Volume 3 (Z:) — 200MB   — EFI System Partition (FAT32)
```

Assigned letter to EFI partition and ran bcdboot:
```cmd
select volume 3
assign letter=Z
exit

bcdboot E:\Windows /s Z: /f UEFI
```

Output: `Boot files successfully created`

**Fix 3 — Corrected BIOS boot setting:**

Identified critical BIOS setting blocking all storage devices from appearing in boot menu:

```
Boot → CSM → Boot from storage devices: Ignore  ← was set to this
                                        ↓
                                  UEFI driver only  ← changed to this
```

This single setting had prevented Windows from appearing in boot priorities throughout the entire process.

---

### Phase 7 — Post-Install Driver Cleanup

Identified unrecognized PCIe WiFi card via Device Manager Hardware ID:
```
PCI\VEN_14E4&DEV_43C3&SUBSYS_86FB1043&REV_04
```

- VEN_14E4 = Broadcom chipset manufacturer
- SUBSYS_86FB1043 = ASUS subsystem ID
- Identified as: **ASUS PCE-AC88** 802.11ac dual-band PCIe WiFi adapter

Downloaded Windows 10 64-bit driver from ASUS support page (fully compatible with Windows 11). Installed successfully — adapter operational.

---

## Skills Demonstrated

| Skill | Application |
|---|---|
| BIOS/UEFI Firmware Management | USB Flashback update, boot configuration, CSM settings |
| RAID Configuration | RAID 0 theory, AMD RAIDXpert2, array identification |
| Windows PE Environment | Driver injection, Shift+F10 command prompt access |
| Disk Management (diskpart) | list disk/volume/partition, select, clean, convert gpt, assign |
| Bootloader Reconstruction | bootrec, bcdboot, EFI partition identification |
| DISM | Offline driver injection into Windows image |
| Linux CLI Diagnostics | lsblk, lspci, fdisk, dmesg, lsmod, lshw, grep |
| Hardware Identification | PCI Vendor/Device ID lookup, Device Manager |
| Systematic Troubleshooting | Hypothesis testing, root cause elimination, phased approach |
| Driver Research | Identifying correct driver packages from vendor pages |

---

## Tools Used

- ASUS EZ Flash 3 / USB BIOS Flashback
- Windows 11 Installation Media (USB)
- AMD RAIDXpert2 Driver Package (v9.3.3.267.0)
- Ubuntu 26.04 LTS Live USB
- Rufus (bootable USB creation)
- diskpart, bootrec, bcdboot, DISM (Windows CLI)
- lsblk, lspci, fdisk, dmesg, lsmod (Linux CLI)
- Windows Device Manager

---

## Key Takeaways

- RAID 0 arrays require vendor-specific drivers in WinPE — standard Windows recovery tools are blind to them by default
- AMD X470 identifies as a RAID bus controller at the hardware level regardless of BIOS SATA mode — AHCI mode alone is insufficient
- BIOS firmware maintenance is not optional on long-running systems — 7 years of deferred updates created cascading compatibility issues
- Driver loading order matters in WinPE — rcbottom → rccfg → rcraid must be loaded in sequence
- DISM offline driver injection is the correct method to make drivers persistent across reboots after a fresh install
- Always verify CSM storage boot settings after major BIOS changes — a single misconfigured setting can make installed OS invisible to the boot manager
- Hardware ID strings (VEN/DEV/SUBSYS) are the definitive method for identifying unknown devices when manufacturer labels are unavailable

---

## What I Would Do Differently

- Implement regular BIOS update schedule (check quarterly)
- Document RAID configuration details at time of setup for future reference
- Consider migrating to RAID 1 (mirroring) or a dedicated backup solution — RAID 0 provides no redundancy
- Run periodic SMART health checks on HDDs given their age
- Maintain a recovery USB with pre-injected AMD RAID drivers for faster future recovery
