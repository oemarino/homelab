# Incident report: simulated multi-stage web attack against the isolated lab

*October 26, 2026 · Wazuh 4.14.9 (Docker, single node) · DVWA + OWASP Juice Shop in the isolated [attack lab](../attack-lab/)*

**Classification:** CONTROLLED LAB SIMULATION (self-authorized, internal). This is **not** a real breach.

> **Scope and authorization.** I ran this exercise entirely inside the isolated practice lab on my homelab server.
> The attacker and the targets were both systems I own. No production service, third party or internet-facing asset
> was touched. DVWA and Juice Shop are intentionally vulnerable training apps on an `internal: true` Docker network
> with no internet egress, and the lab is powered off except during a test. The goal was to measure what my Wazuh
> deployment detects, and to practice writing a SOC-style incident report.

Source IPs and hostnames are redacted: `<server-ts-ip>` is the server's own Tailscale address, which was both the
attack origin and the address the lab proxy listens on.

## 1. Executive summary
I ran a controlled attack chain against the lab: reconnaissance, then SQL injection, cross-site scripting and local
file inclusion against two deliberately vulnerable apps. Wazuh detected every meaningful stage:

- **Reconnaissance** (Nmap service and script scan) was flagged by correlation rule **31151**, level 10 (MITRE **T1595.002**).
- **Exploitation** was flagged by four rules, all mapped to MITRE **T1190**: my **custom** boolean-SQLi rule **100101**
  (level 12), plus stock rules **31106** and **31103**.
- The highest-severity alerts were the three **level 12** custom boolean-SQLi detections, which confirms the
  [detection gap I closed earlier](wazuh-custom-rule-boolean-sqli.md) now fires on a live attack.

Two honest gaps: (1) a SQL injection sent in an **HTTP POST body** (Juice Shop login bypass) was **not detected**,
because the access-log pipeline only sees the URL and query string; (2) a raw TCP port scan with no HTTP component
would not be caught, because there is no network IDS yet. Both are covered in section 8.

The lab was returned to its powered-off baseline right after the test. No production system was changed.

## 2. Classification and severity

| Field | Value |
|---|---|
| Incident type | Web-application attack chain (recon + exploitation) |
| Environment | Isolated practice lab (DVWA + OWASP Juice Shop behind an nginx reverse proxy) |
| Detection source | Wazuh 4.14.9, agent `homelab-server` (ID 001), reading the lab's nginx access log |
| Highest alert severity | **Level 12** (custom rule 100101: boolean SQLi likely successful) |
| Business impact | **None**: isolated lab, intentionally vulnerable targets, no real data |
| Simulated severity | **High** if the same pattern hit a real public app |
| Status | Closed: lab powered off and verified |

## 3. Environment

| Component | Detail |
|---|---|
| Host | Homelab server, Ubuntu Server 24.04 |
| Attacker origin | The server itself, over Tailscale (`<server-ts-ip>`) |
| Targets | DVWA (`:8090`) and Juice Shop (`:8091`), reachable only on loopback and the Tailscale address; internal-only Docker network, no internet egress |
| Reverse proxy | `nginx:alpine`; its access log is the Wazuh log source |
| SIEM | Wazuh 4.14.9 single node (Docker) |
| Lab control | `lab up` / `lab down` (off by default) |

All times are America/New_York (EDT, UTC-4). Raw log timestamps are UTC.

## 4. Timeline

| Time (EDT) | Stage | Action | Detection |
|---|---|---|---|
| 14:55 | Setup | `lab up`; apps polled until HTTP 200 | — |
| 14:58:08 | — | Attack simulation started | — |
| 14:58:21 | A: Recon | `nmap -Pn -sV -sC -p 8090,8091 <server-ts-ip>` | Rule **31101** ×~16 (HTTP 400/404 probes) → correlation rule **31151**, level 10 (**T1595.002**) |
| 14:58:21 | B: SSH brute force | **Not run.** The lab has no SSH service, and I deliberately did not target production SSH | n/a |
| 14:58:21–22 | C: Access | DVWA login, database reset, security level low | Benign setup traffic |
| 14:58:22 | C1: UNION SQLi | `?id=1' UNION SELECT user,password FROM users-- -` → 200, user/hash table returned | Rule **31106**, level 6 (**T1190**) |
| 14:58:22 | C2: Boolean SQLi ×3 | `?id=1' OR '1'='1`, `?id=1' OR 1=1-- -`, `?id=x' OR 'a'='a` → all 200 | Rule **100101**, level 12, ×3 (**T1190**) |
| 14:58:22 | C3: Reflected XSS | `/vulnerabilities/xss_r/?name=<script>alert(1)</script>` → 200 | Rule **31106**, level 6 (**T1190**) |
| 14:58:22 | C4: LFI / path traversal | `/vulnerabilities/fi/?page=../../../../etc/passwd` → 200 | Rule **31106**, level 6 (**T1190**) |
| 14:58:22 | D1: Juice Shop SQLi (POST) | `POST /rest/user/login {"email":"' OR 1=1--"}` → 200 | **NOT DETECTED** (payload in body, not logged) |
| 14:58:22 | D2: Juice Shop SQLi (GET) | `/rest/products/search?q=apple')) UNION SELECT 1--` → 500 (SQL error) | Rule **31103**, level 7 (**T1190**) |
| 14:58:23 | Detection | Wazuh generated the exploitation alerts | — |
| 15:01 | Recovery | `lab down`; verified no lab containers and nothing listening on 8090/8091 | — |

