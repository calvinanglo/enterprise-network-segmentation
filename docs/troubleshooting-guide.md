# Troubleshooting Guide — Project 1: Enterprise Network Segmentation

**Environment:** GNS3/EVE-NG  
**Devices:** R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2, pfSense FW-01

Each issue below follows the **OSI model top-down or bottom-up method** — a structured troubleshooting approach required for CCNA and used in production NOC environments.

---

## Issue 1: OSPF Neighbors Not Forming (Stuck in INIT or EXSTART)

### Symptoms
```
R1-CORE# show ip ospf neighbor
(empty or shows INIT/EXSTART state)
```

### Diagnosis Steps

**Step 1 — Verify Layer 1/2 (Physical and Data Link)**
```
R1-CORE# show interfaces GigabitEthernet0/1
  ! Check: line protocol is up — if down, physical issue
  SW-DIST# show interfaces GigabitEthernet0/1
  ```

  **Step 2 — Verify IP connectivity on the OSPF link**
  ```
  R1-CORE# ping 10.0.1.2
  SW-DIST# ping 10.0.1.1
  ! Both must succeed — if fail, check IP addresses and subnet mask
  ```

  **Step 3 — Verify OSPF is enabled on the link**
  ```
  R1-CORE# show ip ospf interface GigabitEthernet0/1
    ! Check: OSPF enabled, area 0, hello/dead timers
    SW-DIST# show ip ospf interface GigabitEthernet0/1
      ! Timers must match: Hello 10, Dead 40 (defaults)
      ```

      **Step 4 — Check for mismatched OSPF parameters**
      ```
      ! Verify same area number (area 0)
      R1-CORE# show run | section router ospf
      SW-DIST# show run | section router ospf

      ! Verify network statements cover the 10.0.1.0/30 link
      ! R1-CORE must have: network 10.0.1.0 0.0.0.3 area 0
      ! SW-DIST must have: network 10.0.1.0 0.0.0.3 area 0
      ```

      **Step 5 — Check MTU mismatch (common in GNS3)**
      ```
      R1-CORE# show interfaces GigabitEthernet0/1 | include MTU
      SW-DIST# show interfaces GigabitEthernet0/1 | include MTU
      ! MTU must match on both ends (default 1500)
      ! Fix: interface GigabitEthernet0/1 -> ip ospf mtu-ignore (workaround)
      ```

      ### Resolution
      - MTU mismatch: add `ip ospf mtu-ignore` on the interface
      - Timer mismatch: standardize `ip ospf hello-interval` and `ip ospf dead-interval`
      - Area mismatch: correct the `network` statement area number

      ### Root Cause Documentation (for ITIL Incident Record)
      ```
      Category: Network — Routing Protocol
      Priority: High (loss of dynamic routing)
      Resolution: [describe fix applied]
      KEDB Update: Yes/No
      ```

      ---

      ## Issue 2: VLAN Traffic Not Passing (Hosts Cannot Reach Gateway)

      ### Symptoms
      ```
      STAFF-PC> ping 10.10.10.1
      ! Timeout — cannot reach default gateway
      ```

      ### Diagnosis Steps

      **Step 1 — Verify host VLAN assignment**
      ```
      SW-ACC-1# show vlan brief
      ! Confirm port Gi0/1 is assigned to VLAN 10
      ! If showing VLAN 1 (default), port was never configured

      SW-ACC-1# show interfaces GigabitEthernet0/1 switchport
      ! Verify: Access Mode VLAN = 10
      ```

      **Step 2 — Verify trunk is passing VLAN 10**
      ```
      SW-DIST# show interfaces trunk
      ! Confirm Gi0/2 trunk shows VLAN 10 in "VLANs allowed and active in management domain"
      ! If VLAN 10 missing: check "switchport trunk allowed vlan" on both ends
      ```

      **Step 3 — Verify SVI is up/up on SW-DIST**
      ```
      SW-DIST# show interfaces Vlan10
      ! Must show: Vlan10 is up, line protocol is up
      ! If down/down: no active ports in VLAN 10 OR VLAN not in database

      SW-DIST# show ip interface brief | include Vlan10
      ! Verify: 10.10.10.1 assigned, status Up
      ```

      **Step 4 — Verify VLAN exists in database on SW-DIST**
      ```
      SW-DIST# show vlan brief
      ! VLAN 10 must be present and active
      ! If missing: vlan 10 / name STAFF
      ```

      ### Resolution
      - VLAN missing from trunk: `switchport trunk allowed vlan add 10`
      - SVI down because no active ports: ensure at least one access port is up in the VLAN
      - Host wrong VLAN: `switchport access vlan 10` on the access port

      ---

      ## Issue 3: Guest Can Reach Internal Servers (ACL Not Working)

      ### Symptoms
      ```
      GUEST-PC> ping 10.10.20.10
      ! Ping succeeds — should be BLOCKED
      ```

      ### Diagnosis Steps

      **Step 1 — Verify ACL exists**
      ```
      SW-DIST# show ip access-lists GUEST-RESTRICT
      ! If empty or missing, ACL was never created
      ```

      **Step 2 — Verify ACL is applied to the correct interface/direction**
      ```
      SW-DIST# show ip interface Vlan30
      ! Look for: Inbound access list is GUEST-RESTRICT
      ! If shows "not set": ACL exists but was never applied
      ! Fix: interface Vlan30 -> ip access-group GUEST-RESTRICT in
      ```

      **Step 3 — Verify ACL direction (CRITICAL)**
      ```
      ! ACL must be INBOUND on Vlan30
      ! If applied OUTBOUND, it filters traffic LEAVING Vlan30 (wrong direction)
      ! Inbound = filters traffic arriving from Guest devices
      ```

      **Step 4 — Check ACL hit counters**
      ```
      SW-DIST# show ip access-lists GUEST-RESTRICT
      ! Matches column should increment when Guest pings server
      ! If 0 matches: traffic isn't hitting the ACL (wrong interface or direction)
      ! If matches but still passing: check permit statement order
      ```

      **Step 5 — Verify permit statement is not too broad**
      ```
      ! Correct:
        deny ip 10.10.30.0 0.0.0.255 10.10.20.0 0.0.0.255 log
          permit ip 10.10.30.0 0.0.0.255 any

          ! WRONG (would allow everything):
            permit ip any any     <- never use this on a security ACL
            ```

            ### Resolution
            - ACL not applied: `ip access-group GUEST-RESTRICT in` on Vlan30
            - ACL applied outbound: remove and re-apply inbound
            - ACL rule order wrong: resequence with `ip access-list resequence`

            ### Security+ Exam Note
            This scenario tests Security+ domain: Network Architecture — Access Control Lists. The key concepts are: ACL direction (inbound vs outbound), implicit deny at end of ACL, and ACL placement (as close to source as possible).

            ---

            ## Issue 4: SSH Connection Refused to Cisco Device

            ### Symptoms
            ```
            $ ssh admin@10.10.99.11
            ! ssh: connect to host 10.10.99.11 port 22: Connection refused
            ```

            ### Diagnosis Steps

            **Step 1 — Verify SSH is enabled**
            ```
            SW-ACC-1# show ip ssh
            ! Must show: SSH Enabled - version 2.0
            ! If "SSH disabled": RSA key was never generated or was deleted
            ! Fix: crypto key generate rsa modulus 2048
            ```

            **Step 2 — Verify VTY transport**
            ```
            SW-ACC-1# show run | section line vty
            ! Must show: transport input ssh
            ! If shows: transport input none — SSH is explicitly blocked
            ! If shows: transport input telnet — SSH disabled in favor of Telnet
            ```

            **Step 3 — Verify IP domain-name is set (required for RSA key)**
            ```
            SW-ACC-1# show run | include domain-name
            ! Must show: ip domain-name lab.internal
            ! Without this, crypto key generate will fail
            ```

            **Step 4 — Verify local user account exists**
            ```
            SW-ACC-1# show run | include username
            ! Must show: username admin privilege 15 algorithm-type scrypt secret ...
            ! Without a local user, SSH authentication fails even if SSH is enabled
            ```

            **Step 5 — Verify VTY login method**
            ```
            SW-ACC-1# show run | section line vty
            ! Must show: login local (use local database)
            ! If shows: login (no local) — uses line password, not username
            ```

            **Step 6 — Test connectivity to management IP**
            ```
            ! From management workstation:
            ping 10.10.99.11
            ! If fails: check MGMT VLAN SVI, default gateway on switch
            ```

            ### Resolution
            - No RSA key: `crypto key generate rsa modulus 2048`
            - Transport set to none/telnet: `transport input ssh` on VTY lines
            - No domain name: `ip domain-name lab.internal` then regenerate keys

            ---

            ## Issue 5: Internet Not Working from Any VLAN (Default Route Missing)

            ### Symptoms
            ```
            STAFF-PC> ping 8.8.8.8
            ! Timeout from all VLANs
            ```

            ### Diagnosis Steps

            **Step 1 — Check routing table on R1-CORE**
            ```
            R1-CORE# show ip route 0.0.0.0
            ! Must show: S* 0.0.0.0/0 [1/0] via 10.0.0.1 (static to pfSense)
            ! If missing: ip route 0.0.0.0 0.0.0.0 10.0.0.1
            ```

            **Step 2 — Check default route is redistributed into OSPF**
            ```
            R1-CORE# show run | section router ospf
            ! Must include: default-information originate
            SW-DIST# show ip route 0.0.0.0
            ! SW-DIST must have O*E2 0.0.0.0/0 via 10.0.1.1 (learned via OSPF)
            ```

            **Step 3 — Verify pfSense is reachable from R1-CORE**
            ```
            R1-CORE# ping 10.0.0.1
            ! If fails: check WAN interface on pfSense, IP addressing on 10.0.0.0/24
            ```

            **Step 4 — Verify pfSense NAT is working**
            ```
            ! Log into pfSense GUI > Diagnostics > Ping
            ! Source: LAN, Destination: 8.8.8.8
            ! If this works but hosts cannot reach internet: NAT rule issue
            ```

            **Step 5 — Check VLAN traffic is reaching pfSense**
            ```
            ! On pfSense: Status > System Logs > Firewall
            ! Filter for traffic from 10.10.10.x
            ! If not appearing: routing issue between VLAN and pfSense
            ```

            ### Resolution
            - Missing default route on R1-CORE: add static route to 10.0.0.1
            - Default route not in OSPF: add `default-information originate` to R1-CORE OSPF
            - pfSense NAT broken: verify Outbound NAT rules cover all VLAN subnets

            ---

            ## Issue 6: Port Security Violation — Access Port Goes to err-disabled

            ### Symptoms
            ```
            SW-ACC-1# show interfaces GigabitEthernet0/5
            ! GigabitEthernet0/5 is err-disabled
            ```

            ### Diagnosis Steps

            **Step 1 — Identify violation reason**
            ```
            SW-ACC-1# show port-security interface GigabitEthernet0/5
            ! Look for: Last Source Address / VLAN, Security Violation Count
            ! Violation mode: Shutdown means any violation err-disables the port
            ```

            **Step 2 — Check what triggered the violation**
            ```
            SW-ACC-1# show port-security address
            ! Shows all learned/sticky MACs
            ! Compare against the MAC of the device currently connected
            ! If different MAC: either device changed or unauthorized device connected
            ```

            **Step 3 — Check logs for the event**
            ```
            SW-ACC-1# show log | include SECURITY
            ! Will show: %PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred
            ! Includes the offending MAC address and port
            ```

            ### Resolution — Clearing err-disabled Port

            **If the violation was legitimate (authorized device):**
            ```
            SW-ACC-1# configure terminal
            SW-ACC-1(config)# interface GigabitEthernet0/5
            SW-ACC-1(config-if)# shutdown
            SW-ACC-1(config-if)# no shutdown
            ! This clears err-disabled state
            ! Update sticky MAC if device legitimately changed:
            SW-ACC-1(config-if)# no switchport port-security mac-address sticky
            SW-ACC-1(config-if)# switchport port-security mac-address sticky
            ```

            **If the violation was a security event (unauthorized device):**
            ```
            ! Document as security incident
            ! Report to SIEM/Wazuh
            ! Investigate physical access to the port
            ! Consider changing violation mode to "restrict" for softer enforcement
            ```

            ### ITIL Incident Note
            ```
            Category: Security — Unauthorized Access Attempt
            Priority: Medium (single port affected)
            Impact: Low (one user)
            Actions: Clear port, investigate, document
            ```

            ---

            ## Issue 7: Spanning Tree — Wrong Root Bridge

            ### Symptoms
            ```
            SW-ACC-1# show spanning-tree vlan 10
            ! Root ID: Address = [SW-ACC-1 MAC] <- WRONG
            ! This switch should not be root
            ```

            ### Diagnosis Steps

            **Step 1 — Identify current STP root**
            ```
            SW-DIST# show spanning-tree vlan 10
            ! Should show: This bridge is the root
            ! Priority should be 4096 (lower = preferred root)
            ```

            **Step 2 — Check priorities on all switches**
            ```
            SW-DIST# show spanning-tree vlan 10 | include Priority
            ! SW-DIST: 4096 (correct)
            SW-ACC-1# show spanning-tree vlan 10 | include Priority
            ! SW-ACC-1: 32768 (default — should be higher than SW-DIST)
            ```

            **Step 3 — Verify Root Guard on uplinks**
            ```
            SW-ACC-1# show spanning-tree interface GigabitEthernet0/24 detail
            ! Should show: Root guard is enabled on the port
            ! If disabled: a BPDU from SW-ACC-1 could claim root
            ```

            ### Resolution
            ```
            SW-DIST(config)# spanning-tree vlan 10 priority 4096
            SW-DIST(config)# spanning-tree vlan 20 priority 4096
            SW-DIST(config)# spanning-tree vlan 30 priority 4096
            SW-DIST(config)# spanning-tree vlan 99 priority 4096
            ! Or use: spanning-tree vlan 10,20,30,99 root primary
            ```

            ---

            ## Quick Reference — Key Show Commands

            ### Routing
            ```
            show ip route                        # Full routing table
            show ip route ospf                   # Only OSPF routes
            show ip route 0.0.0.0               # Default route
            show ip ospf neighbor               # OSPF neighbor states
            show ip ospf interface               # OSPF interface details
            ```

            ### Switching
            ```
            show vlan brief                      # All VLANs and port assignments
            show interfaces trunk                # All trunk ports and allowed VLANs
            show spanning-tree vlan 10          # STP topology for VLAN 10
            show port-security interface gi0/1  # Port security on specific port
            show port-security address          # All learned/sticky MACs
            ```

            ### Security
            ```
            show ip access-lists                 # All ACLs with hit counters
            show ip interface Vlan30             # Shows applied ACL on interface
            show ip ssh                          # SSH status and version
            show ssh                             # Active SSH sessions
            show users                           # All connected users
            ```

            ### Management
            ```
            show snmp user                       # SNMPv3 users
            show snmp group                      # SNMP groups
            show ntp status                      # NTP sync status
            show logging                         # Syslog configuration and buffer
            show clock                           # Current time (check NTP sync)
            ```

            ### Error Recovery
            ```
            show interfaces status               # All ports: connected/notconnect/err-disabled
            show log | include ERR               # Error log entries
            show log | include SECURITY         # Security events
            show cdp neighbors                   # Layer 2 topology (if CDP enabled)
            ```

            ---

            ## Structured Troubleshooting Methodology

            This guide uses the **OSI Layer Approach** — standard for CCNA troubleshooting and used in real production NOC environments:

            **Bottom-Up (Physical to Application):**
            1. **Layer 1** — Cable connected? Interface up? `show interfaces`
            2. **Layer 2** — VLAN assigned? Trunk passing VLAN? `show vlan brief`, `show interfaces trunk`
            3. **Layer 3** — IP address correct? Gateway reachable? Routing table populated? `show ip route`
            4. **Layer 4+** — ACL blocking? Firewall rule? Port security violation?

            **Top-Down (Application to Physical):**
            Start with the symptom (e.g., "can't reach server"), verify the service, then work down to identify where the failure is.

            **Divide and Conquer:**
            Test connectivity at the midpoint of the path first to halve the troubleshooting space. For a host-to-server failure: ping the default gateway first. If that works, move to the next hop.

            ### ITIL Incident Management Mapping

            | Troubleshooting Step          | ITIL Practice         | Action                                    |
            |-------------------------------|-----------------------|-------------------------------------------|
            | Identify symptoms             | Incident Management   | Log incident with user-reported symptoms  |
            | Diagnose root cause           | Problem Management    | Use structured methodology (OSI layers)   |
            | Apply fix                     | Change Enablement     | Emergency or Standard change record       |
            | Verify resolution             | Incident Management   | Confirm with user, close incident         |
            | Document for future           | Knowledge Management  | Update KEDB (Known Error Database)        |
