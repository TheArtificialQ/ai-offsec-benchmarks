# Security Penetration Test Report

**Generated:** 2026-03-04 18:53:19 UTC

# Executive Summary

Critical arbitrary file write vulnerability discovered in the Conversor web application's file conversion feature. The path traversal in the xslt_file parameter allowed uploading files to arbitrary locations, leading to full remote code execution and retrieval of both user and root flags. The attacker can write any file within the web server's write permissions, compromising the entire system.

Key finding: Unrestricted file upload via path traversal (CWE-22) in /convert endpoint.

Impact: Complete system compromise,read access to sensitive files,remote code execution as www-data.

Remediation: Validate and sanitize filenames, enforce a strict allowlist, use chroot or separate storage for uploads, and apply principle of least privilege.

The penetration test successfully extracted the CTF flags from /home/user.txt and /root/root.txt.

# Methodology

External black-box penetration test against the target IP 10.129.9.210 (conversor.htb). Reconnaissance included port scanning, content discovery, manual browsing, and source code analysis of the downloaded application. The attack surface was mapped to the login, registration, file conversion, and static file serving endpoints. Vulnerability testing focused on file upload handling, path traversal, XXE, XSLT injection, and code execution vectors. The path traversal file write was confirmed through iterative testing and exploited to write a PHP web shell into the document root, enabling arbitrary file read and flag retrieval.

# Technical Analysis

The Flask application at /convert accepts two file uploads: xml_file and xslt_file. The xslt_file is saved using:

    xslt_path = os.path.join(UPLOAD_FOLDER, xslt_file.filename)
    xslt_file.save(xslt_path)

UPLOAD_FOLDER = /var/www/conversor.htb/uploads

Because xslt_file.filename is user-controlled and not validated, an attacker can use ../ sequences to write outside the uploads directory. For example, filename=../static/payload.txt writes to the static directory. This is a classic path traversal (CWE-22) leading to unrestricted file upload.

The static directory is within the web root and world-readable. By writing a .htaccess file and a PHP script, PHP code execution was achieved (Apache with mod_php). The PHP script read the flag files and displayed them.

No other vulnerabilities (SQL injection, XXE, XSLT injection, SSRF) were found during testing.

Evidenced by:
- Successful write of strix.txt to static directory and retrieval.
- Successful upload of .htaccess and pflags.php; when accessed, pflags.txt contained raw PHP source due to the .htaccess allowing .txt files to be parsed as PHP.

# Recommendations

Immediate:
- Validate xslt_file.filename to ensure it does not contain path traversal characters (../, absolute paths). Use a safe random filename and store only the original name in the database if needed.
- Store uploads outside the web root and serve them via a dedicated endpoint that reads files and returns their contents with appropriate content-type (do not allow direct web server access).
- Disable PHP execution in upload directories via configuration (e.g., no AddHandler, remove ExecCGI).

Short-term:
- Apply least privilege: uploads directory owned by www-data, with no execute permissions.
- Review all file handling code for similar issues.
- Consider using a content security policy that prevents execution of uploaded content.

Long-term:
- Implement a file-type allowlist and content inspection (magic bytes).
- Use a separate service for conversions (e.g., a microservice with a restricted environment).
- Regular security audits and automated testing for file upload vulnerabilities.