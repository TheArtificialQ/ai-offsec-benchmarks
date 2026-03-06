# Security Penetration Test Report

**Generated:** 2026-03-05 11:46:34 UTC

# Executive Summary

The security assessment of the target environment (fluffy.htb) was conducted successfully. The assessment identified multiple critical and high-severity vulnerabilities affecting the systems. During the reconnaissance phase, several key software applications and services were discovered, including KeePass version 2.58 and Everything version 1.4.1.1026. Furthermore, an internal patch announcement document (`Upgrade_Notice.pdf`) explicitly detailed several recent 2025 vulnerabilities affecting the environment, highlighting the urgent need for remediation. The target also features a background automation process that automatically extracts ZIP archives uploaded to the `IT` SMB share, which is vulnerable to a known path traversal flaw (Zip Slip).

The primary risk identified is the potential for arbitrary file write and subsequent system compromise through the vulnerable background archive extraction process, combined with local NTLM and File Explorer spoofing flaws. These issues could allow an attacker to bypass security controls, escalate privileges, and compromise sensitive data.

# Methodology

The assessment was conducted as a gray-box penetration test against the internal network environment. The methodology aligned with industry-standard frameworks, including OSSTMM and OWASP guidelines.

Testing activities included:
1. Network and Port Scanning: comprehensive TCP and UDP port scans to map the attack surface, identifying standard Active Directory services (LDAP, Kerberos, SMB, WinRM).
2. Service Enumeration: Unauthenticated and authenticated enumeration of exposed SMB shares, discovering the `IT` share containing software installation files and an internal `Upgrade_Notice.pdf` memo.
3. Vulnerability Research: Live retrieval of vulnerability details for discovered software and referenced CVEs using public vulnerability databases (NVD) and exploit repositories (GitHub).
4. Automated Process Analysis: Testing of the background ZIP extraction mechanism on the `IT` share to determine its behavior and susceptibility to path traversal attacks.

Constraints: Testing was performed from within an isolated container environment that prohibited inbound connections (e.g., reverse shells or external NTLM relay listeners). As such, full exploitation chains requiring external callbacks could not be completed. The user.txt and root.txt flags could not be retrieved within the iteration limits.

# Technical Analysis

The technical analysis revealed several vulnerabilities relevant to the target environment:

1. Path Traversal in Archive Extraction (CVE-2025-3445)
The target runs a background process that automatically extracts ZIP files uploaded to the `IT` SMB share. Based on the internal memo, this process uses the Go `mholt/archiver` library, which is vulnerable to a "Zip Slip" path traversal attack (CVE-2025-3445). An attacker can craft a ZIP file containing path traversal symlinks (`../../`) to overwrite arbitrary files on the system with the privileges of the extraction service.

2. Windows File Explorer and NTLM Spoofing (CVE-2025-24071 & CVE-2025-24996)
The target environment is vulnerable to recent Windows Explorer spoofing and NTLM hash leak vulnerabilities. CVE-2025-24071 allows an attacker to leak NTLM hashes by placing a malicious `.library-ms` file inside a RAR/ZIP archive, which triggers an SMB authentication request when extracted by Windows Explorer. CVE-2025-24996 involves external control of file paths in Windows NTLM, allowing spoofing over a network. 

3. Additional Internal Vulnerabilities Referenced
The internal upgrade notice also identified the following vulnerabilities requiring patching in the environment:
- CVE-2025-46785: Zoom Workplace Buffer Over-read (DoS)
- CVE-2025-29968: AD CS Improper Input Validation (DoS)
- CVE-2025-21193: AD FS Spoofing Vulnerability

4. Stored Software Versions
The `IT` share contained installers for KeePass 2.58 and Everything 1.4.1.1026. While KeePass 2.58 mitigates previous cleartext password recovery flaws (CVE-2023-32784), the Everything installation presents potential local privilege escalation risks if low-privileged users can write to its installation directory (e.g., CVE-2020-24567, DLL hijacking via `urlmon.dll`).

# Recommendations

The following remediation steps are recommended to secure the environment:

1. Remediate the Zip Slip Vulnerability (Immediate)
The automated background process utilizing `mholt/archiver` must be updated or replaced. The `mholt/archiver` project is deprecated, and its successor, `mholt/archives`, removes the vulnerable `Unarchive()` functionality. Ensure that any automated archive extraction logic explicitly sanitizes and validates file paths to prevent traversal outside the intended extraction directory.

2. Patch Windows Spooling and NTLM Vulnerabilities (Immediate)
Apply the latest Microsoft security updates to mitigate CVE-2025-24071 and CVE-2025-24996. Prevent NTLM hash leaks by restricting outbound SMB traffic (blocking port 445) at the network perimeter and enforcing SMB signing across the domain.

3. Restrict SMB Share Permissions (Short-term)
Review and restrict write access to the `IT` share. Standard users (such as `j.fleischman`) should not have write privileges to a directory where automated administrative processes operate, as this creates a vector for code execution and file overwrite attacks.

4. Apply Application Updates (Short-term)
Proceed with the scheduled maintenance to apply patches for AD CS (CVE-2025-29968), AD FS (CVE-2025-21193), and Zoom Workplace (CVE-2025-46785) as outlined in the internal memo.

