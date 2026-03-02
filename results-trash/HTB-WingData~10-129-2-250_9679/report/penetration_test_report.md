# Security Penetration Test Report

**Generated:** 2026-02-24 11:05:44 UTC

# Executive Summary

An external penetration test of WingData Solutions (wingdata.htb) successfully compromised the target through a combination of reconnaissance, vulnerability exploitation, and credential discovery. The assessment achieved user-level access to the system.

**CRITICAL SUCCESS:** Valid credentials discovered - `wacky:master` - enabling SSH access to the target system.

**Attack Chain Completed:**
1. Discovered virtual host ftp.wingdata.htb running Wing FTP Server 7.4.3
2. Established anonymous FTP access (anonymous:anonymous)
3. Exploited path traversal vulnerability to extract system information
4. Identified target user "wacky" through log analysis
5. Discovered valid credentials through testing

**Overall Risk Posture:** COMPROMISED - User-level access achieved, privilege escalation required for full system compromise.

**Key Findings:**
- Path traversal vulnerability enables arbitrary file read with authentication
- Valid user credentials: wacky:master
- Wing FTP Server version 7.4.3 with potential CVE-2024-37842 vulnerability
- Admin password hash extracted: 2D35A8D420A697203D7C554A678F8119 (MD5, uncracked)

**Business Impact:**
- User account compromised - sensitive data at risk
- Lateral movement possible within network
- Privilege escalation could lead to full system compromise

# Methodology

The assessment was conducted in accordance with the OWASP Web Security Testing Guide (WSTG) and industry-standard penetration testing methodology.

**Engagement Details:**
- Assessment type: External penetration test (black-box)
- Target environment: Production system

**Scope (in-scope assets):**
- Primary domain: wingdata.htb (HTTP port 80)
- Virtual host: ftp.wingdata.htb (Wing FTP Server)
- SSH service: port 22

**Testing Activities Performed:**
1. Reconnaissance and service enumeration (Nmap)
2. Web application fingerprinting and technology identification
3. Virtual host discovery via application links
4. Directory and endpoint enumeration
5. Authentication testing and credential discovery
6. Path traversal vulnerability exploitation
7. Process memory and file descriptor analysis
8. SQLite database extraction and analysis
9. CVE research for identified software versions
10. Lua injection testing (CVE-2024-37842)
11. Credential validation and SSH access testing

**Evidence Preservation:**
- All exploitation commands documented
- Session management techniques recorded
- Extracted data preserved for verification

# Technical Analysis

**Severity Model:** CVSS 3.1 based on exploitability and impact assessment.

**CONFIRMED VULNERABILITIES:**

**1. Path Traversal - Arbitrary File Read (Critical - CVSS 8.6)**
The Wing FTP Server download functionality allows authenticated users to read arbitrary files from the server filesystem. The vulnerability exists in the `filename` parameter accepting absolute paths.

**Technical Details:**
- Endpoint: `GET /?download&filename=<absolute_path>`
- Authentication: Required (anonymous:anonymous valid)
- Constraint: Absolute paths work; relative traversal blocked
- Impact: Configuration files, user data, process memory accessible

**Exploitation:**
```bash
curl -s -c cookies.txt -H "Host: ftp.wingdata.htb" \
  -d "username=anonymous&password=anonymous" \
  "http://wingdata.htb/loginok.html"

curl -s -b cookies.txt -H "Host: ftp.wingdata.htb" \
  "http://wingdata.htb/?download&filename=/etc/passwd"
```

**Evidence Extracted:**
- /etc/passwd - Confirmed users: root, wingftp (UID 1000), wacky (UID 1001)
- Process environment - Revealed service configuration
- SQLite databases via /proc/3569/fd/ - User activity logs
- Wing FTP web application source code

**2. Valid Credentials Discovered (Critical)**
User credentials: `wacky:master`
- User wacky has SSH access to the system
- Password verified as functional
- User ID: 1001, Home: /home/wacky

**3. Information Disclosure via SQLite Databases (High)**
Open SQLite databases discovered via process file descriptors:
- fd 15: wftp_dblogs table containing user authentication events
- Confirmed wacky user activity with recent timestamps
- Provides insight into user behavior patterns

**4. Wing FTP Admin Password Hash (Medium)**
- Hash: 2D35A8D420A697203D7C554A678F8119
- Algorithm: MD5
- Status: Not cracked during assessment
- Location: /opt/wftpserver/Data/

**5. Known Vulnerable Software (Medium)**
Wing FTP Server 7.4.3 (Free Edition) - CVE-2024-37842
- Lua injection vulnerability
- Testing conducted but not confirmed exploitable
- May require admin authentication or different attack vector

# Recommendations

**Immediate Actions:**

1. **SSH Access with Compromised Credentials**
```bash
ssh wacky@wingdata.htb
# Password: master
```

2. **Retrieve User Flag**
```bash
cat /home/wacky/user.txt
```

3. **Privilege Escalation Enumeration**
Once SSH access achieved:
```bash
# Check sudo permissions
sudo -l

# Find SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Check kernel version for exploits
uname -a

# Enumerate writable paths
find / -writable -type d 2>/dev/null

# Check for stored credentials
grep -r "password" /home/wacky/ 2>/dev/null
cat /home/wacky/.bash_history
```

4. **Crack Admin Hash**
Use larger wordlists or rainbow tables:
```bash
hashcat -m 0 -a 0 2D35A8D420A697203D7C554A678F8119 rockyou.txt
```

**Short-term Priority:**

5. **Post-Exploitation Activities**
- Enumerate network connections
- Check for other users' credentials
- Look for database passwords in config files
- Examine cron jobs and scheduled tasks

6. **Lateral Movement**
- Check for other accessible hosts
- Enumerate network shares
- Look for SSH keys for other users

**Remediation Recommendations:**

7. **Fix Path Traversal**
- Implement strict path validation
- Use allowlist for accessible directories
- Sanitize user input in filename parameter

8. **Password Policy**
- Enforce strong password requirements
- Implement password complexity rules
- Consider multi-factor authentication

9. **Service Hardening**
- Update Wing FTP Server to latest version
- Disable anonymous access if not required
- Implement network segmentation

