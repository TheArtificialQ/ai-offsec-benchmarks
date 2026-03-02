# Security Penetration Test Report

**Generated:** 2026-02-28 14:31:02 UTC

# Executive Summary

A comprehensive security assessment of the cap.htb target environment was performed. The engagement identified a critical attack path that allowed for complete system compromise starting from an unauthenticated perspective.

Overall risk posture: Critical.

Key findings:
- An Insecure Direct Object Reference (IDOR) vulnerability was discovered in the web application's packet capture analysis feature. This allowed unauthorized access to sensitive PCAP files.
- Analysis of a compromised PCAP file revealed plaintext FTP credentials, which were successfully reused to gain SSH access to the system as a standard user.
- A privilege escalation vulnerability was identified due to insecure Linux capability assignments on the system's Python binary, allowing the standard user to execute arbitrary commands as the root user.

Business impact:
- The IDOR vulnerability presents a direct risk of exposing sensitive network traffic and credentials to unauthorized users.
- The combination of exposed credentials and the privilege escalation flaw results in a total loss of confidentiality, integrity, and availability of the target system, allowing an attacker full control over the host.

Remediation theme:
Immediate remediation efforts should focus on correcting the overly permissive Linux capabilities assigned to system binaries and implementing strict authorization controls on the web application's data retrieval endpoints. Furthermore, password reuse across services should be prohibited, and plaintext transmission of credentials must be eliminated.

# Methodology

The assessment followed industry-standard penetration testing methodologies, aligning with the OWASP Web Security Testing Guide (WSTG) for application-layer analysis and general network penetration testing practices for infrastructure assessment.

Engagement details:
- Assessment type: Black-box external penetration test, progressing to an internal authenticated review post-exploitation.
- Target environment: cap.htb (10.129.7.96)

Scope:
- All exposed network services (FTP, SSH, HTTP).
- The "Security Dashboard" web application hosted on port 80.
- Local privilege escalation vectors post-initial access.

Testing activities:
- Open port enumeration and service fingerprinting.
- Web application attack surface mapping and functional analysis.
- Authorization and access control testing (parameter tampering, IDOR).
- Post-exploitation enumeration (capabilities, SUID binaries, misconfigurations).

Validation standard:
All findings were manually validated. Concrete proof-of-concept exploits were developed to demonstrate the practical impact without causing operational disruption.

# Technical Analysis

This section details the confirmed vulnerabilities that formed the successful attack chain leading to full system compromise.

Severity model:
Findings are rated based on their exploitability and potential impact to the system's confidentiality, integrity, and availability.

1) Insecure Direct Object Reference (IDOR) in /data/ Endpoint (High)
The web application, which provides network analysis features, failed to enforce proper access controls on the `/data/<id>` endpoint. By altering the sequential ID parameter (e.g., accessing `/data/0`), it was possible to retrieve historical packet capture (PCAP) files without authentication. This lack of object-level authorization directly led to the disclosure of sensitive information.

2) Plaintext Credential Exposure and Reuse (High)
Analysis of the improperly accessed PCAP file (`0.pcap`) revealed an FTP session containing the plaintext credentials for the user `nathan`. These credentials (`Buck3tH4TF0RM3!`) were found to be valid for SSH access to the server, highlighting the risks of transmitting credentials over unencrypted protocols and reusing passwords across different services.

3) Privilege Escalation via Insecure Linux Capabilities (High)
Upon gaining SSH access as the user `nathan`, local enumeration revealed that the `/usr/bin/python3.8` binary was configured with the `cap_setuid` capability. This capability allows the executable to alter its process UID. An attacker can leverage this misconfiguration by invoking Python and instructing it to set its UID to 0 (root), effectively bypassing standard system access controls and achieving arbitrary code execution with root privileges.

Systemic themes:
- A failure to implement a robust, deny-by-default authorization model in the web application.
- Over-reliance on insecure configurations (broad capabilities) to facilitate system functionality, violating the principle of least privilege.

# Recommendations

The following remediation steps are recommended, prioritized by their impact on reducing the immediate risk of compromise.

Immediate priority:
1. Remediate Insecure Linux Capabilities: Immediately remove the `cap_setuid` capability from the `/usr/bin/python3.8` binary. This can be accomplished using the command: `sudo setcap -r /usr/bin/python3.8`. If the binary legitimately requires network binding capabilities, apply only those specific capabilities (e.g., `cap_net_bind_service+eip`) and ensure script execution is strictly controlled.
2. Enforce Strict Authorization Controls: Modify the web application to implement robust authorization checks on the `/data/` and `/download/` endpoints. Ensure that a user can only access resources they explicitly own or have been granted permission to view. Implement session-based verification before serving any sensitive files.

Short-term priority:
3. Address Plaintext Protocols and Credential Reuse: Deprecate the use of plaintext FTP in favor of secure alternatives such as SFTP or SCP. Initiate a mandatory password reset for the `nathan` user and enforce policies against password reuse across different services.
4. Implement Unpredictable Resource Identifiers: Replace the sequential integer IDs used for PCAP files with universally unique identifiers (UUIDs) to mitigate the risk of resource enumeration, providing defense-in-depth alongside proper authorization checks.

Medium-term priority:
5. Regular Security Audits: Institute a schedule for regular security audits of the system configuration, specifically focusing on SUID/SGID binaries and Linux capabilities, to detect and remediate similar misconfigurations proactively.
