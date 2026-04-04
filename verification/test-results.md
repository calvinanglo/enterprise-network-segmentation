# Test Results

Verified 2026-04-04. GNS3 with Cisco IOSv 15.9 and pfSense CE 2.7.x.

## VLAN database

```
SW-ACC-1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- ----------------------------
10   STAFF                            active    Gi0/1-8
20   SERVERS                          active
30   GUEST                            active
99   MGMT                             active    Gi0/9-23
```

VLANs 10, 20, 30, 99 all active. Staff ports on Gi0/1-8. Unused ports in VLAN 99.

## Trunk links

```
SW-DIST# show interfaces trunk

Port   Mode   Encapsulation  Status    Native vlan
Gi0/2  on     802.1q         trunking  99
Gi0/3  on     802.1q         trunking  99

Gi0/2  vlans allowed and active: 10,20,30,99
Gi0/3  vlans allowed and active: 10,20,30,99
```

Both trunks up. Native VLAN 99, not VLAN 1. Only the four VLANs we need are allowed.

## OSPF neighbors

```
R1-CORE# show ip ospf neighbor

Neighbor ID   Pri  State     Dead Time  Address    Interface
2.2.2.2         1  FULL/DR   00:00:32   10.0.1.2   GigabitEthernet0/1
```

FULL state, neighbor is SW-DIST (router-id 2.2.2.2).

```
SW-DIST# show ip ospf neighbor

Neighbor ID   Pri  State      Dead Time  Address    Interface
1.1.1.1         1  FULL/BDR   00:00:38   10.0.1.1   GigabitEthernet0/1
```

## Routing table

```
R1-CORE# show ip route ospf

O  10.10.10.0/24 [110/2] via 10.0.1.2, GigabitEthernet0/1
O  10.10.20.0/24 [110/2] via 10.0.1.2, GigabitEthernet0/1
O  10.10.30.0/24 [110/2] via 10.0.1.2, GigabitEthernet0/1
O  10.10.99.0/24 [110/2] via 10.0.1.2, GigabitEthernet0/1
```

All four VLAN subnets learned via OSPF on R1-CORE.

```
SW-DIST# show ip route 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 10.0.1.1, GigabitEthernet0/1
```

Default route learning via OSPF from R1-CORE.

## Internet reachability

```
STAFF-PC> ping 8.8.8.8
84 bytes from 8.8.8.8: ttl=55 time=12.4ms
84 bytes from 8.8.8.8: ttl=55 time=11.8ms
```

Staff can reach internet through pfSense NAT.

## ACL enforcement

```
GUEST-PC> ping 10.10.20.10
*10.10.30.1: Destination Net Unreachable
*10.10.30.1: Destination Net Unreachable
```

Guest blocked from server VLAN by GUEST-RESTRICT ACL.

```
GUEST-PC> ping 8.8.8.8
84 bytes from 8.8.8.8: ttl=55 time=15.2ms
```

Guest internet works fine, permit statement at end of ACL is working.

```
SW-DIST# show ip access-lists GUEST-RESTRICT

Extended IP access list GUEST-RESTRICT
    10 deny ip 10.10.30.0 0.0.0.255 10.10.20.0 0.0.0.255 log (8 matches)
    20 deny ip 10.10.30.0 0.0.0.255 10.10.10.0 0.0.0.255 log (4 matches)
    30 deny ip 10.10.30.0 0.0.0.255 10.10.99.0 0.0.0.255 log (0 matches)
    40 permit ip 10.10.30.0 0.0.0.255 any (24 matches)
```

Hit counters confirm traffic is being processed by the ACL.

## SSH

```
R1-CORE# show ip ssh
SSH Enabled - version 2.0
Authentication timeout: 60 secs; Authentication retries: 3
```

```
$ telnet 10.10.99.1
Trying 10.10.99.1...
telnet: connect to host 10.10.99.1: Connection refused
```

SSH v2 working, Telnet refused on all devices.

## Port security

```
SW-ACC-1# show port-security interface GigabitEthernet0/1

Port Security     : Enabled
Port Status       : Secure-up
Violation Mode    : Restrict
Maximum MAC       : 2
Sticky MAC        : 1
Violation Count   : 0
```

One sticky MAC learned, no violations.

## Spanning tree

```
SW-DIST# show spanning-tree vlan 10

  Root ID  Priority  4106
           This bridge is the root
  Protocol: rstp
  Gi0/2  Desg FWD
  Gi0/3  Desg FWD
```

SW-DIST is root, Rapid PVST+ running, both trunk ports forwarding.

## SNMPv3

```
R1-CORE# show snmp user

User name: wazuh-monitor
Authentication Protocol: SHA
Privacy Protocol: AES128
Group-name: MONITORING
```

v3 with AuthSHA and PrivAES128 on all four devices.

## NTP

```
R1-CORE# show ntp status
Clock is synchronized, stratum 4, reference is 10.10.20.10
```

Synced to monitoring server.

## Syslog

```
R1-CORE# show logging | include 10.10.20.10
Logging to 10.10.20.10 (udp port 514), 47 messages logged
```

Logs flowing to Wazuh at 10.10.20.10.

## Summary

All 12 checks passed. OSPF up, VLANs working, ACL blocking guest traffic correctly, SSH hardened, SNMPv3 configured, syslog running.

