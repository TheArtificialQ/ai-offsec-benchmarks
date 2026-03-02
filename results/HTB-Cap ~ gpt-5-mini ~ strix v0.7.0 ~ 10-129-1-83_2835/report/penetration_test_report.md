# Security Penetration Test Report

**Generated:** 2026-02-21 07:42:59 UTC

# Executive Summary

An external, non-destructive security assessment was conducted against the target host 10.129.1.83 to map exposed services, enumerate the web application surface, and validate high-impact vulnerability classes. The assessment confirmed the host is reachable and operating the following services: FTP (vsftpd 3.0.3), SSH (OpenSSH 8.2p1), and an HTTP application served by a Python WSGI process on port 80.

Web-focused discovery identified approximately fifty unique URLs and nineteen JavaScript assets that build dynamic API calls and parameter templates. Non-destructive validation (marker-based XSS probes, passive SQL heuristics, conservative directory discovery, and Playwright-driven browser checks) found no confirmed critical vulnerabilities (no confirmed SQL injection, no confirmed SSRF with out-of-band verification, and no DOM-executed XSS under instrumented tests). Key items requiring follow-up include a reflected-but-likely-escaped query parameter (literal token reflection observed but no script execution), multiple candidate sensitive-path patterns (redirects/403 responses and backup-like filename matches), and several client-side URL-building patterns that create SSRF risk if server-side fetch logic accepts unvalidated destinations.

Overall risk posture: Moderate. The existing exposure and client-side behavior create plausible, high-impact attack chains (notably SSRF → internal access and SQLi → data disclosure) if follow-up confirmation validates these vectors. Immediate remediation of obvious configuration issues and prioritized follow-up testing are recommended.

# Methodology

The assessment followed industry-standard external web and network testing practices informed by the OWASP Web Security Testing Guide.

Phases executed:
- Reconnaissance and service enumeration: External host discovery and TCP service fingerprinting to identify open ports and server software.
- Web discovery: HTTP probing and crawling to map endpoints, static JavaScript analysis to extract API endpoints/parameter names and URL templates, and directory enumeration to identify candidate administrative, upload, and backup paths.
- Non-destructive validation: Context-aware marker testing for reflected XSS, passive/error-based SQLi probes, conservative use of an SQL detection tool in confirmation mode (where environment allowed), and Playwright-driven browser instrumentation for runtime verification of reflection and script-execution contexts.
- Controlled fuzzing: Recursive directory discovery with tuned lists, constrained recursion and rate limits to avoid disruption.
- Evidence triage: All test artifacts and per-test evidence were preserved for review and to seed targeted follow-up validation.

Safety constraints
All testing was explicitly non-destructive and read-only: no credential guessing, no mass data extraction, no automated retrieval of large binary artifacts, and conservative rate limits. Blind/OOB tests were not executed without a provisioned interaction listener; blind SSRF or blind SQLi verification was deferred pending an OOB channel.

# Technical Analysis

Network and service posture
- Open services: FTP (vsftpd 3.0.3), SSH (OpenSSH 8.2p1), HTTP (Python WSGI). Each service should be reviewed for configuration hardening, unnecessary exposure, and up-to-date patching.

Web application discovery and client-side analysis
- The application surface includes numerous static and dynamically-generated endpoints. Nineteen JavaScript assets were found that construct API calls and parameterized URLs; these assets were used to seed targeted tests.
- No HTML forms were discovered during the initial automated crawl; query-string parameters and AJAX/JSON endpoints were the primary input vectors.

XSS findings
- Marker-based testing and Playwright instrumentation detected literal reflections of unique tokens in a query parameter used by the site's visible search control. These reflections were observed to be rendered in a non-executable context during instrumented runs (i.e., no console execution or DOM-executed script was observed for context-appropriate payloads).
- Classification: reflected-but-likely-escaped. Risk remains if the same input is inserted into alternate contexts (HTML attribute, inline script, or dynamically constructed DOM insertion) without proper context encoding.

