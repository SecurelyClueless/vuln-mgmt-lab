# Risk Acceptance Register

## RA-001 — Apache Tomcat 8.0.43 (end-of-life) on dmz-host

| Field                        | Value                                                                                                                                    |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Related finding              | F-002 (triage.csv)                                                                                                                       |
| Asset                        | dmz-host (10.10.20.10), port 8080/tcp                                                                                                    |
| Asset criticality / exposure | 3 / DMZ (simulated internet-facing)                                                                                                      |
| Risk                         | Tomcat 8.0.x is end-of-life and receives no security fixes. Scan reports 8 version-based findings (2 Critical, 3 High, 2 Medium, 1 Low). |
| Priority before controls     | P2                                                                                                                                       |
| Business reason              | Application depends on Tomcat 8.0; migration to a supported branch requires application testing not yet scheduled.                       |
| Owner                        | Web/application team                                                                                                                     |
| Approver                     | Security manager                                                                                                                         |
| Accepted on                  | 2026-10-18                                                                                                                               |
| Expires                      | 2026-11-18 (30 days) — re-review required; no automatic renewal                                                                          |

### Why this was not treated as P1

The highest-risk finding, CVE-2017-12617 (CISA KEV, EPSS 0.9999), is only exploitable when HTTP PUT is enabled on the default servlet. On 2026-10-18 the `readonly` parameter was confirmed not overridden in `conf/web.xml` (default `true`), so PUT and DELETE requests are rejected. The KEV item is not exploitable in the current configuration.

### Compensating controls

1. **HTTP PUT disabled** — verified 2026-10-18 (readonly = default true).
2. **Manager, host-manager, examples and docs apps removed** — enforced at every container start via docker-compose (F-009, CHG-002). Verified: webapps contains ROOT only.
3. **No outbound connectivity from the DMZ** — router firewall drops all DMZ-initiated traffic and DMZ is not NATed (verified by isolation tests T1–T3).
4. **Inbound limited** — only Internal (TCP 80/443/8080) and Management segments can reach the DMZ.

### Residual risk

Remaining version-based vulnerabilities in an unsupported branch, including any disclosed after end-of-life that will never be patched. Likelihood reduced by the controls above; impact unchanged.

### Conditions

- Acceptance is void if HTTP PUT is enabled, the manager app is restored, or DMZ egress is opened outside an approved change.
- Re-scan weekly; any new KEV finding on this asset ends the acceptance immediately.
- Remediation path: upgrade to a supported Tomcat branch (9.0.x / 10.1.x) before 2026-11-18.
