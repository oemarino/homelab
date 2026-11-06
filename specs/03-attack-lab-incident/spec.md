# Spec: Attack lab and incident report

**Phase:** spec. Written 2026-11-06 to document work already built. Every number below is taken from [docs/incident-report-simulated-attack.md](../../docs/incident-report-simulated-attack.md), [docs/wazuh-soc-lab.md](../../docs/wazuh-soc-lab.md), or [attack-lab/compose.yaml](../../attack-lab/compose.yaml). Plan: [plan.md](plan.md). Tasks: [tasks.md](tasks.md).

## Problem

I needed a place to run web attacks on purpose and a SOC-style record of what the SIEM caught. The targets had to be deliberately vulnerable, and they had to be unable to reach the internet or the rest of the homelab. The exercise also had to say what was missed.

## Goals

- Run DVWA and OWASP Juice Shop behind one nginx proxy, on a Docker network with no egress, off unless a test is in progress.
- Reach the proxy only from the server itself or over Tailscale.
- Feed the proxy access log to the Wazuh agent already specified in the [Wazuh spec](../02-wazuh-detection/spec.md).
- Execute one controlled chain — reconnaissance, then SQL injection, reflected XSS, and local file inclusion — from this server against that proxy, and write the incident report from the alerts.
- Return the lab to a powered-off state and verify it.

## Non-goals

- Attacking production SSH, the portfolio site, Cloudflare, the game server, or any system I do not own. The report's stage B (SSH brute force) was not run, because the lab has no SSH service and production SSH was out of scope.
- Leaving the lab up after the test.
- Claiming the POST-body Juice Shop login bypass was detected. It was not.
- Adding a WAF, Suricata, per-technique XSS/LFI rules, or active response as part of the completed exercise. The incident report lists those as follow-ups. Of that list, the README schedules the WAF here and schedules Suricata on the [Wazuh tasks](../02-wazuh-detection/tasks.md). Labelled XSS/LFI rules and active response are not README next steps, so they are not tasks yet.
- Publishing the raw alert export. It contains host-specific addresses. The report quotes redacted excerpts.

## Success metrics

### Isolation (built with the lab, confirmed in the SOC write-up)

| Check | Result |
|---|---|
| App network | `lab-internal` with `internal: true` |
| Egress from the DVWA container | Failed, as intended |
| Who can reach the apps | Only the nginx proxy, and the proxy only on `127.0.0.1` and the Tailscale address, ports 8090 (DVWA) and 8091 (Juice Shop) |
| Restart policy | `restart: "no"` |
| Memory caps | DVWA 256 MB, MariaDB 384 MB, Juice Shop 512 MB, proxy 64 MB |

### Simulated attack (2026-10-26, 14:58 EDT)

Classification in the report: controlled lab simulation, self-authorized, internal. Agent `homelab-server` (ID 001). Highest alert: level 12.

| Stage | Action | Detection |
|---|---|---|
| A. Recon | `nmap -Pn -sV -sC -p 8090,8091` against the server's own Tailscale address. Nmap reported both ports open (nginx, DVWA title and Juice Shop title) in 13.37 seconds. | About **16** rule **31101** alerts (HTTP 400/404), then correlation rule **31151**, level 10, MITRE **T1595.002**, count 1. |
| B. SSH brute force | Not run. | n/a |
| C1. UNION SQLi | DVWA `id=1' UNION SELECT user,password FROM users-- -` returned HTTP 200 and the user/hash table. | Rule **31106**, level 6, T1190. |
| C2. Boolean SQLi | Three payloads (`OR '1'='1`, `OR 1=1-- -`, `OR 'a'='a`), each HTTP 200. | Rule **100101**, level 12, **3 of 3**, T1190. |
| C3. Reflected XSS | `name=<script>alert(1)</script>`, HTTP 200. | Rule **31106**, level 6, T1190. |
| C4. LFI | `page=../../../../etc/passwd`, HTTP 200. | Rule **31106**, level 6, T1190. |
| D1. Juice Shop login | `POST /rest/user/login` with body `{"email":"' OR 1=1--","password":"x"}`, HTTP 200. | **Not detected.** The body is not in the access log. |
| D2. Juice Shop search | `GET /rest/products/search?q=apple')) UNION SELECT 1--`, HTTP 500. | Rule **31103**, level 7, T1190. |

Rule 31106 fired **3** times (C1, C3, C4). When a DVWA attack matched a web signature and returned 200, Wazuh used 31106 instead of the generic attempt rule. The Juice Shop GET returned 500, so it stayed at 31103.

### Recovery

`lab down` removed all four containers (dvwa, dvwa-db, juice-shop, proxy) and both lab networks. `docker compose ps -a` showed no lab containers. Nothing was listening on TCP 8090 or 8091. Temporary cookies and dumped output were removed from the host. Production SSH, ufw, Tailscale ACLs, Cloudflare, the website, and the game server were not changed.

### Impact, as assessed in the report

Confidentiality, integrity, and availability impact on real data: none. The UNION output was DVWA's seeded demo accounts in a throwaway container. The only write was DVWA's database reset during setup. The report's hypothetical is separate and is labeled as hypothetical: the same UNION and boolean pattern against a real application would expose usernames and password hashes.
