# Enterprise-Layer-2-Resiliency-Hardening-Cisco-IOS-

[![Cisco IOS](https://img.shields.io/badge/Cisco-IOS%2015.x-blue?logo=cisco)](https://www.cisco.com)
[![Protocol](https://img.shields.io/badge/Protocols-LACP%20%7C%20Rapid--PVST%2B-green)](#overview)
[![Security](https://img.shields.io/badge/Security-BPDU%20Guard%20%7C%20PortFast-red)](#hardening--security)

A production-grade Cisco Layer 2 switching infrastructure designed for high availability, deterministic traffic engineering, and loop prevention with sub-second failover.

---

## 🏗️ Architecture Overview

In a typical multi-switch broadcast domain, default configurations cause link idling and non-deterministic Root Bridge elections based purely on MAC addresses. This project addresses those pitfalls by implementing:

* **Link Aggregation (LACP):** Bundling physical interfaces to multiply bandwidth and bypass STP port-blocking on active bundles.
* **Rapid-PVST+ (802.1w):** Deterministic per-VLAN spanning-tree convergence under 2 seconds.
* **Hierarchical STP Design:** Enforced Primary & Secondary Root Bridges via deterministic bridge priority tuning.
* **Access Layer Hardening:** Instant edge link transitions with PortFast and protection against unauthorized switch/rogue bridge injection via BPDU Guard.
```text
+-------------------------+
|     SW1 (Core / Dist)   |
|   Root Primary (4096)   |
+-------------------------+
/ /           \ \
LACP (Po1)  / /             \ \  LACP (Po2)
/ /               \ \
v v                 v v
+-------------------------+     +-------------------------+
|   SW2 (Distribution)    |<===>|       SW3 (Access)      |
|  Root Secondary (8192)  | Po3 |    Access Edge Ports    |
+-------------------------+     +-------------------------+
|                                |
   [Access Edge]                    [Access Edge]
   PortFast + BPDU Guard            PortFast + BPDU Guard

⚙️ Configuration Blueprint
1. EtherChannel Aggregation (LACP 802.3ad)

Ensures full line-rate utilization and seamless trunk negotiation across physical uplinks.

        

cisco
SW1(config)# interface range FastEthernet 0/3 - 4
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# exit

SW1(config)# interface port-channel 1
SW1(config-if)# switchport mode trunk
SW1(config-if)# no shutdown

2. Spanning-Tree Optimization (Rapid-PVST+)

Migrating from legacy 802.1D to Rapid-PVST+ for instant synchronization:

        

cisco
! Enable Rapid-PVST+ globally
SW(config)# spanning-tree mode rapid-pvst

3. Deterministic Hierarchy (Root Election)

Preventing non-deterministic STP re-elections caused by aging hardware:

        

cisco
! Primary Root (SW1)
SW1(config)# spanning-tree vlan 1 priority 4096

! Secondary / Backup Root (SW2)
SW2(config)# spanning-tree vlan 1 priority 8192

    Note: Priorities are explicitly configured in increments of 4096 to align with the 4-bit priority field + 12-bit Extended System ID.

4. Edge Hardening (PortFast & BPDU Guard)

Applied exclusively to access/host ports to speed up DHCP acquisition and shut down rogue BPDU generators:

        

cisco
SW(config)# interface range FastEthernet 0/1 - 2, FastEthernet 0/5 - 10
SW(config-if-range)# switchport mode access
SW(config-if-range)# spanning-tree portfast
SW(config-if-range)# spanning-tree bpduguard enable
SW(config-if-range)# exit

🔍 Verification & Diagnostics
Validate LACP Bundle Integrity

        

cisco
SW1# show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)           LACP   Fa0/3(P) Fa0/4(P)

    (SU): Layer 2, In-Use.
    (P): Port bundled in Port-channel.

Validate STP Topology & Edge Status

        

cisco
SW1# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: VLAN0001
PortFast BPDU Guard is enabled by default
...

        

cisco
SW1# show spanning-tree vlan 1
VLAN0001
  Spanning tree enabled protocol rstp
  Root ID    Priority    4097
Address     5000.0001.0000
This bridge is the root
...

🛡️ Security / Threat Mitigation
Risk 	Mitigation 	Outcome
STP Spoofing / Hijacking 	spanning-tree bpduguard enable 	Any rogue bridge/tool (e.g. Yersinia) attempting to inject BPDUs triggers immediate err-disable state on the edge port.
DHCP / PXE Timeout 	spanning-tree portfast 	Bypasses traditional 30-50s Listening/Learning delay, switching state directly to Forwarding.
Spanning-Tree Loops 	LACP Link Bundling + RSTP 	Resolves topology loops within <2 seconds, maintaining uninterrupted traffic flows.
👨‍💻 Author
EHSAN SALEHI
Configured and maintained by Prince as part of advanced Cisco routing and switching security lab
