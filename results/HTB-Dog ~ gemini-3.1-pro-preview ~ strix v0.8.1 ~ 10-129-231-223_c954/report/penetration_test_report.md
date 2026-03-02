# Security Penetration Test Report

**Generated:** 2026-02-28 07:54:58 UTC

# Executive Summary

The security assessment of the target application (dog.htb) and its underlying infrastructure revealed several critical vulnerabilities that cumulatively allowed for full system compromise. The most significant issue was the exposure of a Git repository within the web root, which leaked hardcoded database credentials and sensitive administrative details. An attacker could exploit these findings to gain unauthorized administrative access to the Backdrop CMS platform. From there, the application permitted authenticated administrators to execute arbitrary code via the manual module installation feature, resulting in remote code execution (RCE) on the server. Furthermore, a local privilege escalation vulnerability was identified in the `sudo` configuration, allowing a low-privileged user (`johncusack`) to execute a Backdrop command-line utility (`bee`) as root. By combining these vulnerabilities, complete administrative control over the server environment was achieved. Immediate remediation is required to secure the application, mitigate configuration oversights, and apply the principle of least privilege.

# Methodology

The assessment was conducted as a targeted external penetration test using a gray-box methodology. The engagement included comprehensive reconnaissance, including port scanning and service enumeration. During discovery, a publicly accessible `.git` repository was identified and downloaded for static analysis, which led to the discovery of hardcoded credentials. Dynamic testing against the Backdrop CMS involved leveraging these credentials to bypass authentication. Post-authentication testing focused on administrative functionalities, leading to the deployment of a malicious payload to verify remote code execution. Upon gaining a foothold on the system, local privilege escalation techniques were executed by enumerating user permissions and exploiting insecure `sudo` configurations. All testing followed industry-standard methodologies to evaluate the system's security posture and validate findings without causing destructive impact.

# Technical Analysis

The technical analysis uncovered three primary attack vectors that formed a complete kill chain:

1) Information Disclosure via Exposed Git Repository (Critical): The web server was misconfigured to serve the `.git` directory (`http://dog.htb/.git/`), allowing full repository extraction. Analysis of the downloaded files revealed database connection strings (`mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop`) inside `settings.php` and the administrator email `tiffany@dog.htb` inside active JSON configurations. The database password was reused as the CMS password for the `tiffany` account.

2) Authenticated Remote Code Execution in Backdrop CMS (Critical): Once authenticated as the administrator (`tiffany`), the CMS’s manual module installation feature (`admin/installer/manual`) could be leveraged to upload a crafted `.tar.gz` archive containing a PHP web shell. This resulted in arbitrary command execution in the context of the `www-data` web server user.

3) Privilege Escalation via Sudo Misconfiguration (High): The server was configured to allow the user `johncusack` to execute the `/usr/local/bin/bee` binary as `root` without a password. The `bee` utility (Backdrop CMS CLI) includes an `eval` parameter capable of executing arbitrary PHP code. This functionality was abused to spawn a root-level interactive shell (`sudo /usr/local/bin/bee eval "system('/bin/bash');"`), compromising the entire server.

# Recommendations

Immediate Action Items:
1. Secure the Web Root: Immediately restrict web access to the `.git` directory and other sensitive configuration files using appropriate server configurations (e.g., `.htaccess` or Apache site blocks).
2. Credential Rotation and Hardening: Rotate all compromised credentials, including the database password and all Backdrop CMS user passwords. Ensure that passwords are not reused across different services and enforce strong password policies. Do not store sensitive credentials in plaintext within version control systems; utilize environment variables or secure vault solutions instead.
3. Remediate Sudo Misconfiguration: Remove the `sudo` entry in `/etc/sudoers` (or included files) that grants the `johncusack` user the ability to run `/usr/local/bin/bee` as root. Apply the principle of least privilege.
4. Restrict Module Uploads: In production environments, disable the ability to upload and install modules manually via the CMS web interface to mitigate the risk of post-authentication code execution.

