# Network

## IP plan

| Segment                    | Subnet         | Router interface | Router IP    | Hosts                         |
| -------------------------- | -------------- | ---------------- | ------------ | ----------------------------- |
| WAN (VMware NAT "MOD NAT") | 192.168.1.0/24 | ens33            | 192.168.1.10 | Admin workstation 192.168.1.1 |
| Internal                   | 10.10.10.0/24  | ens37            | 10.10.10.1   | Reserved for v2               |
| DMZ                        | 10.10.20.0/24  | ens38            | 10.10.20.1   | dmz-host 10.10.20.10          |
| Management                 | 10.10.30.0/24  | ens39            | 10.10.30.1   | vm-scan 10.10.30.10           |

## Router interface mapping (authoritative)

Verified by MAC against VMware adapter settings. Re-verify whenever adapters are edited.

| Interface | MAC               | VMware adapter    | VMware network |
| --------- | ----------------- | ----------------- | -------------- |
| ens33     | 00:0c:29:d5:63:6d | Network Adapter   | MOD NAT        |
| ens37     | 00:0c:29:d5:63:81 | Network Adapter 2 | Internal LAN   |
| ens38     | 00:0c:29:d5:63:77 | Network Adapter 3 | DMZ            |
| ens39     | 00:0c:29:d5:63:8b | Network Adapter 4 | MANAGEMENT LAN |

## Firewall design

- Default deny on input and forward chains; stateful return traffic allowed
- Every allow rule bound to an ingress interface (anti-spoofing), backed by `rp_filter = 1`
- DMZ blocked twice: no forward rule allows DMZ-initiated traffic, and DMZ is never NATed
- Admin access to Management from a single workstation IP only
- Router SSH: key-only, from Management and the admin workstation only
- Dropped traffic counted and logged (`IN-DROP`, `FWD-DROP` prefixes)

Ruleset: [`router-nftables.conf`](router-nftables.conf)
