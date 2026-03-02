# Security Penetration Test Report

**Generated:** 2026-02-22 15:35:45 UTC

# Executive Summary

A security assessment of the target system at 10.129.2.140 identified **one critical vulnerability** that enables complete compromise of user accounts via credential exposure. The application operates as a Security Dashboard with network monitoring capabilities, but implementation flaws expose sensitive authentication credentials to unauthenticated attackers.

**Risk Posture: HIGH**

**Key Findings:**
- **Critical (CVSS 9.1)**: The `/download/{id}` endpoint serves network capture (PCAP) files without authentication. These files contain cleartext FTP credentials (username: nathan, password: Buck3tH4TF0RM3!) intercepted during authentication sessions.
- **Medium (CVSS 5.3)**: System information endpoints (`/ip`, `/netstat`, `/capture`) expose network configuration, active connections, and listening services without requiring authentication.
- **Low**: Server header disclosure reveals the Gunicorn WSGI server in use.

**Business Impact:**
- Complete compromise of the "nathan" user account via FTP
- Unauthorized access to user data and files (user flag/credentials retrieved)
- Exposed network topology aids attackers in lateral movement planning
- System commands accessible for reconnaissance without authentication

**Remediation Theme:**
Prioritize removing sensitive data from capture files and implementing authentication on all system endpoints. The credential exposure represents an immediate risk requiring urgent remediation.

# Methodology

The assessment followed industry-standard penetration testing methodologies aligned with OWASP Testing Guide and PTES standards:

**Engagement Type**: External network penetration test (black-box)

**Testing Activities Performed:**
1. **Network Reconnaissance**: Comprehensive port scanning with nmap to identify open services (ports 21/ftp, 22/ssh, 80/http)
2. **Service Enumeration**: Version detection and banner grabbing for all identified services
3. **Web Application Mapping**: Directory enumeration, endpoint discovery, and attack surface analysis using ffuf
4. **Vulnerability Assessment**: Automated and manual testing for:
   - Command injection vulnerabilities (RCE testing on /capture, /ip, /netstat endpoints)
   - Insecure Direct Object Reference (IDOR) on /data/{id} and /download/{id}
   - Path traversal in file download functionality
   - Information disclosure through error messages, headers, and system output
5. **Traffic Analysis**: PCAP file examination to extract sensitive data from network captures
6. **Credential Testing**: Validation of discovered credentials against FTP service

**Evidence Validation**: All findings were validated through successful exploitation, including FTP authentication with exposed credentials and retrieval of sensitive user files.

# Technical Analysis

The assessment identified confirmed vulnerabilities across multiple severity levels. Detailed reproduction steps and evidence are documented in individual vulnerability reports.

**Severity Model**: CVSS 3.1 base scores reflect exploitability and potential impact considering realistic attacker capabilities.

**Confirmed Findings:**

1) **Cleartext Credential Exposure in PCAP Files (Critical, CVSS 9.1)**
The application captures network traffic and stores it as downloadable PCAP files accessible via `/download/{id}` without authentication. Analysis of these files reveals complete FTP authentication sessions including cleartext passwords. The exposed credentials (nathan:Buck3tH4TF0RM3!) enable full FTP access to the user account, allowing retrieval of sensitive files including user flags.

2) **Unauthenticated Network Information Disclosure (Medium, CVSS 5.3)**
Three endpoints execute system commands and return output without authentication:
- `/ip` executes `ifconfig` - exposes IP addresses (10.129.2.140), MAC addresses, network masks
- `/netstat` executes `netstat` - reveals active connections, listening ports, process IDs
- `/capture` executes `tcpdump` - captures network traffic for 5-second intervals

3) **Server Header Disclosure (Low)**
The `Server: gunicorn` header reveals the Python WSGI HTTP server in use.

4) **Missing Security Headers (Informational)**
The application lacks standard security headers including X-Content-Type-Options, X-Frame-Options, Content-Security-Policy, X-XSS-Protection, and Strict-Transport-Security.

**Areas Tested Without Findings:**
- Command injection (RCE): All system command endpoints properly sanitize user input
- IDOR: Access controls correctly limit data access to existing capture IDs (0-2)
- Path traversal: File download functionality properly sanitizes path parameters
- Source code disclosure: No exposed Git repositories, backup files, or configuration files

# Recommendations

Recommendations are prioritized by urgency and potential risk reduction.

**Immediate Priority (Critical Risk)**

1. **Sanitize PCAP Files Before Distribution**
Implement automated scanning and sanitization of capture files to remove sensitive authentication data before making them available for download. Consider filtering FTP, Telnet, HTTP Basic Auth, and other cleartext protocols.

2. **Implement Authentication on All Endpoints**
Require valid user authentication for access to `/ip`, `/netstat`, `/capture`, `/data/{id}`, and `/download/{id}` endpoints. Implement authorization checks to ensure users can only access their own data.

3. **Use Encrypted Protocols**
Replace FTP with FTPS or SFTP to prevent credential transmission in cleartext. This eliminates the risk of credential exposure through traffic capture.

**Short-term Priority (Medium Risk)**

4. **Security Headers Implementation**
Add standard security headers to all HTTP responses:
- X-Content-Type-Options: nosniff
- X-Frame-Options: DENY
- Content-Security-Policy with appropriate directives
- X-XSS-Protection: 1; mode=block
- Strict-Transport-Security (if HTTPS is implemented)

5. **Access Logging and Monitoring**
Implement comprehensive logging for all access to system information endpoints and file downloads. Configure alerts for suspicious access patterns.

**Medium-term Priority (Defense in Depth)**

6. **Alternative Architecture**
Replace direct system command execution with a privileged statistics-gathering service that returns sanitized, structured data rather than raw command output.

7. **Network Segmentation**
If unauthenticated endpoints must be maintained, restrict access to trusted administrative networks only.

**Retest and Validation**
After remediation, conduct a focused retest to verify:
- PCAP files no longer contain cleartext credentials
- Authentication is enforced on all sensitive endpoints
- Security headers are present on all responses
- The original attack chain (download PCAP → extract creds → FTP login) is no longer viable

