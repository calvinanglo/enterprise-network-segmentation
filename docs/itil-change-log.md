# Change Log

Tracking changes to this environment the same way I would in a production ITIL shop. Standard changes are pre-approved low-risk stuff. Normal changes need an impact assessment and a rollback plan before touching anything.

| ID | Date | What | Type | Impact | Risk | Status |
|----|------|------|------|--------|------|--------|
| CHG-001 | 2026-04-04 | VLAN database on SW-ACC-1 and SW-ACC-2 | Standard | Low | Low | Done |
| CHG-002 | 2026-04-04 | 802.1Q trunks on all uplinks | Standard | Low | Low | Done |
| CHG-003 | 2026-04-04 | IP routing and SVIs on SW-DIST | Normal | Medium | Medium | Done |
| CHG-004 | 2026-04-04 | OSPF on R1-CORE and SW-DIST | Normal | Medium | Medium | Done |
| CHG-005 | 2026-04-04 | GUEST-RESTRICT ACL on Vlan30 inbound | Normal | High | Low | Done |
| CHG-006 | 2026-04-04 | SSH hardening on all devices, disable Telnet | Normal | Medium | Low | Done |
| CHG-007 | 2026-04-04 | pfSense deployment, firewall rules, NAT | Normal | High | Medium | Done |
| CHG-008 | 2026-04-04 | SNMPv3 on all Cisco devices | Standard | Low | Low | Done |

## Notes on each change

**CHG-001 and CHG-002** are standard because adding VLANs and trunks is additive. Nothing breaks if you get it wrong, you just won't have connectivity until the next step.

**CHG-003** is where it gets interesting. Enabling ip routing on SW-DIST means the switch starts forwarding between VLANs. If the SVI IPs are wrong, hosts lose their gateway. Tested each SVI with a ping before moving on.

**CHG-004** OSPF adjacency between R1-CORE and SW-DIST. Main risk is if the network statements don't match the actual interface IPs, OSPF forms but doesn't advertise the right subnets. Verified with `show ip route ospf` on both devices before adding the default route.

**CHG-005** High impact because applying an ACL in the wrong direction or with a typo can block the wrong traffic. Tested the ACL logic in isolation first, then applied to Vlan30 inbound and immediately verified Guest can still reach the internet but can't ping internal hosts. ACL hit counters confirmed it was working.

**CHG-006** The risk here is locking yourself out. Generated RSA keys and verified SSH worked before removing Telnet from the VTY lines. Applied to one device at a time rather than all four at once.

**CHG-007** Biggest change in the project. pfSense touches everything once it's in the path. Built and tested the rules one at a time starting with the Pass rules before enabling the implicit deny, so I could verify each VLAN's connectivity before adding restrictions.

**CHG-008** Standard because SNMPv3 is additive. Added it to all four devices, verified with `show snmp user`, then confirmed Prometheus could pull metrics before removing any v2c config.

## Rollback procedures

Each change has a rollback in case something goes wrong:

- CHG-001: `no vlan 10/20/30/99` on both access switches
- - CHG-002: `switchport mode access` on the trunk interfaces
  - - CHG-003: `no ip routing` on SW-DIST, remove SVI IPs
    - - CHG-004: `no router ospf 1` on both devices
      - - CHG-005: `no ip access-group GUEST-RESTRICT in` on Vlan30
        - - CHG-006: `transport input telnet ssh` on VTY lines as a recovery option
          - - CHG-007: disable firewall rules temporarily, revert default gateway on R1-CORE
            - - CHG-008: re-add community strings if v3 polling breaks
