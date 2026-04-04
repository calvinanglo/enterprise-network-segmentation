# ITIL 4 Change Log — Project 1: Enterprise Network Segmentation

**ITIL 4 Practice:** Change Enablement  
**Service:** Enterprise Network Segmentation (Lab Environment)  
**Environment:** Production-equivalent lab (GNS3/EVE-NG)  
**Change Authority:** Calvin Anglo (Engineer / CAB-equivalent)

---

## Change Types Used in This Project

| Type     | Definition                                                                  | Approval Required        |
|----------|-----------------------------------------------------------------------------|--------------------------|
| Standard | Pre-approved, low-risk, routine changes with defined procedures             | None — pre-approved      |
| Normal   | Requires assessment, impact analysis, and risk rating before implementation | Engineer review required |
| Emergency| Unplanned, urgent changes to restore service or address critical risk       | Post-implementation review|

---

## Change Records

---

### CHG-001
| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| **Change ID**    | CHG-001                                                        |
| **Date**         | 2026-04-04                                                     |
| **Type**         | Standard                                                       |
| **Title**        | Deploy VLAN database on SW-ACC-1 and SW-ACC-2                  |
| **Description**  | Create VLANs 10 (STAFF), 20 (SERVERS), 30 (GUEST), 99 (MGMT) on both access switches. No routing changes — Layer 2 only. |
| **Impact**       | Low — new VLANs have no effect until ports are assigned        |
| **Risk**         | Low — additive change, no existing traffic affected            |
| **Rollback**     | `no vlan 10`, `no vlan 20`, `no vlan 30`, `no vlan 99` on both switches |
| **Status**       | Completed                                                      |
| **Verification** | `show vlan brief` — all four VLANs present, status active      |
| **Cert Mapping** | CCNA: Network Access — VLANs; ITIL 4: Change Enablement        |

---

### CHG-002
| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| **Change ID**    | CHG-002                                                        |
| **Date**         | 2026-04-04                                                     |
| **Type**         | Standard                                                       |
| **Title**        | Configure 802.1Q trunks on all uplinks                         |
| **Description**  | Configure trunk ports on SW-ACC-1 Gi0/24 → SW-DIST Gi0/2 and SW-ACC-2 Gi0/24 → SW-DIST Gi0/3. Allow VLANs 10,20,30,99. Native VLAN 99. Root Guard enabled on access switch uplinks. |
| **Impact**       | Low — new trunk links, no existing traffic on these interfaces |
| **Risk**         | Low — native VLAN mismatch risk mitigated by explicit native VLAN config |
| **Rollback**     | `switchport mode access` on both trunk interfaces              |
| **Status**       | Completed                                                      |
| **Verification** | `show interfaces trunk` on SW-DIST — both Gi0/2 and Gi0/3 in trunking mode, VLANs 10,20,30,99 active |
| **Cert Mapping** | CCNA: Network Access — 802.1Q; Security+: Hardening — native VLAN security |

---

### CHG-003
| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| **Change ID**    | CHG-003                                                        |
| **Date**         | 2026-04-04                                                     |
| **Type**         | Normal                                                         |
| **Title**        | Enable IP routing and configure SVIs on SW-DIST                |
| **Description**  | Enable `ip routing` and `ip cef` on SW-DIST. Create SVIs for all four VLANs: Vlan10 (10.10.10.1/24), Vlan20 (10.10.20.1/24), Vlan30 (10.10.30.1/24), Vlan99 (10.10.99.1/24). Configure routed uplink Gi0/1 to R1-CORE. |
| **Impact**       | Medium — enables inter-VLAN routing; hosts gain gateway access |
| **Risk**         | Medium — routing errors could cause unintended traffic paths    |
| **Pre-change**   | Verify no existing routing table entries conflict               |
| **Rollback**     | `no ip routing` on SW-DIST; remove SVI IPs                    |
| **Status**       | Completed                                                      |
| **Verification** | `show ip route connected` — all four VLAN subnets present; `ping 10.10.10.1` from Staff host succeeds |
| **Cert Mapping** | CCNA: IP Connectivity — L3 switching, SVIs; ISC2 CC: Network security |

---

