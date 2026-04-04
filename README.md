# Project 1: Enterprise Network Segmentation
> **Production-grade segmented network** | CCNA · Security+ · ISC2 CC · ITIL 4

A fully segmented enterprise network built with Cisco IOS devices and a pfSense firewall. Implements VLANs, 802.1Q trunking, OSPF dynamic routing, ACLs, and SSH hardening — mirroring real SMB and enterprise production environments.

---

## Certification Mapping

| Domain | CCNA | Security+ | ISC2 CC | ITIL 4 |
|--------|------|-----------|---------|--------|
| VLANs and Trunking | Network Access | Network segmentation | Boundary protection | Service design |
| OSPF Routing | IP Connectivity | Defense-in-depth | Network security | Availability mgmt |
| ACLs | Security Fundamentals | Access control | Access control | Change enablement |
| Port Security / STP | Network Access | Hardening | Physical security | Risk management |
| SSH / Device Hardening | IP Services | Identity and access | Access control | Config management |
| pfSense Firewall | Security Fundamentals | Firewall / zone-based | Network controls | Service design |

---

## Network Topology

```
+------------------------------+
|       INTERNET / ISP         |
+______________+_______________+
               | WAN (DHCP)
+--------------v---------------+
|   pfSense Firewall (FW-01)   |
|   LAN: 10.0.0.1/24           |
|   NAT + Zone-based rules     |
+--------------+---------------+
               | 10.0.0.0/24
+--------------v---------------+
|    Core Router (R1-CORE)     |
|    Gi0/0: 10.0.0.2/24        |
|    Gi0/1: 10.0.1.1/30        |
|    OSPF Router-ID: 1.1.1.1   |
+--------------+---------------+
               | 10.0.1.0/30
+--------------v---------------+
|  Distribution Switch SW-DIST |
|  Layer 3 - ip routing on     |
|  OSPF Router-ID: 2.2.2.2     |
|  VLAN 10 SVI: 10.10.10.1/24  |
|  VLAN 20 SVI: 10.10.20.1/24  |
|  VLAN 30 SVI: 10.10.30.1/24  |
|  VLAN 99 SVI: 10.10.99.1/24  |
+-------+----------------+-----+
        |                |
+-------v------+  +------v---------+
|  SW-ACC-1    |  |   SW-ACC-2     |
|  10.10.99.11 |  |   10.10.99.12  |
+-+----+----+--+  +-+----+----+----+
  |    |    |       |    |    |
VLAN VLAN VLAN    VLAN VLAN VLAN
 10   20   99      20   30   99
STAF SERV MGMT   SERV GUES MGMT
```

### VLAN Design Table

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| 10 | STAFF | 10.10.10.0/24 | 10.10.10.1 | Employee workstations |
| 20 | SERVERS | 10.10.20.0/24 | 10.10.20.1 | Internal servers (Wazuh, Prometheus) |
| 30 | GUEST | 10.10.30.0/24 | 10.10.30.1 | Guest WiFi — internet-only access |
| 99 | MGMT | 10.10.99.0/24 | 10.10.99.1 | Out-of-band device management |

---

## Security Design Decisions

**Why VLAN 30 (Guest) is isolated:** ACLs on the distribution switch deny Guest-to-Servers at Layer 3. Guest devices get internet access via NAT through pfSense but cannot reach any internal VLAN. This maps directly to Security+ domain: Network Architecture — Segmentation and Defense-in-Depth.

**Why SNMPv3 instead of SNMPv2c:** SNMPv2c community strings are sent in cleartext and are trivially sniffable. SNMPv3 with AuthSHA and PrivAES128 encrypts all management traffic. Required for CIS Benchmark compliance.

**Why OSPF passive-interface on access VLANs:** Prevents a rogue device on an access port from injecting false OSPF routes into the routing table. Only the distribution-to-core link exchanges OSPF hellos.

**Why Rapid PVST+ with BPDU Guard:** Standard 802.1D STP has 30–50 second convergence. Rapid PVST+ converges in under 2 seconds. BPDU Guard shuts down any access port that receives a BPDU, blocking switch spoofing attacks.

**Why SSH v2 with 2048-bit RSA:** SSHv1 has documented MITM vulnerabilities. 2048-bit RSA meets NIST SP 800-57 minimums. Telnet is explicitly blocked on all VTY lines.

---

## Build Guide

### Prerequisites

- GNS3 or EVE-NG with Cisco IOS images (CSR1000v or IOSv)
- pfSense CE 2.7.x ISO or OVA
- Linux VM for management workstation
- 8 GB RAM minimum for full topology

