# pfSense Firewall Rules

pfSense CE 2.7.x running as the perimeter firewall. WAN gets DHCP from the hypervisor, LAN is 10.0.0.1/24 connecting to R1-CORE Gi0/0 at 10.0.0.2.

## Interfaces

- em0 = WAN (DHCP from hypervisor)
- em1 = LAN (10.0.0.1/24, connects to R1-CORE)

## LAN Rules

Rules are top-down, first match wins. Everything is logged.

| # | Action | Source | Destination | Port | Notes |
|---|--------|--------|-------------|------|-------|
| 1 | Pass | 10.10.10.0/24 | any | any | Staff, full access |
| 2 | Pass | 10.10.20.0/24 | any | any | Servers, full access |
| 3 | Block | 10.10.30.0/24 | 10.0.0.0/8 | any | Guest blocked from all internal RFC1918 |
| 4 | Pass | 10.10.30.0/24 | any | 80, 443 | Guest gets HTTP/HTTPS only |
| 5 | Pass | 10.10.99.0/24 | any | 22 | MGMT can SSH out |
| 6 | Block | any | any | any | Implicit deny, log everything |

Rule 3 blocks 10.0.0.0/8 rather than just specific subnets so any future VLANs added to the 10.x space are denied by default until explicitly permitted. Easier to maintain than keeping a list of per-subnet rules.

## WAN Rules

Default pfSense WAN rules block all inbound that's not related to established outbound. No additional inbound rules - this lab doesn't expose anything to the internet.

## Floating Rules

None currently. Using per-interface rules keeps things more readable and debuggable.

## Aliases

- RFC1918 = 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- INTERNAL_VLANS = 10.10.10.0/24, 10.10.20.0/24, 10.10.30.0/24, 10.10.99.0/24
- MANAGEMENT = 10.10.99.0/24

Using aliases keeps the rule table readable. When a new server VLAN gets added, you update the alias once and all rules that reference it update automatically.

## Logging

All rules have logging enabled. Logs go to /var/log/filter.log and are also forwarded via syslog to Wazuh on 10.10.20.10:514. The block rules in particular are useful for catching misconfigurations and lateral movement attempts from the guest VLAN.
