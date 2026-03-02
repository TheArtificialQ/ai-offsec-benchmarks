# Security Penetration Test Report

**Generated:** 2026-02-20 15:52:05 UTC

# Executive Summary

An external, non-destructive security assessment was performed against soulmate.htb (10.129.3.155). The target exposed an HTTP web application and an SSH service. No high-severity vulnerabilities (such as exploitable SQL injection, SSRF to cloud metadata, remote code execution, or exposed credentials) were confirmed during unauthenticated, read-only testing. Client-side JavaScript and external assets were enumerated and produced several low-confidence DOM XSS candidates; controlled runtime checks did not demonstrate reliable script execution. Automated directory and information-level scans returned informational findings that require manual triage. The assessment was limited to unauthenticated, non-destructive techniques; authenticated API and privileged workflows were not exercised and represent the principal residual risk. Recommended priorities are: enable TLS across public endpoints, centralize and harden authorization controls, and perform authenticated follow-up testing focused on API and business-logic flows.

# Methodology

The assessment used an OWASP WSTG-aligned approach combining external reconnaissance, automated non-destructive scanning, static analysis of client-side artifacts, and safe runtime validation. Reconnaissance included service and port discovery and web crawling to enumerate pages, parameters and JavaScript assets. Automated, read-only scanners and directory discovery identified candidate paths for follow-up. Static parsing of JavaScript bundles collected client-side endpoints and potential DOM sinks. Runtime validation used controlled, harmless markers and headless-browser snapshots to verify DOM-based behaviors. Focused, non-destructive validation passes were executed for XSS, IDOR, and SQL injection against discovered unauthenticated input vectors. All activity was intentionally non-destructive and rate-limited; authenticated testing was out of scope.

# Technical Analysis

Observed platform posture: the host served HTTP (nginx) and SSH; no HTTPS service was observed on common ports, increasing exposure of credentials and session tokens in transit. Client-side analysis: multiple client-side endpoints and external scripts were catalogued and statically analyzed. Static analysis identified a small set of low-confidence DOM XSS candidates; controlled headless runtime checks did not confirm exploitable DOM XSS for the tested inputs. Automated content-discovery and information-level vulnerability templates produced informational results and candidate paths but no validated high-confidence vulnerabilities. Authorization and injection testing performed without credentials did not reveal IDOR or SQL injection on the unauthenticated surface; however JSON APIs, authenticated endpoints, header/cookie vectors and privileged workflows were not exercised and represent the most significant residual risk vector.

# Recommendations

Priority 0 (Immediate)
  - Enable HTTPS for all public endpoints with a valid certificate, enforce HTTPS-only access, strong cipher suites, and HSTS to protect confidentiality and session integrity.
  - Patch and harden the web server and host operating system to supported, fully-patched versions and apply secure configuration baselines.

Priority 1 (High)
  - Centralize and enforce authorization: implement deny-by-default object-access policies, require explicit tenant/owner checks for every read/write path, and consolidate authorization logic into testable middleware.
  - Apply strict input validation and context-aware output encoding (HTML, attribute, JavaScript, URL) across rendering paths and templates.
  - Triaging: perform manual review of enumerated JavaScript endpoints and DOM snapshots to resolve low-confidence findings.

Priority 2 (Medium)
  - Conduct authenticated, API-focused testing using scoped test accounts to exercise JSON payloads, header/cookie flows, and privileged actions; prioritize testing of business-logic flows that handle sensitive objects or payments.
  - Expand parameter discovery (POST/JSON bodies and header/cookie vectors) and repeat non-destructive validation for potential injection and IDOR vectors.

Priority 3 (Defense-in-depth)
  - Introduce egress controls and an allowlist for server-side outbound requests to mitigate SSRF risk; re-validate destination IPs after DNS resolution and on redirects.
  - Strengthen logging and alerting for anomalous authorization failures, repeated discovery or crawling activity, and unusual outbound request patterns.

Follow-up
  - After remediation, perform a targeted retest including authenticated API coverage, TLS verification, and focused validation of any previously flagged client-side or server-side candidates to confirm fixes.