### Phase 1 — SW-ACC-1 (Access Layer, VLAN 10 Staff)

```
enable
configure terminal
hostname SW-ACC-1
!
vlan 10
 name STAFF
vlan 20
 name SERVERS
vlan 30
 name GUEST
vlan 99
 name MGMT
!
interface range GigabitEthernet0/1 - 8
 description STAFF-WORKSTATIONS
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
interface range GigabitEthernet0/9 - 23
 description UNUSED
 switchport mode access
 switchport access vlan 99
 shutdown
!
interface GigabitEthernet0/24
 description UPLINK-TO-SW-DIST
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 spanning-tree guard root
 no shutdown
!
interface Vlan99
 description MANAGEMENT
 ip address 10.10.99.11 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.99.1
spanning-tree mode rapid-pvst
spanning-tree loopguard default
!
logging host 10.10.20.10 transport udp port 514
logging trap informational
logging source-interface Vlan99
!
end
write memory
```

### Phase 2 — SW-ACC-2 (Access Layer, VLAN 20 Servers and VLAN 30 Guest)

```
enable
configure terminal
hostname SW-ACC-2
!
vlan 10
 name STAFF
vlan 20
 name SERVERS
vlan 30
 name GUEST
vlan 99
 name MGMT
!
interface range GigabitEthernet0/1 - 8
 description SERVER-PORTS
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
interface range GigabitEthernet0/9 - 16
 description GUEST-PORTS
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
interface GigabitEthernet0/24
 description UPLINK-TO-SW-DIST
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 spanning-tree guard root
 no shutdown
!
interface Vlan99
 description MANAGEMENT
 ip address 10.10.99.12 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.99.1
spanning-tree mode rapid-pvst
logging host 10.10.20.10 transport udp port 514
!
end
write memory
```

### Phase 3 — SW-DIST (Distribution, Layer 3 Routing + ACL)

```
enable
configure terminal
hostname SW-DIST
!
ip routing
!
vlan 10
 name STAFF
vlan 20
 name SERVERS
vlan 30
 name GUEST
vlan 99
 name MGMT
!
interface Vlan10
 description STAFF-GATEWAY
 ip address 10.10.10.1 255.255.255.0
 no shutdown
!
interface Vlan20
 description SERVERS-GATEWAY
 ip address 10.10.20.1 255.255.255.0
 no shutdown
!
interface Vlan30
 description GUEST-GATEWAY
 ip address 10.10.30.1 255.255.255.0
 no shutdown
!
interface Vlan99
 description MGMT-GATEWAY
 ip address 10.10.99.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 description UPLINK-TO-R1-CORE
 no switchport
 ip address 10.0.1.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/2
 description DOWNLINK-TO-SW-ACC-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 no shutdown
!
interface GigabitEthernet0/3
 description DOWNLINK-TO-SW-ACC-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 no shutdown
!
router ospf 1
 router-id 2.2.2.2
 network 10.0.1.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.30.0 0.0.0.255 area 0
 network 10.10.99.0 0.0.0.255 area 0
 passive-interface Vlan10
 passive-interface Vlan20
 passive-interface Vlan30
 passive-interface Vlan99
!
ip access-list extended GUEST-RESTRICT
 remark SECURITY+ - Deny Guest to Server VLAN
 deny   ip 10.10.30.0 0.0.0.255 10.10.20.0 0.0.0.255
 remark Deny Guest to Staff VLAN
 deny   ip 10.10.30.0 0.0.0.255 10.10.10.0 0.0.0.255
 remark Deny Guest to Management VLAN
 deny   ip 10.10.30.0 0.0.0.255 10.10.99.0 0.0.0.255
 remark Allow Guest internet access
 permit ip 10.10.30.0 0.0.0.255 any
!
interface Vlan30
 ip access-group GUEST-RESTRICT in
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,99 priority 4096
logging host 10.10.20.10 transport udp port 514
!
end
write memory
```

### Phase 4 — R1-CORE (Core Router, OSPF + Default Route)

```
enable
configure terminal
hostname R1-CORE
!
interface GigabitEthernet0/0
 description WAN-TOWARD-PFSENSE
 ip address 10.0.0.2 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/1
 description LAN-TOWARD-SW-DIST
 ip address 10.0.1.1 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.0.0.1
!
router ospf 1
 router-id 1.1.1.1
 network 10.0.0.0 0.0.0.255 area 0
 network 10.0.1.0 0.0.0.3 area 0
 default-information originate
!
end
write memory
```

