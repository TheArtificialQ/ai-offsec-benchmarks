# Security Penetration Test Report

**Generated:** 2026-02-22 19:52:44 UTC

# Executive Summary

An external penetration test was conducted against the target dog.htb, a web server hosting Backdrop CMS version 1.27.1. The assessment identified multiple security weaknesses that collectively enabled complete system compromise at the web application level.

Overall risk posture: Critical.

Key findings include an exposed Git repository providing source code and database credentials, an authenticated remote code execution vulnerability in the CMS module installer, and directory listing exposing sensitive configuration files. The combination of these vulnerabilities allowed for initial access, credential theft, and code execution on the target server.

Business impact includes potential unauthorized access to all data stored in the CMS database, the ability to modify or deface web content, and the potential for lateral movement within the hosting environment. While the user flag could not be retrieved due to file permission restrictions, the RCE capability provides a persistent foothold for further exploitation attempts.

Remediation should prioritize eliminating the exposed Git repository, securing module installation mechanisms, and disabling directory listings.

# Methodology

The assessment was conducted in accordance with the OWASP Web Security Testing Guide (WSTG) and industry-standard penetration testing methodology.

Engagement details:
- Assessment type: External penetration test (black-box with gray-box context from exposed source)
- Target environment: Production web server

Scope (in-scope assets):
- Web application: http://dog.htb
- Open services: HTTP (80), SSH (22)

High-level testing activities performed:
- Reconnaissance and attack surface mapping (subdomain enumeration, port scanning, service detection)
- Web application crawling and endpoint discovery
- Exposed Git repository exploitation and credential extraction
- Authentication testing with discovered credentials
- Authenticated vulnerability assessment targeting CMS-specific functionality
- Exploit development and validation for module upload RCE
- Post-exploitation enumeration for privilege escalation paths
- File system and system configuration review from compromised web shell

Evidence handling and validation standard:
Only validated issues with reproducible impact were treated as findings. Each finding was documented with clear reproduction steps and proof-of-concept code.

# Technical Analysis

This section provides a consolidated view of confirmed findings. Detailed reproduction steps are documented in individual vulnerability reports.

Severity model used:
Critical: Immediate compromise without authentication or with minimal effort; complete system compromise
High: Significant access requires authentication; data breach or code execution possible
Medium: Limited impact; information disclosure or requires specific conditions

Confirmed findings:

1) Exposed Git Repository (Critical - CVSS 9.4)
The web server exposes the complete .git directory containing full source code, commit history, and configuration files. This enabled extraction of database credentials (mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop), hash salts, and complete application source. The credentials were valid and provided administrative access to the CMS.

2) Backdrop CMS Authenticated RCE (High - CVSS 7.2)
Backdrop CMS 1.27.1 allows authenticated administrators to upload arbitrary module archives. A malicious archive containing a PHP web shell can be uploaded via the installer at /admin/installer/manual. The shell is extracted to a publicly accessible path, enabling remote command execution as the www-data user (uid=33).

3) Directory Listing (Medium - CVSS 5.3)
Directory listing is enabled for /files/ and /modules/, exposing configuration JSON files, user email addresses, and role definitions. This facilitates information gathering and identification of attack targets.

Systemic themes:
- Sensitive metadata (.git) should never be deployed to production
- File upload mechanisms require strict validation of content
- Default web server configurations often enable directory listing

# Recommendations

The following recommendations are prioritized by urgency.

Immediate priority:
1. Remove or block access to the .git directory
   - Delete the .git directory from production
   - Add Apache/Nginx rules to deny access to .git paths
   - Implement automated checks to detect exposed VCS metadata

2. Restrict module installation functionality
   - Disable manual module upload for non-essential users
   - Implement content validation for uploaded archives
   - Remove write permissions from modules directory for web server user

3. Disable directory listing
   - Add "Options -Indexes" to Apache configuration
   - Verify .htaccess files are present in sensitive directories
   - Move configuration storage outside web root

Short-term priority:
4. Rotate all exposed credentials
   - Change database password
   - Reset CMS administrator passwords
   - Regenerate hash salt

5. Implement principle of least privilege
   - Separate database users for application vs. administrative tasks
   - Restrict web server file system permissions

6. Deploy Web Application Firewall
   - Add rules to detect web shell patterns
   - Block common exploit signatures

Medium-term priority:
7. Security monitoring implementation
   - Log and alert on file creation in web directories
   - Monitor for unusual command execution patterns
   - Implement file integrity monitoring

Retest guidance:
Conduct a focused retest after remediation to verify:
- Git repository is no longer accessible
- Module upload exploits are blocked
- Directory listings are disabled
- Previously exposed credentials are invalidated

