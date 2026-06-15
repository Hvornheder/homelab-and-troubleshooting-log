# Home Lab Infrastructure Build

A self-hosted, enterprise-grade home lab built on a Dell Precision workstation running Proxmox VE. The design was planned for scalability, redundancy, and hands-on development of real-world infrastructure skills.

This homelab project was inspired by my hands-on experience gained during a shadowing at MetroTech, where I observed a lead infrastructure technician. After my shadowing experience, it was made clear to me that no amount of virtual lab practice fully replicates the decision-making required when working with physical hardware in an open environment. This project was built to close that gap, and enhance my capabilities.

---

## Project Status

| Phase | Description | Status |
|-------|-------------|--------|
| Phase 1 | Hardware procurement, physical setup, Proxmox installation | ✅ Complete |
| Phase 2 | TrueNAS SCALE VM + RAID-1 ZFS photo storage | 🔄 In progress |
| Phase 3 | pfSense VM — software firewall and router | 📋 Planned |
| Phase 4 | WireGuard VPN — secure remote access | 📋 Planned |
| Phase 5 | VLAN segmentation and DMZ architecture | 📋 Planned |
| Phase 6 | Web hosting — reverse proxy, SSL, Docker | 📋 Planned |
| Phase 7 | Traffic monitoring and observability | 📋 Planned |

---

## Documentation

| File | Contents |
|------|----------|
| [hardware.md](hardware.md) | Full hardware stack, selection rationale, specs |
| [proxmox-setup.md](proxmox-setup.md) | Installation steps, configuration, commands run |
| [network-topology.md](network-topology.md) | Network diagrams, switch port assignments, IP scheme |
| [planned-phases.md](planned-phases.md) | Roadmap for remaining phases with step-by-step breakdown |
| [troubleshooting.md](troubleshooting.md) | BIOS recovery incident and other issues encountered |

---

## Skills at a Glance

`Proxmox VE` `Type-1 hypervisor` `ZFS / RAID-1` `TrueNAS SCALE` `pfSense` `WireGuard VPN` `VLAN segmentation` `Linux CLI` `Dell firmware management` `UPS / power engineering` `managed switch` `network topology design` `3-2-1 backup strategy` `defense in depth` `CIA triad`

---

## Connection to Security Learning Log

This project applies CIA Triad principles and defense-in-depth concepts in a physical environment — see [security-learning-log](https://github.com/Hvornheder/security-learning-log) for the theoretical foundation behind the security decisions made here.
