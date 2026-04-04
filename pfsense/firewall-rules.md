# pfSense Firewall Rules — FW-01
## Project 1: Enterprise Network Segmentation

**Device:** pfSense CE 2.7.x  
**WAN:** DHCP from hypervisor/ISP  
**LAN:** 10.0.0.1/24 (connects to R1-CORE Gi0/0)  
**Role:** Perimeter firewall, NAT, zone-based traffic enforcement

---

## Interface Assignment

| Interface | pfSense Name | IP Address      | Connected To          |
|-----------|--------------|------------------|-----------------------|
| em0       | WAN          | DHCP             | Hypervisor / ISP      |
| em1       | LAN          | 10.0.0.1/24      | R1-CORE Gi0/0 (10.0.0.2) |

---

## LAN Firewall Rules

Rules are evaluated top-to-bottom. First match wins. All rules logged.

| Priority | Action | Protocol | Source            | Destination       | Port     | Description                          |
|----------|--------|----------|-------------------|-------------------|----------|--------------------------------------|
| 1        | Pass   | Any      | 10.10.10.0/24     | Any               | Any      | STAFF — full internet and internal   |
| 2        | Pass   | Any      | 10.10.20.0/24     | Any               | Any      | SERVERS — full access                |
| 3        | Block  | Any      | 10.10.30.0/24     | 10.0.0.0/8        | Any      | GUEST — block all RFC1918 internal   |
| 4        | Pass   | TCP      | 10.10.30.0/24     | Any               | 80, 443  | GUEST — HTTP/HTTPS internet only     |
| 5        | Pass   | TCP      | 10.10.99.0/24     | Any               | 22       | MGMT — SSH to management targets     |
| 6        | Block  | Any      | Any               | Any               | Any      | Implicit deny — log all blocked      |

### Rule Design Rationale

**Rule 3 — Guest RFC1918 Block:**  
Blocks Guest from reaching any internal address space (10.x.x.x, 172.16.x.x, 192.168.x.x).  
This is the perimeter layer of Guest isolation. The distribution switch ACL (GUEST-RESTRICT) handles Layer 3 enforcement closer to the source — pfSense adds a second enforcement point at the WAN edge.  
*Maps to: Security+ — Network Architecture, Defense-in-Depth; ISC2 CC — Network Security Controls*

**Rule 4 — Guest Internet Only:**  
After blocking RFC1918, this rule explicitly permits Guest traffic to reach only TCP 80/443.  
Guest devices cannot use non-web protocols (FTP, SMTP, SNMP, etc.) — limits the blast radius if a guest device is compromised.  
*Maps to: Security+ — Least Privilege; CCNA — ACL Design*

**Rule 5 — Management SSH Only:**  
MGMT VLAN (10.10.99.0/24) is restricted to SSH outbound. Management devices should not be initiating arbitrary internet connections.  
*Maps to: Security+ — Hardening; ITIL 4 — Service Configuration Management*

**Rule 6 — Implicit Deny with Logging:**  
All unmatched traffic is blocked and logged. Every blocked connection generates a log entry forwarded to Wazuh SIEM.  
*Maps to: Security+ — Log Management; ITIL 4 — Event Management*

---

## Outbound NAT Rules (Hybrid Mode)

Hybrid NAT mode: pfSense applies manual rules first, then auto-rules for anything not explicitly handled.

| Rule | Source Network  | Outbound Interface | Action | Description                        |
|------|-----------------|--------------------|--------|------------------------------------|
| 1    | 10.10.10.0/24   | WAN                | NAT    | Staff internet access              |
| 2    | 10.10.20.0/24   | WAN                | NAT    | Server internet (updates, OSINT)   |
| 3    | 10.10.30.0/24   | WAN                | NAT    | Guest internet only                |
| 4    | 10.10.99.0/24   | —                  | No NAT | MGMT stays internal — no internet  |

### NAT Design Rationale

**Why Hybrid NAT:**  
Automatic NAT would NAT all interfaces including MGMT. Hybrid mode lets us explicitly exclude MGMT (10.10.99.0/24) from internet access, ensuring management traffic never exits the perimeter.

**Why MGMT has No NAT:**  
The management plane should never initiate outbound internet connections. If a management device is calling home, that's a security event — not normal behaviour.  
*Maps to: Security+ — Zero Trust Principles; ITIL 4 — Service Configuration Management*

