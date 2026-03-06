# Security Penetration Test Report

**Generated:** 2026-03-06 20:12:58 UTC

# Executive Summary

The security assessment of the target system (underpass.htb) revealed critical security vulnerabilities that permitted full system compromise. The primary point of entry was the discovery of a publicly exposed daloRADIUS management portal running with default administrative credentials. This initial foothold allowed an attacker to access sensitive user information, including poorly-hashed (MD5) user passwords. Cracking these hashes yielded valid credentials that were reused for SSH access to the underlying operating system. Once authenticated to the server as a low-privileged user, a misconfiguration in the sudo privileges related to the `mosh-server` binary was exploited to easily escalate privileges to full `root` control.

The overall risk posture is assessed as Critical. This combination of default credentials, insecure cryptography, password reuse, and overly permissive system configurations demonstrates a lack of basic security hygiene, directly resulting in complete unauthorized control over the server environment.

To remediate these issues, the organization should immediately enforce strong, unique passwords across all systems and portals, remove unnecessary administrative interfaces from public view, and restrict sudo access to the minimum required for standard operations.

# Methodology

The assessment was performed as a grey-box penetration test against the host at IP address 10.129.231.213, with the internal hostname underpass.htb configured. Testing was conducted remotely via network access and focused on discovering external attack surface, identifying vulnerabilities in exposed web applications and services, and ultimately achieving internal lateral movement and privilege escalation.

High-level testing activities included:
- Full TCP port scanning and service enumeration using Nmap.
- Web application directory brute-forcing and enumeration with ffuf.
- Manual verification and interaction with discovered web portals (daloRADIUS).
- Identification of weak and default authentication credentials.
- Extraction and offline cracking of weak cryptographic hashes (MD5).
- Credential stuffing and password reuse attacks against exposed SSH services.
- Post-exploitation enumeration and privilege escalation via sudo misconfigurations (GTFOBins techniques).

The assessment utilized industry-standard penetration testing methodologies aligned with the OWASP Web Security Testing Guide (WSTG) and OSSTMM.

# Technical Analysis

During the reconnaissance phase, an Apache web server and an OpenSSH server were found running on ports 80 and 22, respectively. Directory enumeration of the web server identified the `/daloradius/` path, hosting a daloRADIUS 2.2 beta management interface.

The following critical vulnerabilities were identified and chained together to compromise the host:

1. Default Credentials in daloRADIUS (Critical - CVSS 9.8)
The daloRADIUS management application at `http://underpass.htb/daloradius/app/operators/login.php` was accessible using the default operator credentials (`administrator`:`radius`). This allowed unauthenticated remote access to the internal administration portal.

2. Insecure Password Storage and Information Disclosure (Medium)
Through the daloRADIUS user management interface (`mng-list-all.php`), the attacker viewed configuration details for the user `svcMosh`. This exposed the user's password hash in the `7__MD5-Password` attribute. The use of unsalted MD5 (an obsolete cryptographic hashing algorithm) made the hash trivial to crack offline, yielding the plaintext password `underwaterfriends`.

3. Password Reuse and Inadequate Remote Access Controls (High)
The cracked password (`underwaterfriends`) was found to be valid for SSH authentication as the `svcMosh` user. This indicated dangerous password reuse across different authentication domains (web application and operating system). Upon logging in via SSH, the flag `user.txt` (31e96a90eb85afc479b91df75d34d4fc) was retrieved.

4. Privilege Escalation via Sudo Misconfiguration of mosh-server (High - CVSS 7.8)
After gaining local shell access, standard post-exploitation enumeration revealed that the `svcMosh` user had `sudo` privileges to execute `/usr/bin/mosh-server` without a password. Because `mosh-server` allows arbitrary command execution as part of session instantiation, it was leveraged to execute `/bin/bash` under the context of `root`. The attacker connected using `mosh-client` and the generated connection key, obtaining a root shell and recovering `root.txt` (a0868ade3e175dbe2e1529d51d4c754b).

# Recommendations

The following remediation steps are recommended, prioritized by urgency:

Immediate Priority:
1. Change Default Passwords: Immediately rotate the passwords for all default accounts within the daloRADIUS application (e.g., `administrator`) to strong, unique, and complex passphrases.
2. Revoke Excessive Sudo Privileges: Modify the `/etc/sudoers` file to remove the `NOPASSWD` entry for `mosh-server` for the user `svcMosh`. If the user strictly requires access to specific functions, wrap the execution in a hardened script that does not permit arbitrary command flags (like `-- COMMAND`).
3. Audit User Credentials: Change the password for the local `svcMosh` user and ensure that passwords are not shared across web applications and operating system accounts.

Short-term Priority:
4. Upgrade Cryptographic Mechanisms: Reconfigure daloRADIUS and backend databases to use strong, modern password hashing algorithms (e.g., Argon2, bcrypt, or at minimum SHA-256 with unique salts) rather than obsolete algorithms like MD5.
5. Limit Management Interface Exposure: Restrict access to the daloRADIUS `/daloradius/` path. Implement IP-based access control lists (ACLs) so that the administrative portal is only accessible from trusted internal management networks or via a VPN.

Medium-term Priority:
6. Implement Multi-Factor Authentication (MFA): Require MFA for SSH access and administrative web portals.
7. Conduct Regular Security Awareness Training: Educate administrators and developers on the risks of default credentials and password reuse.

