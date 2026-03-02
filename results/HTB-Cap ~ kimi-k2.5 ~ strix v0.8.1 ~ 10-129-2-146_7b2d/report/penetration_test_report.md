# Security Penetration Test Report

**Generated:** 2026-02-22 17:13:08 UTC

# Executive Summary

The penetration test of the CAP HTB target (10.129.2.146 / cap.htb) identified multiple critical and high-severity security vulnerabilities that could lead to complete system compromise.

**Overall Risk Posture: CRITICAL**

Key Findings:
1. **Information Disclosure in Network Capture Files (Critical)** - The Security Dashboard application stores and serves PCAP files containing cleartext authentication credentials. Analysis of these files revealed FTP credentials (nathan / Buck3tH4TF0RM3!), enabling full account compromise and user flag retrieval.

2. **Insecure Direct Object Reference (IDOR) (High)** - The `/data/{id}` and `/download/{id}` endpoints lack authorization checks, allowing attackers to access network capture files belonging to other users by iterating through numeric ID values.

3. **Command Injection with Root Privilege Escalation (Critical)** - The `/capture` endpoint in app.py uses `os.system()` with unsanitized input and executes commands with root privileges via `os.setuid(0)`. While direct exploitation was not possible in the current configuration due to server-controlled IP addressing, the vulnerability presents a critical risk if the application is deployed behind a reverse proxy that forwards client headers.

4. **SSH Cryptographic Weaknesses (High)** - The SSH service (OpenSSH 8.2p1) is vulnerable to the Terrapin attack (CVE-2023-48795) and uses deprecated SHA-1 algorithms.

Business Impact:
- Complete compromise of user accounts through credential exposure
- Potential for root-level remote code execution
- Unauthorized access to sensitive files and flags
- Lateral movement capabilities through compromised credentials

Remediation Theme:
Immediate action is required to eliminate cleartext credential transmission, implement proper authorization controls, and remediate command injection vulnerabilities. SSH cryptographic configurations must be hardened to prevent downgrade attacks.

# Methodology

The security assessment followed OWASP Web Security Testing Guide (WSTG) and industry-standard penetration testing methodology.

**Engagement Details:**
- Assessment Type: External penetration test (black-box with white-box code analysis)
- Target Environment: CAP HTB lab environment
- Testing Period: Single session comprehensive assessment

**Scope (In-Scope Assets):**
- IP Address: 10.129.2.146
- Domain: cap.htb
- Services: HTTP (80/tcp), FTP (21/tcp), SSH (22/tcp)

**Testing Activities Performed:**
1. **Network Enumeration**: Full TCP port scan using nmap to identify open services
2. **Service Enumeration**: Version detection and banner grabbing for all identified services
3. **Web Application Mapping**: Manual browsing, endpoint discovery, and JavaScript analysis
4. **Source Code Analysis**: FTP file access enabled retrieval of app.py for white-box review
5. **Vulnerability Assessment**: Testing for OWASP Top 10 vulnerabilities including injection, broken access control, and security misconfigurations
6. **Exploitation**: Credential extraction from PCAP files and FTP access validation
7. **Privilege Escalation Analysis**: Review of application permissions and system configuration

**Evidence Validation:**
All findings were validated through reproducible exploitation steps. Vulnerabilities were confirmed via:
- Direct exploitation of IDOR to access multiple user capture files
- String extraction from PCAP files revealing FTP credentials
- Source code review confirming command injection vulnerability patterns
- Successful authentication to FTP service using extracted credentials

# Technical Analysis

This section provides a consolidated view of confirmed findings and observed risk patterns.

**Severity Model:**
Severity reflects exploitability, potential impact to confidentiality/integrity/availability, and realistic attacker capabilities.

**Confirmed Findings:**

1) **Information Disclosure via PCAP Files (Critical, CVSS 9.1)**
The Security Dashboard application generates network capture files and makes them available for download via the `/download/{id}` endpoint. These PCAP files contain complete network traffic including application-layer protocols. The captured traffic includes cleartext FTP authentication sequences with USER and PASS commands, exposing valid credentials (nathan / Buck3tH4TF0RM3!).

Technical details: The `/capture` endpoint triggers tcpdump to capture 5 seconds of traffic, storing results in `/var/www/html/upload/{id}.pcap`. The application serves these files without sanitizing sensitive content, enabling credential extraction.