---

## pfSense Setup Steps

### Step 1 — Initial Installation
```
1. Download pfSense CE 2.7.x ISO from pfsense.org
2. Deploy as VM in GNS3/EVE-NG or on bare metal
3. Assign interfaces during boot:
   - em0 = WAN (connected to hypervisor NAT adapter)
      - em1 = LAN (connected to R1-CORE Gi0/0)
      4. Access web GUI at https://192.168.1.1 (default)
      5. Run Setup Wizard — set LAN IP to 10.0.0.1/24
      ```

      ### Step 2 — Interface Configuration
      ```
      System > Interfaces > WAN
        - Type: DHCP
          - Block RFC1918 networks: Enabled (blocks private IPs from WAN — spoofing protection)
            - Block bogon networks: Enabled

            System > Interfaces > LAN
              - Type: Static
                - IP: 10.0.0.1
                  - Subnet: 24
                  ```

                  ### Step 3 — LAN Firewall Rules
                  ```
                  Firewall > Rules > LAN

                  Add rules in order (top-to-bottom evaluation):

                  Rule 1: Staff Pass
                    Action: Pass | Protocol: Any
                      Source: Network 10.10.10.0/24
                        Destination: Any
                          Description: STAFF-FULL-ACCESS
                            Logging: Enabled

                            Rule 2: Servers Pass
                              Action: Pass | Protocol: Any
                                Source: Network 10.10.20.0/24
                                  Destination: Any
                                    Description: SERVERS-FULL-ACCESS
                                      Logging: Enabled

                                      Rule 3: Guest Block RFC1918
                                        Action: Block | Protocol: Any
                                          Source: Network 10.10.30.0/24
                                            Destination: Network 10.0.0.0/8
                                              Description: GUEST-BLOCK-INTERNAL-RFC1918
                                                Logging: Enabled (every block generates a SIEM event)

                                                Rule 4: Guest Internet Only
                                                  Action: Pass | Protocol: TCP
                                                    Source: Network 10.10.30.0/24
                                                      Destination: Any
                                                        Destination Port: 80, 443
                                                          Description: GUEST-INTERNET-HTTP-HTTPS-ONLY
                                                            Logging: Enabled

                                                            Rule 5: MGMT SSH Only
                                                              Action: Pass | Protocol: TCP
                                                                Source: Network 10.10.99.0/24
                                                                  Destination: Any
                                                                    Destination Port: 22
                                                                      Description: MGMT-SSH-ONLY
                                                                        Logging: Enabled

                                                                        Rule 6: Implicit Deny (pfSense adds this automatically — ensure logging ON)
                                                                          Firewall > Settings > Advanced > Log packets blocked by default rule: Enabled
                                                                          ```

                                                                          ### Step 4 — Outbound NAT (Hybrid Mode)
                                                                          ```
                                                                          Firewall > NAT > Outbound
                                                                            Mode: Hybrid Outbound NAT

                                                                            Add manual rules:

                                                                            Rule 1: Staff NAT
                                                                              Interface: WAN
                                                                                Source: 10.10.10.0/24
                                                                                  Translation: Interface Address
                                                                                    Description: NAT-STAFF-INTERNET

                                                                                    Rule 2: Servers NAT
                                                                                      Interface: WAN
                                                                                        Source: 10.10.20.0/24
                                                                                          Translation: Interface Address
                                                                                            Description: NAT-SERVERS-INTERNET

                                                                                            Rule 3: Guest NAT
                                                                                              Interface: WAN
                                                                                                Source: 10.10.30.0/24
                                                                                                  Translation: Interface Address
                                                                                                    Description: NAT-GUEST-INTERNET
                                                                                                    
                                                                                                    (Do NOT add a rule for 10.10.99.0/24 — MGMT gets no internet)
                                                                                                    ```
                                                                                                    
                                                                                                    ### Step 5 — Syslog Forwarding to Wazuh SIEM
                                                                                                    ```
                                                                                                    Status > System Logs > Settings
                                                                                                      Remote Logging:
                                                                                                          Enable Remote Logging: Checked
                                                                                                              Source Address: LAN (10.0.0.1)
                                                                                                                  Remote Syslog Server: 10.10.20.10:514
                                                                                                                      Remote Syslog Contents: Everything
                                                                                                                      ```
                                                                                                                      
                                                                                                                      ### Step 6 — SNMP for Prometheus Monitoring
                                                                                                                      ```
                                                                                                                      Services > SNMP
                                                                                                                        Enable: Checked
                                                                                                                          SNMP v2c (pfSense CE limitation — document this)
                                                                                                                            Community: <STRONG-COMMUNITY-STRING>
                                                                                                                              Bind interfaces: LAN
                                                                                                                                
                                                                                                                                Note: pfSense CE does not support SNMPv3 natively. 
                                                                                                                                Workaround: restrict SNMP to MGMT VLAN and use strong community string.
                                                                                                                                For production: consider pfSense Plus or use the pfSense API for metrics.
                                                                                                                                ```
                                                                                                                                
                                                                                                                                ---
                                                                                                                                
                                                                                                                                ## Verification Commands
                                                                                                                                
                                                                                                                                ```bash
                                                                                                                                # From a STAFF workstation (10.10.10.x) — should SUCCEED
                                                                                                                                ping 8.8.8.8
                                                                                                                                ping 10.10.20.10
                                                                                                                                curl http://example.com
                                                                                                                                
                                                                                                                                # From a GUEST device (10.10.30.x) — internet should work
                                                                                                                                curl http://example.com     # PASS
                                                                                                                                curl https://example.com    # PASS
                                                                                                                                
                                                                                                                                # From a GUEST device — internal should FAIL
                                                                                                                                ping 10.10.10.100           # FAIL — blocked by Rule 3
                                                                                                                                ping 10.10.20.10            # FAIL — blocked by Rule 3
                                                                                                                                ssh admin@10.10.99.11       # FAIL — blocked by Rule 3
                                                                                                                                
                                                                                                                                # From MGMT (10.10.99.x) — SSH should work, internet should not
                                                                                                                                ssh admin@10.10.10.100      # PASS — Rule 5
                                                                                                                                curl http://example.com     # FAIL — no NAT rule for MGMT
                                                                                                                                ```
                                                                                                                                
                                                                                                                                ---
                                                                                                                                
                                                                                                                                ## Security Design Notes
                                                                                                                                
                                                                                                                                ### Defense-in-Depth — Three Enforcement Layers
                                                                                                                                
                                                                                                                                This topology enforces Guest isolation at three separate layers, as required by Security+ Network Architecture domain:
                                                                                                                                
                                                                                                                                1. **Layer 2 — SW-ACC-2:** Guest devices land on VLAN 30. Port security limits one MAC per port. No access to other VLANs at Layer 2.
                                                                                                                                2. **Layer 3 — SW-DIST ACL:** `GUEST-RESTRICT` ACL on Vlan30 SVI denies Guest traffic to Server, Staff, and MGMT VLANs before it even reaches the router.
                                                                                                                                3. **Perimeter — pfSense Rule 3:** Even if the ACL were misconfigured, pfSense blocks all RFC1918 traffic from Guest at the firewall level.
                                                                                                                                
                                                                                                                                An attacker would need to defeat all three layers simultaneously — this is defense-in-depth in practice.
                                                                                                                                
                                                                                                                                ### Why Block RFC1918 Specifically (Not Just Internal Subnets)
                                                                                                                                
                                                                                                                                Blocking `10.0.0.0/8` covers all future internal subnets, not just today's VLANs. This future-proofs the rule — adding a new VLAN doesn't require a firewall change.
                                                                                                                                
                                                                                                                                ### Cert Mapping
                                                                                                                                
                                                                                                                                | Rule / Feature              | CCNA                     | Security+                        | ITIL 4                         |
                                                                                                                                |-----------------------------|--------------------------|----------------------------------|--------------------------------|
                                                                                                                                | Zone-based firewall rules   | Security Fundamentals    | Network Architecture             | Service Design                 |
                                                                                                                                | Implicit deny               | ACL design               | Defense-in-Depth                 | Risk Management                |
                                                                                                                                | NAT / PAT                   | IP Services              | Network Architecture             | —                              |
                                                                                                                                | Syslog to SIEM              | Network Management       | Log Management                   | Event Management               |
                                                                                                                                | Guest isolation (3 layers)  | VLANs + ACLs             | Segmentation + Defense-in-Depth  | Risk Management                |
