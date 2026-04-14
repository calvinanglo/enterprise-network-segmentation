# Enterprise Network Segmentation

Built a segmented network using Cisco IOS devices and pfSense. The goal was to get hands-on with VLANs, OSPF, ACLs, and firewall rules in a setup that mirrors what you'd actually see in a small enterprise environment. Everything is version controlled here so I can track changes and roll back if something breaks.

## Project Series

This is **Project 1 of 5** in a production enterprise environment build. Each project builds on the previous one.

| # | Project | What It Adds |
|---|---------|-------------|
| 1 | [Enterprise Network Segmentation](https://github.com/calvinanglo/enterprise-network-segmentation) | VLANs, OSPF, ACLs, pfSense firewall |
| 2 | [Wazuh SIEM Deployment](https://github.com/calvinanglo/wazuh-siem-deployment) | Centralized log collection, threat detection, incident response |
| 3 | [Compliance Hardening Pipeline](https://github.com/calvinanglo/compliance-hardening-pipeline) | Automated CIS benchmarks across all devices |
| 4 | [Network Monitoring Stack](https://github.com/calvinanglo/network-monitoring-stack) | Prometheus, Grafana, SNMP monitoring, SLA dashboards |
| 5 | [DR & BC Simulation](https://github.com/calvinanglo/dr-bc-simulation) | Disaster recovery testing, backup validation, RTO/RPO measurement |

### Prerequisites
- GNS3 or EVE-NG with Cisco IOSv images
- pfSense CE 2.7.x VM
- 8GB RAM minimum

### What's Next
After completing this project, continue to [Project 2: Wazuh SIEM Deployment](https://github.com/calvinanglo/wazuh-siem-deployment) to add centralized security monitoring to this network. The Wazuh server will sit on VLAN 20 (10.10.20.10) and collect syslog from all four Cisco devices and pfSense.

## What I built

Four VLANs across two access switches feeding into a Layer 3 distribution switch, which connects up to a core router, which connects to pfSense as the perimeter firewall. pfSense handles NAT and zone-based rules. All syslog and SNMP traffic goes to a central monitoring host at 10.10.20.10 (Wazuh/Prometheus).

```
Internet
    |
pfSense FW-01 (10.0.0.1)
    |
R1-CORE (10.0.0.2 / 10.0.1.1)
    |
SW-DIST (10.0.1.2) - Layer 3, OSPF, STP root
    |           |
SW-ACC-1      SW-ACC-2
(10.10.99.11) (10.10.99.12)
```

## IP addressing

| VLAN | Name    | Subnet          | Gateway     | What's on it                        |
|------|---------|-----------------|-------------|-------------------------------------|
| 10   | STAFF   | 10.10.10.0/24   | 10.10.10.1  | Employee workstations               |
| 20   | SERVERS | 10.10.20.0/24   | 10.10.20.1  | Internal servers, Wazuh, Prometheus |
| 30   | GUEST   | 10.10.30.0/24   | 10.10.30.1  | Guest wifi, internet only           |
| 99   | MGMT    | 10.10.99.0/24   | 10.10.99.1  | Out of band device management       |

## Design decisions

**Why guest is isolated at three layers:** VLAN 30 is blocked from reaching internal subnets at Layer 2 (separate VLAN), Layer 3 (GUEST-RESTRICT ACL on SW-DIST Vlan30 inbound), and at the perimeter (pfSense rule blocks RFC1918 from guest source). If any one layer is misconfigured, the other two still catch it.

**Why SNMPv3 instead of v2c:** SNMPv2c sends community strings in cleartext. SNMPv3 with AuthSHA + PrivAES128 encrypts the session. All four devices use the same wazuh-monitor user so Prometheus can poll them from one place.

**Why passive-interface on OSPF:** OSPF hellos on access-facing interfaces are unnecessary and slightly extend the attack surface. Passive-interface stops OSPF from advertising on interfaces that have no OSPF neighbors, while still including those networks in the OSPF database.

**Why native VLAN 99 on trunks:** Default native VLAN is 1 and you don't want untagged traffic landing on the management VLAN or mixing with user traffic. Setting native VLAN to 99 and keeping 99 unused for data means untagged frames go somewhere harmless.

## How to build it

You need GNS3 or EVE-NG with Cisco IOSv images and a pfSense CE 2.7.x VM. At least 8GB RAM for the full topology.

Build order matters because each phase depends on the last:

1. Configure VLANs and access ports on SW-ACC-1 and SW-ACC-2
2. Set up trunks between access switches and SW-DIST
3. Enable ip routing on SW-DIST, configure SVIs, bring up the routed link to R1-CORE
4. Configure OSPF on both R1-CORE and SW-DIST
5. Add the GUEST-RESTRICT ACL on SW-DIST Vlan30
6. SSH harden all devices (generate RSA keys, restrict VTY to SSH, set banners)
7. Deploy pfSense, configure LAN rules and NAT
8. Add SNMPv3 and syslog forwarding on all Cisco devices

The config files in /configs have the full running configs for each device. Comments in the files explain why things are configured the way they are.

## Verification

After building, the key things to verify:

```
# OSPF neighbors should be FULL state
R1-CORE# show ip ospf neighbor

# All 4 VLAN subnets should show up as OSPF routes on R1-CORE
R1-CORE# show ip route ospf

# Guest to server should fail, guest to internet should work
SW-DIST# show ip access-lists GUEST-RESTRICT

# SW-DIST should be root for all VLANs
SW-DIST# show spanning-tree vlan 10

# SSH version 2, telnet should be refused
R1-CORE# show ip ssh
```

Full test results with expected outputs are in /verification/test-results.md.

## ITIL change records

Every config change has a corresponding change record in /docs/itil-change-log.md with impact, risk, and rollback procedure. The Git commit history maps directly to those change IDs.

## Certifications this covers

CCNA 200-301 — VLAN design and inter-VLAN routing, OSPF configuration and troubleshooting, ACL syntax and placement, STP root bridge election, SNMPv3 and syslog

CompTIA Security+ — network segmentation as a security control, defense in depth (guest isolation at 3 layers), SNMPv3 vs v2c security implications, ACL-based access control

ISC2 CC — confidentiality through VLAN isolation, integrity of network configs via version control, availability through OSPF redundancy and STP

ITIL 4 — change management process for network modifications, incident management via Wazuh/syslog integration, service level monitoring via Prometheus/Grafana