### CHG-004
| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| **Change ID**    | CHG-004                                                        |
| **Date**         | 2026-04-04                                                     |
| **Type**         | Normal                                                         |
| **Title**        | Deploy OSPF single-area (R1-CORE and SW-DIST)                  |
| **Description**  | Configure OSPF process 1 on R1-CORE (router-id 1.1.1.1) and SW-DIST (router-id 2.2.2.2). Advertise WAN link (10.0.0.0/24), distribution link (10.0.1.0/30), and all VLAN subnets. Enable passive-interface on all VLAN SVIs. Configure `default-information originate` on R1-CORE to distribute default route. |
| **Impact**       | Medium — all hosts gain internet routing via OSPF-learned default route |
| **Risk**         | Medium — incorrect OSPF config could blackhole traffic; mitigated by passive-interface on access VLANs |
| **Pre-change**   | Verify routed link 10.0.1.0/30 is up between R1-CORE and SW-DIST |
| **Rollback**     | `no router ospf 1` on both devices                            |
| **Status**       | Completed                                                      |
| **Verification** | `show ip ospf neighbor` — FULL/DR state between R1-CORE and SW-DIST; `show ip route ospf` on both devices shows all VLAN subnets |
| **Cert Mapping** | CCNA: IP Connectivity — OSPFv2; ISC2 CC: Availability management |

---

### CHG-005
| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| **Change ID**    | CHG-005                                                        |
| **Date**         | 2026-04-04                                                     |
| **Type**         | Normal                                                         |
| **Title**        | Apply GUEST-RESTRICT ACL on Vlan30 inbound interface           |
| **Description**  | Create extended named ACL `GUEST-RESTRICT` on SW-DIST. Deny Guest (10.10.30.0/24) to Servers (10.10.20.0/24), Staff (10.10.10.0/24), and MGMT (10.10.99.0/24). Permit Guest to `any` (internet via pfSense NAT). Apply inbound on Vlan30 SVI. All deny entries include `log` keyword. |
| **Impact**       | High — immediately restricts Guest VLAN traffic; mis-applied ACL could block internet for all VLANs |
| **Risk**         | Low — ACL tested in isolation before applying to SVI; implicit permit any at end avoided intentionally |
| **Pre-change**   | Test ACL logic on paper; confirm permit statement covers internet-bound traffic before applying |
| **Post-change**  | Verify Guest internet still works; confirm `ping 10.10.20.10 source vlan 30` fails |
| **Rollback**     | `no ip access-group GUEST-RESTRICT in` on Vlan30              |
| **Status**       | Completed                                                      |
| **Verification** | `show ip access-lists GUEST-RESTRICT` — deny entries show hit counts after test pings from Guest |
| **Cert Mapping** | CCNA: Security — Extended ACLs; Security+: Network Architecture — Segmentation; ITIL 4: Risk Management |

---

### CHG-006
| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| **Change ID**    | CHG-006                                                        |
| **Date**         | 2026-04-04                                                     |
| **Type**         | Normal                                                         |
| **Title**        | Harden all Cisco devices: SSH v2, disable Telnet, banners, password encryption |
| **Description**  | Apply to R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2: generate RSA 2048-bit keys, enable SSH v2, set auth-retries 3 and timeout 60. Restrict VTY to `transport input ssh`. Add `banner motd`. Enable `service password-encryption`. Disable `ip http server`, `ip http secure-server`, `cdp run`, and small server services. Configure `exec-timeout` on all lines. |
| **Impact**       | Medium — Telnet sessions in progress will be terminated       |
| **Risk**         | Low — SSH keys pre-generated and tested before removing Telnet |
| **Pre-change**   | Verify SSH connectivity works before disabling Telnet          |
| **Rollback**     | `transport input telnet ssh` on VTY lines (partial rollback available) |
| **Status**       | Completed                                                      |
| **Verification** | `show ip ssh` — SSH enabled, version 2; `telnet 10.10.99.11` — connection refused |
| **Cert Mapping** | Security+: Hardening — SSH, Identity Management; CCNA: IP Services; ITIL 4: Service Config Mgmt |

---

### CHG-007
| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| **Change ID**    | CHG-007                                                        |
| **Date**         | 2026-04-04                                                     |
| **Type**         | Normal                                                         |
| **Title**        | Deploy pfSense FW-01 with zone-based firewall rules and NAT    |
| **Description**  | Install pfSense CE 2.7.x. Assign WAN (DHCP) and LAN (10.0.0.1/24). Configure LAN rules: Pass Staff, Pass Servers, Block Guest RFC1918, Pass Guest HTTP/HTTPS, Pass MGMT SSH, Implicit deny. Configure Hybrid Outbound NAT for Staff/Servers/Guest (exclude MGMT). Enable syslog forwarding to 10.10.20.10:514. |
| **Impact**       | High — all internet traffic now routes through pfSense; incorrect rules could block all VLANs |
| **Risk**         | Medium — test each rule in sequence before enabling implicit deny |
| **Pre-change**   | Verify R1-CORE default route points to pfSense (10.0.0.1); test ping to pfSense from R1-CORE |
| **Post-change**  | Test each VLAN's internet access; test Guest isolation; verify SIEM receives firewall logs |
| **Rollback**     | Temporarily disable firewall rules to allow all; revert to previous default gateway if needed |
| **Status**       | Completed                                                      |
| **Verification** | See `pfsense/firewall-rules.md` verification section           |
| **Cert Mapping** | Security+: Network Architecture — Firewalls, NAT; CCNA: Security Fundamentals; ITIL 4: Service Design |

