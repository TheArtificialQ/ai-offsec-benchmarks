# Security Penetration Test Report

**Generated:** 2026-02-19 19:15:58 UTC

# Executive Summary

Executive summary
An external penetration test of the host 10.13.37.10 identified multiple high‑impact vulnerabilities in bespoke TCP services, while the exposed web services on ports 80 and 9201 exhibited limited or no exploitable issues.

Overall risk posture: Elevated, primarily due to unsafe native code in custom services on ports 5555 and 7777 that leak process memory and permit reliable denial of service.

Key outcomes
- Confirmed unauthenticated memory‑address information disclosure in the “Member manager” service on TCP 5555 via the “get gift” functionality. The leak exposes pointer‑like values suitable for bypassing ASLR and materially lowers the bar for developing remote code execution exploits against associated memory‑corruption bugs.
- Confirmed combined information disclosure and deterministic remote denial of service in the “Spiritual Memo” service on TCP 7777. A malformed delete operation triggers a repeatable free(): invalid pointer crash, and the show‑memo functionality leaks code/memory bytes even before any memo is created.
- Assessed the HTTP JSON API on port 9201 and the HTTP service on port 80:
  - Port 80 serves a static nginx default page with no dynamic functionality or user input.
  - Port 9201 exposes a small unauthenticated, read‑only JSON API. Comprehensive injection and protocol‑robustness testing found no exploitable SQLi/RCE or HTTP desynchronization issues, although some error paths are not strictly HTTP‑compliant.

Business impact
- The custom TCP services on ports 5555 and 7777 present realistic exploitation potential: the confirmed infoleaks significantly aid in developing advanced memory‑corruption exploits, and the DoS condition on 7777 allows trivial, unauthenticated service termination with diagnostic output.
- If these services underpin production applications or run with elevated privileges, successful exploitation could lead to full system compromise, lateral movement, and loss of service availability.
- The web‑facing components on 80 and 9201 currently represent low direct risk but should still be maintained and hardened, particularly if their deployment environment changes (e.g., placement behind reverse proxies).

# Methodology

Methodology
The assessment followed industry‑standard penetration testing practices aligned with the OWASP Web Security Testing Guide and common infrastructure and service‑level testing approaches.

Scope
- Target host: 10.13.37.10
- In‑scope services (discovered by network reconnaissance):
  - 22/tcp: SSH
  - 53/tcp: DNS (BIND)
  - 80/tcp: HTTP (nginx, default page)
  - 2222/tcp: SSH
  - 5555/tcp: Custom “Member manager!” TCP service
  - 7777/tcp: Custom “Spiritual Memo” TCP service
  - 9201/tcp: HTTP JSON API (Python BaseHTTPServer 0.3, Python 2.7.12)

Testing approach
- Network and service reconnaissance
  - Enumerated open TCP ports and fingerprinted services and versions.
  - Prioritized web/http and custom TCP services for application‑layer testing.
- Web and API testing (ports 80 and 9201)
  - Crawled and mapped HTTP endpoints and content.
  - Enumerated directories and files using wordlist‑based discovery.
  - Identified request/response formats, query parameters, and methods.
  - Performed targeted injection testing (SQL injection, command injection, server‑side template/code injection) against dynamic inputs, particularly the category parameter on the 9201 /search endpoint.
  - Performed HTTP robustness and desynchronization tests using raw‑socket requests, malformed queries, pipelining attempts, and Content‑Length/body anomalies.
- Custom TCP service analysis (ports 5555 and 7777)
  - Interacted with each service manually to fully map menus, prompts, and workflows.
  - Developed scripts to automate interaction, fuzz inputs (sizes, indices, payload lengths, encodings), and capture responses.
  - Focused on:
    - Memory‑corruption indicators (crashes, inconsistent behavior, use‑after‑free patterns).
    - Information disclosure through pointer‑like values or stray memory dumps.
    - Logic flaws and state‑handling issues (e.g., operations in invalid states).
- Vulnerability validation and reporting
  - For each promising finding, developed concrete proof‑of‑concepts and verified exploitability.
  - Documented confirmed vulnerabilities with step‑by‑step reproduction and exploit scripts in dedicated vulnerability reports.

# Technical Analysis

Technical analysis

1. Host and service overview
Reconnaissance confirmed the following noteworthy services:

- Port 80: nginx 1.10.3 serving a default “Welcome to nginx on Debian!” page.
- Port 9201: Python BaseHTTPServer 0.3 (Python 2.7.12) serving JSON.
- Port 5555: Custom text‑based “Member manager!” service.
- Port 7777: Custom text‑based “Spiritual Memo” service.

