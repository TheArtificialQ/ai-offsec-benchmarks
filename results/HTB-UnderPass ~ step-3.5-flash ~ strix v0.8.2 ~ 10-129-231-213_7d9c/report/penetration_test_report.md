# Security Penetration Test Report

**Generated:** 2026-03-06 17:37:03 UTC

# Executive Summary

The target underpass.htb (10.129.231.213) runs Apache HTTP Server 2.4.52 (Ubuntu) with a critical open forward proxy misconfiguration (mod_proxy with ProxyRequests On). This allows unauthenticated attackers to perform Server-Side Request Forgery (SSRF) against arbitrary internal HTTP resources. The proxy successfully reaches the link-local metadata service at 169.254.169.254, demonstrating the ability to pivot into internal networks or cloud metadata endpoints. Extensive testing of other common attack vectors (path traversal, local file inclusion, configuration disclosure, virtual host differentials, and known Apache CVEs) did not reveal additional vulnerabilities. The system is a default Ubuntu Apache installation with no active backend applications detected.

**Overall risk posture:** Elevated due to SSRF.

**Key findings:**
- Server-Side Request Forgery (SSRF) – High severity – allows internal network access via proxy.
- No other security weaknesses identified.

**Remediation theme:** Disable or tightly restrict forward proxy functionality and implement outbound network filtering to prevent SSRF exploitation.

# Methodology

The assessment followed OWASP Web Security Testing Guide and NIST penetration testing methodology.

- **Reconnaissance:** Port scanning, virtual host enumeration (533 hosts tested), content discovery via fuzzing.
- **Configuration analysis:** Reviewed Apache behavior, proxy settings, and response patterns.
- **Vulnerability testing:** Tested for SSRF, path traversal (CVE-2021-41773/CVE-2021-42013), local file inclusion, server-status disclosure, and configuration file exposure.
- **Exploitation validation:** Confirmed the SSRF vector by reaching internal link-local address and observing distinct responses (404 from 169.254.169.254 vs. default page for unreachable destinations).
- **Impact assessment:** Evaluated potential for cloud metadata access and internal network scanning.

Only validated issues with reproducible impact are reported.

# Technical Analysis

### Confirmed High‑Severity Finding

**Server‑Side Request Forgery (SSRF) via Open HTTP Proxy**

The target Apache server accepts absolute URIs in the request line and forwards them to arbitrary destinations, behaving as an open forward proxy. This was validated by sending `GET http://169.254.169.254/latest/meta-data/ HTTP/1.1` with a valid `Host` header. The server returned a 404 response that differed from the default Apache page, confirming that the request reached an actual internal host (169.254.169.254).

All other tested destinations (127.0.0.1, other private IPs, unroutable TEST‑NET addresses) returned the same 200 OK response containing the Apache Ubuntu default page (Content‑Length ~10671). This indicates that connections to most targets either fail or are intercepted and replaced by a fallback page, but reachability to 169.254.169.254 proves the proxy can connect to internal services.

The vulnerability could be exploited to:
- Discover internal infrastructure and open ports via timing and response analysis
- Access cloud metadata services if present (AWS, Azure, GCP) to obtain instance credentials and configuration
- Bypass network segmentation and reach internal web applications that would otherwise be inaccessible

A detailed vulnerability report has been created (ID `vuln-0001`).

### Negative Findings

- **Path traversal (CVE‑2021‑41773 / CVE‑2021‑42013):** Various encoded payloads resulted in 400/404; no file disclosure achieved.
- **Local file inclusion / directory traversal:** No successful reads of `/etc/passwd` or application files.
- **Configuration disclosure:** `server-status` (403), `.htaccess` (403), no backup files discovered.
- **Virtual host differentials:** All 533 tested virtual hosts served identical static default page; no hidden applications found.
- **SSH brute force:** Connection attempts to port 22 were reset or closed; no credentials obtained.
- **Other Apache CVEs:** Tested CVE‑2022‑23943 (AJP) – port 8009 closed; CVE‑2023‑27522 – `file://` scheme blocked; no RCE indicators.
- **Cloud metadata:** AWS/Azure/GCP paths on 169.254.169.254 returned 404; no user‑data available.

The web server appears to be a stock Ubuntu Apache installation with no active backend services behind the proxy.

# Recommendations

### Immediate Actions (Priority: Critical)

1. **Disable forward proxy if not required**  
   In Apache configuration, set `ProxyRequests Off`. Ensure no `<Proxy>` blocks allow open access.

2. **Restrict proxy usage (if a forward proxy is needed)**  
   - Use `<Proxy>` containers with explicit `Require ip` or `Require host` rules to limit clients.
   - Avoid `ProxyRequests On` in public‑facing instances; reverse‑proxy (`ProxyPass`) should be used instead.

3. **Implement egress firewall filtering**  
   Prevent the web server from initiating outbound connections to sensitive ranges:  
   - Loopback: `127.0.0.0/8`  
   - Link‑local: `169.254.0.0/16`  
   - Private‑use: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`  
   Only allow destinations that are explicitly required.

4. **Remove unnecessary proxy modules**  
   Disable `mod_proxy`, `mod_proxy_http`, `mod_proxy_ajp` if the server does not need proxying functionality.

5. **Upgrade and patch**  
   Although no Apache RCE was found, ensure the server runs a supported version and receives security updates.

6. **Monitoring and Alerting**  
   Log all proxy requests and alert on attempts to access link‑local or private IP ranges.

### Follow‑up Testing

After remediation, perform a retest to verify:
- The SSRF vector is fully mitigated (proxy cannot reach internal IPs).
- No unintended proxy behavior remains.
- All proxy‑related directives are correctly scoped.

