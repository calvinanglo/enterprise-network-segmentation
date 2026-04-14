# Enterprise Network Segmentation - Setup Guide

Step-by-step build guide for the full topology: two access switches, one Layer 3 distribution switch, one core router, and pfSense as the perimeter firewall. Every section ends with verification commands and a screenshot reference so you can document your build as you go.

Full device configs are in `/configs/`. This guide walks through them in build order with explanations.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Building the GNS3 Topology](#2-building-the-gns3-topology)
3. [Device Configuration](#3-device-configuration)
   - [3a. SW-ACC-1: Staff Access Switch](#3a-sw-acc-1-staff-access-switch)
   - [3b. SW-ACC-2: Server and Guest Access Switch](#3b-sw-acc-2-server-and-guest-access-switch)
   - [3c. Trunks Between Access Switches and SW-DIST](#3c-trunks-between-access-switches-and-sw-dist)
   - [3d. SW-DIST: Distribution Switch (Layer 3)](#3d-sw-dist-distribution-switch-layer-3)
   - [3e. R1-CORE: Core Router](#3e-r1-core-core-router)
   - [3f. pfSense: Perimeter Firewall](#3f-pfsense-perimeter-firewall)
   - [3g. SSH Hardening on All Devices](#3g-ssh-hardening-on-all-devices)
   - [3h. SNMPv3 and Syslog on All Devices](#3h-snmpv3-and-syslog-on-all-devices)
4. [Verification](#4-verification)
5. [Troubleshooting Common Issues](#5-troubleshooting-common-issues)

---

## 1. Prerequisites

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| GNS3 | 2.2.x or newer | Network simulation platform |
| GNS3 VM | Matching GNS3 version | Runs IOS images and pfSense |
| Cisco IOSv L2 | 15.9 | Access and distribution switches (SW-ACC-1, SW-ACC-2, SW-DIST) |
| Cisco IOSv L3 | 15.9 | Core router (R1-CORE) |
| pfSense CE ISO | 2.7.x | Perimeter firewall |
| VMware Workstation or VirtualBox | Latest | Hypervisor for GNS3 VM |

### Hardware Requirements

- **RAM:** 8 GB minimum (GNS3 VM needs 4 GB, host needs headroom)
- **CPU:** 4 cores recommended (2 minimum)
- **Disk:** 20 GB free for images and VM snapshots

### Downloading Images

**Cisco IOSv images:** You need a valid Cisco CML or VIRL license. Download `vios-adventerpriselk9-m.vmdk` (L3 router) and `vios_l2-adventerpriselk9-m.SSA.high_iron_*.qcow2` (L2 switch) from your Cisco account.

**pfSense ISO:** Download from https://www.pfsense.org/download/ - select AMD64, ISO Installer, closest mirror.

### GNS3 Installation

1. Download GNS3 from https://www.gns3.com/software/download
2. Run the installer, accept defaults
3. When prompted, install the GNS3 VM (VMware or VirtualBox)
4. Launch GNS3, go to **Edit > Preferences > GNS3 VM** and verify the VM is detected and running
5. Go to **Edit > Preferences > QEMU VMs** to import the Cisco IOSv images
6. Go to **Edit > Preferences > QEMU VMs** to import the pfSense ISO as a QEMU VM (set RAM to 1024 MB, 1 vCPU, 2 NICs)

> Screenshot: GNS3 Preferences showing the GNS3 VM connected and running
> Save as: docs/screenshots/01-gns3-vm-connected.png

> Screenshot: QEMU VM list showing all imported images (IOSv L2, IOSv L3, pfSense)
> Save as: docs/screenshots/02-qemu-images-imported.png

---

## 2. Building the GNS3 Topology

### Target Topology

```
Internet
    |
pfSense FW-01 (em0=WAN DHCP, em1=LAN 10.0.0.1/24)
    |
    | em1 <---> Gi0/0
    |
R1-CORE (Gi0/0: 10.0.0.2/24, Gi0/1: 10.0.1.1/30)
    |
    | Gi0/1 <---> Gi0/1
    |
SW-DIST (Gi0/1: 10.0.1.2/30, Gi0/2: trunk, Gi0/3: trunk)
    |               |
    | Gi0/2         | Gi0/3
    |               |
SW-ACC-1          SW-ACC-2
(Gi0/24: trunk)   (Gi0/24: trunk)
```

### Step-by-Step

1. **Create a new project:** File > New Blank Project, name it `enterprise-network-segmentation`

2. **Place devices on the canvas:**
   - Drag 1x IOSv L3 router from the device panel. Label it `R1-CORE`
   - Drag 3x IOSv L2 switches. Label them `SW-DIST`, `SW-ACC-1`, `SW-ACC-2`
   - Drag 1x pfSense QEMU VM. Label it `FW-01`
   - Drag 1x NAT cloud from End Devices (this simulates the internet uplink for pfSense WAN)

3. **Connect cables:**

   | From Device | From Interface | To Device | To Interface | Purpose |
   |-------------|---------------|-----------|-------------|---------|
   | NAT Cloud | nat0 | FW-01 | em0 | pfSense WAN (DHCP) |
   | FW-01 | em1 | R1-CORE | Gi0/0 | pfSense LAN to core router |
   | R1-CORE | Gi0/1 | SW-DIST | Gi0/1 | Core to distribution (routed /30) |
   | SW-DIST | Gi0/2 | SW-ACC-1 | Gi0/24 | Distribution to access trunk |
   | SW-DIST | Gi0/3 | SW-ACC-2 | Gi0/24 | Distribution to access trunk |

4. **Start all devices:** Right-click the canvas > Start All Nodes. Wait 2-3 minutes for all devices to boot fully.

> Screenshot: Completed GNS3 topology with all devices placed, labeled, and cabled. All link lights should be green.
> Save as: docs/screenshots/03-gns3-topology-complete.png

> Screenshot: GNS3 topology with all nodes started (green indicators on all devices)
> Save as: docs/screenshots/04-all-nodes-started.png

---

## 3. Device Configuration

**Build order matters.** Configure from the bottom up so you can verify each layer before adding the next:

1. Access switches (VLANs and ports)
2. Trunks to distribution
3. Distribution switch (SVIs, routing, OSPF, ACL)
4. Core router (interfaces, OSPF, default route)
5. pfSense (LAN/WAN, rules, NAT)
6. SSH hardening (all devices)
7. SNMPv3 and syslog (all devices)

Double-click any device in GNS3 to open its console.

---

### 3a. SW-ACC-1: Staff Access Switch

SW-ACC-1 handles VLAN 10 (Staff) workstation ports. See `/configs/SW-ACC-1.txt` for the full config.

**Set the hostname and disable unnecessary services:**

```
enable
configure terminal

hostname SW-ACC-1
no service finger
no service udp-small-servers
service password-encryption
```

**Create the VLAN database:**

```
vlan 10
 name STAFF
vlan 20
 name SERVERS
vlan 30
 name GUEST
vlan 99
 name MGMT
```

All four VLANs are created on every switch even though SW-ACC-1 only uses VLAN 10 for access ports. This is so the trunk passes all VLANs and the VLAN database is consistent across the network.

**Configure staff access ports (Gi0/1-8):**

```
interface range GigabitEthernet0/1 - 8
 description STAFF-WORKSTATION
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

- **Port security max 2:** Allows a PC plus an IP phone on the same port.
- **Violation restrict:** Logs the violation and drops the frame but does NOT shut the port down. Better for workstations where people occasionally plug in a second device.
- **Sticky MAC:** Learns the first MAC(s) and writes them to running-config. Persists across reboots if you `write memory`.
- **PortFast + BPDU Guard:** PortFast skips STP listening/learning (faster link-up for PCs). BPDU Guard shuts the port if it receives a BPDU, preventing rogue switches.

**Shut down unused ports and move them off VLAN 1:**

```
interface range GigabitEthernet0/9 - 23
 description unused
 switchport mode access
 switchport access vlan 99
 shutdown
```

Unused ports go to VLAN 99 (MGMT) and are shut down. Never leave unused ports in VLAN 1 or in an enabled state.

**Configure the trunk uplink to SW-DIST:**

```
interface GigabitEthernet0/24
 description uplink to SW-DIST Gi0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 spanning-tree guard root
 no shutdown
```

- **Root guard:** Prevents this access switch from ever becoming STP root, even if someone misconfigures priorities.
- **Native VLAN 99:** Moves untagged traffic off VLAN 1.
- **Allowed VLANs:** Only the four VLANs we use. Prunes everything else.

**Configure the management interface:**

```
interface Vlan99
 description management
 ip address 10.10.99.11 255.255.255.0
 no shutdown

ip default-gateway 10.10.99.1
```

**Set STP mode:**

```
spanning-tree mode rapid-pvst
spanning-tree loopguard default
```

**Save the config:**

```
end
write memory
```

**Verify SW-ACC-1:**

```
show vlan brief
show interfaces GigabitEthernet0/24 switchport
show port-security interface GigabitEthernet0/1
show spanning-tree vlan 10
```

> Screenshot: `show vlan brief` output on SW-ACC-1 showing VLANs 10, 20, 30, 99 with correct port assignments
> Save as: docs/screenshots/05-sw-acc-1-vlan-brief.png

> Screenshot: `show port-security interface Gi0/1` on SW-ACC-1 showing port security enabled with restrict mode
> Save as: docs/screenshots/06-sw-acc-1-port-security.png

---

### 3b. SW-ACC-2: Server and Guest Access Switch

SW-ACC-2 handles VLAN 20 (Servers) and VLAN 30 (Guest). See `/configs/SW-ACC-2.txt` for the full config.

**Set hostname and base config:**

```
enable
configure terminal

hostname SW-ACC-2
no service finger
no service udp-small-servers
service password-encryption
```

**Create the VLAN database (same on every switch):**

```
vlan 10
 name STAFF
vlan 20
 name SERVERS
vlan 30
 name GUEST
vlan 99
 name MGMT
```

**Configure server ports (Gi0/1-8):**

```
interface range GigabitEthernet0/1 - 8
 description SERVER-PORT-VLAN20
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

- **Max 1 MAC + violation shutdown:** Servers have known MACs. If a different MAC shows up, something is wrong. Shutdown mode forces manual investigation.

**Configure guest ports (Gi0/9-16):**

```
interface range GigabitEthernet0/9 - 16
 description GUEST-PORT-VLAN30
 switchport mode access
 switchport access vlan 30
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

- **No sticky MAC on guest ports:** Guests rotate too often. Sticky would fill up the MAC table with stale entries.
- **Restrict mode:** Logs the violation but keeps the port up. Guests plugging in extra devices should not require a network admin to recover the port.

**Shut down unused ports:**

```
interface range GigabitEthernet0/17 - 23
 description unused
 switchport mode access
 switchport access vlan 99
 shutdown
```

**Configure the trunk uplink to SW-DIST:**

```
interface GigabitEthernet0/24
 description uplink to SW-DIST Gi0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 spanning-tree guard root
 no shutdown
```

**Management interface:**

```
interface Vlan99
 description management
 ip address 10.10.99.12 255.255.255.0
 no shutdown

ip default-gateway 10.10.99.1
```

**STP and save:**

```
spanning-tree mode rapid-pvst
spanning-tree loopguard default

end
write memory
```

**Verify SW-ACC-2:**

```
show vlan brief
show interfaces GigabitEthernet0/24 switchport
show port-security interface GigabitEthernet0/1
show port-security interface GigabitEthernet0/9
show spanning-tree vlan 20
show spanning-tree vlan 30
```

> Screenshot: `show vlan brief` output on SW-ACC-2 showing server ports in VLAN 20 and guest ports in VLAN 30
> Save as: docs/screenshots/07-sw-acc-2-vlan-brief.png

> Screenshot: `show port-security interface Gi0/1` on SW-ACC-2 showing shutdown violation mode (server port)
> Save as: docs/screenshots/08-sw-acc-2-server-port-security.png

> Screenshot: `show port-security interface Gi0/9` on SW-ACC-2 showing restrict violation mode (guest port)
> Save as: docs/screenshots/09-sw-acc-2-guest-port-security.png

---

### 3c. Trunks Between Access Switches and SW-DIST

The trunk interfaces on the access switches are already configured above. Now configure the matching trunk ports on SW-DIST.

On **SW-DIST**, open the console:

```
enable
configure terminal

vlan 10
 name STAFF
vlan 20
 name SERVERS
vlan 30
 name GUEST
vlan 99
 name MGMT
```

You must create the VLANs on SW-DIST before the trunks will pass tagged traffic for those VLANs. This is the most common mistake in the build -- the trunk comes up but VLANs show as "allowed" without being "active" because the VLAN does not exist in the local database.

```
interface GigabitEthernet0/2
 description trunk to SW-ACC-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 no shutdown

interface GigabitEthernet0/3
 description trunk to SW-ACC-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 no shutdown

end
write memory
```

**Verify trunks:**

```
show interfaces trunk
```

Expected output:

```
Port   Mode   Encapsulation  Status    Native vlan
Gi0/2  on     802.1q         trunking  99
Gi0/3  on     802.1q         trunking  99

Gi0/2  vlans allowed and active: 10,20,30,99
Gi0/3  vlans allowed and active: 10,20,30,99
```

Both trunks should show `trunking` status with native VLAN 99 and all four VLANs in the "allowed and active" section. If a VLAN shows as "allowed" but not "active," it does not exist in the VLAN database on SW-DIST.

> Screenshot: `show interfaces trunk` on SW-DIST showing both trunks up with correct VLANs and native VLAN 99
> Save as: docs/screenshots/10-sw-dist-trunks-verified.png

Also verify from the access switch side:

```
SW-ACC-1# show interfaces trunk
SW-ACC-2# show interfaces trunk
```

> Screenshot: `show interfaces trunk` on SW-ACC-1 confirming trunk is up to SW-DIST
> Save as: docs/screenshots/11-sw-acc-1-trunk-verified.png

---

### 3d. SW-DIST: Distribution Switch (Layer 3)

SW-DIST is the Layer 3 boundary. It performs inter-VLAN routing, runs OSPF with R1-CORE, enforces the guest ACL, and is STP root for all VLANs. See `/configs/SW-DIST.txt` for the full config.

**Enable IP routing:**

```
enable
configure terminal

hostname SW-DIST
no service finger
no service udp-small-servers
service password-encryption

ip routing
ip cef
```

`ip routing` turns this L2 switch into a Layer 3 switch. Without it, SVIs cannot route traffic between VLANs.

**Configure SVIs (VLAN gateways):**

```
interface Vlan10
 description STAFF gateway
 ip address 10.10.10.1 255.255.255.0
 no shutdown

interface Vlan20
 description SERVERS gateway
 ip address 10.10.20.1 255.255.255.0
 no shutdown

interface Vlan30
 description GUEST gateway
 ip address 10.10.30.1 255.255.255.0
 no shutdown

interface Vlan99
 description MGMT gateway
 ip address 10.10.99.1 255.255.255.0
 no shutdown
```

Each SVI is the default gateway for its VLAN. Hosts in VLAN 10 use 10.10.10.1, hosts in VLAN 20 use 10.10.20.1, etc.

**Configure the routed uplink to R1-CORE:**

```
interface GigabitEthernet0/1
 description uplink to R1-CORE
 no switchport
 ip address 10.0.1.2 255.255.255.252
 no shutdown
```

`no switchport` converts this from a Layer 2 port to a routed interface. The /30 subnet (10.0.1.0/30) provides a point-to-point link between SW-DIST (.2) and R1-CORE (.1).

**Shut down unused ports:**

```
interface range GigabitEthernet0/4 - 24
 description unused
 switchport mode access
 switchport access vlan 99
 shutdown
```

**Configure OSPF:**

```
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
```

- **Router-ID 2.2.2.2:** Manually set so it does not change if a loopback is added later.
- **Network statements:** Advertise all VLAN subnets and the /30 transit link into OSPF area 0.
- **Passive interfaces on all SVIs:** OSPF hellos on access VLANs serve no purpose (no OSPF neighbors there) and slightly increase the attack surface. Passive-interface stops hellos while still advertising the network.

**Configure the guest isolation ACL:**

```
ip access-list extended GUEST-RESTRICT
 deny ip 10.10.30.0 0.0.0.255 10.10.20.0 0.0.0.255 log
 deny ip 10.10.30.0 0.0.0.255 10.10.10.0 0.0.0.255 log
 deny ip 10.10.30.0 0.0.0.255 10.10.99.0 0.0.0.255 log
 permit ip 10.10.30.0 0.0.0.255 any
```

- Denies guest (10.10.30.0/24) from reaching servers, staff, and management subnets.
- The `log` keyword on deny entries sends hits to syslog (picked up by Wazuh later).
- The final `permit` allows guest traffic to reach the internet (via R1-CORE and pfSense).

**Apply the ACL to the VLAN 30 SVI inbound:**

```
interface Vlan30
 ip access-group GUEST-RESTRICT in
```

Direction matters. `in` means traffic entering the switch from VLAN 30 hosts is filtered. This is the correct direction.

**Configure STP root priority:**

```
spanning-tree mode rapid-pvst
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 4096
spanning-tree vlan 30 priority 4096
spanning-tree vlan 99 priority 4096
spanning-tree loopguard default
```

Priority 4096 is lower than the default 32768 on the access switches, so SW-DIST always wins the root election.

**Save:**

```
end
write memory
```

**Verify SW-DIST:**

```
show ip interface brief
show ip route
show ip ospf interface brief
show spanning-tree vlan 10
show ip access-lists GUEST-RESTRICT
```

> Screenshot: `show ip interface brief` on SW-DIST showing all SVIs up/up and the Gi0/1 routed link up/up
> Save as: docs/screenshots/12-sw-dist-interfaces.png

> Screenshot: `show spanning-tree vlan 10` on SW-DIST confirming "This bridge is the root"
> Save as: docs/screenshots/13-sw-dist-stp-root.png

> Screenshot: `show ip access-lists GUEST-RESTRICT` on SW-DIST showing the ACL entries
> Save as: docs/screenshots/14-sw-dist-acl.png

---

### 3e. R1-CORE: Core Router

R1-CORE is the WAN edge. It connects to pfSense on one side and SW-DIST on the other. It runs OSPF to share the default route with the rest of the network. See `/configs/R1-CORE.txt` for the full config.

**Base config:**

```
enable
configure terminal

hostname R1-CORE
no service finger
no service udp-small-servers
no service tcp-small-servers
service password-encryption
```

**Configure interfaces:**

```
interface GigabitEthernet0/0
 description WAN to pfSense
 ip address 10.0.0.2 255.255.255.0
 no shutdown

interface GigabitEthernet0/1
 description LAN to SW-DIST Gi0/1
 ip address 10.0.1.1 255.255.255.252
 no shutdown
```

- **Gi0/0 (10.0.0.2/24):** Connects to pfSense LAN interface (10.0.0.1).
- **Gi0/1 (10.0.1.1/30):** Connects to SW-DIST routed uplink (10.0.1.2).

**Configure the static default route to pfSense:**

```
ip route 0.0.0.0 0.0.0.0 10.0.0.1 name DEFAULT-TO-PFSENSE
```

All traffic with no more specific route goes to pfSense for NAT and internet access.

**Configure OSPF:**

```
router ospf 1
 router-id 1.1.1.1
 network 10.0.0.0 0.0.0.255 area 0
 network 10.0.1.0 0.0.0.3 area 0
 default-information originate
```

- **default-information originate:** Redistributes the static default route into OSPF so SW-DIST (and all VLANs behind it) learn a default route via OSPF. Without this, only R1-CORE knows how to reach the internet.

**Save:**

```
end
write memory
```

**Verify R1-CORE:**

```
show ip interface brief
show ip route
show ip ospf neighbor
show ip route ospf
```

Expected OSPF neighbor output:

```
Neighbor ID   Pri  State     Dead Time  Address    Interface
2.2.2.2         1  FULL/DR   00:00:32   10.0.1.2   GigabitEthernet0/1
```

The neighbor should be in **FULL** state. If it shows INIT, 2WAY, or EXSTART, see the troubleshooting section.

Expected OSPF routes on R1-CORE:

```
O  10.10.10.0/24 [110/2] via 10.0.1.2, GigabitEthernet0/1
O  10.10.20.0/24 [110/2] via 10.0.1.2, GigabitEthernet0/1
O  10.10.30.0/24 [110/2] via 10.0.1.2, GigabitEthernet0/1
O  10.10.99.0/24 [110/2] via 10.0.1.2, GigabitEthernet0/1
```

All four VLAN subnets learned from SW-DIST via OSPF.

> Screenshot: `show ip ospf neighbor` on R1-CORE showing FULL adjacency with SW-DIST (2.2.2.2)
> Save as: docs/screenshots/15-r1-core-ospf-neighbor.png

> Screenshot: `show ip route ospf` on R1-CORE showing all four VLAN subnets learned via OSPF
> Save as: docs/screenshots/16-r1-core-ospf-routes.png

Also verify the default route is propagating to SW-DIST:

```
SW-DIST# show ip route 0.0.0.0
```

Expected:

```
O*E2  0.0.0.0/0 [110/1] via 10.0.1.1, GigabitEthernet0/1
```

> Screenshot: `show ip route 0.0.0.0` on SW-DIST showing the OSPF-learned default route (O*E2)
> Save as: docs/screenshots/17-sw-dist-default-route.png

---

### 3f. pfSense: Perimeter Firewall

pfSense handles NAT and perimeter security. See `/pfsense/firewall-rules.md` for the full rule set.

#### Initial Installation

1. Start the pfSense VM in GNS3 (double-click to open console)
2. Boot from the ISO
3. Select **Install pfSense**
4. Accept default UFS filesystem
5. Choose the target disk
6. Wait for installation to complete and reboot

> Screenshot: pfSense installer completed, ready to reboot
> Save as: docs/screenshots/18-pfsense-install-complete.png

#### Interface Assignment

After reboot, pfSense drops to a console menu. Assign interfaces:

1. Select option **1 - Assign Interfaces**
2. Do NOT configure VLANs when prompted (type `n`)
3. WAN interface: `em0` (this connects to the GNS3 NAT cloud for internet)
4. LAN interface: `em1` (this connects to R1-CORE Gi0/0)

> Screenshot: pfSense console showing interface assignments (em0=WAN, em1=LAN)
> Save as: docs/screenshots/19-pfsense-interface-assignment.png

#### Set LAN IP Address

1. Select option **2 - Set interface(s) IP address**
2. Select the LAN interface (em1)
3. Enter IPv4 address: `10.0.0.1`
4. Subnet mask: `24`
5. No upstream gateway for LAN (press Enter to skip)
6. No IPv6 (press Enter to skip)
7. Enable DHCP on LAN: `n` (R1-CORE has a static IP)
8. Revert to HTTP for webConfigurator: `y` (for initial setup)

> Screenshot: pfSense console showing LAN configured as 10.0.0.1/24
> Save as: docs/screenshots/20-pfsense-lan-ip-set.png

#### Verify Connectivity from R1-CORE to pfSense

```
R1-CORE# ping 10.0.0.1
```

This should succeed. If it fails, check that Gi0/0 on R1-CORE is up and in the correct subnet.

#### Web GUI Configuration

Open a browser from a host that can reach 10.0.0.1 (or use a VPCS node on the 10.0.0.0/24 network in GNS3) and navigate to `http://10.0.0.1`.

1. **Login:** admin / pfsense (default credentials)
2. Complete the setup wizard:
   - Hostname: `FW-01`
   - Domain: `lab.internal`
   - Primary DNS: `8.8.8.8`
   - Secondary DNS: `8.8.4.4`
   - Timezone: your timezone
   - WAN interface: DHCP (leave as default)
   - LAN interface: `10.0.0.1/24` (should already be set)
   - Set a new admin password

> Screenshot: pfSense dashboard after completing the setup wizard
> Save as: docs/screenshots/21-pfsense-dashboard.png

#### Add Static Route for Internal Subnets

pfSense needs to know how to reach the VLAN subnets behind R1-CORE.

1. Go to **System > Routing > Static Routes**
2. Add a route:
   - Destination: `10.10.0.0/16`
   - Gateway: `10.0.0.2` (R1-CORE)
   - Description: `Internal VLANs via R1-CORE`
3. Click Save and Apply

> Screenshot: pfSense static route showing 10.10.0.0/16 via 10.0.0.2
> Save as: docs/screenshots/22-pfsense-static-route.png

#### Create Aliases

1. Go to **Firewall > Aliases**
2. Create alias `RFC1918`:
   - Type: Network(s)
   - Networks: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
3. Create alias `INTERNAL_VLANS`:
   - Type: Network(s)
   - Networks: `10.10.10.0/24`, `10.10.20.0/24`, `10.10.30.0/24`, `10.10.99.0/24`
4. Create alias `MANAGEMENT`:
   - Type: Network(s)
   - Networks: `10.10.99.0/24`

> Screenshot: pfSense Aliases page showing RFC1918, INTERNAL_VLANS, and MANAGEMENT aliases
> Save as: docs/screenshots/23-pfsense-aliases.png

#### Configure LAN Firewall Rules

Go to **Firewall > Rules > LAN**. Delete the default "allow LAN to any" rule. Add rules in this order (top-down, first match wins):

| # | Action | Source | Destination | Port | Description |
|---|--------|--------|-------------|------|-------------|
| 1 | Pass | 10.10.10.0/24 | any | any | Staff full access |
| 2 | Pass | 10.10.20.0/24 | any | any | Servers full access |
| 3 | Block | 10.10.30.0/24 | RFC1918 alias | any | Guest blocked from internal |
| 4 | Pass | 10.10.30.0/24 | any | 80, 443 | Guest HTTP/HTTPS only |
| 5 | Pass | 10.10.99.0/24 | any | 22 | MGMT SSH out |
| 6 | Block | any | any | any | Implicit deny (log) |

Enable logging on all rules.

> Screenshot: pfSense LAN firewall rules showing all 6 rules in correct order with logging enabled
> Save as: docs/screenshots/24-pfsense-lan-rules.png

#### Verify NAT

Go to **Firewall > NAT > Outbound**. Ensure outbound NAT is set to **Automatic** (default). This will NAT all internal traffic to the WAN IP for internet access.

> Screenshot: pfSense Outbound NAT showing Automatic mode
> Save as: docs/screenshots/25-pfsense-nat-outbound.png

#### Configure Syslog Forwarding

1. Go to **Status > System Logs > Settings**
2. Check **Enable Remote Logging**
3. Remote log server: `10.10.20.10:514`
4. Check all log categories (especially Firewall Events)
5. Save

> Screenshot: pfSense remote logging settings showing 10.10.20.10:514 configured
> Save as: docs/screenshots/26-pfsense-syslog.png

---

### 3g. SSH Hardening on All Devices

Apply these commands on **all four Cisco devices** (SW-ACC-1, SW-ACC-2, SW-DIST, R1-CORE). The commands are identical on each.

```
configure terminal

ip domain-name lab.internal
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
ip ssh logging events

username admin privilege 15 algorithm-type scrypt secret <SET-YOUR-PASSWORD-HERE>

banner motd #
*************************************************************
* AUTHORIZED ACCESS ONLY - ALL ACTIVITY IS MONITORED       *
*************************************************************
#

line vty 0 4
 login local
 transport input ssh
 exec-timeout 5 0
 logging synchronous

line console 0
 login local
 exec-timeout 10 0
 logging synchronous

no ip http server
no ip http secure-server
no cdp run

end
write memory
```

Key points:
- **RSA 2048-bit key:** Required for SSH v2. The `ip domain-name` must be set before generating the key.
- **transport input ssh:** Restricts VTY lines to SSH only. Telnet is rejected.
- **exec-timeout 5 0:** SSH sessions time out after 5 minutes idle.
- **no ip http server / no ip http secure-server:** Disables the web GUI on Cisco devices (attack surface reduction).
- **no cdp run:** Disables CDP globally. CDP leaks device info to adjacent devices.

**Verify SSH on each device:**

```
show ip ssh
```

Expected:

```
SSH Enabled - version 2.0
Authentication timeout: 60 secs; Authentication retries: 3
```

**Verify telnet is blocked:**

From any other device, try:

```
telnet 10.10.99.1
```

Expected: `Connection refused`

> Screenshot: `show ip ssh` on R1-CORE confirming SSH version 2.0 enabled
> Save as: docs/screenshots/27-r1-core-ssh-enabled.png

> Screenshot: Failed telnet attempt to SW-DIST (10.10.99.1) showing connection refused
> Save as: docs/screenshots/28-telnet-refused.png

---

### 3h. SNMPv3 and Syslog on All Devices

Apply these commands on **all four Cisco devices**. Replace the SNMP credentials if you want different ones.

```
configure terminal

snmp-server group MONITORING v3 priv
snmp-server user wazuh-monitor MONITORING v3 auth sha Auth@2026 priv aes 128 Priv@2026
snmp-server host 10.10.20.10 version 3 priv wazuh-monitor
snmp-server contact noc@lab.internal
snmp-server location Lab-Rack-A1

ntp server 10.10.20.10

logging host 10.10.20.10 transport udp port 514
logging trap informational
logging source-interface Vlan99
logging buffered 16384 informational

end
write memory
```

Note: On R1-CORE, use `logging source-interface GigabitEthernet0/1` instead of `Vlan99` (R1-CORE has no VLAN interfaces). On the access switches, use `logging buffered 8192` (less memory available).

Key points:
- **SNMPv3 with AuthSHA + PrivAES128:** Encrypted SNMP. No community strings in cleartext.
- **NTP:** All devices sync time to 10.10.20.10 so syslog timestamps are consistent.
- **Syslog at informational level:** Captures config changes, interface up/down, ACL hits, and port security violations.
- **Source interface:** Ensures syslog traffic always comes from a predictable IP that the monitoring server can filter on.

**Verify SNMP on each device:**

```
show snmp user
```

Expected:

```
User name: wazuh-monitor
Authentication Protocol: SHA
Privacy Protocol: AES128
Group-name: MONITORING
```

**Verify syslog:**

```
show logging | include 10.10.20.10
```

Expected:

```
Logging to 10.10.20.10 (udp port 514), XX messages logged
```

**Verify NTP:**

```
show ntp status
```

Expected (after sync completes):

```
Clock is synchronized, stratum 4, reference is 10.10.20.10
```

> Screenshot: `show snmp user` on R1-CORE showing wazuh-monitor with SHA/AES128
> Save as: docs/screenshots/29-r1-core-snmpv3.png

> Screenshot: `show logging` on SW-DIST showing syslog messages being sent to 10.10.20.10
> Save as: docs/screenshots/30-sw-dist-syslog.png

> Screenshot: `show ntp status` on any device showing clock synchronized
> Save as: docs/screenshots/31-ntp-synchronized.png

---

## 4. Verification

Run all of these checks after the full build is complete. Expected outputs are from `/verification/test-results.md`.

### 4.1 VLAN Database

```
SW-ACC-1# show vlan brief
```

Expected: VLANs 10 (STAFF Gi0/1-8), 20, 30, 99 (MGMT Gi0/9-23) all active.

```
SW-ACC-2# show vlan brief
```

Expected: VLANs 10, 20 (SERVERS Gi0/1-8), 30 (GUEST Gi0/9-16), 99 (unused Gi0/17-23) all active.

> Screenshot: `show vlan brief` on both access switches side by side
> Save as: docs/screenshots/32-vlan-database-both-switches.png

### 4.2 Trunk Links

```
SW-DIST# show interfaces trunk
```

Expected: Gi0/2 and Gi0/3 trunking, native VLAN 99, VLANs 10,20,30,99 allowed and active.

> Screenshot: `show interfaces trunk` on SW-DIST showing both trunks healthy
> Save as: docs/screenshots/33-trunk-verification.png

### 4.3 OSPF Adjacency

```
R1-CORE# show ip ospf neighbor
```

Expected: Neighbor 2.2.2.2 in FULL state on Gi0/1.

```
SW-DIST# show ip ospf neighbor
```

Expected: Neighbor 1.1.1.1 in FULL state on Gi0/1.

> Screenshot: `show ip ospf neighbor` on both R1-CORE and SW-DIST showing FULL adjacency
> Save as: docs/screenshots/34-ospf-adjacency.png

### 4.4 Routing Table

```
R1-CORE# show ip route ospf
```

Expected: All four VLAN subnets (10.10.10.0/24, 10.10.20.0/24, 10.10.30.0/24, 10.10.99.0/24) learned via OSPF.

```
SW-DIST# show ip route 0.0.0.0
```

Expected: Default route learned via OSPF (O*E2) pointing to 10.0.1.1.

> Screenshot: `show ip route` on R1-CORE showing the complete routing table with OSPF and static routes
> Save as: docs/screenshots/35-r1-core-routing-table.png

> Screenshot: `show ip route` on SW-DIST showing the complete routing table with connected, OSPF, and default route
> Save as: docs/screenshots/36-sw-dist-routing-table.png

### 4.5 Internet Reachability

From a STAFF host (or ping sourced from VLAN 10):

```
ping 8.8.8.8
```

Expected: Replies from 8.8.8.8.

> Screenshot: Successful ping to 8.8.8.8 from a staff VLAN host
> Save as: docs/screenshots/37-internet-reachability.png

### 4.6 Guest Isolation (ACL)

From a GUEST host, try to reach the server VLAN:

```
ping 10.10.20.10
```

Expected: `Destination Net Unreachable` (blocked by GUEST-RESTRICT ACL).

From the same GUEST host, try the internet:

```
ping 8.8.8.8
```

Expected: Replies from 8.8.8.8 (permitted by the ACL and pfSense rule 4).

Verify ACL hit counters:

```
SW-DIST# show ip access-lists GUEST-RESTRICT
```

Expected: Match counters incrementing on deny entries.

> Screenshot: Failed ping from guest to 10.10.20.10 showing "Destination Net Unreachable"
> Save as: docs/screenshots/38-guest-isolation-blocked.png

> Screenshot: Successful ping from guest to 8.8.8.8 showing replies
> Save as: docs/screenshots/39-guest-internet-works.png

> Screenshot: `show ip access-lists GUEST-RESTRICT` showing hit counters on deny lines
> Save as: docs/screenshots/40-acl-hit-counters.png

### 4.7 Spanning Tree

```
SW-DIST# show spanning-tree vlan 10
```

Expected: SW-DIST shows "This bridge is the root" with priority 4096+VLAN. Both trunk ports in Designated Forwarding state.

> Screenshot: `show spanning-tree vlan 10` on SW-DIST confirming root bridge status
> Save as: docs/screenshots/41-stp-root-verification.png

### 4.8 Port Security

```
SW-ACC-1# show port-security interface GigabitEthernet0/1
```

Expected: Port Security Enabled, Secure-up, Violation Mode Restrict, 0 violations.

> Screenshot: `show port-security` summary on SW-ACC-1
> Save as: docs/screenshots/42-port-security-status.png

### 4.9 SSH Verification

```
R1-CORE# show ip ssh
```

Expected: SSH Enabled - version 2.0.

Attempt telnet from any device:

```
telnet 10.10.99.1
```

Expected: Connection refused.

> Screenshot: `show ip ssh` output and failed telnet attempt
> Save as: docs/screenshots/43-ssh-verification.png

### 4.10 SNMPv3

```
R1-CORE# show snmp user
```

Expected: User wazuh-monitor, Auth SHA, Priv AES128, Group MONITORING.

> Screenshot: `show snmp user` on all four devices
> Save as: docs/screenshots/44-snmpv3-all-devices.png

### 4.11 Syslog

```
R1-CORE# show logging | include 10.10.20.10
```

Expected: `Logging to 10.10.20.10 (udp port 514), XX messages logged`

> Screenshot: `show logging` on R1-CORE confirming messages are being sent
> Save as: docs/screenshots/45-syslog-active.png

### 4.12 pfSense Firewall Logs

In the pfSense GUI, go to **Status > System Logs > Firewall**.

Check for:
- Blocked entries from 10.10.30.0/24 to RFC1918 addresses (rule 3 working)
- Passed entries from 10.10.10.0/24 (staff traffic)

> Screenshot: pfSense firewall log showing blocked guest traffic and passed staff traffic
> Save as: docs/screenshots/46-pfsense-firewall-logs.png

---

## 5. Troubleshooting Common Issues

See `/docs/troubleshooting-guide.md` for the full troubleshooting reference. Below is a quick summary of the most common problems.

### OSPF Neighbors Not Forming

**Symptoms:** `show ip ospf neighbor` is empty on R1-CORE or SW-DIST.

**Check these in order:**

```
R1-CORE# show interfaces GigabitEthernet0/1
! Line protocol must be up/up

R1-CORE# ping 10.0.1.2
! Basic L3 connectivity to SW-DIST

R1-CORE# show ip ospf interface GigabitEthernet0/1
! Verify hello/dead timers match (10/40 default)

R1-CORE# show run | section router ospf
! Network statements must cover 10.0.1.0/30
```

**GNS3-specific fix:** If OSPF is stuck in EXSTART, add `ip ospf mtu-ignore` on both sides of the link. This is a known GNS3 quirk with IOSv.

### VLANs Not Passing Traffic

**Symptoms:** Host in VLAN 10 cannot ping its gateway (10.10.10.1).

```
SW-ACC-1# show vlan brief
! Confirm port is in the correct VLAN

SW-DIST# show interfaces trunk
! VLAN must appear in "allowed and active" section

SW-DIST# show interfaces Vlan10
! SVI must be up/up. If down/down, no active ports in that VLAN
```

**Most common cause:** VLAN not created in the SW-DIST VLAN database. The trunk will show the VLAN as "allowed" but not "active."

### Guest Can Reach Internal Hosts

**Symptoms:** Ping from VLAN 30 to 10.10.20.10 succeeds when it should fail.

```
SW-DIST# show ip interface Vlan30
! Look for "Inbound access list is GUEST-RESTRICT"
! If it says "not set," the ACL exists but was never applied

SW-DIST# show ip access-lists GUEST-RESTRICT
! Check hit counters. If 0, traffic is not hitting the ACL.
```

**Most common cause:** ACL was created but never applied to the interface with `ip access-group GUEST-RESTRICT in`.

### No Internet from Any VLAN

**Symptoms:** Hosts can reach each other across VLANs but cannot ping 8.8.8.8.

```
R1-CORE# show ip route 0.0.0.0
! Must show static route to 10.0.0.1

SW-DIST# show ip route 0.0.0.0
! Must show O*E2 default via OSPF
! If missing: check "default-information originate" under R1-CORE OSPF config

R1-CORE# ping 10.0.0.1
! If pfSense is unreachable, nothing will work
```

**Also check pfSense:**
- Static route for 10.10.0.0/16 via 10.0.0.2 must exist
- Outbound NAT must be enabled
- LAN rules must permit traffic from internal subnets

### Port Goes err-disabled

**Symptoms:** A port on SW-ACC-2 shows `err-disabled` status.

```
SW-ACC-2# show port-security interface GigabitEthernet0/5
! Shows violation count and the MAC that triggered it

SW-ACC-2# show log | include SECURITY
! Timestamp and details of the violation
```

**To recover:**

```
SW-ACC-2(config)# interface GigabitEthernet0/5
SW-ACC-2(config-if)# shutdown
SW-ACC-2(config-if)# no shutdown
```

If the MAC legitimately changed (server hardware swap), clear sticky entries first:

```
SW-ACC-2(config-if)# no switchport port-security mac-address sticky
SW-ACC-2(config-if)# switchport port-security mac-address sticky
```

### Wrong STP Root

**Symptoms:** An access switch becomes STP root, causing suboptimal paths.

```
SW-DIST# show spanning-tree vlan 10
! Should say "This bridge is the root"
! If not, set the priority:
SW-DIST(config)# spanning-tree vlan 10,20,30,99 priority 4096
```

### SSH Connection Refused

**Symptoms:** Cannot SSH to a device.

```
show ip ssh
! Must say "SSH Enabled - version 2.0"
! If disabled: check that ip domain-name is set and RSA key was generated

show run | section line vty
! transport input must say "ssh"

show run | include username
! A local user must exist
```

If you are locked out of SSH, console access still works (login local is configured on the console line).

---

## Quick Reference: All Device IPs

| Device | Interface | IP Address | Purpose |
|--------|-----------|-----------|---------|
| pfSense | em0 (WAN) | DHCP | Internet uplink |
| pfSense | em1 (LAN) | 10.0.0.1/24 | Gateway for R1-CORE |
| R1-CORE | Gi0/0 | 10.0.0.2/24 | WAN side to pfSense |
| R1-CORE | Gi0/1 | 10.0.1.1/30 | LAN side to SW-DIST |
| SW-DIST | Gi0/1 | 10.0.1.2/30 | Uplink to R1-CORE |
| SW-DIST | Vlan10 | 10.10.10.1/24 | Staff gateway |
| SW-DIST | Vlan20 | 10.10.20.1/24 | Servers gateway |
| SW-DIST | Vlan30 | 10.10.30.1/24 | Guest gateway |
| SW-DIST | Vlan99 | 10.10.99.1/24 | Management gateway |
| SW-ACC-1 | Vlan99 | 10.10.99.11/24 | Management |
| SW-ACC-2 | Vlan99 | 10.10.99.12/24 | Management |
