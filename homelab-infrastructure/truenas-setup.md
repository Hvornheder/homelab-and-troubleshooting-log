# TrueNAS SCALE Setup

Documentation of the TrueNAS SCALE VM configuration, RAID-1 storage pool creation, dataset setup, UPS integration, and automated maintenance scheduling.

---

## Overview

TrueNAS SCALE runs as a VM inside Proxmox and acts as the NAS (Network Attached Storage) operating system for the home lab. Its primary role is managing the two Seagate IronWolf drives in a RAID-1 mirror configuration using ZFS, providing a self-hosted photo library accessible over the local network via SMB, which will replace paid cloud storage.

---

## Step 1 — ISO Download into Proxmox

Rather than downloading the ISO to a separate machine and transferring it, Proxmox can pull it directly via URL:

- Navigated to `truenas.com` → Community Edition → TrueNAS SCALE stable
- Copied the direct download link
- In Proxmox web UI: Local (pve) → ISO Images → Download from URL
- Pasted URL, clicked Query URL, then Download
- Proxmox downloaded the ISO directly into local storage

**ISO used:** TrueNAS-SCALE-25.10.4

---

## Step 2 — VM Creation

**General settings:**

| Setting | Value | Notes |
|---------|-------|-------|
| VM ID | 101 | |
| Name | TrueNAS-Scale | |
| Start at Boot | Yes | TrueNAS starts automatically when Proxmox boots |
| HA | Disabled | HA requires a multi-node cluster, because I am running a single node setup, this makes it non-functional and potentially disruptive |

**OS:** TrueNAS-SCALE-25.10.4.iso, Linux, kernel 2.6+

**Disk:**

| Setting | Value | Notes |
|---------|-------|-------|
| Bus | SCSI | |
| Storage | local-lvm | Proxmox LVM on the Kingston NVMe |
| Size | 16 GiB | Initial 8GiB was insufficient, resized via Disk Action → Resize Disk |
| SSD Emulation | Enabled | Boot disk resides on NVMe, emulation improves performance |
| IO Thread | Enabled | |

**CPU:**

| Setting | Value |
|---------|-------|
| Sockets | 1 |
| Cores | 2 |
| Type | x86-64-v2-AES |

**Memory:**

| Setting | Value | Notes |
|---------|-------|-------|
| RAM | 16384 MiB (16 GiB) | ZFS ARC requires adequate RAM, 16GiB provides strong caching while preserving headroom for future VMs |
| Ballooning | Disabled | ZFS manages memory aggressively via ARC and does not handle dynamic memory reallocation gracefully, ballooning must be off |
| Minimum Memory | 16384 MiB | Fixed allocation — no dynamic scaling |

> **Why ballooning is disabled:** Proxmox's memory ballooning dynamically reclaims RAM from VMs when the host needs it. ZFS ARC assumes stable memory availability and can become unstable or corrupt under dynamic memory pressure. This is a known incompatibility, ballooning must always be disabled for any VM running ZFS.

**Network:** Default (virtio, vmbr0)

---

## Step 3 — Disk Passthrough

The IronWolf drives are physically installed in the Dell T3650 and visible to Proxmox, but TrueNAS running as a VM cannot access them unless Proxmox explicitly passes them through. Disk passthrough gives TrueNAS direct, exclusive ownership of the physical drives, bypassing Proxmox's storage layer entirely so ZFS can manage them natively.

**Why `/dev/disk/by-id/` instead of `/dev/sda`:**

`/dev/sda` and `/dev/sdb` are assigned by the kernel at boot based on detection order. That order can change; a reboot, a cable swap, a new drive, and the labels shift. `/dev/disk/by-id/` paths are derived from each drive's factory serial number and never change regardless of port, cable, or boot sequence. Using them prevents the pool from failing to import because a drive's label changed.

**Identifying drive IDs:**

```bash
ls /dev/disk/by-id
```

IronWolf drive IDs identified:
- `ata-ST4000VN006-3CW104_ZW640YJC`
- `ata-ST4000VN006-3CW104_ZW640YMY`

**Attaching drives to VM 101:**

```bash
qm set 101 -scsi1 /dev/disk/by-id/ata-ST4000VN006-3CW104_ZW640YJC,serial=ZW640YJC
qm set 101 -scsi2 /dev/disk/by-id/ata-ST4000VN006-3CW104_ZW640YMY,serial=ZW640YMY
```

> **Note on serial parameter:** TrueNAS threw a "duplicate serial numbers" error when creating the ZFS pool, a known issue with Proxmox disk passthrough where serial numbers aren't passed through to the guest OS. Adding `,serial=SERIALNUMBER` explicitly to each passthrough command resolves this by making the drive's serial visible to TrueNAS. A full VM power cycle (not just reboot) was required after the change for TrueNAS to recognize the updated serial information.

**Verification:** Confirmed both IronWolf drives visible under TrueNAS UI → Storage → Disks.

---

## Step 4 — TrueNAS Installation

TrueNAS was installed inside the VM from the ISO:

- Selected: Install/Upgrade → Install to disk
- Target: `sda` (QEMU virtual disk — the 16GiB boot disk)
- Web UI authentication: configured via web UI
- EFI boot: enabled
- Installation succeeded — system rebooted into TrueNAS

**TrueNAS web UI accessible at:** `http://192.168.0.47`

