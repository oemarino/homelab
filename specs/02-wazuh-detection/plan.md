# Plan: Wazuh SOC and custom detection

**Phase:** plan. Written 2026-11-06 against the stack already in the repo. Spec: [spec.md](spec.md). Tasks: [tasks.md](tasks.md).

## Stack

| Piece | Role |
|---|---|
| Wazuh manager, indexer, dashboard **4.14.9** | Single-node SIEM in Docker |
| Wazuh agent on the Ubuntu 24.04 host | Reads local logs, FIM, SCA, and Docker events |
| OpenSearch indexer | Alert store. Published on localhost only. |
| Custom rules 100100 / 100101 | Boolean SQL injection in web access logs |
| Lab nginx access log | Web detection source. The lab itself is specified under [attack lab](../03-attack-lab-incident/plan.md). |

The dashboard listens on **5443** because 443 is already used by Greenbone on this host. Agent enrollment uses an authd password file on the server. That file is not in the repo.

## Architecture

```mermaid
flowchart TB
  dash[dashboard :5443]
  idx[indexer localhost:9200]
  mgr[manager API localhost:55000]
  dash --- idx --- mgr
  agent[Wazuh agent on the host] -->|127.0.0.1:1514| mgr
  auth[/var/log/auth.log] --> agent
  ufw[/var/log/ufw.log] --> agent
  syslog[/var/log/syslog] --> agent
  fim["/etc real-time FIM"] --> agent
  sca[CIS Ubuntu 24.04 SCA] --> agent
  docker[docker-listener] --> agent
  access["/opt/attack-lab/logs/nginx/access.log"] --> agent
```

Binding is part of the design. Docker-published ports skip ufw, so [wazuh/docker-compose.yml](../../wazuh/docker-compose.yml) publishes:

- Agent ports 1514 and 1515 on `127.0.0.1`, `${LAN_IP}`, and `${TAILSCALE_IP}` only.
- API 55000 and indexer 9200 on `127.0.0.1` only.
- Dashboard host port 5443 on those same three addresses, mapped to container port 5601.
- Syslog 514/udp is not published.

Demo passwords for the indexer admin, kibanaserver, the API user, and the other built-in users were replaced with random 26-character secrets before first boot. They live in a root-only file (mode 600). A check confirmed the old defaults are rejected with HTTP 401. [wazuh/.env.example](../../wazuh/.env.example) has placeholders only.

The agent on the host talks to the manager container at `127.0.0.1:1514`. The sanitized excerpt is [wazuh/agent-ossec.conf.snippet.xml](../../wazuh/agent-ossec.conf.snippet.xml). On the server the full file is `/var/ossec/etc/ossec.conf` (original kept as `ossec.conf.orig`). The compose project on the server is `/opt/wazuh/wazuh-docker/single-node/`, with upstream files kept as `*.orig`.

## What the agent collects

| Source | Config | Rules named in the SOC write-up |
|---|---|---|
| `/var/log/auth.log` | syslog localfile | 5710, 5712, and also 5715, 5760, 5402, 5501 |
| `/var/log/ufw.log` | syslog localfile | ufw block events |
| `/var/log/syslog` | syslog localfile, UFW lines ignored so they are not double-counted | |
| Docker engine | `docker-listener` wodle | 87901–87932 |
| `/etc` | syscheck, `realtime="yes"` | 554, 550, 553 |
| CIS Ubuntu 24.04 | SCA, scan on start, 12h interval | 19003 at 50% on the first scan |
| Lab proxy access log | apache-format localfile at `/opt/attack-lab/logs/nginx/access.log` | 31106, then 100100/100101 |

## Custom rule

[wazuh/rules/100100-boolean_sqli_rules.xml](../../wazuh/rules/100100-boolean_sqli_rules.xml) matches the shape of a boolean injection (a quote, digit, or closing parenthesis, whitespace or an inline comment, `OR` / `AND` / `XOR` / `||`, then a comparison). It uses a case-insensitive PCRE2 `<url>` match. It does not add more SQL keywords.

Deployment on the server: the file is copied to `/var/ossec/etc/rules/` in the manager container, on the `wazuh_etc` volume, so it survives restart. The change procedure was: archive `etc/rules`, `etc/decoders`, and `ossec.conf`; add the file; restart only the manager container; wait for the agent to reconnect. The rule file in git is the sanitized copy of that logic.

Parent rules stay 31100 and 31108, the same parents the stock SQLi rule uses, so a UNION payload keeps rule 31106.

## Detection limits this architecture still has

- The rule sees the URL in the access log. POST bodies never reach it. The next control for that gap is a WAF in front of the lab, tracked in [the attack-lab tasks](../03-attack-lab-incident/tasks.md).
- Web correlation rule 31151 sees HTTP 400/404 bursts, which is how the Nmap HTTP probes were caught. A port scan that never speaks HTTP needs Suricata. That is an open task in [tasks.md](tasks.md).
- One crafted filter query (`year>2000 and year<2010`) matches the boolean pattern. The write-up keeps it as a known edge. A level-0 child rule is the documented remedy if an app ever uses that style. None of mine do.

## Write-ups

- [docs/wazuh-soc-lab.md](../../docs/wazuh-soc-lab.md) — deployment, hardening decisions, and the first four detection tests.
- [docs/wazuh-custom-rule-boolean-sqli.md](../../docs/wazuh-custom-rule-boolean-sqli.md) — gap, logtest, live re-test, and the 29,970-request false-positive check.
- [docs/incident-report-simulated-attack.md](../../docs/incident-report-simulated-attack.md) — later confirmation that 100101 fired on a scripted attack, plus the POST-body miss.
