# Rules of Engagement — Vulnerability Management Lab

Version: 1.0
Last updated: 11/10/2026

## Purpose
Identify, prioritise, remediate and verify vulnerabilities across the lab environment on a continuous basis, aligned to the ASD Essential Eight patching strategies and CIS Control 7.

## Scope

### In scope
- 10.10.10.0/24 — Internal segment
- 10.10.20.0/24 — DMZ segment (simulated internet-facing)
- 10.10.30.0/24 — Management segment
- Router WAN interface 192.168.1.10
- Assets marked `in_scope = yes` in inventory/assets.csv

### Out of scope
- The Windows host machine (192.168.1.1)
- The remainder of 192.168.1.0/24 (VMware NAT network)
- The home LAN and any address outside the lab
- dc1, v1 and log-server: out of scope for v1, pending migration into the segmented network (v2)

## Asset criticality scale
- 4 — Compromise impacts the whole environment (router/firewall, domain controller)
- 3 — Important server or security tooling
- 2 — Standard server or workstation
- 1 — Test or disposable system

## Scan types and schedule
Target schedule:
- Unauthenticated discovery scan: weekly, all in-scope subnets
- Authenticated scan: weekly, all in-scope hosts
- DMZ: twice weekly (exposed assets require tighter timeframes)
- Verification re-scan: on demand after remediation

Lab note: during v1, scans are run manually on demand. The schedule above is the intended operating model.

## Scan accounts
- Linux: `svc-vulnscan` — SSH key authentication, scoped sudo, present on router and dmz-host
- Windows: `svc-vulnscan` domain account (v2, not yet created). Local admin on scanned hosts via GPO; never Domain Admin
- Credentials are stored only in Nessus and on vm-scan; never committed to Git
- Credentials rotated every 90 days

## Safe-scan settings
- "Safe checks" enabled; no denial-of-service or dangerous plugins
- No brute-force or password-guessing plugins
- Scans run from vm-scan (10.10.30.10) only

## Scan data handling
- Raw scan exports are stored on vm-scan and excluded from Git
- Only sanitised sample results are published in reports/

## Remediation timeframes
Defined in remediation-sla.md.

## Owners and approvals
- Router / domain infrastructure: Infrastructure team
- Internal Linux servers: Platform team
- DMZ services: Web/application team
- Scanner and SIEM: Security team
- Risk acceptance approver: Security manager
- Vulnerability management (scan, triage, track, verify — does not patch): VM analyst

## Stop conditions
If a scan causes service degradation:
1. Stop the scan
2. Notify the asset owner
3. Record the affected host and plugin
4. Exclude that host/plugin until reviewed