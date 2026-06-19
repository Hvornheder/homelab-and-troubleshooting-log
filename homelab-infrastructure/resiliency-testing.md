# Resiliency Testing

Documented stress tests performed on the home lab infrastructure prior to deploying any production VMs or containers. Testing was conducted at this stage specifically because no live workloads existed yet; therefore, any failure during testing would carry zero data risk, making this the ideal window to validate the infrastructure's failure tolerance before anything of value depends on it.

---

## Why Test Resiliency Before Adding Services

Resiliency (the ability of a system to maintain availability and recover cleanly from disruption) is one of the three pillars of the CIA Triad. It's easy to assume a UPS or a network connection works correctly simply because it's plugged in. Verifying that assumption under controlled, low-risk conditions is standard practice before trusting infrastructure with real workloads. This round of testing validates three distinct failure domains: power loss, network interruption, and unexpected system shutdown.

---

## Test 1 — UPS Power Failover

**Objective:** Confirm the UPS sustains the full server + network stack during a simulated power outage with zero service interruption, and establish a real-world runtime estimate.

**Pre-test verification:**
- Confirmed server and switch were connected to the **Battery and Surge Protected** outlets on the UPS, not the Full-Time Surge Protection–only outlets (verified against the CyberPower CP1500PFCLCD manual, these are two separate outlet banks with different failover behavior)
- Enabled the UPS LCD display to monitor real-time battery percentage and load during the test
- Started a continuous ping to `8.8.8.8` (Google DNS) which was chosen deliberately over a local target to validate the *entire* chain simultaneously: server, switch, router, and the WAN/internet connection through the coax modem, all riding the UPS at once

**Procedure:**
1. Pulled the UPS power cord from the wall outlet to simulate a complete utility power loss
2. Monitored continuous ping output, UPS LCD status, and Proxmox web UI accessibility throughout
3. Allowed the system to run on battery for approximately 5 minutes
4. Restored utility power by reconnecting the UPS to the wall outlet

**Results:**

| Metric | Observation |
|--------|-------------|
| Ping continuity | Zero dropped packets: no disruption throughout the outage |
| Proxmox web UI | Remained accessible for the full duration, confirming server stayed powered |
| UPS output voltage | Stable 120V to connected equipment (AVR regulation active) |
| Estimated runtime at time of pull | ~73 minutes remaining at start of test |
| Battery capacity after ~5 min runtime | 79% |
| Power restoration | UPS returned to utility power (bypass) mode automatically, no manual intervention required |

**Conclusion:** The UPS fully sustained the server, switch, and router with zero observable service interruption. Based on the runtime estimate at full load, the infrastructure has approximately **70+ minutes of battery runtime** during a utility outage which is sufficient time for either power to be restored or for a manual/automated graceful shutdown to be triggered before battery depletion.

**Note on output voltage vs. battery charge:** The 120V reading reflects the UPS's *regulated output* to connected equipment via Automatic Voltage Regulation (AVR) — it is constant regardless of battery state. Battery charge is tracked separately via the capacity percentage (79% in this test), not the voltage reading. These are two distinct metrics and shouldn't be conflated.

---

## Test 2 — Network Interruption and Recovery

**Objective:** Confirm the server's network connection recovers automatically after a physical disconnect, with no manual intervention required.

**Procedure:**
1. Identified the server's static IP, which was confirmed as `192.168.0.9/24` both via the Sercomm gateway's connected devices list and directly within the Proxmox network configuration
2. Started a continuous ping to `192.168.0.9` from a separate machine on the network
3. Physically disconnected the ethernet cable from the switch port serving the server
4. Waited approximately 15 seconds
5. Reconnected the ethernet cable

**Results:**

| Metric | Observation |
|--------|-------------|
| Disconnect behavior | Ping immediately returned "Request timed out" |
| Reconnect behavior | Ping resumed almost immediately upon reconnection with no manual restart required |

**Conclusion:** The network interface recovers automatically from a physical link interruption with no need to restart networking services or the host. This confirms basic link-layer resiliency at the server's network interface.

---

## Test 3 — Shutdown and Recovery Time

**Objective:** Measure recovery time from a system shutdown and confirm Proxmox returns to a fully accessible state without manual intervention.

**Procedure:**
1. Pressed the physical power button on the Dell T3650 workstation
2. Workstation powered down completely
3. Waited and monitored for Proxmox web UI accessibility
4. Recorded time to successful reconnection

**Results:**

| Metric | Observation |
|--------|-------------|
| Shutdown method | Power button press |
| Time powered off | ~10 seconds before next step |
| Connection state during shutdown | Proxmox web UI unreachable, as expected |
| Recovery time | ~45 seconds from power-on to full Proxmox web UI accessibility |

**Conclusion:** Server returns to a fully operational, network-accessible state in under a minute from a cold start with no manual reconfiguration needed.
