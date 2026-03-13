# 🏨 Hotel Network Security Lab

## Objective

This project aimed to build a realistic hotel network environment in Cisco Packet Tracer to simulate, generate, and analyze real security logs. The primary focus was configuring security protocols across a multi-floor, multi-VLAN network, forwarding all logs to a centralized Syslog server, and analyzing captured events to understand how unauthorized access and network violations appear in real log data.

### Skills Learned

- Practical understanding of VLAN segmentation and inter-VLAN routing across multiple network floors
- Configuration and hardening of SSH v2 for secure remote device management
- Deployment and operation of a centralized Syslog server for network-wide log collection
- Configuration of port security to detect and log unauthorized device connections by MAC address
- Troubleshooting switch management IPs (SVI) and network connectivity for log forwarding

### Tools Used

- **Cisco Packet Tracer 8.x** — Network simulation and device configuration environment

## Network Architecture

### Topology Overview

| Floor | Router | Switch | VLANs | Departments | Subnets |
|-------|--------|--------|-------|-------------|---------|
| 1st Floor | Cisco 2901 (F1-Router) | Cisco 2960 (F1-Switch) | 60, 70, 80 | Logistics · Store · Reception | 192.168.6-8.0/24 |
| 2nd Floor | Cisco 2911 (F2-Router) | Cisco 2960 (F2-Switch) | 30, 40, 50 | Sales · HR · Finance | 192.168.3-5.0/24 |
| 3rd Floor | Cisco 2911 (F3-Router) | Cisco 2960 (F3-Switch) | 10, 20 | IT · Admin | 192.168.1-2.0/24 |

**WAN point-to-point links between routers:**

| Link | Subnet |
|------|--------|
| F1-Router ↔ F2-Router | 10.10.10.0/30 |
| F1-Router ↔ F3-Router | 10.10.10.4/30 |
| F2-Router ↔ F3-Router | 10.10.10.8/30 |

**Syslog Server:** `192.168.1.100` — VLAN 10 (IT Department, 3rd Floor)

---

## Steps

### Step 1 — Build the Network Topology

Designed and built the full hotel network in Cisco Packet Tracer: 3 floors, 3 routers, 3 switches, 8 VLANs, and all end devices (PCs, printers, laptops, tablets, access points). Connected routers with point-to-point WAN links and configured inter-VLAN routing on each floor.

*Ref 1: Full network topology showing all 3 floors, routers, switches, VLANs, and the Syslog server connected to F3-Switch in VLAN 10*

---

### Step 2 — Configure Centralized Syslog Server

Added a Server-PT device to VLAN 10 (IT — 192.168.1.100). Enabled the Syslog service on the server. Configured all routers and switches to forward logs to the server with timestamps enabled.

**Commands used on routers:**
```bash
logging on
logging 192.168.1.100
logging trap debugging
service timestamps log datetime msec
```

**Commands used on switches:**
```bash
logging on
logging 192.168.1.100
logging trap
service timestamps log datetime msec
```

**Important:** Each switch required a management IP on its VLAN SVI to be able to reach the Syslog server:
```bash
interface vlan 10
  ip address 192.168.1.50 255.255.255.0
  no shutdown
exit
ip default-gateway 192.168.1.1
```

*Ref 2: Syslog server interface showing the service enabled and logs being received from network devices*

---

### Step 3 — Harden SSH v2 on All Routers

Configured SSH v2 on all three routers to replace Telnet. Generated RSA 2048-bit keys, set authentication limits, and enabled login attempt logging. All VTY lines were restricted to SSH only.

```bash
ip domain-name hotellab.local
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh authentication-retries 3
ip ssh time-out 60
username admin privilege 15 secret StrongPass123!
login on-success log
login on-failure log every 1
line vty 0 4
  transport input ssh
  login local
  exec-timeout 5 0
exit
```

*Ref 3: SSH configuration on F1-Router showing SSHv2 enabled, RSA keys generated, and VTY lines restricted to SSH only*

---

### Step 4 — Configure Port Security on Switches

Applied port security to all access ports on the switches. Used sticky MAC learning so each port automatically learns and locks to its authorized device. Set violation mode to `restrict` so unauthorized connections are dropped and logged without shutting down the port — this generates maximum log data for analysis.

```bash
interface FastEthernet0/1
  switchport mode access
  switchport port-security
  switchport port-security maximum 1
  switchport port-security mac-address sticky
  switchport port-security violation restrict
exit
```

*Ref 4: Port security configuration on F3-Switch showing sticky MAC addresses learned on active ports*

---

### Step 5 — Simulate a Security Incident

To generate real security logs, an unauthorized device was connected to a port that already had an authorized device's MAC address learned. This triggered port security violation alerts that were forwarded to the Syslog server.

**What happened — timeline reconstruction from logs:**

| Timestamp | Event | Analysis |
|-----------|-------|---------|
| 15:37:41 | `%LINK-3-UPDOWN` → **down** | Authorized device was unplugged |
| 15:37:44 | `%LINK-3-UPDOWN` → **up** | Unknown device was plugged in |
| 15:38:04 | `%PORT_SECURITY-2-PSECURE_VIOLATION` | Switch rejected the unknown MAC |
| 15:38:14 → 15:39:04 | Violation repeated ×6 | Device remained connected, blocked every 10 seconds |

*Ref 5: Switch CLI showing the port security violation log messages appearing in real time*

---

### Step 6 — Analyze Logs on Syslog Server

Opened the Syslog server interface and reviewed all collected log messages. Identified and correlated events by timestamp to reconstruct the full incident timeline. The logs confirmed that the unauthorized device (MAC `0003.E479.BACB`) was detected, blocked, and logged successfully.

**Raw logs captured:**

```
*Mar 02, 15:37:41.3737: %LINK-3-UPDOWN: Interface FastEthernet0/4, changed state to down
*Mar 02, 15:37:41.3737: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/4, changed state to down
*Mar 02, 15:37:44.3737: %LINK-3-UPDOWN: Interface FastEthernet0/4, changed state to up
*Mar 02, 15:37:44.3737: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/4, changed state to up
*Mar 02, 15:38:04.3838: %PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred,
  caused by MAC address 0003.E479.BACB on port FastEthernet0/4.
*Mar 02, 15:38:14.3838: %PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred,
  caused by MAC address 0003.E479.BACB on port FastEthernet0/4.
*Mar 02, 15:38:24.3838: %PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred,
  caused by MAC address 0003.E479.BACB on port FastEthernet0/4.
```

**Key observations:**
- The 3-second gap between port going down (15:37:41) and coming back up (15:37:44) indicates a physical device swap
- Severity level **2 (Critical)** on the port security messages confirms this is a high-priority alert
- The violation repeating every 10 seconds confirms the device remained connected — the port was not shut down (restrict mode)
- In a real SOC environment, this sequence would trigger an immediate investigation ticket

*Ref 6: Syslog server displaying collected logs including the port security violation messages with timestamps*

---

## Summary

| Security Control | Configured | Logs Generated | Outcome |
|-----------------|-----------|----------------|---------|
| SSH v2 Hardening | ✅ All 3 routers | Login success / failure events | Remote access secured |
| Syslog Server | ✅ 192.168.1.100 | All device events centralized | Full network visibility |
| Port Security | ✅ All switches | `%PORT_SECURITY-2-PSECURE_VIOLATION` | Unauthorized device detected & blocked |
| Interface Monitoring | ✅ Automatic | `%LINK-3-UPDOWN` / `%LINEPROTO-5-UPDOWN` | Physical events logged |