---

### CHG-008
| Field            | Value                                                          |
|------------------|----------------------------------------------------------------|
| **Change ID**    | CHG-008                                                        |
| **Date**         | 2026-04-04                                                     |
| **Type**         | Standard                                                       |
| **Title**        | Configure SNMPv3 on all Cisco devices (replace SNMPv2c)        |
| **Description**  | On R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2: create SNMP group `MONITORING` v3 priv; create user `wazuh-monitor` with AuthSHA (Auth@2026) and PrivAES128 (Priv@2026); set trap destination to 10.10.20.10 (Wazuh/Prometheus host). Remove any SNMPv2c community strings. |
| **Impact**       | Low — SNMP monitoring change; network traffic unaffected       |
| **Risk**         | Low — SNMPv3 is additive; community strings removed after v3 confirmed working |
| **Rollback**     | Re-add community strings; remove SNMPv3 user                  |
| **Status**       | Completed                                                      |
| **Verification** | `show snmp user` — wazuh-monitor present with auth SHA, priv AES128; SNMP walk from monitoring host succeeds |
| **Cert Mapping** | CCNA: Network Management — SNMPv3; Security+: Secure Protocols — encryption in transit |

---

## ITIL 4 Practice Analysis

### Change Enablement Practice

Every change in this project follows the ITIL 4 Change Enablement practice, which defines:

- **Scope:** All changes to production configurations, whether additive or modifying, require a change record
- **Purpose:** Maximize successful changes by ensuring risks are assessed, appropriate authorization obtained, and rollback procedures defined
- **Change Models:** Standard and Normal changes use different authorization flows matching their risk profile

**What this demonstrates for interviews:**

1. **Standard changes are pre-approved by design** — VLANs and SNMPv3 (CHG-001, CHG-008) follow documented procedures with known outcomes. They don't need CAB review because the risk is understood.

2. **Normal changes require risk assessment** — Routing (CHG-003, CHG-004), ACLs (CHG-005), and the firewall (CHG-007) have impact ratings of Medium-High. Each has a documented pre-change verification step and rollback procedure.

3. **High-impact changes are sequenced** — CHG-007 (pfSense deployment) is last because it depends on all previous changes being stable. This mirrors how a real CAB would sequence a production change window.

### Service Configuration Management Practice

This Git repository functions as the CMDB (Configuration Management Database) for this lab:

- Every device configuration is version-controlled
- Each commit represents a verified, tested configuration state
- Commit messages map to change IDs, creating traceability: `git log` shows the full change history
- Configuration drift can be detected by comparing `show running-config` output to the committed baseline

**ITIL 4 CI (Configuration Item) mapping:**

| CI Name      | CI Type         | File                        | Relationships                        |
|--------------|-----------------|-----------------------------|--------------------------------------|
| R1-CORE      | Network Device  | configs/R1-CORE.txt         | Connected to pfSense FW-01, SW-DIST  |
| SW-DIST      | Network Device  | configs/SW-DIST.txt         | Connected to R1-CORE, SW-ACC-1, SW-ACC-2 |
| SW-ACC-1     | Network Device  | configs/SW-ACC-1.txt        | Connected to SW-DIST                 |
| SW-ACC-2     | Network Device  | configs/SW-ACC-2.txt        | Connected to SW-DIST                 |
| FW-01        | Security Device | pfsense/firewall-rules.md   | Connected to WAN, R1-CORE            |

---

## CAB Process (Simulated)

For Normal changes in this lab, the following CAB process was followed:

1. **Change request raised** — Change record created with description, impact, and risk rating
2. **Technical review** — Configuration reviewed for correctness before implementation
3. **Rollback documented** — Specific rollback commands listed before changes applied
4. **Change window defined** — All changes applied during a defined maintenance window (lab session)
5. **Post-implementation review** — Verification commands run; results documented in `verification/test-results.md`
6. **Change closed** — Status updated to Completed; configuration committed to Git repository