## 5. Detection and evidence

### 5.1 Alerts that fired

| Rule | Level | Description | Count | MITRE | Stage |
|---|---|---|---|---|---|
| 31151 | 10 | Multiple web server 400 error codes from same source IP | 1 (correlation) | T1595.002 Vulnerability Scanning | A |
| 31101 | 5 | Web server 400 error code | ~16 | — | A (Nmap NSE probes) |
| 31106 | 6 | A web attack returned code 200 (success) | 3 | T1190 Exploit Public-Facing Application | C1, C3, C4 |
| **100101** | **12** | Boolean-based SQL injection returned code 200 (likely successful) | 3 | T1190 Exploit Public-Facing Application | C2 |
| 31103 | 7 | SQL injection attempt | 1 | T1190 Exploit Public-Facing Application | D2 |

When a DVWA attack matched a web-attack signature **and** returned 200, Wazuh escalated it to child rule 31106
instead of the generic "attempt" rule. The Juice Shop GET injection returned 500, so it stayed at 31103. The boolean
payloads matched my custom chain 100100 → 100101.

### 5.2 Custom rule 100101 on a live attack
All three boolean variants raised **100101 at level 12**. Representative alert (source IP redacted):

```json
{
  "timestamp": "2026-10-26T18:58:23.512+0000",
  "agent": { "id": "001", "name": "homelab-server" },
  "rule": {
    "id": "100101", "level": 12,
    "description": "Boolean-based SQL injection returned code 200 (likely successful) from <server-ts-ip>.",
    "mitre": { "id": ["T1190"], "tactic": ["Initial Access"], "technique": ["Exploit Public-Facing Application"] },
    "groups": ["web","accesslog","attack","sql_injection","local"]
  },
  "data": { "protocol": "GET", "srcip": "<server-ts-ip>", "id": "200",
            "url": "/vulnerabilities/sqli/?id=1%27+OR+%271%27%3D%271&Submit=Submit" }
}
```

### 5.3 Other representative alerts (trimmed, source IP redacted)

**Recon, rule 31151 (T1595.002):**
```json
{ "timestamp": "2026-10-26T18:58:21.534+0000",
  "rule": { "id": "31151", "level": 10,
    "description": "Multiple web server 400 error codes from same source ip.",
    "mitre": { "id": ["T1595.002"], "tactic": ["Reconnaissance"], "technique": ["Vulnerability Scanning"] },
    "groups": ["web","accesslog","web_scan","recon"] },
  "data": { "srcip": "<server-ts-ip>", "id": "404", "url": "/evox/about" } }
```

**UNION SQLi, rule 31106 (T1190):**
```json
{ "timestamp": "2026-10-26T18:58:23.510+0000",
  "rule": { "id": "31106", "level": 6, "description": "A web attack returned code 200 (success).",
    "mitre": { "id": ["T1190"], "tactic": ["Initial Access"], "technique": ["Exploit Public-Facing Application"] } },
  "data": { "srcip": "<server-ts-ip>", "id": "200",
    "url": "/vulnerabilities/sqli/?id=1%27+UNION+SELECT+user%2Cpassword+FROM+users--+-&Submit=Submit" } }
```

**Juice Shop GET SQLi, rule 31103 (T1190):**
```json
{ "timestamp": "2026-10-26T18:58:23.524+0000",
  "rule": { "id": "31103", "level": 7, "description": "SQL injection attempt.",
    "mitre": { "id": ["T1190"], "tactic": ["Initial Access"], "technique": ["Exploit Public-Facing Application"] } },
  "data": { "srcip": "<server-ts-ip>", "id": "500",
    "url": "/rest/products/search?q=apple%27))%20UNION%20SELECT%201--" } }
```

