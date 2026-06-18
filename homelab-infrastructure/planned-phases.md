# Planned Phases

Remaining implementation roadmap for the home lab. 

---

## Phase 2 — TrueNAS SCALE + RAID-1 Storage

**Goal:** Replace paid cloud photo storage with a self-hosted NAS running ZFS RAID-1. Two Seagate IronWolf 4TB drives are physically installed and visible to Proxmox, this phase configures them.

**Steps:**
1. Download TrueNAS SCALE ISO into Proxmox local storage via URL (no intermediate download required)
2. Create TrueNAS VM: allocate 8GB RAM minimum for ZFS ARC cache
3. Configure disk passthrough using stable `/dev/disk/by-id/` identifiers rather than `/dev/sda` naming (device names can change across reboots; disk IDs are permanent)
4. Install TrueNAS SCALE from ISO inside the VM
5. Create ZFS mirror pool: both IronWolf drives as a RAID-1 mirror vdev
6. Create dataset for photo library
7. Enable SMB sharing for Windows client access
8. Configure UPS USB passthrough to TrueNAS for power event detection and graceful shutdown
9. Schedule automated scrubs (monthly) and ZFS snapshots (daily / weekly)
10. Configure backup job to external USB drive: completes on-site layer of 3-2-1 strategy

**Key concept — RAID is not a backup:**
The RAID-1 mirror protects against a single drive failure while keeping the system running. It does not protect against accidental deletion, ransomware, or physical disaster at the same location. The external USB drive and planned off-site copy address those scenarios independently.

---

## Phase 3 — pfSense Firewall and Router

**Goal:** Replace the ISP gateway's routing function with a dedicated software firewall VM, giving full control over traffic rules, DNS, DHCP, and network segmentation.

**Steps:**
1. Enable Intel VT-d and IOMMU in Proxmox: required for PCIe NIC passthrough to a VM
2. Pass the 82576 dual-port NIC through to the pfSense VM so pfSense owns the hardware directly
3. Download pfSense ISO into Proxmox, create VM
4. Install pfSense, assign WAN and LAN interfaces (nic1 and nic2 on the 82576 card)
5. Physically move nic0 (onboard) cable from switch to gateway, this becomes pfSense's WAN connection
6. Connect nic1 (82576 port 1) to the switch, this becomes pfSense's LAN
7. Enable bridge mode on the Sercomm gateway, it becomes a dumb modem passing the public IP to pfSense
8. Configure firewall rules, default deny inbound, allow established outbound
9. Configure Unbound DNS resolver inside pfSense
10. Verify all LAN devices can reach the internet through pfSense before proceeding

---

## Phase 4 — WireGuard VPN

**Goal:** Secure remote access to the home lab from anywhere; phone, laptop, any external network without exposing internal services directly to the internet.

**Steps:**
1. Install or enable WireGuard package inside pfSense
2. Create tunnel interface, generate server keypair
3. Configure tunnel subnet (e.g. 10.0.0.0/24) separate from LAN subnet
4. Add firewall rule allowing inbound UDP on port 51820 at the WAN interface
5. Install DuckDNS dynamic DNS client in pfSense, resolves home public IP to a stable hostname even when ISP rotates the address
6. Generate individual keypairs for each client device (phone, laptop, additional machines)
7. Configure WireGuard app on each client with server public key and endpoint hostname
8. Configure split tunneling via `AllowedIPs` only LAN-destined traffic routes through the VPN; general internet traffic exits locally
9. Test from outside the network (mobile data, not home WiFi) to verify connectivity

**Security note:**
WireGuard uses ChaCha20 symmetric encryption and Curve25519 key exchange. Authentication is entirely via cryptographic keypairs; no passwords, no certificates to manage. Each client device has its own unique keypair, so individual devices can be revoked without affecting others.

---

## Phase 5 — VLAN Segmentation and DMZ

**Goal:** Isolate public-facing web services from the personal LAN so a compromised web server cannot reach the NAS or workstation.

**Steps:**
1. Configure 802.1Q VLANs on the TP-Link TL-SG108E
2. Create VLAN interfaces in pfSense, VLAN 10 (LAN, 192.168.10.0/24) and VLAN 20 (DMZ, 192.168.20.0/24)
3. Configure inter-VLAN firewall rules, DMZ devices cannot initiate connections to LAN; LAN devices can reach DMZ
4. Migrate personal devices and NAS to VLAN 10
5. Place web-facing VMs and reverse proxy in VLAN 20

---

## Phase 6 — Web Hosting and Reverse Proxy

**Goal:** Self-host websites and web applications with HTTPS, accessible from the public internet, isolated in the DMZ.

**Steps:**
1. Create reverse proxy VM: Nginx or Traefik
2. Register domain name via Cloudflare Registrar (at-cost pricing, no markup)
3. Configure Cloudflare DNS with proxied records, hides home public IP behind Cloudflare's network, provides DDoS protection
4. Automate Let's Encrypt SSL certificate issuance and renewal (90-day certs, auto-renew)
5. Configure pfSense port forwarding, ports 80 and 443 inbound to reverse proxy VM only
6. Create web server VMs or Docker containers per project
7. Configure reverse proxy routing rules, domain/subdomain maps to the correct backend service
8. Test HTTPS access from outside the network

---

## Phase 7 — Traffic Monitoring and Observability

**Goal:** Visibility into what's happening on the network; bandwidth usage, device activity, anomaly detection, and system health metrics.

**Steps:**
1. Create a monitoring VM or LXC container in Proxmox
2. Install ntopng (community edition) for network flow analysis
3. Configure pfSense to export NetFlow data to ntopng
4. Install Grafana + InfluxDB or Prometheus for system-level metrics (CPU, RAM, disk, network per VM)
5. Configure alerting for anomalous conditions, unexpected bandwidth spikes, new unknown devices, high resource utilization
6. Build dashboards for at-a-glance network health

---

## 3-2-1 Backup Strategy — Full Implementation

The 3-2-1 rule requires three copies of data, on two different media types, with one copy off-site.

| Copy | Media | Location | Status |
|------|-------|----------|--------|
| 1 — Live data | RAID-1 ZFS mirror (2 × IronWolf) | Server, home | 📋 Phase 2 |
| 2 — On-site backup | External USB drive (5TB) | Home, separate from server | 🔄 Drive owned, sync pending |
| 3 — Off-site backup | Cloud or physically remote storage | Off-site | 📋 Planned |

ZFS snapshots provide an additional recovery layer on top of the mirror; point-in-time recovery from accidental deletion without requiring a full restore from backup.
