# Network Isolation Tests

Date: 11/10/2026
Firewall: network/router-nftables.conf

| # | Source | Destination | Test | Expected | Actual | Evidence |
|---|---|---|---|---|---|---|
| T1 | dmz-host | Internet (1.1.1.1) | ICMP | Blocked | | screenshots/t1.png |
| T2 | dmz-host | vm-scan | ICMP | Blocked | | |
| T3 | dmz-host | vm-scan:8834 | HTTPS | Blocked | | |
| T4 | vm-scan | dmz-host:80 | HTTP | Allowed | | |
| T5 | vm-scan | dmz-host:8080 | HTTP | Allowed | | |
| T6 | vm-scan | dmz-host:8983 | HTTP | Allowed | | |
| T7 | Admin host | dmz-host | ICMP | Blocked | | |
| T8 | Internal | DMZ:80/443/8080 | HTTP | Allowed | Pending v2 (no internal host yet) | |