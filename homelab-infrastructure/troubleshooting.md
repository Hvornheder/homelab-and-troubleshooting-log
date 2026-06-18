# Troubleshooting Log

Issues encountered during the home lab build, diagnostic process, and resolutions.

---

## Incident 01 — BIOS Flash Caused No Display Output

**Date:** June 2026
**System:** Dell Precision T3650
**Severity:** High: system unbootable, no display

### What Happened

After running a critical BIOS update (June 2026 release) via Dell SupportAssist on Windows, the system rebooted and produced no display output. The machine powered on; the fans were spinning, no beeps, no POST output, but the monitor showed no signal regardless of which DisplayPort was used or which monitor was connected.

### Diagnostic Process

**Step 1 — Ruled out simple causes:**
- Tried both DisplayPort outputs on the motherboard, no signal from either
- Tried a second monitor, same result
- Confirmed the machine was powering on (fans active, power light solid white)
- Confirmed no amber diagnostic blinks, white power light only indicates the system is running, not a hardware fault

**Step 2 — CMOS reset:**
- Drained power by unplugging and holding power button for 30 seconds
- Located and removed the CR2032 coin cell battery from the motherboard
- Waited 5 minutes to allow CMOS to fully clear
- Reinserted battery and attempted boot, no change

**Step 3 — Minimum hardware boot:**
- Removed dual-port NIC from PCIe slot
- Disconnected both IronWolf drives
- Reduced to one RAM stick
- Attempted boot, no change

**Step 4 — RAM reseating:**
- Fully removed and reseated RAM sticks
- Attempted boot, no change

**Step 5 — Dell USB BIOS recovery:**
After determining the BIOS flash had likely resulted in a corrupted or incomplete flash, attempted Dell's built-in recovery procedure:

1. Formatted a USB drive as FAT32 on a working PC
2. Downloaded the same BIOS update `.exe` from Dell support using the T3650 service tag
3. Renamed the file to `BIOS_IMG.rcv` (exact filename required, extension change, no other modifications)
4. Placed the file at the root of the USB drive (not inside any folder)
5. Inserted USB into a rear USB 2.0 port on the T3650
6. Held `Ctrl + Esc` on the keyboard and pressed the power button simultaneously
7. Held `Ctrl + Esc` for approximately 30 seconds after pressing power

The system detected the recovery file, USB drive activity light began flashing. The system power-cycled several times (expected behavior during BIOS reflash). After approximately 3 minutes, the system stayed off.

**Step 6 — First boot after recovery:**
Pressed power normally. System booted and displayed:

```
Invalid configuration information - please run setup program
Time-of-day not set - please run setup program
Alert: The amount of system memory has changed.

Continue | BIOS-Setup | Diagnostics
```

These messages are expected after a CMOS clear, the BIOS had lost its configuration including date/time and RAM configuration. Selected **BIOS-Setup**, loaded defaults, saved and exited. System rebooted normally.

### Root Cause

The initial BIOS flash either completed with a corrupted image or the BIOS reset display output priority in a way that prevented POST output on the available DisplayPort connections. The USB recovery procedure successfully reflashed the BIOS with a clean image, restoring normal operation.

### Resolution

BIOS recovery via Dell's `BIOS_IMG.rcv` USB recovery method fully restored the system. No hardware damage occurred. Total downtime: approximately 3 hours including diagnosis.

### What I Would Do Differently

- Before flashing BIOS on a machine without redundant display options, confirm which output the BIOS uses for POST, some Dell systems output POST exclusively through one port and switch to another after the OS loads
- Keep a bootable USB with the previous BIOS version available before flashing, as a faster rollback option
- Note the exact BIOS version before updating so rollback is straightforward if needed

---

## Incident 02 — Windows Display Loss After BIOS Recovery

**Date:** June 2026
**System:** Dell Precision T3650
**Severity:** Low: affected Windows only; Proxmox installation proceeded normally

### What Happened

After BIOS recovery, Windows loaded partially. Dell logo and spinning wheel appeared then the display lost signal when Windows attempted to initialize its graphics driver. The system appeared to be running (white power light, disk activity) but the monitor showed no signal.

### Diagnostic Process

Attempted multiple recovery approaches including Windows automatic repair, safe mode via interrupted boot cycles, and Windows recovery environment. Automatic repair ran for over 25 minutes without completing. Safe mode was not accessible through the standard 3x interrupted boot method.

**Key observation:** Hardware diagnostics run from the Windows recovery environment reported no hardware issues, the machine was healthy. The failure was isolated to the Windows graphics driver initialization, specifically the Intel UHD Graphics driver conflicting with the post-BIOS-recovery display state.

### Resolution

Since Windows was scheduled to be wiped for Proxmox installation, I made the decision to skip Windows recovery entirely and proceed directly to Proxmox installation. Proxmox uses completely different display initialization code and the issue did not manifest at all during the Proxmox installer or after installation.

**The Proxmox installer loaded and displayed correctly on first attempt.**

### Lesson

When the end goal is installing a different OS, time spent recovering the interim OS has diminishing returns. Identifying that the hardware was healthy (confirmed by Dell diagnostics) was the critical decision point, once hardware was ruled out, proceeding to the target OS was the most efficient path.

---

## Incident 03 — Package Repository Error on First Proxmox Boot

**Date:** June 2026
**System:** Dell Precision T3650 — Proxmox VE 9.2.2
**Severity:** Low: cosmetic, no functional impact

### What Happened

The Proxmox task log showed a red error on first login:

```
Error: command 'apt-get upd...' failed
```

### Cause

Proxmox ships configured to use its enterprise update repository, which requires a paid subscription. The free community version does not have subscription credentials, so the repository update fails.

### Resolution

Switched to the no-subscription community repository:

```bash
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt-get update && apt-get upgrade -y
```

All packages updated successfully. Error resolved.
