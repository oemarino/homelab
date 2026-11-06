# Constitution

These phase documents were written on 2026-11-06, after the October 2026 build. They organize work that is already in this repository and list the next work. The evidence stays in [`docs/`](docs/) and in the configs those write-ups point to. Nothing here replaces a write-up, and nothing here adds a second copy of a config.

## Principles

The rules below are the ones the README and the write-ups already follow.

### Sanitized configs, no secrets

Published files are examples. Passwords, API keys, tunnel credentials, TLS private keys, the Wazuh enrollment password, public addresses, Tailscale addresses, and tunnel IDs are not committed. They appear as `${VARIABLES}`, `<PLACEHOLDERS>`, or `.env.example` files. The LAN is shown as `192.168.1.0/24`. See [SECURITY.md](SECURITY.md).

### Exposure first

When a service cannot drop a weak algorithm, limit who can reach it and write down what remains. The AMP SFTP servers on ports 2223 and 2224 still offer SHA-1 key exchange and CBC ciphers, because the vendor config has no cipher setting. Rules in `before.rules` accept those ports only on `tailscale0` and drop them on every other interface. The hardening write-up records that as exposure reduction. The weak algorithms stay until the vendor changes them. See [docs/hardening-greenbone.md](docs/hardening-greenbone.md).

### One control at a time, with an SSH lifeline

Back up the file, change one control, and confirm the open SSH session still works before the next change. For OpenSSH, validate with `sshd -t`, reload so existing sessions stay up, and complete a new login before closing the old one. After the set of changes, rescan with the same Greenbone task and target. The temporary ufw allow for the scanner's Docker subnet comes off when the scan finishes.

### Evidence for false positives

A finding that stays open is explained with a check. The remaining Greenbone Medium, "Linux Home Folder Accessible," is HTTP 200 from AMP's single-page app, which returns its login page for every unknown path. A request for `/.ssh/known_hosts` does not return `known_hosts` content. The finding moved from port 8080 to 8443 together with the panel, which ties it to AMP's routing. An override that hides it is still future work, listed in the hardening tasks.

### Prove detections

A detection is finished when the missed attack alerts, alerts the stock rules already raised still fire, and the new pattern does not match real benign traffic. Custom rules 100100 and 100101 were checked with `wazuh-logtest` (18 crafted lines), a live re-test in the lab (3 of 3 boolean payloads at rule 100101, UNION still rule 31106), and the same regex against 29,970 requests from the portfolio site (0 matches). See [docs/wazuh-custom-rule-boolean-sqli.md](docs/wazuh-custom-rule-boolean-sqli.md).

### The lab has no egress

DVWA and Juice Shop sit on a Docker network with `internal: true`. An egress test from the DVWA container to the internet failed, which is the intended result. Only the nginx proxy is reachable, and only from the server itself or over Tailscale. `restart: "no"` keeps the lab stopped unless it is started for a test. See [attack-lab/compose.yaml](attack-lab/compose.yaml) and [docs/wazuh-soc-lab.md](docs/wazuh-soc-lab.md).

### Authorized, self-targeted testing only

Attack traffic in this repo is against systems I own. The October 26 simulation ran from the homelab server against the lab proxy. No production service, third party, or internet-facing asset was a target. SSH on the host was left out of that exercise on purpose. The lab was powered off and checked afterward. See [docs/incident-report-simulated-attack.md](docs/incident-report-simulated-attack.md).

### Bind addresses explicitly

Docker-published ports skip ufw. Every published service in this repo binds to localhost, the LAN address, or the Tailscale address, and the listening address is checked with `ss`. The Wazuh indexer (9200) and API (55000) are localhost-only. The website listens on `127.0.0.1` only. The Cloudflare Tunnel is outbound, so the site has no inbound firewall rule.

## Phases

| Phase | What it is in this repo |
|---|---|
| Constitution | This file. Rules for all three projects. |
| Spec | `specs/<project>/spec.md`. Problem, goals, non-goals, and results already measured. |
| Plan | `specs/<project>/plan.md`. Stack and architecture, with links to the files that implement them. |
| Tasks | `specs/<project>/tasks.md`. Checked items are finished work. Unchecked items are the README's next steps, each with a done-when line. |
| Implement | The configs, compose files, rules, and write-ups linked from each task. Checked work is already implemented. Unchecked work has no config in the repo yet. |

## Projects

1. [Hardening and exposure](specs/01-hardening-exposure/spec.md) — [`host-hardening/`](host-hardening/), [Greenbone before/after](docs/hardening-greenbone.md), [outside-in check](docs/external-exposure-check.md).
2. [Wazuh SOC and custom detection](specs/02-wazuh-detection/spec.md) — [`wazuh/`](wazuh/), [SOC lab](docs/wazuh-soc-lab.md), [boolean SQLi rule](docs/wazuh-custom-rule-boolean-sqli.md).
3. [Attack lab and incident report](specs/03-attack-lab-incident/spec.md) — [`attack-lab/`](attack-lab/), [incident report](docs/incident-report-simulated-attack.md).

[`website/`](website/) and [`amp-https/`](amp-https/) are shared infrastructure. The panel TLS proxy is part of the hardening result, and the portfolio site is the outbound-only public host. They have no spec of their own.
