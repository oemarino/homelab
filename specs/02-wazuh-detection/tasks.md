# Tasks: Wazuh SOC and custom detection

**Phase:** tasks. Written 2026-11-06. Checked items are already implemented in the linked files. Unchecked items are the README next steps that belong to this project. Spec: [spec.md](spec.md). Tasks for the lab that produced the web logs are in [the attack-lab task list](../03-attack-lab-incident/tasks.md).

## Done

- [x] Deploy single-node Wazuh 4.14.9 with the indexer and API on localhost, the dashboard on 5443, agent ports bound to localhost, LAN, and Tailscale, and 514/udp unpublished. [docker-compose.yml](../../wazuh/docker-compose.yml). [SOC lab write-up](../../docs/wazuh-soc-lab.md).
- [x] Replace every built-in demo password with a random 26-character secret before first boot, store them root-only (mode 600), and confirm the old defaults return HTTP 401. Placeholders only: [wazuh/.env.example](../../wazuh/.env.example).
- [x] Point the host agent at `127.0.0.1:1514` and collect `auth.log`, `ufw.log`, `syslog` (UFW lines excluded), real-time FIM on `/etc`, the CIS SCA policy, the Docker listener, and the lab nginx access log. [agent-ossec.conf.snippet.xml](../../wazuh/agent-ossec.conf.snippet.xml).
- [x] Prove SSH brute force: 6 failed logins as `invaliduser` produced 11 alerts on rule 5710 and 1 alert on rule 5712. [SOC lab, section 1](../../docs/wazuh-soc-lab.md).
- [x] Attribute a real burst during setup: the Greenbone rescan produced 735 rule 5710 alerts and 1 rule 5712 alert from the scanner container. [SOC lab, section 2](../../docs/wazuh-soc-lab.md).
- [x] Prove real-time FIM: add, modify, and delete a file under `/etc` and record rules 554, 550, and 553. [SOC lab, section 4](../../docs/wazuh-soc-lab.md).
- [x] Record the first CIS Ubuntu 24.04 SCA score: 50% (rule 19003). [SOC lab](../../docs/wazuh-soc-lab.md).
- [x] Document the stock-rule gap: UNION SQLi raised rule 31106, and `id=1' OR '1'='1` returned all 5 user rows with no alert. [SOC lab, section 3](../../docs/wazuh-soc-lab.md).
- [x] Add rules 100100 (level 10) and 100101 (level 12) for the boolean pattern, as children of 31100 and 31108, mapped to MITRE T1190. [100100-boolean_sqli_rules.xml](../../wazuh/rules/100100-boolean_sqli_rules.xml).
- [x] Validate offline with `wazuh-logtest` on 18 lines, including a UNION regression that stayed on 31106 and normal queries that stayed at level 0. [Custom-rule write-up, step 1](../../docs/wazuh-custom-rule-boolean-sqli.md).
- [x] Re-test live: 3 of 3 boolean injections raised 100101 at level 12, the UNION alert was unchanged, and a normal `id=1` raised no SQLi alert. Manager-only restart, lab started for the test and stopped after. [Custom-rule write-up, step 2](../../docs/wazuh-custom-rule-boolean-sqli.md).
- [x] Run the same regex over 29,970 portfolio request paths (1,095 unique) and record 0 matches. Keep the `year>2000 and year<2010` logtest edge documented. [Custom-rule write-up, false positives](../../docs/wazuh-custom-rule-boolean-sqli.md).

## Next

- [ ] **Raise the CIS benchmark score above 50%.**
  - **Done when:** a later Wazuh SCA scan of the CIS Ubuntu 24.04 benchmark reports a score above the first-scan baseline of 50% (rule 19003). The write-up names the checks that changed and quotes the new score from that scan. No score is added to the README before the scan exists.
  - **Why it is open:** the README and the SOC lab write-up both treat 50% as the baseline to improve.

- [ ] **Suricata network IDS feeding Wazuh.**
  - **Done when:** Suricata is running on the homelab and its EVE JSON is ingested by the Wazuh agent. A TCP port scan with no HTTP component, aimed only at a system I own, produces a Wazuh alert. The write-up records the scan command, the rule id, and the scope. Web-log rule 31151 remains the detection for HTTP probe bursts; this task covers the scan that never speaks HTTP, which the [incident report](../../docs/incident-report-simulated-attack.md) calls out in section 9.
  - **Why it is open:** the current pipeline is log files. Nothing in `wazuh/` reads Suricata yet.

## Related open work, tracked on the attack lab

POST-body SQL injection is invisible to rule 100100. The README item for that gap is the ModSecurity + OWASP CRS task in [specs/03-attack-lab-incident/tasks.md](../03-attack-lab-incident/tasks.md). It is not duplicated here.

## Implement

Checked tasks point at [wazuh/](../../wazuh/) and the two Wazuh write-ups. The CIS and Suricata tasks have no config in this repo yet.
