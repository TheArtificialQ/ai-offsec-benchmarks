# Security Penetration Test Report

**Generated:** 2026-03-03 18:24:45 UTC

# Executive Summary

An external penetration test of the Dog.HTB machine revealed multiple security vulnerabilities stemming primarily from improper deployment practices and configuration management. The most critical finding is the exposure of the .git/ directory, which allowed complete extraction of the application source code and led to the discovery of hardcoded database credentials. While the database is not directly accessible from external networks, the exposed credentials combined with other potential vulnerabilities create significant risk. The overall risk posture is HIGH due to the ease of exploitation and the sensitivity of exposed information.

# Methodology

The assessment was conducted using black-box testing methodology with source code analysis made possible through the discovered .git/ exposure. Testing included: network reconnaissance using nmap, web application scanning using custom scripts and manual testing, source code analysis of the extracted Backdrop CMS 1.27.1 application, authentication testing, and vulnerability exploitation attempts. The OWASP Testing Guide v4 was followed for web application security testing.

# Technical Analysis

Three confirmed vulnerabilities were identified and documented:

1) Exposed Git Repository (HIGH - CVSS 7.5): The .git/ directory was publicly accessible, allowing full source code extraction via git-dumper. This represents a critical deployment security failure that exposed 2873 files including core application logic and configuration.

2) Hardcoded Database Credentials (HIGH - CVSS 7.5): MySQL root credentials (root:BackDropJ2024DS2024) were found hardcoded in settings.php, extracted from the exposed repository. While MySQL listens only on localhost, these credentials could be leveraged through other vulnerabilities.

3) Exposed Configuration Directory (MEDIUM - CVSS 5.3): The Backdrop CMS configuration directory at /files/config_*/ was publicly browsable, revealing 75 configuration files containing system settings, user roles, and permissions.

Additional testing was performed for SQL injection, XSS, SSRF, LFI, RCE, and authentication bypass without success. The Backdrop CMS 1.27.1 application appears to have secure coding practices in these areas.

# Recommendations

Immediate Priority:
1. Remove .git/ directory from web root immediately to prevent further source code exposure
2. Rotate all database credentials exposed in the repository
3. Move configuration directories outside web root and disable directory listing
4. Review and audit all files committed to version control for secrets

Short-term Priority:
5. Implement proper error handling to prevent username enumeration through login error messages
6. Deploy web application firewall rules to block access to sensitive paths
7. Enable security headers (X-Content-Type-Options, X-Frame-Options, CSP)
8. Create dedicated MySQL user with minimal privileges instead of using root

Medium-term Priority:
9. Implement automated scanning in CI/CD pipeline to detect exposed directories and secrets
10. Conduct regular security audits and penetration testing
11. Review file and directory permissions on production systems
12. Implement intrusion detection to monitor for suspicious access patterns

Validation:
After remediation, verify that:
- .git/ returns 403 Forbidden or 404 Not Found
- Configuration directories are inaccessible
- No sensitive files are exposed in web root
- New credentials are not present in source code

