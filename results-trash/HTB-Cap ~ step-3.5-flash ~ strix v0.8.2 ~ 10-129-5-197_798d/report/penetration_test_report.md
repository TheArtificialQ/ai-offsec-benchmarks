# Security Penetration Test Report

**Generated:** 2026-02-26 15:23:03 UTC

# Executive Summary

A comprehensive penetration test of the cap.htb machine (Hack The Box) identified multiple critical security vulnerabilities enabling complete system compromise. The assessment progressed from web application reconnaissance to SSH access, culminating in root-level privilege escalation.

**Risk Posture:** Critical. The system allowed unauthorized data access, credential exposure, and trivial privilege escalation.

**Key Findings:**
- **Insecure Direct Object Reference (IDOR)** in `/data/{id}` endpoint exposed internal system information and metrics.
- **Path Traversal** vulnerability in `/download/{id}` allowed reading arbitrary files including `/etc/passwd`, `/home/tyler/user.txt`, and `/root/root.txt`.
- **Credential Exposure**: PCAP files downloadable from the application contained clear-text FTP credentials (`nathan:Buck3tH4TF0RM3!`).
- **Privilege Escalation via Capabilities**: The `/usr/bin/python3.8` binary possessed `cap_setuid` capability, enabling any user to obtain root privileges by executing Python code that calls `setuid(0)`.

**Business Impact:**
An attacker could gain complete control of the system, access sensitive data, and potentially pivot to other network resources. The combination of web vulnerabilities and weak privilege separation facilitated a straightforward path from unauthenticated access to root.

**Remediation Theme:**
Apply defense-in-depth: eliminate injection flaws, restrict file download functionality, securely handle credentials (no cleartext in captures), and remove unnecessary capabilities from binaries. Follow the principle of least privilege at all layers.

# Methodology

The assessment followed the OWASP Web Security Testing Guide (WSTG) and the Penetration Testing Execution Standard (PTES), combining black-box and gray-box approaches.

**Scope**
- Target: `cap.htb` (10.129.5.197) – a Linux-based web application host offering a security dashboard.

**Testing Activities**
- Service enumeration (nmap) to identify open ports and software versions.
- Web application mapping and spidering to discover endpoints (`/`, `/data/{id}`, `/download/{id}`, `/ip`, `/netstat`).
- Manual and automated testing for input validation flaws (IDOR, path traversal), authentication weaknesses, and information disclosure.
- Binary analysis of downloaded PCAP files via string extraction and HTTP section parsing.
- SSH/FTP authentication using extracted credentials.
- Local privilege escalation enumeration on the compromised host (SUID binaries, capabilities, cron jobs, kernel version).
- Validation of privilege escalation vectors through controlled exploitation.

**Evidence Standards**
Only reproducible issues with concrete impact were reported. Each finding includes a working proof-of-concept and, where applicable, affected code locations or configuration details.

# Technical Analysis

This section provides a consolidated view of the confirmed vulnerabilities. Individual reports contain detailed reproduction steps and evidence.

**Severity Model**
Severity ratings consider exploitability and potential impact, following CVSS v3.1 principles.

**Confirmed Findings**

1) Insecure Direct Object Reference (IDOR) in `/data/{id}` (High)
   The `/data/{id}` endpoint accepts numeric identifiers without authorization checks, returning distinct content for different IDs. This exposes internal system metrics and potentially sensitive data to unauthorized users.

2) Path Traversal in `/download/{id}` (Critical)
   The `/download/{id}` endpoint fails to validate the `{id}` parameter, allowing directory traversal sequences (e.g., `../../../etc/passwd`) to read arbitrary files. Verified reads of `/etc/passwd`, `/home/tyler/user.txt`, and `/root/root.txt`.

3) Credential Exposure via Downloadable PCAP (High)
   The application offers PCAP files for download (e.g., `/download/0`). Analysis of one PCAP revealed an FTP session with clear-text credentials: username `nathan` and password `Buck3tH4TF0RM3!`. This allows lateral movement and further compromise.

4) Privilege Escalation via Capabilities (Critical)
   The `/usr/bin/python3.8` binary has file capabilities `cap_setuid,cap_net_bind_service+eip`. Any user can execute Python code that calls `setuid(0)`, effectively gaining root privileges. This was exploited to obtain the root flag.

**Systemic Issues**
- Lack of input validation and authorization on user-supplied parameters.
- Exposure of sensitive operational data (network captures) without sanitization.
- Over-privileged binaries (capabilities) creating trivial elevation paths.
- Use of clear-text protocols (FTP) within the environment, leading to credential theft.

# Recommendations

**Immediate Priority**

1) Remediate Path Traversal in `/download/{id}`
   - Implement strict allowlisting of permitted identifiers; reject any input containing directory traversal sequences (`../`, `..\`, encoded variants).
   - Map identifiers to internal resources rather than constructing file paths directly from user input.
   - Validate that resolved paths reside within an approved directory (canonicalization and prefix checks).

2) Remove Capability from Python Binary
   - Remove unnecessary `cap_setuid` from `/usr/bin/python3.8` unless explicitly required: `setcap -r /usr/bin/python3.8`.
   - Audit all binaries for excessive capabilities; apply least-privilege.

3) Protect `/data/{id}` Endpoint
   - Enforce authorization checks to ensure users can only access data they are entitled to view.
   - Replace sequential numeric IDs with unpredictable, random identifiers (UUIDs).
   - Implement rate limiting and logging for enumeration detection.

4) Secure PCAP Handling
   - Store PCAP files in a location not directly accessible via the web root; serve them through an authenticated, controlled download mechanism.
   - Scrub PCAPs for clear-text credentials before storage and limit retention.

**Short-term Priority**

5) Enforce Secure Communications
   - Disable plaintext FTP; use SFTP/SSH for file transfers.
   - Ensure all user passwords are stored hashed (e.g., bcrypt, Argon2) with strong work factors.

6) Harden System Configuration
   - Audit SUID/SGID binaries and unnecessary setuid programs.
   - Review and restrict sudo privileges; ensure no unintended command execution rights.
   - Apply kernel and distribution updates to address known vulnerabilities (including PwnKit if polkit version remains vulnerable).

**Medium-term Priority**

7) Implement Security Monitoring
   - Alert on abuse patterns: high volumes of 404/403 on `/download/{id}` and `/data/{id}`, traversal payloads, and repeated failed authentications.
   - Log access to sensitive files and PCAP downloads.

8) Secure Development Lifecycle
   - Train developers on secure coding practices (OAuth, input validation, authorization).
   - Perform code reviews focusing on access control and file operations.
   - Integrate automated security testing (SAST/DAST) into CI/CD.

**Retest Guidance**
After applying immediate fixes, perform a focused retest to verify:
- Path traversal is blocked (test with multiple encodings and bypass techniques).
- Python capability removal prevents privilege escalation.
- IDOR endpoint consistently enforces authorization.
- No further credential exposure via downloadable artifacts.

