# Security Penetration Test Report

**Generated:** 2026-02-28 19:24:09 UTC

# Executive Summary

The assessment of dog.htb identified a full compromise path from internet-facing exposure to root-level control of the host.

Overall risk posture is Critical. The most significant issues were public source control metadata exposure, credential disclosure with cross-service reuse, and unsafe privileged command delegation. Together, these enabled an attacker to obtain valid credentials, access the application and SSH services, and escalate privileges to root.

Business impact in a real environment would include complete confidentiality, integrity, and availability compromise of the affected system, with high potential for lateral movement if credentials are reused elsewhere.

Required lab artifacts were successfully obtained:
user.txt: 03231651465136bfd091b71b85f4897b
root.txt: 29682252ae41f8a67d5eebe5953770dc

# Methodology

The engagement followed a black-box penetration testing approach aligned with OWASP Web Security Testing Guide principles and standard adversarial workflow.

Scope and targets:
- Host: 10.129.231.223
- Application domain: dog.htb

Method summary:
- External reconnaissance and service enumeration (ports, service banners, web fingerprinting)
- Web attack-surface mapping (routes, access-control boundaries, exposed files/directories)
- Source exposure validation and secret extraction from publicly accessible artifacts
- Credential attack testing across application and SSH authentication surfaces
- Post-authentication abuse-path validation and privilege escalation testing
- Evidence-based validation only: findings were recorded when reproducible with clear technical proof

Threat intelligence validation:
Detected software and versions were correlated against live public advisories and vulnerability sources with applicability checks before inclusion.

# Technical Analysis

Confirmed findings were documented as formal vulnerability reports:

- vuln-0001: Publicly Accessible Git Metadata Directory Exposes Source Code and Secrets (High)
The web server exposed repository metadata under .git, allowing retrieval of source material and sensitive configuration data.

- vuln-0003: Credential Exposure in Public Repository Enables Cross-Service Account Takeover (Critical)
A disclosed credential from exposed source/configuration was reusable across services, enabling authenticated access to both web and SSH contexts.

- vuln-0002: Local Privilege Escalation via Sudo Permission on Bee CLI (High)
A non-root user could execute a privileged command path that allowed arbitrary command execution as root.

Attack chain validation:
Public .git exposure -> secret extraction -> credential reuse for authenticated access -> host access as standard user -> sudo-based privilege escalation -> root compromise.

Additional CVE intelligence (live-sourced and applicability-checked):
- Backdrop CMS XSS affecting versions before 1.27.3, applicable to detected 1.27.1 with privilege preconditions:
  NVD: https://nvd.nist.gov/vuln/detail/CVE-2024-41709
  Vendor advisory: https://backdropcms.org/security/backdrop-sa-core-2024-001
  Release context: https://github.com/backdrop/backdrop/releases/tag/1.27.3
- Apache HTTP Server 2.4.41 is within historical vulnerable ranges for selected issues, but exploitability is configuration-dependent and was not required for compromise:
  CVE-2021-44790 NVD: https://nvd.nist.gov/vuln/detail/CVE-2021-44790
  Apache advisory index: https://httpd.apache.org/security/vulnerabilities_24.html
- OpenSSH 8.2p1 is within version range for some historical issues, with non-default configuration prerequisites:
  CVE-2021-41617 NVD: https://nvd.nist.gov/vuln/detail/CVE-2021-41617
  Upstream discussion: https://www.openwall.com/lists/oss-security/2021/09/26/1

Systemic root causes:
- Sensitive operational data exposed through deployment misconfiguration
- Inadequate secret segregation and rotation discipline
- Excessive privilege delegation in sudo policy

# Recommendations

Immediate priority:
- Remove public access to version-control artifacts (.git, backup files, temporary files) at the web tier.
- Rotate all exposed credentials immediately (database, application, SSH, API/service accounts), invalidate active sessions, and review historical access logs for abuse.
- Disable credential reuse across services and enforce unique secrets per service/account boundary.
- Restrict sudoers entries to least privilege; remove or harden command paths that permit arbitrary code execution contexts.

Short-term priority:
- Implement secret management controls: centralized vaulting, runtime secret injection, no plaintext secrets in repository-tracked files.
- Enforce strong authentication policy for SSH and administrative application accounts, including key-based SSH controls and account lockout/rate-limiting.
- Harden administrative feature controls to reduce abuse potential after account compromise (defense-in-depth authorization and high-risk action gating).

Medium-term priority:
- Establish secure SDLC controls to prevent secret leakage and sensitive file publication (pre-commit hooks, repository scanning, CI policy gates).
- Implement continuous external exposure monitoring for high-risk artifacts and misconfigurations.
- Maintain patch and vulnerability intelligence workflows for core platform components and validate exploit preconditions during hardening cycles.

Retest guidance:
- Conduct focused verification after remediation to confirm:
  - .git and similar artifacts are no longer accessible
  - previously exposed credentials are invalidated and replaced
  - cross-service authentication with old credentials fails
  - privilege escalation path via sudo command delegation is eliminated
- Perform a follow-up end-to-end adversarial retest to ensure the original compromise chain is fully broken.

