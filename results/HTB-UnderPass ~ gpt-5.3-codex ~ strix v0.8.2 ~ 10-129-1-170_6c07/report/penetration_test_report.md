# Security Penetration Test Report

**Generated:** 2026-03-06 19:41:07 UTC

# Executive Summary

An authorized penetration test of underpass.htb identified a critical, chained compromise path from unauthenticated web access to full root-level system control.

The assessment confirmed three high-impact weaknesses:
An exposed administrative login using weak/default credentials in daloRADIUS
Sensitive credential and password-hash disclosure within authenticated administrative functionality
A local privilege escalation path via a sudo NOPASSWD misconfiguration for mosh-server

These issues enabled complete host compromise with direct access to both CTF flags:
user.txt: `REDACTED`
root.txt: `REDACTED`

Overall risk posture is Critical. The primary business impact theme is credential hygiene and privilege-boundary failure: weak authentication controls and over-privileged local execution combined to allow full system takeover.

# Methodology

The assessment was conducted as an external black-box penetration test aligned with OWASP WSTG and standard adversarial testing workflow.

Scope and approach:
Target host: 10.129.1.170
Mapped domain: underpass.htb
Testing included network/service enumeration, web attack-surface mapping, authentication and session testing, post-authenticated functionality review, credential abuse validation, and local privilege escalation validation.

Execution phases:
Reconnaissance and service fingerprinting
HTTP application discovery and endpoint mapping
Authentication and authorization testing
Sensitive data exposure verification
Credential cracking/validation where applicable
Privilege escalation validation and impact proof

CVE and exploit lookup mode was executed with live-source validation. Product/version context was established from discovered services and application banners, and candidate vulnerabilities were checked against:
NVD JSON API: https://services.nvd.nist.gov/rest/json/cves/2.0
Vendor/project security source: https://github.com/lirantal/daloradius/security/advisories

Only validated, reproducible findings with demonstrated impact were treated as confirmed vulnerabilities.

# Technical Analysis

Severity model combined exploitability, required privileges, and confidentiality/integrity impact, with emphasis on realistic chaining.

Confirmed findings summary:
Critical: Weak Default Operator Credentials Allow Unauthorized daloRADIUS Administrative Access
The operator portal accepted a default/weak credential pair, providing immediate administrative dashboard access without prior foothold.

High: Authenticated daloRADIUS Interface Exposes Database Credentials and User Password Hashes
Within the authenticated interface, sensitive configuration and identity material was disclosed, including database credentials and password hash data. This materially increased lateral movement risk and enabled credential compromise.

Critical: Sudo NOPASSWD Misconfiguration Allows Root Privilege Escalation via mosh-server
A local user could execute mosh-server as root without password. This allowed escalation to uid=0 and full system compromise.

Attack chain validated end-to-end:
Unauthenticated access to operator portal
Administrative login via weak/default credentials
Extraction of sensitive credential/hash material from admin interface
Credential use to obtain shell access as low-privileged user
Escalation to root via sudo mosh-server misconfiguration
Read access to both flags:
user.txt = `REDACTED`
root.txt = `REDACTED`

CVE applicability review:
NVD results for daloRADIUS returned historical candidates (including CVE-2009-4347 and CVE-2022-23475), but this engagement’s primary confirmed issues were environment-specific misconfiguration and credential weaknesses directly demonstrated during testing.
Source used: https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=daloRADIUS&resultsPerPage=20
Vendor advisory source reviewed: https://github.com/lirantal/daloradius/security/advisories

Detailed reproduction and evidence are documented in the three created vulnerability reports (vuln-0001, vuln-0002, vuln-0003).

# Recommendations

Immediate priority:
Remove weak/default credentials and rotate all operator, database, and related service credentials.
Disable password-based reuse across web admin, system users, and backend services.
Restrict administrative interfaces to trusted management networks and require strong authentication controls.

Remove the sudo privilege for /usr/bin/mosh-server from non-administrative users immediately.
Apply least privilege in sudoers, requiring explicit command justification and no broad NOPASSWD execution rights for interactive/session-launch binaries.
Audit all sudoers entries for similar privilege-escalation primitives.

Short-term priority:
Eliminate sensitive secret exposure in administrative pages.
Do not render database credentials or password hash material in UI responses.
Store and handle secrets via protected configuration channels and strict role-gated access paths.
Enforce robust password policy and optional MFA for operator accounts.

Medium-term priority:
Implement centralized credential lifecycle management, periodic rotation, and secret scanning.
Add detection/alerting for anomalous operator logins, credential-access pages, and suspicious sudo invocations.
Perform regular hardening reviews for exposed management protocols and interfaces.

Retest guidance:
Conduct a focused retest after remediation to confirm:
Default/weak credentials no longer provide access
Sensitive credential/hash disclosures are removed from all admin workflows
Sudo misconfiguration is fully corrected and root escalation path is closed
No alternative privilege-escalation path remains via comparable sudo-enabled binaries

