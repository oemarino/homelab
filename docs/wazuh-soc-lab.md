# Home SOC Lab: Wazuh SIEM + Isolated Attack Lab

*Built October 20, 2026 on my homelab server (Ubuntu 24.04, 4 CPU cores, 16 GB RAM)*

## Summary
I deployed a single-node **Wazuh 4.14.9** SIEM in Docker, enrolled the homelab server as a monitored endpoint, and built an **isolated, network-segmented practice lab** (DVWA + OWASP Juice Shop) whose web traffic feeds into the SIEM. Then I ran controlled attacks and checked what the SIEM detected, and what it missed.

## Architecture
```
 LAN / Tailscale (private only)
        │  5443/tcp (dashboard)  1514-1515/tcp (agent enrollment/events)
        ▼
 ┌──────────────── Docker: Wazuh single-node (v4.14.9) ────────────────┐
 │ wazuh.dashboard ── wazuh.indexer (OpenSearch, localhost:9200 only)  │
 │ wazuh.manager  (API on localhost:55000 only)                        │
 └─────────────────────────────▲───────────────────────────────────────┘
                               │ encrypted agent channel (127.0.0.1:1514)
 Wazuh agent on the host: auth.log, ufw.log, syslog, Docker events,
 real-time file integrity monitoring (FIM) on /etc, CIS benchmark (SCA), lab web logs
                               ▲
 ┌──────────── Docker: attack-lab (off by default) ────────────┐
 │ nginx proxy (localhost + Tailscale only, :8090 / :8091)     │──► access.log
 │    │  "lab-internal" network: internal=true (no internet)   │
 │    ├─ DVWA (PHP/Apache) ── MariaDB                          │
 │    └─ OWASP Juice Shop                                      │
 └─────────────────────────────────────────────────────────────┘
```

**Hardening decisions**
- Every published port is bound to specific interfaces (localhost, LAN and Tailscale). Nothing listens on `0.0.0.0`. I bound by IP because Docker-published ports skip host firewall (ufw) rules. I checked the result with `ss`.
- The indexer (9200) and the Wazuh API (55000) are localhost-only. The unused syslog port (514/udp) is not published.
- All the default demo passwords (indexer admin, kibanaserver, the API user and the other built-in users) were replaced with random 26-character secrets **before first boot**. The secrets are stored in a root-only file (mode 600). I confirmed the old default passwords are rejected (HTTP 401).
- Dashboard runs on 5443 because 443 is already used by the Greenbone/OpenVAS scanner on the same host.
- The lab apps sit on a Docker `internal` network. An egress test from the DVWA container to the internet failed (blocked). Only the nginx proxy is reachable, and only from the server itself or the Tailscale network.
- The lab containers have CPU, memory and process limits (DVWA 256 MB, MariaDB 384 MB, Juice Shop 512 MB, proxy 64 MB). They use `restart: "no"`, so the lab is off unless I start it.

## What's monitored
| Source | How | Example rules seen |
|---|---|---|
| SSH / sudo / PAM (`/var/log/auth.log`) | logcollector (syslog) | 5710, 5712, 5715, 5760, 5402, 5501 |
| Firewall (`/var/log/ufw.log`) | logcollector (syslog) | ufw block events |
| System log (`/var/log/syslog`, UFW lines filtered out to avoid duplicates) | logcollector | |
| Docker engine | `docker-listener` wodle | 87901-87932 (container create/start/die, exec, network, volumes) |
| `/etc` file integrity | syscheck, **real-time** | 554 added, 550 modified, 553 deleted |
| CIS Ubuntu 24.04 benchmark | SCA module | 19003 (score 50% on first scan), 19007-19009 |
| Attack-lab web traffic | nginx access log, `apache` format | 31106 |

## Detection results (real data from this build)

### 1. SSH brute force with a nonexistent user (controlled test)
I made 6 failed password logins as `invaliduser` against `127.0.0.1`. That's 12 auth.log events (an "Invalid user" line and a "Failed password" line for each attempt).
- **Rule 5710** (level 5, "sshd: Attempt to login using a non-existent user", MITRE T1110.001 / T1021.004): **11 alerts**
- **Rule 5712** (level 10, "sshd: brute force trying to get access to the system", MITRE T1110): **1 alert**. This correlation rule fired on the 8th matching event, in place of that event's 5710 alert, which is why there are 11 + 1.

### 2. Unplanned real-world catch: vulnerability scanner traffic
While I was setting this up, a Greenbone/OpenVAS rescan of the server was running. Its default-credential SSH checks, sent from the scanner's container, produced **735 rule 5710 alerts** (usernames like `manuf` and `3comcso`) and **1 rule 5712 level-10 brute-force alert** within about a minute. The SIEM traced the activity to the scanner's container IP, so I could confirm it was authorized scanning and not an attack. That's the kind of triage an analyst does every day.

### 3. SQL injection against DVWA (security level: low)
I ran these from the server through the lab proxy:
| Payload | Attack worked? | Wazuh alert? |
|---|---|---|
| `id=1' OR '1'='1` (URL-encoded with `+` for spaces) | Yes, all 5 user rows returned | **No** |
| `id=1' UNION SELECT user,password FROM users-- -` | Yes, 5 password hashes returned | **Yes: rule 31106**, level 6, "A web attack returned code 200 (success)", MITRE T1190 |

**Lesson:** Wazuh's signature rules for web logs caught the UNION-based injection but missed the boolean `OR '1'='1'` variant, so signature detection alone has gaps. Possible next steps: write a custom rule, or add a WAF such as ModSecurity with the OWASP Core Rule Set (CRS) in front of the lab.

### 4. File integrity monitoring
I created, modified and deleted a test file in `/etc`. Real-time FIM raised **554** (file added, level 5), **550** (checksum changed, level 7) and **553** (file deleted, level 7) within seconds.

## How to use it
- **Dashboard:** `https://<server LAN IP>:5443` from home, or `https://<server Tailscale IP>:5443` from anywhere on the tailnet. The certificate is self-signed, so accept the browser warning. Credentials live in a root-only file on the server.
- **Useful searches (Threat Hunting → Events):** `rule.id:(5710 or 5712)`, `rule.id:31106`, `rule.groups:syscheck`, `rule.groups:docker`.
- **Lab:** `lab up` / `lab down` / `lab status` / `lab logs`
  - DVWA: `http://127.0.0.1:8090` (or the Tailscale IP). Log in with DVWA's well-known default credentials; security level defaults to *low*.
  - Juice Shop: `http://127.0.0.1:8091`
  - Turn the lab off when you're done. It's intentionally vulnerable.
- **Workflow:** attack in the lab → find the alert in Wazuh → write up the attack and the detection.

## Files
- `/opt/wazuh/wazuh-docker/single-node/`: Wazuh compose stack (original files kept as `*.orig`)
- `/var/ossec/etc/ossec.conf`: agent config (original saved as `ossec.conf.orig`)
- `/opt/attack-lab/`: lab compose, nginx config, `lab` script, logs
