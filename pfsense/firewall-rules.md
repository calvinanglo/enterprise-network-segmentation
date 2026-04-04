# pfSense Firewall Rules

pfSense CE 2.7.x running as the perimeter firewall. WAN gets DHCP from the hypervisor, LAN is 10.0.0.1/24 connecting to R1-CORE Gi0/0 at 10.0.0.2.

## Interfaces

- em0 = WAN (DHCP from hypervisor)
- - em1 = LAN (10.0.0.1/24, connects to R1-CORE)
 
  - ## LAN rules
 
  - Rules are top-down, first match wins. Everything is logged.
 
  - | # | Action | Source | Destination | Port | Notes |
  - |---|--------|--------|-------------|------|-------|
  - | 1 | Pass | 10.10.10.0/24 | any | any | Staff, full access |
  - | 2 | Pass | 10.10.20.0/24 | any | any | Servers, full access |
  - | 3 | Block | 10.10.30.0/24 | 10.0.0.0/8 | any | Guest blocked from all internal RFC1918 |
  - | 4 | Pass | 10.10.30.0/24 | any | 80, 443 | Guest gets HTTP/HTTPS only |
  - | 5 | Pass | 10.10.99.0/24 | any | 22 | MGMT can SSH out |
  - | 6 | Block | any | any | any | Implicit deny, log everything |
 
  - Rule 3 blocks 10.0.0.0/8 rather than just our specific subnets so any future VLANs I add are automatically covered without touching this rule.
 
  - Rule 4 comes after rule 3 so guest traffic to internal IPs hits the block first. Guest devices can only reach the public internet on web ports.
 
  - MGMT intentionally has no internet NAT rule. If a management device is trying to reach the internet that's worth investigating, not facilitating.
 
  - ## Outbound NAT
 
  - Using Hybrid mode so I can control which sources get NATted and which don't.
 
  - | Source | Interface | Action |
  - |--------|-----------|--------|
  - | 10.10.10.0/24 | WAN | NAT |
  - | 10.10.20.0/24 | WAN | NAT |
  - | 10.10.30.0/24 | WAN | NAT |
  - | 10.10.99.0/24 | none | no NAT |
 
  - ## Setup steps
 
  - 1. Install pfSense CE 2.7.x, assign em0 to WAN and em1 to LAN
    2. 2. Run the setup wizard, set LAN IP to 10.0.0.1/24
       3. 3. WAN: enable Block RFC1918 and Block bogons
          4. 4. Add the LAN rules above in order under Firewall > Rules > LAN
             5. 5. Switch NAT to Hybrid mode under Firewall > NAT > Outbound, add the three NAT rules
                6. 6. Enable remote syslog under Status > System Logs > Settings, point to 10.10.20.10:514
                   7. 7. Enable SNMP under Services > SNMP, bind to LAN only
                     
                      8. Note: pfSense CE doesn't support SNMPv3 natively. Using a strong community string and restricting SNMP to the LAN interface as a workaround. If this were production I'd look at the pfSense API or upgrade to Plus.
                     
                      9. ## Verification
                     
                      10. ```bash
                          # from a staff workstation
                          ping 8.8.8.8          # should work
                          ping 10.10.20.10      # should work

                          # from a guest device
                          curl https://example.com    # should work
                          ping 10.10.10.100           # should fail, blocked by rule 3
                          ping 10.10.20.10            # should fail
                          ssh admin@10.10.99.11       # should fail

                          # from mgmt
                          ssh admin@10.10.10.100      # should work, rule 5
                          curl https://example.com    # should fail, no NAT for mgmt
                          ```

