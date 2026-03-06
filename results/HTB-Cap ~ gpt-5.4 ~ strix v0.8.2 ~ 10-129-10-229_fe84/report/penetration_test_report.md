# Security Penetration Test Report

**Generated:** 2026-03-05 19:49:06 UTC

# Executive Summary

An external penetration test of the target host at 10.129.10.229 identified a clear attack chain from unauthenticated web access to full root compromise.

Overall risk posture: High.

The most significant weakness was an insecure direct object reference in the web-based packet capture functionality. Historical PCAP files were accessible through predictable numeric identifiers without appropriate authorization checks. One exposed capture contained cleartext FTP credentials for a valid local user. Those credentials provided authenticated access to the host.

A second high-severity weakness was a dangerous Linux capability assigned to the Python interpreter. Because the interpreter had CAP_SETUID, the recovered low-privilege account could elevate directly to root with a single command. This resulted in complete host compromise.

Confirmed flag recovery:
user.txt: `REDACTED`
root.txt: `REDACTED`

The overall remediation theme is to eliminate direct object reference exposure in the web application, remove unsafe privilege assignments from the operating system, and retire insecure protocols that leak reusable credentials.

# Methodology

The assessment was conducted as an external black-box penetration test against the in-scope host 10.129.10.229 and the associated virtual host cap.htb.

Methodology aligned with standard web and infrastructure testing practices, including:
Reconnaissance and service enumeration
Web application route discovery and response analysis
Object access testing for insecure direct references
Sensitive data exposure review of downloadable artifacts
Credential validation across exposed services
Privilege escalation analysis following authenticated access
Live vulnerability applicability checks against current public sources, including NVD and Ubuntu security notices, where relevant to detected product versions

In-scope services identified during testing:
FTP on TCP/21 running vsftpd 3.0.3
SSH on TCP/22 running OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
HTTP on TCP/80 serving a Flask application behind Gunicorn

Only validated issues with demonstrated impact were reported as findings.

# Technical Analysis

Severity ratings reflect realistic exploitability and impact to confidentiality, integrity, and availability.

Confirmed findings

1. Insecure Direct Object Reference in PCAP download and analysis endpoints
Severity: High
The web application exposed packet capture files through predictable numeric identifiers at /download/<id> and /data/<id> without enforcing ownership or access control. Accessing a prior capture returned sensitive traffic from another session, including cleartext FTP credentials. This enabled authenticated access as a valid local user. Detailed reproduction and evidence are documented in vulnerability report vuln-0001.

2. Dangerous Linux capability assignment on Python interpreter enabling root escalation
Severity: High
The host assigned CAP_SETUID to /usr/bin/python3.8. After authenticated access was obtained, this allowed immediate escalation from the nathan account to root by invoking os.setuid(0) from Python. Root-level command execution and disclosure of root-only content were confirmed. This is an environmental privilege management failure rather than a vendor software defect. Detailed reproduction and evidence are documented in vulnerability report vuln-0002.

Observed product and version context
The target exposed vsftpd 3.0.3, OpenSSH 8.2p1 Ubuntu 4ubuntu0.2, and a Python 3.8.5-based Flask application. Live retrieval against NVD identified CVE-2021-30047 as a denial-of-service issue affecting vsftpd 3.0.3, with Exploit-DB reference https://www.exploit-db.com/exploits/49719. NVD source: https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=vsftpd%203.0.3
This issue was not required for compromise and was not validated as part of the attack chain.
Live retrieval against NVD and Ubuntu notices did not identify an applicable Python product CVE for the confirmed privilege escalation scenario; the root cause was unsafe host capability configuration, not an upstream Python flaw.
No directly applicable OpenSSH CVE was validated from the live checks performed for the detected Ubuntu package version.

Systemic themes and root causes
Authorization controls were absent from object-backed download paths.
Sensitive network artifacts were exposed and contained reusable secrets because insecure FTP was in use.
Privilege boundaries on the host were weakened by assigning a powerful capability to a general-purpose interpreter.

# Recommendations

Immediate priority

1. Remove unauthorized access to packet captures
Enforce authentication and object-level authorization on all capture-related routes. Ensure each capture is bound to its owner and cannot be accessed by requesting a different identifier.

2. Remove dangerous capabilities from Python
Revoke CAP_SETUID from /usr/bin/python3.8 and any other general-purpose interpreter or shell. Reassess the design of privileged capture functions so they do not depend on broad interpreter-level privilege changes.

3. Rotate exposed credentials
Treat the recovered FTP credentials as compromised. Reset affected passwords and review any credential reuse across services.

Short-term priority

4. Replace FTP with encrypted alternatives
Disable FTP for interactive access and migrate to SFTP or SSH-based file transfer. This prevents reusable credentials from appearing in traffic captures.

5. Review stored and historical captures
Identify and securely remove sensitive captures that may contain credentials or internal traffic. Establish retention limits and access logging for future capture artifacts.

6. Harden privileged workflows
Where packet capture or other privileged actions are required, use a narrowly scoped helper with minimal privileges rather than direct execution through Python or shell commands.

Medium-term priority

7. Add authorization regression testing
Implement routine testing for object-level access control failures across all routes that accept user-controlled identifiers.

8. Review host hardening baseline
Audit file capabilities, sudo rights, service configurations, and exposed management pathways to ensure no additional privilege-escalation opportunities remain.

Retest guidance
A focused retest should confirm that historical captures cannot be accessed cross-user, that FTP is disabled or replaced, that compromised credentials are invalidated, and that Python no longer permits setuid-based privilege escalation.