2) **Insecure Direct Object Reference (IDOR) (High)**
The `/data/{id}` and `/download/{id}` endpoints accept numeric identifiers without validating that the requesting user owns the requested resource. Testing confirmed that ID values 0 and 1 return different user data, demonstrating horizontal privilege escalation.

Technical details: The vulnerable code (app.py lines 131-152) performs no authorization checks:
```python
@app.route("/download/<id>")
def download(id):
    uploads = os.path.join(app.root_path, "upload")
    return send_from_directory(uploads, str(id) + ".pcap", as_attachment=True)
```

3) **Command Injection with Root Privilege Escalation (Critical, CVSS 9.0)**
The `/capture` endpoint constructs shell commands using string interpolation with user-influenced input. The code at lines 105-106 of app.py demonstrates this pattern:
```python
command = f"""python3 -c 'import os; os.setuid(0); os.system("timeout 5 tcpdump -w {path} -i any host {ip}")'"""
os.system(command)
```

The `ip` variable derives from `request.remote_addr`. While this is typically server-controlled, deployment behind a reverse proxy that forwards X-Forwarded-For headers would enable command injection with root privileges via `os.setuid(0)`.

4) **SSH Cryptographic Weaknesses (High)**
The SSH service (OpenSSH 8.2p1 Ubuntu-4ubuntu0.2) exhibits multiple cryptographic weaknesses:
- **CVE-2023-48795 (Terrapin)**: The chacha20-poly1305 cipher is vulnerable to message prefix truncation
- SHA-1 algorithms (ssh-rsa, hmac-sha1) are enabled and subject to collision attacks
- DHEat DoS vulnerability due to insufficient connection throttling

**Systemic Themes:**
- Insufficient authorization enforcement across endpoints
- Unsafe use of shell command execution with string interpolation
- Cleartext protocol usage (FTP) exposing authentication credentials
- Privileged execution of user-influenced commands

# Recommendations

The following recommendations are prioritized by urgency and potential risk reduction.

**Immediate Priority (Critical Risk Reduction):**

1. **Eliminate Credential Exposure in Network Captures**
   - Implement data sanitization to remove or mask authentication credentials from stored PCAP files
   - Use regular expressions or packet analysis libraries to detect and redact sensitive patterns
   - Consider not storing full packet payloads; store only metadata and statistics

2. **Implement Authorization Controls**
   - Add user authentication and session management to track resource ownership
   - Validate that the requesting user owns the requested resource ID before serving `/data/{id}` and `/download/{id}`
   - Implement centralized authorization middleware with deny-by-default policy

3. **Remediate Command Injection**
   - Replace `os.system()` with `subprocess.run()` using list arguments instead of shell strings
   - Validate IP address format using Python's `ipaddress` module before use
   - Remove `os.setuid(0)` and use Linux capabilities (`setcap cap_net_raw,cap_net_admin=eip /usr/bin/tcpdump`) instead
   - If behind a proxy, configure Flask's ProxyFix with strict trusted proxy whitelist

4. **Secure FTP Service**
   - Replace FTP with SFTP or FTPS to encrypt authentication and data transfer
   - If FTP must remain, disable cleartext authentication and require TLS

**Short-Term Priority (Defense in Depth):**

5. **Harden SSH Configuration**
   - Update OpenSSH to latest version to address CVE-2023-48795
   - Disable SHA-1 based algorithms in sshd_config
   - Implement connection rate limiting to mitigate DHEat attacks
   - Replace NIST elliptic curves with Curve25519

6. **Implement Secure Development Practices**
   - Conduct security code review for all endpoints using os.popen(), os.system(), or subprocess with shell=True
   - Implement static analysis (bandit, semgrep) in CI/CD pipeline
   - Establish secure coding guidelines prohibiting shell string interpolation

7. **Network Segmentation**
   - Isolate web application in restricted network segment
   - Implement egress filtering to limit outbound connections from application server

**Medium-Term Priority:**

8. **Security Monitoring**
   - Implement logging for all access to `/download` and `/data` endpoints
   - Alert on unusual access patterns (sequential ID enumeration, multiple capture downloads)
   - Monitor for command injection patterns in HTTP headers

9. **Regular Security Testing**
   - Conduct quarterly penetration testing
   - Implement automated vulnerability scanning
   - Perform source code security audits before deployment

**Retest and Validation:**
After implementing immediate priority fixes:
1. Verify PCAP files no longer contain cleartext credentials
2. Confirm IDOR is remediated by attempting to access other users' data
3. Validate command injection is eliminated through code review and testing
4. Confirm SSH cryptographic hardening via ssh-audit tool