### Phase 5 — SSH Hardening (Apply to All Cisco Devices)

```
configure terminal
!
ip domain-name lab.internal
crypto key generate rsa modulus 2048
!
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
ip ssh logging events
!
username admin privilege 15 algorithm-type scrypt secret <STRONG-PASSWORD>
!
line vty 0 4
 transport input ssh
 login local
 exec-timeout 5 0
 logging synchronous
!
line console 0
 login local
 exec-timeout 10 0
!
service password-encryption
no service finger
no ip http server
no ip http secure-server
no cdp run
!
banner motd #
*************************************************************
*  AUTHORIZED ACCESS ONLY - ALL ACTIVITY IS MONITORED      *
*  Unauthorized access is prohibited and prosecutable      *
*************************************************************
#
!
snmp-server group MONITORING v3 priv
snmp-server user wazuh-monitor MONITORING v3 auth sha Auth@2026 priv aes 128 Priv@2026
snmp-server host 10.10.20.10 version 3 priv wazuh-monitor
!
ntp server 10.10.20.10
!
end
write memory
```

---

## pfSense Firewall Rules

Install pfSense CE 2.7.x. Assign WAN (DHCP from hypervisor) and LAN (10.0.0.1/24).

### LAN Firewall Rules

| Priority | Action | Source | Destination | Port | Description |
|----------|--------|--------|-------------|------|-------------|
| 1 | Pass | 10.10.10.0/24 | Any | Any | Staff — full access |
| 2 | Pass | 10.10.20.0/24 | Any | Any | Servers — full access |
| 3 | Block | 10.10.30.0/24 | 10.10.0.0/8 | Any | Guest — block all internal RFC1918 |
| 4 | Pass | 10.10.30.0/24 | Any | 80,443 | Guest — HTTP/HTTPS only |
| 5 | Pass | 10.10.99.0/24 | Any | 22 | Management — SSH only |
| 6 | Block | Any | Any | Any | Implicit deny (log all) |

### Outbound NAT (Hybrid Mode)

| Source Network | Interface | Action | Note |
|---------------|-----------|--------|------|
| 10.10.10.0/24 | WAN | NAT | Staff internet access |
| 10.10.20.0/24 | WAN | NAT | Server internet access |
| 10.10.30.0/24 | WAN | NAT | Guest internet only |
| 10.10.99.0/24 | — | No NAT | Management stays internal |

---

## Verification Commands

```
! Verify VLAN database
SW-ACC-1# show vlan brief

! Verify trunk links
SW-DIST# show interfaces trunk

! Verify OSPF neighbors are UP/FULL
R1-CORE# show ip ospf neighbor

! Verify routing table has all VLANs via OSPF
R1-CORE# show ip route ospf

! Verify ACL hit counters (confirm Guest traffic is being blocked)
SW-DIST# show ip access-lists GUEST-RESTRICT

! Verify STP root and port roles
SW-DIST# show spanning-tree vlan 10

! Verify port security sticky MACs learned
SW-ACC-1# show port-security interface gi0/1

! Verify SSH is the only remote access method
R1-CORE# show ip ssh
R1-CORE# show ssh

! Test inter-VLAN routing (Staff to Server — should SUCCEED)
STAFF-PC> ping 10.10.20.10

! Test ACL enforcement (Guest to Server — should FAIL)
GUEST-PC> ping 10.10.20.10

! Verify OSPF is redistributing default route
R1-CORE# show ip route 0.0.0.0
SW-DIST# show ip route 0.0.0.0
```

---

## ITIL 4 Change Log

| Change ID | Date | Description | Type | Impact | Status | Risk |
|-----------|------|-------------|------|--------|--------|------|
| CHG-001 | 2026-04-04 | Deploy VLAN database on SW-ACC-1 and SW-ACC-2 | Standard | Low | Completed | Low |
| CHG-002 | 2026-04-04 | Configure 802.1Q trunks on all uplinks | Standard | Low | Completed | Low |
| CHG-003 | 2026-04-04 | Enable IP routing and SVIs on SW-DIST | Normal | Medium | Completed | Medium |
| CHG-004 | 2026-04-04 | Deploy OSPF single-area (R1-CORE + SW-DIST) | Normal | Medium | Completed | Medium |
| CHG-005 | 2026-04-04 | Apply GUEST-RESTRICT ACL on Vlan30 inbound | Normal | High | Completed | Low |
| CHG-006 | 2026-04-04 | Harden all devices: SSH only, disable Telnet | Normal | Medium | Completed | Low |
| CHG-007 | 2026-04-04 | Deploy pfSense with zone-based firewall rules | Normal | High | Completed | Medium |
| CHG-008 | 2026-04-04 | Configure SNMPv3 (replace SNMPv2c) | Standard | Low | Completed | Low |

