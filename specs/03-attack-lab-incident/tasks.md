# Tasks: Attack lab and incident report

**Phase:** tasks. Written 2026-11-06. Checked items are already implemented in the linked files. The unchecked item is the README next step that belongs to this project. Spec: [spec.md](spec.md). Plan: [plan.md](plan.md).

## Done

- [x] Define an isolated lab: DVWA, MariaDB 11, and Juice Shop on `lab-internal` (`internal: true`), with only the nginx proxy published on loopback and the Tailscale address. Memory caps and `restart: "no"` match the SOC write-up. [compose.yaml](../../attack-lab/compose.yaml), [nginx/lab.conf](../../attack-lab/nginx/lab.conf).
- [x] Confirm egress from the DVWA container to the internet fails. [SOC lab write-up](../../docs/wazuh-soc-lab.md).
- [x] Add a `lab up|down|status|logs` helper that runs the compose project from `/opt/attack-lab` and follows the proxy access log. [lab](../../attack-lab/lab).
- [x] Ship passwords and the Tailscale address only through an uncommitted `.env`. [attack-lab/.env.example](../../attack-lab/.env.example).
- [x] Send the proxy access log to the Wazuh agent in apache format. [agent snippet](../../wazuh/agent-ossec.conf.snippet.xml).
- [x] Run the 2026-10-26 chain from this server against the lab only: Nmap recon, three boolean SQLi payloads, UNION SQLi, reflected XSS, LFI, one POST-body SQLi, and one GET SQLi. Leave production SSH out. [Incident report](../../docs/incident-report-simulated-attack.md).
- [x] Record what fired: rule 31151 (level 10, T1595.002) on the HTTP probes, rule 100101 (level 12) on 3 of 3 boolean payloads, rule 31106 (level 6) on UNION, XSS, and LFI, and rule 31103 (level 7) on the Juice Shop GET. [Incident report, section 5](../../docs/incident-report-simulated-attack.md).
- [x] Record the miss: the Juice Shop `POST /rest/user/login` body `' OR 1=1--` returned HTTP 200 and produced no alert. [Incident report, stage D1](../../docs/incident-report-simulated-attack.md).
- [x] Power the lab off, confirm no lab containers, confirm nothing listening on 8090 or 8091, and remove temporary attack files from the host. [Incident report, section 8](../../docs/incident-report-simulated-attack.md).

## Next

- [ ] **WAF (ModSecurity + OWASP CRS) in front of the lab, for POST-body injections.**
  - **Done when:** ModSecurity with the OWASP Core Rule Set inspects requests in front of DVWA and Juice Shop. Repeating the missed request from the incident report — `POST /rest/user/login` with body `{"email":"' OR 1=1--","password":"x"}`, which returned HTTP 200 and no Wazuh alert on 2026-10-26 — produces a WAF event that Wazuh alerts on, or a documented block by the WAF. The GET detections from that same exercise still fire: boolean SQLi on rule 100101, UNION/XSS/LFI on rule 31106, and the Juice Shop GET on rule 31103. The lab stays on the internal network, stays off unless the retest is running, and is powered off afterward. A write-up records the counts from that retest. Those counts are not known yet.
  - **Why it is open:** the access-log pipeline cannot see a body. Both the custom-rule write-up and incident-report section 9 name ModSecurity and the OWASP Core Rule Set as the follow-up. No WAF config is in the repo.

## Related open work, tracked on the Wazuh project

A TCP port scan with no HTTP component is the other gap named in the incident report. It is the Suricata task in [specs/02-wazuh-detection/tasks.md](../02-wazuh-detection/tasks.md). It is not duplicated here.

The report also recommends custom rules that label XSS and path traversal on their own, and a lab-only test of Wazuh active response. Those items are not in the README next steps, so they have no task until they are scheduled.

## Implement

Checked tasks point at [attack-lab/](../../attack-lab/) and the incident report. The WAF task has no config in this repo yet.