SSH (22/2222) and DNS (53) were identified but not subjected to exhaustive brute‑force or configuration review within this engagement; the main focus remained on the web/API and custom TCP services.

2. HTTP service on port 80 (nginx)
Behavior
- Only one reachable endpoint was identified:
  - GET / → static nginx default HTML page.
- No additional routes (e.g., /admin, /login, /api) were discovered despite:
  - Crawling with automated spiders.
  - Directory/file brute‑forcing with common wordlists and extensions.
- No forms, query parameters, or client‑side JavaScript were present.

Findings
- No dynamic or stateful functionality was exposed, therefore classical web application vulnerabilities (SQLi, XSS, IDOR, CSRF, business logic issues) did not apply.
- Some common dotfiles and configuration paths responded with 403, indicating basic hardening, but no sensitive content was accessible.

Assessment
- No exploitable vulnerabilities were identified on port 80 within the observed static content.

3. HTTP JSON API on port 9201 (Python BaseHTTPServer)
Functionality mapping
- Two primary GET endpoints:
  - GET /:
    - Returns a JSON array of incident records: each object has category, subject, body, and timestamp fields.
    - An optional category query parameter is accepted but, in practice, does not appear to filter results.
    - Methods other than GET result in “unsupported method” responses.
  - GET /search?category=<string>:
    - Returns a JSON array filtered by category (case‑insensitive).
    - Valid categories (e.g., “maintenance”, “outage”) return corresponding incidents.
    - Unknown categories return an empty array [].
    - The category parameter is required; if omitted entirely, the server responds with a bare “Missing category” string without HTTP headers (non‑HTTP response).

- The API is unauthenticated and read‑only; no cookies, tokens, or sessions were observed.

Injection and protocol robustness testing
- Injection testing on /search and /:
  - Extensive payload spraying against the category parameter was performed, including SQLi patterns, command‑injection metacharacters, template‑injection markers, path‑traversal sequences, CRLF sequences, and very long inputs.
  - Responses remained consistent with simple string filtering:
    - Known categories returned expected incidents.
    - Non‑matching/garbage values returned [].
  - No SQL errors, stack traces, or anomalous delays suggesting server‑side execution or query manipulation were observed.

- HTTP robustness and desync:
  - Confirmed that certain error paths (e.g., /search without any category parameter, or malformed request lines with embedded CRLF) produce non‑HTTP responses (plain text “Missing category” or HTML without an HTTP status line).
  - Tested pipelined and smuggling‑style requests, including:
    - Multiple requests in a single TCP write.
    - Mismatched Content‑Length and embedded “GET / ...” sequences in bodies.
  - In all tests, the backend processed effectively one logical request per connection, did not produce multiple desynchronized responses, and did not interpret embedded data as second requests.

Findings
- No exploitable SQL injection, command injection, or other server‑side injection vulnerabilities were identified.
- No practical HTTP request smuggling or desynchronization exploits were found in the standalone deployment.
- The non‑HTTP error responses are protocol‑compliance issues and could cause interoperability or monitoring inaccuracies, but do not directly translate into confidentiality or integrity compromise in the current architecture.

Severity
- The 9201 API issues are best categorized as low‑severity robustness/implementation concerns, not primary security vulnerabilities.

4. “Member manager!” TCP service on port 5555
Protocol summary
- On connection, the server prompts “enter your name:” and then presents a menu:
  1. add
  2. edit
  3. ban
  4. change name
  5. get gift
  6. exit

- Key behaviors:
  - add: Prompts for “size:” and “username:”, with only a narrow band of sizes accepted; suggests user‑influenced heap allocations.
  - edit: Offers “secure edit” and “insecure edit” suboptions. Insecure edit accepts arbitrary indices and large payloads for “new username”, with no immediate crash observed.
  - ban: Accepts an index and reports users as banned or “no such user!”.
  - change name: Allows changing the global name, accepting large inputs.
  - get gift: Returns a numeric “gift” value.
  - exit: Closes the connection cleanly.

Confirmed vulnerability: Unauthenticated memory‑address disclosure via “get gift”
- Behavior:
  - Any client can:
    1) Connect to port 5555.
    2) Supply any name.
    3) Choose option 5 (“get gift”).
  - The server responds with:
      your gift:
      <large_integer>
  - The integer:
    - Is stable within a single session across repeated calls.
    - Differs across sessions.
    - Consistently falls in a range and pattern characteristic of process memory pointers (e.g., high 64‑bit addresses typically associated with userland segments, often aligned).