### 5.4 Recon result (Nmap, trimmed)
```
PORT     STATE SERVICE VERSION
8090/tcp open  http    nginx
| http-title: Login :: Damn Vulnerable Web Application (DVWA)
8091/tcp open  http    nginx
|_http-title: OWASP Juice Shop
Nmap done: 1 IP address (1 host up) scanned in 13.37 seconds
```

The raw alert export is not published because it contains host-specific addresses and paths; the alerts above are
the relevant excerpts.

## 6. Impact assessment

| Dimension | Assessment |
|---|---|
| Confidentiality | None. The "user table" dumped by the UNION payload holds only DVWA's seeded demo accounts in a throwaway container. |
| Integrity | None. The only write was DVWA's "reset database" step, part of normal lab setup. |
| Availability | None. Lab containers are resource-capped and were torn down after the test. |
| Production systems | None. SSH, ufw, Tailscale ACLs, Cloudflare, the website and the game server were untouched. |
| Hypothetical | Against a real app, the UNION + boolean SQLi would expose the user credential table (usernames + password hashes): a critical confidentiality breach. |

## 7. Root cause
The targets are **intentionally vulnerable** (DVWA at security level "low", OWASP Juice Shop):

- **SQL injection:** input (`id`, search `q`) concatenated into SQL without parameterized queries.
- **Reflected XSS:** `name` reflected into the page without output encoding.
- **LFI / path traversal:** `page` used in a file include without an allow-list or path canonicalization.

The real subject of the exercise is **detection coverage**, not the app flaws.

## 8. Containment, eradication and recovery
**Containment (by design):** targets ran on an internal-only Docker network with no internet egress, reachable only
over loopback and Tailscale, and only while the test ran.

**Recovery:**
1. `lab down`: all four containers (dvwa, dvwa-db, juice-shop, proxy) and both lab networks stopped and removed.
2. Verified `docker compose ps -a` shows no lab containers.
3. Verified nothing listens on TCP 8090 or 8091.
4. Removed temporary attack artifacts (cookies, dumped output) from the host.

## 9. Lessons learned

**What worked**
- Recon was caught by **correlation** rule 31151, not just individual 400s, giving an early "someone is scanning" signal.
- Exploitation produced high-severity, correctly MITRE-mapped alerts.
- The **custom boolean-SQLi rule fired on every variant**, validating the earlier gap fix on a live attack.

**Gaps and recommendations**
1. **POST-body SQLi was invisible.** The nginx access log records method, URL and status, not the body.
   *Fix:* a WAF or body inspection in front of the lab (ModSecurity/Coraza + OWASP CRS) or app-level request logging.
2. **No network-layer visibility.** A pure TCP port scan would not be detected by a log-only pipeline.
   *Fix:* deploy **Suricata** and feed its EVE JSON into Wazuh.
3. **Generic classification.** XSS and LFI were both only "web attack returned 200" (31106).
   *Fix:* custom rules so XSS and path traversal get their own labelled, MITRE-mapped alerts.
4. **No automated response.** *Fix:* test Wazuh active response to block a source after recon + exploitation, in the lab only.

## Appendix: attack commands (in order)
```
# Stage A: recon
nmap -Pn -sV -sC -p 8090,8091 <server-ts-ip>

# Stage B: SSH brute force: not run (no SSH service in the lab)

# Stage C: DVWA (authenticated, security=low)
GET  /vulnerabilities/sqli/?id=1' UNION SELECT user,password FROM users-- -&Submit=Submit
GET  /vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit
GET  /vulnerabilities/sqli/?id=1' OR 1=1-- -&Submit=Submit
GET  /vulnerabilities/sqli/?id=x' OR 'a'='a&Submit=Submit
GET  /vulnerabilities/xss_r/?name=<script>alert(1)</script>
GET  /vulnerabilities/fi/?page=../../../../etc/passwd

# Stage D: Juice Shop
POST /rest/user/login            {"email":"' OR 1=1--","password":"x"}   -> 200 (not detected: body not logged)
GET  /rest/products/search?q=apple')) UNION SELECT 1--                   -> 500 (rule 31103)
```

---
*This describes an authorized, isolated lab simulation performed by the system owner for training and
detection validation. It does not describe a real-world compromise.*