SQL injection findings
- Passive, non-destructive probes were performed against prioritized parameters; no confirmed SQL injection was identified in-scope. Several probable indicators were present in aggregated data but referenced hosts outside the tested target and therefore were not validated here.
- Recommendation: targeted POST/JSON endpoint testing with conservative confirmation tools and OOB-supported blind testing (where necessary) is required to fully exclude SQLi.

SSRF findings
- Static analysis of client-side code and parameter patterns identified inputs that could be forwarded to server-side fetch routines. No SSRF was confirmed because blind/OOB tests were not executed without an available interaction listener.
- SSRF remains a medium-to-high impact risk until endpoints are validated or protected by strict destination filtering and egress controls.

Directory and sensitive-file discovery
- Recursive enumeration returned a mix of redirects and access-denied responses and flagged candidate sensitive-file patterns (repository artifacts, environment/config names, and backup patterns). No automatic retrieval or content inspection of potentially sensitive files was performed to preserve a non-destructive posture.

Limitations and evidence
- The assessment intentionally limited destructive interactions and blind OOB testing. Detection of blind SSRF and blind SQLi requires an OOB interaction service and extended confirmation. These limitations mean that high-impact blind vectors cannot be conclusively excluded without the recommended follow-up testing.

# Recommendations

Prioritized remediation and validation roadmap

Priority: High
- Harden exposed services
  - Disable or restrict FTP access (remove anonymous access); ensure secure configuration and monitoring for the FTP service.
  - Harden SSH: remove weak host key types/ciphers, enforce modern key-exchange algorithms, and monitor SSH authentication events.
  - Patch and harden the application runtime and dependencies.

- Prevent SSRF and restrict outbound fetches
  - Apply deny-by-default outbound policies for any server-side URL fetching feature.
  - Implement destination allowlists and verify resolved IP addresses against private/link-local ranges; re-validate after redirects.
  - Enforce outbound egress through a policy-enforcing proxy with logging and destination policies.

- Fix input/output handling and XSS mitigations
  - Apply context-sensitive output encoding everywhere user input is rendered (HTML encode, attribute encode, JavaScript-string encode, URL encode as appropriate).
  - Adopt a strong Content-Security-Policy and enforce secure cookie attributes.

- Protect sensitive files and configuration
  - Ensure web server does not expose repository artifacts, environment files, backups, or config files via HTTP.
  - Verify file-system permissions and restrict access to configuration files; rotate any credentials that may have been exposed.

Priority: High — Focused follow-up validation
- Provision an out-of-band interaction service (e.g., Interact/OOB collector) and re-run SSRF and blind SQLi verification to conclusively confirm or rule out blind vectors.
- Execute an exhaustive Playwright-enabled crawl with full console and network instrumentation to:
  - Confirm whether reflected tokens can execute in any runtime context.
  - Discover runtime-generated API endpoints and parameter shapes for precise validation.

Priority: Medium
- Conduct targeted SQLi confirmation (conservative sqlmap or scripted boolean/time probes) against any POST/JSON endpoints identified by the JS analysis and Playwright runs. Restrict tests to confirmation-only (no mass extraction).
- Run deeper recursive directory fuzzing with comprehensive wordlists to exhaustively identify admin/upload/backups. For any candidate sensitive files located, perform controlled read-only retrieval of text files only and rotate secrets immediately if found.

Priority: Low to Medium (policy & process)
- Centralize authorization checks for object access (deny-by-default).
- Add automated regression tests for authorization decisions and input-handling edge cases.
- Improve monitoring and alerting for anomalous outbound fetch attempts, repeated authorization failures, and suspicious access to administrative paths.

Retest and validation
- After remediation, perform a focused retest covering SSRF, SQL injection, context-aware XSS, and any exposed file retrievals. Confirmation should include both unauthenticated and authenticated workflows if applicable.

If the client authorizes it, the testing team can deliver a prioritized verification plan, exact, non-destructive test scripts for follow-up validation, and a controlled sensitive-file retrieval and analysis service to confirm or eliminate the remaining candidate risks.