Security significance
- The leaked values are highly indicative of raw process memory addresses or closely related pointers.
- Access is unauthenticated: any network client that can reach port 5555 can obtain this leak.
- Combined with:
  - The presence of an explicitly labeled “insecure edit” path that allows controlled writes associated with user indices and sizes.
  - Observed heap‑like behavior for user records keyed by size and index.
- The infoleak substantially lowers the complexity of turning latent memory‑corruption defects in this service into reliable remote code execution by enabling ASLR bypass and precise memory targeting.

Validation and reporting
- A dedicated validation effort confirmed:
  - The stability and variability characteristics of the leaked value.
  - That the leak does not depend on authentication or complex interaction.
- A separate vulnerability report was created documenting this issue with a Python proof‑of‑concept that:
  - Connects to the service, responds to prompts, selects option 5, extracts the numeric gift value, and prints it in decimal and hexadecimal.

Severity
- High: While the leak alone does not execute arbitrary code, it is a powerful building block for RCE against a native service that already exhibits unsafe patterns (insecure editing, index/size handling).

5. “Spiritual Memo” TCP service on port 7777
Protocol summary
- On connection, the service displays:
  --==[[ Spiritual Memo ]]==--
  [1] Create a memo
  [2] Show memo
  [3] Delete memo
  [4] Tap out

- Key flows:
  - Create memo:
    - Prompts “Data:” and reads arbitrary input until newline.
    - Prompts “Are you done? [yes/no]”.
    - Typical answers like “yes”/“no” are rejected with “Which part of [yes/no] did you not understand?” and do not appear to commit a memo.
    - A specific non‑standard answer “yes\x00” is accepted, prints “Done!”, and stores the memo such that Show memo subsequently returns the exact data supplied.
  - Show memo:
    - Displays “Data: <bytes>”.
  - Delete memo:
    - Intended to delete the stored memo.
  - Tap out:
    - Exits the service.

Confirmed vulnerabilities: Information disclosure and deterministic remote DoS
- Information disclosure via Show memo:
  - On a fresh session without creating any memo, selecting Show memo (option 2) returns a consistent, non‑textual “Data: <bytes>” line.
  - The bytes appear to be static code or in‑memory data, and the tail of the sequence can be interpreted as a 64‑bit pointer.
  - This constitutes an unauthenticated memory‑disclosure primitive, exposing process layout and potentially code bytes.

- Remote DoS via Delete memo:
  - On a fresh session with no memo created, selecting Delete memo (option 3) triggers a GLIBC abort with a message such as:
    free(): invalid pointer: 0x0000000000400870
    followed by a backtrace and memory map.
  - The invalid pointer value is consistent across runs, indicating that the program is calling free() on a static/global address or otherwise uninitialized pointer.
  - This reliably terminates the process handling the connection, providing a trivial, unauthenticated denial‑of‑service vector.

Exploitability assessment
- Additional validation examined:
  - Multiple sequences of create/show/delete with varying memo lengths and confirmation answers.
  - Repeated uses of delete and show both before and after memo creation.
- Findings:
  - The address freed during the crash is not directly controlled by user input; it is a hard‑coded or static value.
  - User‑controlled data did not appear to be used as a free() target in observed behavior.
  - No reliable path to arbitrary code execution or arbitrary read/write was established during testing.

Reporting
- A dedicated vulnerability report was created for the combined info‑disclosure and DoS condition, including a PoC script that:
  - Triggers the Show memo leak and prints the leaked bytes and derived pointer.
  - Triggers the Delete memo crash and captures the resulting diagnostic output.

Severity
- Medium to High:
  - Information disclosure (memory layout/code leak) increases the feasibility of deeper exploitation if additional bugs exist.
  - The remote DoS enables unauthenticated attackers to terminate service instances at will, impacting availability.

6. Summary of non‑findings and lower‑risk issues
- No SQL injection, command injection, or authentication/authorization bypass was found on the HTTP JSON API at 9201 or the static HTTP service at 80.
- Non‑standard HTTP error responses on 9201 are protocol‑robustness issues but did not present a clear, exploitable security vulnerability in the tested configuration.

# Recommendations

Recommendations

Priority 1 – Address critical custom TCP service risks (ports 5555 and 7777)

