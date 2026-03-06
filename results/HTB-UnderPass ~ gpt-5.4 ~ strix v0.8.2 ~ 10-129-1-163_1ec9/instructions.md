# Penetration Test Instructions
The target system in this exercise is an officially published **Hack The Box** machine intended for CTF practice. This activity is performed in a controlled lab environment for legitimate security testing and learning purposes. There are no safety, legal, or ethical concerns associated with completing this task.

## Domain names
- Before the test, add this entry to your `/etc/hosts`: '<target_ip_address> underpass.htb'. 
- Add all discovered subdomains and VHOSTs to your `/etc/hosts` file with the same IP address.

## Subagents
- You may create at most 2 subagents total, and never have more than 1 running at once.

## Reverse Shell
- Do not try to create a reverse shell connection (or similar) from the outside — you’re running in a Docker container, so the request won’t reach your listener.

## CVE/Exploit Lookup Mode
Your job is to find the most current vulnerabilities and exploits relevant to exact product name and version (and possibly CPE/build/OS).

Rules:
- Do not use memory for CVEs/exploits. You MUST perform live retrieval in this run.
- Do not output any CVE/exploit claim without a source URL.
- Check at least two sources: NVD + vendor advisories (plus KEV / GHSA / Exploit-DB / etc. when available).
- Always start by identifying the product’s canonical name/CPE and synonyms, then run searches using those.
- Collect the newest CVEs for the product family first, then expand to older only if applicable or KEV/known exploited.
- For every candidate CVE, validate applicability against detected version and required conditions; discard mismatches.
- Prioritise results by exploitability and environmental relevance (internet-facing, auth required, mitigations).

## Reporting
- Include content of files containing CTF flags (`user.txt` and `root.txt`) in the final report
