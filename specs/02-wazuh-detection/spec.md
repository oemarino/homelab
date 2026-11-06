# Spec: Wazuh SOC and custom detection

**Phase:** spec. Written 2026-11-06 to document work already built. Every number below is taken from [docs/wazuh-soc-lab.md](../../docs/wazuh-soc-lab.md), [docs/wazuh-custom-rule-boolean-sqli.md](../../docs/wazuh-custom-rule-boolean-sqli.md), the incident report where it confirms the same rules, or the README. Plan: [plan.md](plan.md). Tasks: [tasks.md](tasks.md).

## Problem

The homelab had host controls and an isolated lab, and no place that turned those logs into alerts. I needed a single-node SIEM on the same Ubuntu Server 24.04 host (4 CPU cores, 16 GB RAM) that could see SSH, the firewall, file integrity, a CIS benchmark, Docker events, and the lab's web access log. The first web test then showed a hole: stock rules alerted on UNION-based SQL injection and missed a boolean payload that returned every DVWA user row.

## Goals

- Run Wazuh 4.14.9 in Docker, with the manager, indexer, and dashboard bound to explicit addresses, and enroll this host as the agent.
- Replace the built-in demo passwords before the stack's first boot.
- Prove four detections with traffic I generated or that I could attribute: SSH brute force, real-time file integrity on `/etc`, UNION-based SQL injection, and (after the gap) boolean-based SQL injection.
- Show that the custom rule does not swallow the stock UNION alert and does not match the portfolio site's real request log.
- Record a CIS Ubuntu 24.04 SCA score as a baseline.

## Non-goals

- Detecting SQL injection in an HTTP body, header, or JSON payload. nginx `combined` logs do not contain them. The Juice Shop POST login bypass is the measured miss, and the WAF follow-up is a task on the [attack lab](../03-attack-lab-incident/tasks.md).
- A network IDS. A TCP port scan with no HTTP component is invisible to the web-log rules. Suricata is an open task below.
- Raising the CIS score in this phase. The first scan is a baseline.
- A multi-node Wazuh cluster, or publishing the indexer and API off localhost.
- Custom rules that give XSS and local file inclusion their own labels. The incident report notes that both currently land on stock rule 31106. That recommendation is not in the README next steps, so it is not a task yet.
- Putting secrets, enrollment passwords, or raw alert exports that contain host addresses into git.

## Success metrics

### Host detections (2026-10-20)

| Test | Result |
|---|---|
| 6 failed SSH passwords as `invaliduser` against `127.0.0.1` (12 `auth.log` lines) | Rule **5710** level 5, **11** alerts. Rule **5712** level 10, **1** alert. 5712 fired on the 8th matching event and replaced that event's 5710 alert. |
| Greenbone rescan SSH default-credential checks, during setup | **735** rule 5710 alerts and **1** rule 5712 alert in about a minute, traced to the scanner container. |
| Create, modify, and delete a file in `/etc` | Real-time FIM rules **554** (added, level 5), **550** (checksum changed, level 7), and **553** (deleted, level 7), within seconds. |
| CIS Ubuntu 24.04 benchmark, first SCA scan | **50%**. Reported as rule 19003, with 19007–19009 also seen. |

### Web detections, before the custom rule

| Payload against DVWA (security level low) | Attack | Wazuh |
|---|---|---|
| `id=1' UNION SELECT user,password FROM users-- -` | Yes, 5 password hashes | Rule **31106**, level 6, MITRE T1190 |
| `id=1' OR '1'='1` | Yes, all 5 user rows | **No alert.** The event stopped at rule 31100. |

Stock rules 31103 and 31104 are keyword and encoding lists. The boolean payload is logged as `id=1%27+OR+%271%27%3D%271`, which contains none of those keywords.

### Custom rules 100100 and 100101 (2026-10-26)

| Check | Result |
|---|---|
| Rule shape | 100100 level 10 on the boolean pattern, any status. 100101 level 12 when the status is 200. Both are children of 31100 and 31108. MITRE T1190. |
| `wazuh-logtest`, 18 crafted lines | The missed payload and the other boolean forms in the write-up alert. A normal DVWA lookup and ordinary "and"/"or" search text stay at level 0. UNION stays on **31106**. HTTP 404 on the boolean payload stays at **100100**. |
| Live re-test through the lab proxy | **3 of 3** boolean injections alerted at rule **100101**, level 12, within 3 seconds. UNION stayed on rule **31106**, level 6. A normal `id=1` raised no SQLi alert. |
| Portfolio access log | The same regex matched **0** of **29,970** request paths (**1,095** unique), including internet noise and scanner probes. |
| Known logtest edge | `?filter=year>2000 and year<2010` matches 100101. None of my apps use that filter style. |

Juice Shop was still booting during the live re-test and returned 502. Those lines produced rule 31120 only, not 100100 or 100101.

The [incident report](../../docs/incident-report-simulated-attack.md) later fired the same 100101 rule on three boolean variants during a scripted attack. That confirmation is counted in the attack-lab spec. It is the same rule, not a second gap closure.