### ITIL 4 Practices Demonstrated

**Change Enablement** — Every change is categorized (Standard for low-risk routine changes, Normal for changes requiring assessment), documented with impact and risk ratings, and recorded before implementation. This mirrors how real-world CAB (Change Advisory Board) processes work.

**Service Configuration Management** — All device configurations are version-controlled in this Git repository. Each commit represents a verified, tested configuration state — the CMDB equivalent for this lab environment.

**Service Design** — Network segmentation was designed against defined service requirements: Staff need full internal access, Servers need controlled access, Guest needs internet-only, Management plane needs isolation from data plane.

**Risk Management** — Each change has a risk rating. High-impact changes (ACL deployment, firewall rules) were tested in isolation before applying to the production VLAN. Rollback procedures were documented in the change ticket.

---

## Skills Demonstrated for Resume

| Skill | Tool or Protocol | Primary Cert | Secondary Cert |
|-------|-----------------|--------------|----------------|
| Layer 2 segmentation | VLANs, 802.1Q trunking | CCNA — Network Access | Security+ — Segmentation |
| Loop prevention | Rapid PVST+, BPDU Guard, Root Guard, Loop Guard | CCNA — Network Access | Security+ — Hardening |
| Dynamic routing | OSPFv2 single-area, passive-interface | CCNA — IP Connectivity | ISC2 CC — Availability |
| Traffic filtering | Extended named ACLs, inbound on SVI | CCNA — Security | Security+ — Access Control |
| Device hardening | SSH v2, RSA 2048, exec-timeout, banner | Security+ — Hardening | ISC2 CC — Access Control |
| Perimeter security | pfSense, zone-based rules, NAT, implicit deny | Security+ — Network Arch | ISC2 CC — Network Security |
| Secure management | SNMPv3 with authSHA and privAES128 | CCNA — IP Services | Security+ — Protocols |
| Change documentation | ITIL change records, CAB approval workflow | ITIL 4 — Change Enablement | All certs |
| Config versioning | Git-based configuration management | ITIL 4 — Service Config Mgmt | DevOps practices |

---

## How to Talk About This in an Interview

**The one-liner:**
> "I built a four-VLAN enterprise network with OSPF routing, zone-based firewall segmentation, and SNMPv3 monitoring — version-controlled with ITIL change records for every configuration change."

**For a CCNA interview question about VLANs:**
> "I deployed VLANs across two access switches with 802.1Q trunks to a Layer 3 distribution switch. The distribution switch handles inter-VLAN routing via SVIs, with OSPF redistributing all connected subnets up to the core router and out through pfSense."

**For a Security+ question about defense-in-depth:**
> "I implemented three layers: VLANs segment at Layer 2, named ACLs enforce policy at Layer 3 on the SVI, and pfSense provides stateful inspection at the perimeter. The Guest VLAN is denied access to the Server VLAN at all three layers."

**For an ITIL interview question about change management:**
> "Every configuration change has a corresponding change record categorized by type and impact. Standard changes — routine, pre-approved, low risk — went directly to implementation. Normal changes required impact assessment, risk rating, and documented rollback procedure before being applied."

---

## Repository Structure

```
enterprise-network-segmentation/
├── README.md                    <- Full documentation (this file)
├── configs/
│   ├── R1-CORE.txt              <- Core router running configuration
│   ├── SW-DIST.txt              <- Distribution switch running configuration
│   ├── SW-ACC-1.txt             <- Access switch 1 running configuration
│   └── SW-ACC-2.txt             <- Access switch 2 running configuration
├── pfsense/
│   └── firewall-rules.md        <- pfSense rule documentation and rationale
├── docs/
│   ├── itil-change-log.md       <- Full ITIL 4 change management records
│   └── troubleshooting-guide.md <- Common issues and resolution procedures
└── verification/
    └── test-results.md          <- Verification command outputs and screenshots
```

---

*Project 1 of 5 — Production Infrastructure Portfolio | Calvin Anglo*
*Related: [Project 2 — Wazuh SIEM](../enterprise-siem-wazuh) | [Project 3 — Ansible Compliance](../ansible-compliance-automation) | [Project 4 — Prometheus Monitoring](../prometheus-grafana-monitoring)*
