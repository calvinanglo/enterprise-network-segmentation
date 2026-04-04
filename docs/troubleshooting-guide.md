# Troubleshooting Notes

Running log of issues I hit building this and how I fixed them. General approach is OSI bottom-up: check physical/Layer 2 first, then Layer 3, then check ACLs and firewall if routing is fine.

## OSPF neighbors not forming

Usually one of three things: the link between R1-CORE and SW-DIST is down, hello/dead timers don't match, or the network statements don't cover the actual interface IPs.

```
R1-CORE# show ip ospf neighbor
# if empty, start here
R1-CORE# show interfaces GigabitEthernet0/1
# line protocol needs to be up

R1-CORE# ping 10.0.1.2
# basic connectivity first

R1-CORE# show ip ospf interface GigabitEthernet0/1
# check hello/dead timers, both sides need to match

R1-CORE# show run | section router ospf
# verify network statements actually cover 10.0.1.0/30
```

GNS3 also has an MTU quirk where OSPF gets stuck in EXSTART. Fix is `ip ospf mtu-ignore` on the interface if that happens.

## VLANs not passing traffic

If a host can't reach its gateway, work through this order:

```
SW-ACC-1# show vlan brief
# confirm the port is actually in the right VLAN, not VLAN 1

SW-ACC-1# show interfaces GigabitEthernet0/1 switchport
# access mode VLAN should be 10, not 1

SW-DIST# show interfaces trunk
# VLAN 10 needs to be in the "active" section, not just "allowed"

SW-DIST# show interfaces Vlan10
# SVI needs to be up/up. if down/down, no active ports in that VLAN
```

Most common mistake was forgetting to add the VLAN to the database on SW-DIST separately. The VLAN config on the access switches doesn't automatically create it on the distribution switch when ip routing is on.

## Guest can reach internal hosts (ACL not working)

```
SW-DIST# show ip interface Vlan30
# look for "Inbound access list is GUEST-RESTRICT"
# if it says "not set" the ACL exists but was never applied to the interface

SW-DIST# show ip access-lists GUEST-RESTRICT
# match counters should be incrementing when guest pings internal
# if counters are 0, traffic isn't hitting the ACL at all
```

The direction matters. ACL has to be inbound on Vlan30, not outbound. Outbound would filter traffic leaving Vlan30 which is the opposite of what we want.

## SSH refused

```
SW-ACC-1# show ip ssh
# needs to say "SSH Enabled - version 2.0"
# if it says disabled, either no domain-name is set or the RSA key was never generated

SW-ACC-1# show run | section line vty
# transport input needs to say ssh, not none or telnet

SW-ACC-1# show run | include username
# local user needs to exist
```

If you accidentally lock yourself out, console access still works since we have `login local` on console line with a local username.

## No internet from any VLAN

```
R1-CORE# show ip route 0.0.0.0
# needs the static default pointing to 10.0.0.1 (pfSense)

SW-DIST# show ip route 0.0.0.0
# needs to learn it via OSPF (O*E2)
# if missing, check that R1-CORE has "default-information originate" under router ospf

R1-CORE# ping 10.0.0.1
# if pfSense is unreachable from R1-CORE nothing will work
```

## Port goes err-disabled

Port security violation with shutdown mode. Happens when a new device is connected or someone swaps hardware.

```
SW-ACC-1# show port-security interface GigabitEthernet0/5
# shows the violation count and last MAC that triggered it

SW-ACC-1# show log | include SECURITY
# timestamp and MAC address of whatever caused it
```

To recover:
```
SW-ACC-1(config)# interface GigabitEthernet0/5
SW-ACC-1(config-if)# shutdown
SW-ACC-1(config-if)# no shutdown
```

If it's a server port with a known MAC that changed, clear the sticky entries first then let it relearn.

## Wrong STP root

SW-DIST should be root for all VLANs. If an access switch has a lower bridge ID it can win the root election.

```
SW-DIST# show spanning-tree vlan 10
# should say "This bridge is the root"
# SW-DIST priority is set to 4096, access switches default to 32768 so SW-DIST wins
```

If SW-DIST isn't root: `spanning-tree vlan 10,20,30,99 priority 4096` fixes it.

## Useful commands

```
show ip ospf neighbor           ospf adjacency state
show ip route ospf              routes learned via OSPF
show vlan brief                 VLAN to port mapping
show interfaces trunk           trunk status and allowed VLANs
show ip access-lists            ACL hit counters
show port-security interface    per-port security status
show spanning-tree vlan 10      STP topology
show ip ssh                     SSH version and status
show snmp user                  SNMPv3 user config
show ntp status                 NTP sync state
show interfaces status          quick view of all port states
show log | include ERR          error log entries
```
