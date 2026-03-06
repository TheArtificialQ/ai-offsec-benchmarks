# Security Penetration Test Report

**Generated:** 2026-03-06 19:02:10 UTC

# Executive Summary

An external assessment of the target host identified a complete compromise path from unauthenticated network access to root-level code execution.

Overall risk posture: Critical.

The attack chain began with exposed SNMP using the default community string `public`, which disclosed sensitive environment details and revealed the presence of a daloRADIUS deployment. The web management application was then found at a non-obvious path and accepted default administrative credentials (`administrator:radius`). After authentication, the application exposed additional credential material, including a reusable user password hash that was cracked offline, enabling SSH access as a local user. Local privilege escalation was then possible because the user had passwordless sudo rights to execute `mosh-server`, which can be abused to start a root shell.

Business impact is severe. An attacker could obtain administrative application access, recover user credentials, establish system-level access over SSH, and escalate to root. This enables full loss of confidentiality, integrity, and availability of the host and the services it supports.

Recovered flags:
- user.txt: `REDACTED`
- root.txt: not recovered during this run, although root shell execution via the sudo-allowed `mosh-server` path was demonstrated.

# Methodology

The assessment was conducted as an authorized external penetration test against the in-scope host using a black-box approach, with validation performed through direct exploitation.

Methodology aligned with OWASP Web Security Testing Guide principles and standard penetration testing workflow:
- Network reconnaissance and service enumeration across TCP and UDP exposure
- Service fingerprinting and protocol-specific enumeration
- Web application discovery, hidden-path identification, and authentication testing
- Credential validation and attack-chain development
- Local privilege escalation assessment following confirmed shell access
- Live vulnerability and exploit lookup for the identified product family using public authoritative sources

Testing activities included:
- TCP and UDP port scanning
- SNMP enumeration
- HTTP content and path discovery
- Application login and authenticated feature review
- Credential exposure validation
- SSH access validation
- Sudo privilege enumeration and privilege-escalation testing
- Product-specific CVE review using NVD, CVE.org, upstream GitHub advisory data, and related public references

Evidence standards:
Only issues that were directly reproduced and validated against the target were treated as confirmed findings. CVE references were checked live during the engagement and cross-referenced with at least two sources before inclusion.

# Technical Analysis

The following confirmed findings were validated during the assessment.

1) Exposed SNMP with default community string enables sensitive information disclosure
Severity: High

UDP/161 was reachable and accepted the default SNMP v2c community string `public`. This exposed system metadata including hostname context, contact information, and an explicit reference to the deployed application stack. The disclosure materially accelerated attack-surface discovery by revealing that the host was running daloRADIUS.

2) daloRADIUS management interface exposed with default operator credentials
Severity: Critical

The daloRADIUS operator interface was available at `/daloradius/app/operators/login.php`. Authentication succeeded with the default credentials `administrator:radius`. This provided immediate administrative access to the platform without prior compromise. The issue is a direct improper-authentication weakness with critical impact because it exposed configuration, user-management, and maintenance functionality.

3) Authenticated credential disclosure within daloRADIUS
Severity: High

After login, the application exposed sensitive credential material in management pages:
- The operator edit page disclosed the operator password in cleartext.
- The user management page exposed the `svcMosh` account password as an MD5 value.
- Maintenance functionality also exposed the RADIUS shared secret `testing123`.

The disclosed MD5 value for `svcMosh` was cracked offline to `underwaterfriends`, which was then successfully used for SSH login. This converted application compromise into operating-system access.

4) SSH access as local user via cracked credential
Severity: High

Using the recovered `svcMosh` credential, SSH access to the host was confirmed. This demonstrated a successful transition from web compromise to system compromise.

5) Passwordless sudo to `mosh-server` allows root shell execution
Severity: Critical

The `svcMosh` account had the following sudo privilege:

`(ALL) NOPASSWD: /usr/bin/mosh-server`

