# Security Penetration Test Report

**Generated:** 2026-03-05 20:27:27 UTC

# Executive Summary

An external assessment of the Dog web application identified multiple high-impact security weaknesses that materially weaken the target’s security posture. The most significant issue is public exposure of sensitive deployment artifacts, including the application’s version-control metadata and live Backdrop CMS configuration, which enabled recovery of internal configuration details and database credentials without authentication. In parallel, the authentication workflow discloses valid usernames through differential responses in both password reset and login flows, enabling targeted attacks against real user accounts.

Overall risk posture: High.

The exposed repository and configuration data provide attackers with exact product versioning, internal architecture, user-role definitions, flood-control thresholds, configuration paths, and secrets that should never be publicly reachable. This substantially lowers the effort required to perform follow-on attacks such as targeted password guessing, exploit selection, and credential reuse attempts. The authentication enumeration issue further compounds this risk by allowing attackers to identify valid users, including confirmed active accounts, before conducting focused attacks.

The immediate remediation theme is to eliminate sensitive file exposure from the web root, rotate disclosed credentials and secrets, and standardize authentication error handling to prevent user enumeration. A full compromise was not completed during this assessment, but the confirmed weaknesses provide a credible path toward account takeover and broader application compromise.

# Methodology

The assessment was performed as an external black-box web application and host evaluation with limited gray-box insight derived only from artifacts publicly exposed by the target itself.

Testing approach
The assessment followed principles aligned with OWASP Web Security Testing Guide, PTES, and standard web application penetration testing practice. Activities included:
- Service discovery and web fingerprinting
- HTTP route and functionality enumeration
- Review of application-generated content and metadata
- Authentication workflow analysis
- Version identification and advisory validation
- Public artifact and source disclosure assessment
- Configuration analysis from publicly exposed files
- Limited credential validation against exposed services

In-scope assets tested
- HTTP service at http://dog.htb
- SSH service on the target host
- Publicly exposed application resources and repository artifacts

Validation standard
Only reproducible findings with direct evidence were treated as confirmed. Version-specific vulnerability intelligence was cross-checked against live vendor advisory material and NVD where applicable. Sensitive disclosures were validated through direct retrieval of exposed files and reconstruction of referenced repository objects.

# Technical Analysis

Severity was assessed based on realistic exploitability and likely impact to confidentiality, integrity, and availability.

Confirmed findings

1) Public exposure of Backdrop configuration and source repository via web root (High)
The application exposes Git metadata and active Backdrop CMS configuration files over HTTP without authentication. Accessible resources included .git metadata and JSON configuration files under the public files path. These disclosures enabled recovery of repository metadata, file inventory, active configuration, role permissions, flood-control settings, exact Backdrop version, and the application settings file through Git object retrieval. The recovered settings data included a database connection string, hash salt, and configuration directory paths. This is a serious information disclosure issue that meaningfully accelerates exploitation and follow-on compromise. This finding was formally documented in vulnerability report vuln-0001.

2) Username enumeration in authentication and password reset workflows (Medium)
The application returns different responses for valid and invalid users. In the password reset workflow, invalid usernames remained on the reset page while valid users produced a different flow and an “Unable to send email” condition. In the login workflow, invalid usernames returned an unrecognized-user response, whereas valid users returned an incorrect-password response and were subject to per-account lockout behavior after repeated failures. This allowed confirmation of at least two valid usernames: john and tiffany. Although no account compromise was achieved, this weakness substantially improves the efficiency of targeted attacks.

Observed systemic issues
- Sensitive deployment and operational files were placed inside public web paths or otherwise published in the deployed web root.
- Security boundaries between development artifacts and production assets were not enforced.
- Authentication error handling was inconsistent and leaked account validity.
- Publicly exposed configuration materially improved attacker understanding of access controls, rate limits, versioning, and technology choices.

Additional context
Backdrop CMS 1.27.1 was identified on the target. Vendor and NVD research confirmed that this version falls in the affected range for CVE-2024-41709, a Backdrop core issue addressed by BACKDROP-SA-CORE-2024-001. However, the advisory requires elevated privileges related to field administration, and no authenticated foothold was obtained to validate exploitability in this environment. As such, it was not reported as a confirmed vulnerability in this assessment.

# Recommendations

Immediate priority

1. Remove all version-control metadata from the web root
Purge the .git directory and any other development artifacts from the deployed application. Confirm that repository metadata, commit history, object files, and related references are inaccessible over HTTP.

2. Relocate active configuration outside the public document root
Move Backdrop active and staging configuration directories to non-web-accessible locations in accordance with secure deployment practice. Verify that no configuration JSON, backup files, or settings artifacts can be fetched directly.

3. Rotate all exposed secrets
Immediately rotate database credentials, salts, and any other secrets that were present in exposed files or repository history. Review for credential reuse across SSH, administrative accounts, scheduled tasks, and supporting services.

4. Normalize authentication responses
Ensure that login and password reset workflows return uniform responses regardless of whether a username exists. Remove all differential error conditions that disclose account validity.

Short-term priority

5. Harden deployment controls
Implement a deployment process that publishes only required runtime assets. Exclude repository directories, configuration exports, development files, backups, and temporary content from production deployments.

6. Restrict direct access to sensitive paths at the web server layer
Add explicit deny rules for:
- .git and related version-control paths
- configuration export directories
- backup, archive, and temporary files
- internal administrative or installer artifacts not intended for public access

7. Review account protections
Maintain per-account lockout or throttling controls, but combine them with enumeration-safe messaging, monitoring, and alerting. Validate that rate-limit policies cannot be bypassed through response differentials.

Medium-term priority

8. Conduct secret exposure and access review
Audit the environment for unauthorized access using disclosed credentials and configuration intelligence. Review database, SSH, and application logs for suspicious activity.

9. Upgrade and maintain Backdrop CMS securely
Upgrade from Backdrop CMS 1.27.1 to a supported security-fixed release. Reassess the environment for any version-specific issues after upgrade, including previously disclosed vendor advisories.

10. Perform focused retesting
After remediation, perform a focused retest to verify:
- .git and configuration paths are no longer reachable
- rotated credentials invalidate prior disclosures
- login and password reset no longer permit username enumeration
- no residual sensitive files remain exposed in public paths