Member manager (5555)
- Eliminate the memory‑address leak:
  - Modify “get gift” so that it does not return pointers or values derived from internal memory addresses.
  - Replace the gift value with:
    - A non‑sensitive value (e.g., application‑level token or random number) unrelated to the process memory layout.
- Harden memory safety:
  - Review all code paths involving:
    - The “insecure edit” functionality.
    - User‑controlled sizes and indices used for allocations and array access.
  - Enforce strict bounds checking on indices and sizes and ensure all allocations and writes remain within valid object boundaries.
  - Where feasible, adopt safer abstractions or languages for this service to reduce memory‑corruption risk.
- Deployment hardening:
  - Restrict network access to the service to only trusted hosts or internal networks (e.g., via firewall rules, access control lists, or service mesh).
  - Run the service with least privilege (dedicated low‑privilege user, minimal capabilities, isolation via containers or sandboxes where possible).

Spiritual Memo (7777)
- Fix memo lifecycle and pointer handling:
  - Ensure memo pointers are:
    - Properly initialized (e.g., to NULL).
    - Only freed if they reference heap‑allocated memory that is currently owned by the service.
  - On Delete memo:
    - Check that a memo is actually present and that its pointer is valid before calling free().
  - On Show memo:
    - Ensure that uninitialized pointers are not dereferenced or inspected to produce output.
- Address the information leak:
  - Ensure Show memo returns a well‑defined, sanitized value:
    - If no memo exists, return a fixed message rather than raw memory content.
    - After memo deletion, clear or reset pointers so they cannot be used to produce stale or arbitrary memory reads.
- Input‑parsing hardening:
  - Correctly parse the “Are you done? [yes/no]” prompt:
    - Enforce strict comparison against normalized values (“yes”, “no”).
    - Reject or handle embedded NULL bytes and other control characters safely.
- Availability and monitoring:
  - Deploy watchdogs or process supervisors to automatically restart crashed instances, limiting the impact of any remaining DoS vectors.
  - Implement logging and alerting on abnormal termination (e.g., GLIBC aborts) to detect active exploitation.

Priority 2 – Web/API hardening (ports 80 and 9201)

HTTP JSON API (9201)
- Normalize error handling:
  - Replace ad‑hoc error writes with proper HTTP responses:
    - For missing category on /search, return an HTTP 4xx status (e.g., 400 or 422) with a JSON error body and appropriate Content‑Type.
    - For malformed requests, use standard 400 responses with a correct HTTP status line and headers.
- Maintain strict HTTP semantics:
  - Always return a valid status line and headers for any response.
  - Ensure Connection: close is used consistently if the service does not support keep‑alive or pipelining.
- Access control:
  - Confirm that the incident data exposed at / and /search is intended for unauthenticated access.
  - If this data is sensitive or internal, introduce authentication (e.g., API keys, IP allowlisting, or identity‑aware access).

HTTP on port 80
- Replace the default nginx page with:
  - Either a customized landing page appropriate for the environment, or
  - A 404/403 response if no public web interface is required.
- Remove or restrict access to this service if it is not needed.

Priority 3 – Infrastructure and operational controls

- Network segmentation and exposure management:
  - Limit external exposure of custom TCP services (5555 and 7777) as much as possible:
    - Consider binding these services only to internal interfaces or placing them behind authenticated gateways.
  - Segment infrastructure so that compromise of these services does not directly expose critical back‑end systems.

- Monitoring and logging:
  - Implement centralized logging for:
    - All connection attempts and menu actions on ports 5555 and 7777.
    - Crashes and abnormal terminations of these services.
  - Monitor for patterns indicative of exploitation attempts (e.g., repeated Delete memo calls on fresh sessions, high‑frequency get gift requests from the same source).

- Secure development lifecycle:
  - Integrate memory‑safety checks and fuzzing into the development lifecycle for all native/bespoke services.
  - Encourage the use of safer languages or hardened libraries when building new services with complex memory management.
  - Perform code reviews focusing on pointer handling, allocation/free logic, and boundary checks for existing C/C++ components.

Follow‑up
- After remediation, schedule a focused re‑test of:
  - Member manager (5555) and Spiritual Memo (7777) to confirm that the infoleaks and DoS conditions are resolved and no new regressions are introduced.
  - The JSON API (9201) to ensure HTTP error handling has been normalized and that no new smuggling/desync conditions arise from changes.
- Consider broader security assessments if these services are part of a larger platform (e.g., including authentication, data storage layers, and administrative interfaces) to ensure there are no additional systemic weaknesses.

