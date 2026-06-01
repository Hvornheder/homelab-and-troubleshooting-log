# Homelab & Troubleshooting Log

**Maintainer:** Hunter Vornheder
**GitHub:** [github.com/Hvornheder](https://github.com/Hvornheder)
**Related Repo:** [security-learning-log](https://github.com/Hvornheder/security-learning-log)

---

## About This Repository

This repository documents real-world IT troubleshooting incidents, homelab builds, and hands-on technical work completed as part of my independent transition into IT and cybersecurity.

Each entry is written to be employer-readable — covering the problem, the diagnostic process, the tools and commands used, the resolution, and what I would do differently. The goal is not just to record what happened, but to demonstrate how I think through technical problems systematically.

I come from a long background in operations and facilities management. I am currently pursuing CompTIA Security+ (SY0-701) and building practical experience through independent lab work, hardware projects, and real troubleshooting scenarios on my personal systems.

---

## Repository Structure

```
homelab-and-troubleshooting-log/
│
├── README.md
│
└── incidents/
    └── incident-01-windows-boot-failure-raid-recovery.md
```

---

## Incident Log

| # | Title | Date | Skills Covered | Status |
|---|---|---|---|---|
| 01 | [Windows Boot Failure & RAID 0 Recovery](incidents/incident-01-windows-boot-failure-raid-recovery.md) | May 2026 | BIOS firmware, AMD RAID, WinPE driver injection, BCD reconstruction, Linux CLI diagnostics | ✅ Resolved |

---

## Skills & Tools Covered Across This Repo

This section will grow with each new entry. Current coverage includes:

**Operating Systems & Environments**
- Windows 11 Pro installation and configuration
- Windows PE (WinPE) recovery environment
- Ubuntu Linux (Live environment diagnostics)

**Hardware & Firmware**
- ASUS BIOS/UEFI firmware management
- USB BIOS Flashback
- RAID 0/1 configuration (AMD X470 chipset)
- SATA and PCIe hardware identification

**Command Line Tools**
- diskpart, bootrec, bcdboot, DISM (Windows)
- lsblk, lspci, fdisk, dmesg, lsmod, lshw (Linux)

**Troubleshooting Methodology**
- Root cause analysis
- Hypothesis-driven elimination
- Hardware ID lookup and driver research
- EFI/UEFI boot configuration

---

## Certifications In Progress

- **CompTIA Security+ (SY0-701)** — CompTIA CertMaster Learn, actively studying

---

## Connect

- **LinkedIn:** [linkedin.com/in/huntervornheder](https://linkedin.com/in/huntervornheder)
- **GitHub:** [github.com/Hvornheder](https://github.com/Hvornheder)
