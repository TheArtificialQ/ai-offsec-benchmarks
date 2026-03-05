# Security Penetration Test Report

**Generated:** 2026-03-04 16:25:35 UTC

# Executive Summary

The penetration test of expressway.htb is complete. Only SSH port 22 is open. The service exhibits abnormal protocol behavior (DISCONNECT after KEXINIT) preventing standard handshake. No vulnerabilities were exploited, no credentials obtained, and flags not retrieved. The target appears to be a Cisco Expressway device with restricted SSH access. All standard attack vectors (brute force, default credentials, CVEs) failed. The attack surface is minimal and effectively non-exploitable given current capabilities.

# Methodology

Black-box testing per OWASP WSTG and NIST SP 800-115. Comprehensive port scanning (TCP 1-65535, UDP key ports), service enumeration, protocol analysis, vulnerability research (OpenSSH, Cisco Expressway), authentication testing (default credentials, KEXINIT bypass technique), web interface probing, DNS/subdomain enumeration, and filesystem inspection. All findings validated through reproducible testing.

# Technical Analysis

Single open port: 22/tcp SSH (OpenSSH_10.0p2 Debian-8). Port 22 behavior: server sends SSH banner, then upon client identification returns SSH_MSG_DISCONNECT containing an embedded KEXINIT packet. This prevents standard SSH clients (OpenSSH, paramiko) from completing key exchange and reaching authentication. The anomaly suggests a Cisco Expressway product with custom security configuration (tunneling-only or ForceCommand). All other ports (TCP 1-65535 except 22, UDP 5060/5061/1720/etc.) are closed. Web ports (80,443,8080,8443,9443,9090) are closed. No unauthenticated vulnerabilities in OpenSSH 10.0p2. Tested 40+ credentials via KEXINIT bypass; all rejected. CTF flags not accessible.

# Recommendations

Immediate: 1) Verify Expressway firmware version; if pre-X7.2.4 or X8 pre-X8.1.2, patch CVE-2015-0653 (CVSS 10.0 auth bypass). 2) Audit SSH config (ForceCommand, Match blocks, PermitTTY, AllowTcpForwarding). 3) Ensure strong unique passwords; disable default accounts. Short-term: 4) Restrict SSH forwarding if unused; 5) Implement egress firewall rules; 6) Enable SSH audit logging; 7) Isolate web management interface to internal/admin networks. Long-term: 8) Deploy IDS/IPS for SSH brute-force detection; 9) Consider MFA; 10) Regular key rotation. Retest after patches/config changes.

