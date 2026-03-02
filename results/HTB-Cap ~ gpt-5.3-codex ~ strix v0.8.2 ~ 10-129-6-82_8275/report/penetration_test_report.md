# Security Penetration Test Report

**Generated:** 2026-02-27 09:29:26 UTC

# Executive Summary

An external penetration test was conducted against the target host and associated web application. The assessment identified a critical attack path that enabled full system compromise.

Overall risk posture: Critical.

The most significant weakness was broken access control in the packet capture workflow, where capture objects were accessible via predictable identifiers without effective authorization. This exposed downloadable network capture files containing sensitive credentials. Those credentials were then reused to gain SSH access as a valid user, followed by local privilege escalation to root due to an unsafe Linux capability assignment on the Python interpreter.

Business impact is severe: unauthorized data disclosure, credential compromise, remote shell access, and full administrative control of the host. Immediate remediation should focus on access control enforcement, credential hygiene, and host hardening.

# Methodology

The assessment followed industry-aligned penetration testing methodology consistent with OWASP WSTG and PTES principles.

Scope included:
- Target host: 10.129.6.82
- Application hostname mapping: cap.htb
- Exposed services and web application attack surface

Testing approach:
- Black-box reconnaissance and attack-surface mapping
- Service fingerprinting and endpoint enumeration
- Authentication and authorization testing
- Business logic and object access control validation
- Exploitation and post-exploitation validation
- Live CVE and exploit intelligence lookup for detected software/version context

CVE intelligence process:
- Canonical product mapping (name/CPE/synonyms)
- Cross-source validation using NVD and vendor advisories
- Supplemental review of KEV/GHSA sources
- Applicability triage against observed versions and exploitation prerequisites

Validation standard:
Only reproducible, impact-demonstrated issues were considered confirmed findings and documented through formal vulnerability reports.

# Technical Analysis

Severity model considered exploitability and impact to confidentiality, integrity, and availability.

Confirmed findings:

1) Unauthenticated IDOR / broken access control in capture resources (High)
The web application exposed capture objects through predictable numeric IDs under data and download routes. Access controls did not enforce authentication or object ownership, allowing unauthorized enumeration and retrieval of other users’ packet capture files.
This issue was formally reported as a confirmed vulnerability.

2) Local privilege escalation via dangerous file capability on Python interpreter (High)
After obtaining a standard user shell, the host allowed privilege escalation because the Python interpreter had cap_setuid capability. This enabled setting effective UID to root and executing commands as root.
This issue was formally reported as a confirmed vulnerability.

Attack chain validation:
The two findings chained into full compromise:
IDOR exposure -> packet capture download -> credential disclosure -> SSH login as user -> local privilege escalation to root.

Flag evidence:
- user.txt: 3f0e677583f93e94446e0032d4e2062c
- root.txt: 157b3f5e76179406ee3b6d29b8fbb87c

Service/CVE intelligence highlights (live-sourced and applicability-triaged):
- vsftpd 3.0.3 and OpenSSH 8.2p1 were assessed against current advisories.
- Highest practical risk during this engagement came from application logic flaws and host misconfiguration rather than directly weaponized remote CVE exploitation.

Referenced sources used in lookup:
- NVD CPE query (vsftpd): https://services.nvd.nist.gov/rest/json/cves/2.0?cpeName=cpe:2.3:a:vsftpd_project:vsftpd:3.0.3:*:*:*:*:*:*:*
- NVD CPE query (OpenSSH): https://services.nvd.nist.gov/rest/json/cves/2.0?cpeName=cpe:2.3:a:openbsd:openssh:8.2:*:*:*:*:*:*:*
- OpenSSH security advisories: https://www.openssh.org/security.html
- Ubuntu CVE tracker example: https://ubuntu.com/security/CVE-2021-41617
- CISA KEV catalog: https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json

Operational metrics request:
- Test duration (minutes): not reliably available from environment telemetry
- Model name: not exposed in assessment runtime
- Token usage and estimated cost: not exposed in assessment runtime

# Recommendations

Immediate priority:
1. Enforce strict authorization on capture object access
   Require authentication for all capture-related routes and enforce per-object ownership checks server-side for every request.
2. Remove dangerous Linux capability from Python interpreter
   Remove cap_setuid from interpreter binaries and confirm no equivalent privilege-escalation capability remains on user-accessible executables.
3. Rotate exposed credentials
   Invalidate and rotate all credentials found in captures or potentially derived from disclosed traffic. Force password reset for affected accounts.
4. Restrict sensitive protocol usage
   Eliminate plaintext credential exposure in operational traffic. Use secure protocols and disable insecure credential transmission patterns.

Short-term priority:
5. Replace predictable identifiers with unguessable, access-scoped references
   Use non-enumerable identifiers in addition to authorization checks.
6. Implement centralized access-control middleware and negative authorization tests
   Add regression tests for cross-user object access and unauthenticated object retrieval attempts.
7. Harden SSH and host privilege boundaries
   Review sudo rights, file capabilities, and setuid binaries; enforce least privilege and continuous configuration auditing.

Medium-term priority:
8. Improve detection and monitoring
   Alert on anomalous object ID enumeration, unusual download patterns, credential abuse, and privilege-escalation behaviors.
9. Conduct focused retest
   Perform a verification retest after remediation to validate:
   - IDOR closure and ownership enforcement
   - credential exposure elimination
   - removal of local privilege-escalation paths

