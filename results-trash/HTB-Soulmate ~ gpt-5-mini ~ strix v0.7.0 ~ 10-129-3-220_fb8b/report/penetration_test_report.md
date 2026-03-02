# Security Penetration Test Report

**Generated:** 2026-02-20 20:12:36 UTC

# Executive Summary

Executive summary
An external security assessment of the soulmate.htb service (10.129.3.220) identified multiple configuration and discovery-level issues that increase the likelihood of sensitive admin/API exposure, but no confirmed, fully-exploitable application-layer vulnerabilities were validated during this non-destructive engagement.

Key outcomes
- Network exposure: The host is reachable on standard management/service ports (notably SSH and HTTP), and service/version fingerprinting shows common server software.
- TLS / VHOST anomalies: Multiple TLS and SNI irregularities were observed (unexpected 502 responses, handshake errors, and differing responses by SNI/Host). These anomalies can hide SNI-gated admin or API surfaces and materially impede safe automated discovery.
- Application discovery: Automated crawling and aggressive discovery surfaced a broad set of endpoints (dozens of candidate pages) including ~14 authentication/admin-related endpoints and multiple responsive virtual-host candidates. Several pages were flagged as authentication-related and warrant interactive review.
- Validation status: No high-confidence SQL injection, cross-site scripting, SSRF, or IDOR was confirmed during non-destructive, read-only testing. Reconnaissance-level candidates were captured for triage and follow-up validation.

Business impact
- If any of the discovered admin/API surfaces are misconfigured or lack strict authorization, an external attacker could enumerate or access sensitive administrative or tenant-scoped data.
- TLS/SNI misconfiguration can enable accidental exposure of internal endpoints or allow attackers to bypass normal routing protections.
- The combination of exposed administrative interfaces, inconsistent TLS behavior, and incomplete discovery coverage elevates the risk posture and justifies prioritized follow-up work (interactive proxy review and focused validations).

# Methodology

Methodology
The assessment followed a structured external (black-box) methodology aligned with the OWASP Web Security Testing Guide and common web/API testing practices.

Phases performed
- Reconnaissance: DNS and host mapping, port and service fingerprinting, TLS/ certificate inspection.
- Surface mapping: Comprehensive crawling of public endpoints, static analysis of client-side JavaScript, and enumeration of potential virtual-host names and host-header variants.
- Discovery: Recursive directory and parameter discovery using wordlists and targeted host-header fuzzing to identify hidden paths and vhosts.
- Parameter analysis: Non-destructive reflection and parameter probing (GET-based markers) to identify potential injection/reflection sinks for follow-up validation.
- Interactive triage: Proxy-driven manual inspection of prioritized auth/admin candidates to extract cookie/JWT/session behaviors and identify dynamic flows that require authenticated or manual testing.

Testing constraints and safety
- All active testing was non-destructive. The engagement deliberately avoided state-changing operations (no destructive POSTs, no password resets, no data modification).
- Testing used conservative concurrency and backoff logic to prevent service disruption.
- Evidence for all probes was recorded to enable reproducible follow-up validation and reporting.

# Technical Analysis

Technical analysis
1) Network & services
- Reachable services: SSH (port 22) and HTTP (port 80) were confirmed as externally accessible.
- Service versions observed during enumeration are consistent with widely used server stacks; no direct, unauthenticated service-level exploits were validated as part of this engagement.

2) TLS / SNI observations
- Multiple probes produced inconsistent TLS behavior: TLS handshakes failing for some hostnames, HTTPS requests to non-standard ports returning 502 or handshake errors, and differing responses depending on SNI/Host combination.
- Impact: These behaviors can conceal SNI-gated admin/API endpoints from standard scans and may indicate misconfigured TLS front-ends or reverse proxies. This impedes automated discovery and increases attack surface uncertainty.

3) Application discovery and attack surface
- The assessment enumerated a large sitemap and identified numerous candidate endpoints, including authentication and administrative pages. Several vhost/host-header candidates responded to probes and produced 2xx/3xx/401/403 responses.
- Static JS analysis and crawls exposed limited client-side API hints; however, automated scans did not find reliable in-band evidence of SQLi, XSS, SSRF, or IDOR during non-destructive testing.

