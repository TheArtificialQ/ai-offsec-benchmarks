# Security Penetration Test Report

**Generated:** 2026-02-22 17:32:12 UTC

# Executive Summary

An external penetration test of the CAP Security Dashboard application identified two critical security vulnerabilities that enable complete system compromise. The assessment discovered an Insecure Direct Object Reference (IDOR) vulnerability that exposes sensitive network capture data, combined with a Linux capabilities misconfiguration that allows privilege escalation to root.

Overall risk posture: Critical.

Key findings include an IDOR vulnerability in the PCAP download feature that exposed cleartext FTP credentials, and a Python interpreter with cap_setuid capability that enables trivial privilege escalation. Together, these vulnerabilities create a complete attack chain from unauthenticated external access to full root control.

Business impact includes potential unauthorized access to all system data, installation of persistent backdoors, and use of the compromised system as a pivot point for attacking additional network resources.

# Methodology

The assessment was conducted following industry-standard penetration testing methodology aligned with OWASP Web Security Testing Guide (WSTG).

Engagement details:
- Assessment type: External penetration test (black-box)
- Target environment: Production-equivalent Linux server

Scope (in-scope assets):
- Web application: http://cap.htb
- SSH service: port 22
- FTP service: port 21

Testing activities performed:
- Port scanning and service enumeration
- Web application discovery and mapping
- Authorization and access control testing
- Network traffic analysis
- Linux capabilities enumeration
- Privilege escalation testing

# Technical Analysis

Two confirmed vulnerabilities were identified that chain together for complete system compromise:

1) IDOR in PCAP Download Endpoint (Critical - CVSS 9.1)
The Security Dashboard application's capture feature assigns sequential numeric IDs to packet captures. The /data/{id} and /download/{id} endpoints lack authorization checks, allowing any user to access captures belonging to other users. One such capture contained cleartext FTP authentication credentials that provided initial system access.

2) Python cap_setuid Privilege Escalation (High - CVSS 7.8)
The Python 3.8 interpreter binary was assigned the cap_setuid Linux capability, which allows any process executing Python to change its effective user ID to root. This misconfiguration enables any user with shell access to trivially escalate privileges.

Attack chain executed:
- Access /download/0 to retrieve PCAP file from another user's session
- Extract cleartext FTP credentials from captured network traffic
- Authenticate via SSH using obtained credentials
- Exploit Python cap_setuid capability to escalate to root

# Recommendations

Immediate priority:
1. Implement session-based authorization on /data/{id} and /download/{id} endpoints to verify capture ownership
2. Remove cap_setuid capability from Python: setcap -r /usr/bin/python3.8
3. Use secure file transfer protocols (SFTP/FTPS) instead of FTP to prevent credential exposure

Short-term priority:
4. Implement authentication for the Security Dashboard application
5. Audit all Linux capabilities assignments with getcap -r /
6. Apply network segmentation to isolate security monitoring tools

Medium-term priority:
7. Implement logging and alerting for access to sensitive capture data
8. Conduct security review of capability assignments on all systems
9. Establish regular security assessments and configuration audits