This can be abused by launching `mosh-server` as root with a chosen command, such as `/bin/bash`, yielding root shell execution. A root mosh prompt was successfully initiated, demonstrating practical root compromise. Although the root flag could not be cleanly captured through the terminal interaction model during this run, the privilege-escalation path itself was validated.

Product-specific vulnerability lookup results:
Live CVE review for daloRADIUS found limited public CVE coverage. The most relevant current advisory identified was CVE-2022-23475, which affects daloRADIUS versions up to 1.3 and describes an XSS and CSRF account-takeover condition in `mng-del.php`. Sources reviewed included:
- NVD: https://nvd.nist.gov/vuln/detail/CVE-2022-23475
- GitHub Security Advisory: https://github.com/lirantal/daloradius/security/advisories/GHSA-c9xx-6mvw-9v84
- Fix commit: https://github.com/lirantal/daloradius/commit/ec3b4a419e20540cf28ce60e48998b893e3f1dea

A legacy XSS issue, CVE-2009-4347, was also found in public records but appears relevant only to very old releases:
- NVD: https://nvd.nist.gov/vuln/detail/CVE-2009-4347
- CVE record: https://cveawg.mitre.org/api/cve/CVE-2009-4347

These CVEs were reviewed for applicability, but the compromise achieved in this assessment did not depend on them. The confirmed compromise path relied on exposed SNMP, default credentials, credential disclosure, and unsafe sudo delegation.

Systemic themes observed:
- Default or weak secrets remained in production-facing services
- Administrative interfaces were insufficiently protected
- Sensitive credentials were exposed within the management application
- Privilege separation was undermined by unsafe sudo configuration
- Defense-in-depth controls were insufficient to prevent chaining from application access to root compromise

# Recommendations

Immediate priority

1. Disable or restrict SNMP exposure
Remove external access to UDP/161 unless strictly required. If SNMP is necessary, disable v2c, use SNMPv3 with strong authentication and encryption, restrict source IPs, and rotate all community strings.

2. Remove default and weak credentials
Immediately change all daloRADIUS operator credentials, RADIUS shared secrets, and user passwords that may have been exposed or derived from the application. Review for password reuse across system and service accounts.

3. Restrict or remove the daloRADIUS management interface
Move the interface behind a VPN, bastion, or strict IP allowlist. Enforce strong authentication and consider adding MFA for administrative access.

4. Remove unsafe sudo delegation
Revoke passwordless sudo access to `/usr/bin/mosh-server`. If privileged operational actions are required, replace them with tightly scoped wrapper scripts that do not permit arbitrary command execution.

Short-term priority

5. Eliminate credential disclosure in the application
Ensure operator passwords are never retrievable in cleartext. Do not display reusable password hashes to administrative users. Remove or tightly restrict features that reveal internal secrets such as RADIUS shared secrets.

6. Rebuild password storage and credential handling
Store passwords using modern password hashing for interactive logins. Do not permit MD5-based secrets to serve as reusable authentication material for other services. Enforce credential rotation for all accounts touched by this compromise path.

7. Audit all local accounts and SSH exposure
Review `svcMosh`, administrative accounts, and authorized keys for unauthorized persistence. Rotate credentials, review shell history and logs, and restrict SSH access to necessary users only.

Medium-term priority

8. Patch and update daloRADIUS
Confirm the deployed version and upgrade to a supported, current release. Review the upstream advisory for CVE-2022-23475 and verify remediation where applicable.
Relevant sources:
- https://nvd.nist.gov/vuln/detail/CVE-2022-23475
- https://github.com/lirantal/daloradius/security/advisories/GHSA-c9xx-6mvw-9v84

9. Improve segmentation and service hardening
Limit access to administrative web applications and local-only services such as MariaDB and RADIUS management functions. Apply host firewall policy to reduce exposed management interfaces and protocols.

10. Retest after remediation
A focused retest should verify:
- SNMP is no longer externally enumerable with default settings
- daloRADIUS no longer accepts default credentials
- credential disclosure paths are removed
- SSH access using recovered credentials is no longer possible
- sudo no longer permits root code execution via `mosh-server`
- root flag retrieval is no longer achievable through the demonstrated attack path
