P1 Internet-facing (DMZ) and actively exploited (on KEV or public exploit) 48 hours
P2 Internet-facing with no known exploit, or internal criticality-4 asset with a known exploit 2 weeks
P3 Internal, CVSS High/Critical, no known exploit 30 days
P4 Everything else (Medium/Low, internal) 90 days or next maintenance cycle

The clock starts at first detection, not when someone gets around to triaging the finding.
"Fixed" means verified. A finding closes only when a re-scan confirms it's gone, not when someone says they've patched it.
Missed deadlines escalate. An SLA breach goes to the asset owner's manager and appears in the executive report.
Exceptions require a risk acceptance. It needs a business reason, a compensating control, an approver, and an expiry date (max 90 days, then re-review). P1 findings can't be risk-accepted without a compensating control in place.
Exposure trumps severity. A Medium on a DMZ host can be P2. A Critical on an isolated internal test box can be P3.