4) Authentication & session handling
- The interactive triage pass located multiple authentication-related endpoints (login/password-related indicators). No clear authentication bypass or privilege escalation was validated without authenticated testing.
- Session and cookie behaviors were inspected; no systemic absence of standard protections (Secure/HttpOnly flags) was validated in the read-only pass, but further authenticated testing is required to fully assess token lifecycle, JWKS/JWT handling, and authorization enforcement.

5) Parameter & input analysis
- Automated reflection tests and parameter enumeration produced primarily reconnaissance-level candidates. The current candidate list does not include validated exploit payloads. Focused validation is required (authenticated and/or stateful flows) to confirm or exclude high-risk issues.

Conclusion: The environment contains credible, high-priority misconfiguration and discovery findings (TLS/SNI anomalies, numerous auth/admin endpoints, responsive vhosts) that materially increase risk. However, no fully validated critical vulnerability was confirmed in this non-destructive assessment. Further interactive and authenticated testing is required to complete the validation stage.

# Recommendations

Recommendations
Priority 0 — Immediate (reduce attacker visibility and exposure)
- Investigate and remediate TLS/SNI anomalies
  - Inspect reverse-proxy/load-balancer and TLS termination configuration to ensure SNI is handled predictably.
  - Remove or correct any unintended TLS services bound to management ports (e.g., confirm whether TLS is intentionally served on port 22).
  - Ensure certificates and SAN entries are accurate and do not inadvertently expose internal hostnames.

- Apply network-level egress and access controls
  - Restrict application runtime egress to necessary destinations. Route any required outbound fetches through a policy-enforcing egress proxy to block internal ranges and link-local metadata endpoints.
  - Harden exposure of administrative interfaces via IP allowlists, VPN-only access, or MFA-gated admin paths where feasible.

Priority 1 — High (authorization and discovery hardening)
- Perform an interactive proxy review (Burp/ZAP) of the highest-priority auth/admin hosts
  - Manually exercise login and admin workflows (using a test account) to identify hidden POST parameters, CSRF token usage, JWT/JWKS endpoints, and any stateful flows missed by GET-only scans.
  - Confirm whether any administrative functionality is accessible without appropriate authorization checks.

- Centralize and enforce authorization
  - Implement deny-by-default authorization middleware for all object access paths.
  - Add consistent ownership/tenant checks for all read/write APIs and add automated regression tests to prevent regressions.

Priority 2 — Medium (input handling and discovery improvements)
- Complete exhaustive discovery with full toolset (deep JS crawling, ffuf with large SecLists, arjun) routed through an evidence-collection proxy, then re-run targeted validation against any high-confidence candidates (SQLi/XSS/SSRF/IDOR).
- Harden input handling and output encoding:
  - Enforce strict input validation and parameterized database queries.
  - Apply context-aware output encoding for user-controllable data rendered in HTML/JS.

Priority 3 — Monitoring and governance
- Improve logging and alerting for anomalous admin/API access, unusual outbound fetch attempts, and repeated authorization failures.
- Maintain an evidence-driven retest after remediation: re-run deep discovery and focused validation to confirm fixes.

Follow-up validation plan
- After interactive triage, run narrowly scoped, non-destructive validation tests for any confirmed candidates:
  - SQLi/XSS: validate with safe, minimal payloads and capture reproducible PoCs.
  - IDOR: validate read-only object access tampering and produce exact request/response PoCs.
  - SSRF: test only with in-band measures or controlled OOB monitoring services.
- Create formal vulnerability reports only after successful validation with reproducible PoCs.

Closing
This engagement produced a broad, evidence-based map of the external attack surface and identified high-priority misconfiguration and exposure issues to address first. The recommended next step is an immediate interactive proxy-based review of the flagged authentication/admin hosts and exhaustive SNI/TLS confirmation, followed by narrowly scoped validation of any actionable parameter candidates.