**Credentials configured:** truenas_admin + strong password

---

## Step 5 — RAID-1 ZFS Mirror Pool

With both IronWolf drives visible to TrueNAS, the storage pool was created:

- Storage → Create Pool
- Pool name: `IronWolf-RAID1-Config`
- Layout: **Mirror** both 3.64 TiB drives in a RAID-1 mirror vdev
- Encryption: **Disabled**

> **Why no encryption:** ZFS pool-level encryption requires manual passphrase entry (or stored key retrieval) on every reboot to unlock the pool. In a home lab where automatic recovery from power events is important, pool encryption adds operational complexity without meaningful security benefit for this threat model. Individual dataset encryption can be added later if needed. Pool-level encryption cannot be added after creation, this decision is permanent.

**Result:** 3.64 TiB usable capacity, fully redundant, one drive can fail completely with zero data loss.

---

## Step 6 — Dataset Creation

A dataset was created inside the pool for photo library storage:

- Storage → Datasets → Add Dataset
- Parent path: `IronWolf-RAID1-Config`
- Name: `Photography-Library`
- Preset: **SMB** — optimizes dataset settings for Windows file sharing

SMB share created simultaneously with the dataset for immediate accessibility.

---

## Step 7 — SMB Sharing

SMB (Server Message Block) is the Windows-native file sharing protocol. With the SMB preset applied during dataset creation, the share was ready with minimal additional configuration:

- Share name: `Photography-Library`
- SMB service started and set to start automatically

The photo library is now accessible from Windows machines on the local network as a standard network share.

---

## Step 8 — UPS USB Passthrough

The CyberPower CP1500PFCLCD connects to the Dell T3650 via USB for power monitoring. TrueNAS needs direct access to this USB connection to monitor battery status and trigger graceful shutdowns during outages.

**Identifying the UPS on the Proxmox host:**

```bash
lsusb
# Output: Bus 001 Device 002: ID 0764:0601 Cyber Power System, Inc. PR1500LCDRT2U UPS
```

**Adding USB passthrough to VM 101:**

Proxmox web UI → VM 101 → Hardware → Add → USB Device → selected CyberPower unit (port 1-6)

**Verification in TrueNAS:**

```bash
lsusb
# CyberPower unit now visible in TrueNAS
```

**NUT (Network UPS Tools) configuration in TrueNAS:**

- System → Services → UPS
- Driver: `usbhid-ups`
- Port: `auto`
- Mode: Master (UPS directly connected to this system)
- Shutdown mode: UPS reaches low battery
- Shutdown timer: 60 seconds
- Start automatically: enabled

> **On driver selection:** The CP1500PFCLCD is not explicitly listed in TrueNAS's UPS driver list but the `usbhid-ups` driver supports the full CyberPower PFC Sinewave line via standardized USB HID protocol. The CP1000PFCLCD and CP1500PFCLCD use identical USB communication, the driver doesn't require an exact model match.

**Verification:**

```bash
upsc ups@localhost
# Returns full UPS telemetry; battery charge, voltage, load, estimated runtime
```

---

## Step 9 — Automated Maintenance Scheduling

### Scrub schedule

A ZFS scrub reads every block on the pool and verifies checksums, detecting and correcting silent data corruption (bit rot) that RAID alone cannot protect against.

- Storage → IronWolf-RAID1-Config → Scrub
- Schedule: **Monthly**
- Threshold: 28 days

### Snapshot schedule

ZFS snapshots capture point-in-time copies of the dataset state, enabling recovery from accidental deletion or corruption without a full backup restore.

- Data Protection → Periodic Snapshot Tasks → Add
- Dataset: `IronWolf-RAID1-Config` (entire pool)
- Schedule: **Weekly**
- Lifetime: 1 month
- Naming schema: auto

---

## Step 10 — External USB Backup (Pending)

The WD Elements 5TB external drive is physically connected and visible to TrueNAS. However TrueNAS SCALE does not include NTFS write support, the drive's current NTFS formatting cannot be mounted for write access within the TrueNAS environment.

**Root cause confirmed:**

```bash
# ntfs-3g not installed
which ntfs-3g
# ntfs-3g not found

# Package installation blocked by TrueNAS
sudo apt-get install ntfs-3g
# Package management tools are disabled on TrueNAS appliances.
```

**Resolution:** The WD Elements will be reformatted to ext4 before the photo library is populated. ext4 is natively supported by TrueNAS SCALE and the local rsync task will then be configured to sync `Photography-Library` to the external drive on a scheduled basis. No data currently exists on the NAS to back up, so this step does not block phase 2 completion.

---

## Phase 2 Summary

| Step | Description | Status |
|------|-------------|--------|
| 1 | TrueNAS SCALE ISO downloaded into Proxmox | ✅ |
| 2 | TrueNAS VM created with correct resource allocation | ✅ |
| 3 | IronWolf drives passed through via disk-by-id | ✅ |
| 4 | TrueNAS SCALE installed and accessible | ✅ |
| 5 | ZFS RAID-1 mirror pool created | ✅ |
| 6 | Photography-Library dataset created | ✅ |
| 7 | SMB sharing enabled | ✅ |
| 8 | UPS USB passthrough and NUT monitoring configured | ✅ |
| 9 | Monthly scrubs and weekly snapshots scheduled | ✅ |
| 10 | External USB backup job | ⏸ Pending drive reformat |
