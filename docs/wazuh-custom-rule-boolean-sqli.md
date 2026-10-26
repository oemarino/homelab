# Closing a detection gap: custom Wazuh rule for boolean-based SQL injection

*October 26, 2026 · Wazuh 4.14.9 (Docker, single node) · DVWA in the isolated [attack lab](../attack-lab/)*

## The gap
In the [first SOC lab test](wazuh-soc-lab.md#3-sql-injection-against-dvwa-security-level-low), two SQL injections worked
against DVWA, but Wazuh only alerted on one of them:

| Payload | Attack worked? | Wazuh alert? |
|---|---|---|
| `id=1' UNION SELECT user,password FROM users-- -` | Yes | Rule 31106 (level 6) |
| `id=1' OR '1'='1` | Yes, all 5 user rows | **None** |

**Why it was missed:** the stock web rules in `0245-web_rules.xml` are keyword lists. Rule 31103 looks for
`select`, `union`, `insert`, `where`, `from`, `null,null` and so on, and rule 31104 looks for odd encodings such as `%027`, `%00` or `../..`.
The boolean payload shows up in the nginx access log as `id=1%27+OR+%271%27%3D%271`. None of those keywords is in it, so the
event stopped at rule 31100 (level 0, "Access log messages grouped") and no alert was raised.

## The rule
File: [`wazuh/rules/100100-boolean_sqli_rules.xml`](../wazuh/rules/100100-boolean_sqli_rules.xml), deployed to
`/var/ossec/etc/rules/` in the manager container (the `wazuh_etc` volume, so it persists across restarts).

Rather than adding more keywords, the rule matches the **shape** of a boolean injection:

```
<quote | digit | ")">  <space>  OR | AND | XOR | ||  <space>  <operand>  <comparison>
```

- **space:** `+`, `%20`, `%09`, `%0A`, `%0D`, or an inline comment such as `/**/` (a common filter-evasion trick)
- **quote:** `'`, `"` or their URL-encoded forms `%27` and `%22`
- **comparison:** `=`, `%3D`, `<>`, `!=`, `<`, `>` or `LIKE`
- Matching is case-insensitive, and the pattern uses PCRE2 (`<url type="pcre2">`).

| Rule | Level | Fires when | MITRE |
|---|---|---|---|
| **100100** | 10 | Boolean SQLi pattern in the URL, any status code | T1190 (Exploit Public-Facing Application) |
| **100101** | 12 | Same, and the server answered **200**, so the injection probably worked | T1190 |

Both are children of 31100 and 31108, the same parents the stock SQLi rule uses. Requests that already match the stock
rules, such as UNION SELECT, keep their existing alerts.

## Step 1: offline validation with `wazuh-logtest`
I tested 18 crafted access-log lines before restarting anything:

| Case | Example URL | Result |
|---|---|---|
| Original missed payload | `?id=1%27+OR+%271%27%3D%271` | **100101**, level 12 |
| False condition | `?id=1%27+AND+1%3D2--+-` | **100101**, level 12 |
| String tautology | `?id=1%27+AND+%27a%27%3D%27a` | **100101**, level 12 |
| `%20` spaces, raw `=` | `?id=1%27%20OR%201=1--%20-` | **100101**, level 12 |
| Numeric, no quotes | `?id=1+AND+1=2` | **100101**, level 12 |
| Comment evasion | `?id=admin%27/**/or/**/1=1` | **100101**, level 12 |
| Same payload, HTTP 404 | `?id=1%27+OR+%271%27%3D%271` | **100100**, level 10 |
| UNION SQLi (regression check) | `?id=1%27+UNION+SELECT+user%2Cpassword+FROM+users--+-` | **31106**, level 6 (unchanged) |
| Normal: DVWA lookup | `?id=1&Submit=Submit` | 31100, level 0 (no alert) |
| Normal: search text with "and" or "or" | `?q=rock+and+roll`, `?q=black+or+white`, `?q=terms+and+conditions&page=2`, `?q=O%27Reilly+and+sons`, `?q=1+or+2` | 31100, level 0 (no alert) |
| Normal: other pages | `/login.php`, `/rest/products/search?q=apple`, `/blog/2026/10/20/homelab-and-wazuh/` | 31100 / 31108, level 0 (no alert) |
| Known edge case | `?filter=year%3E2000+and+year%3C2010` | **100101** (false positive, see below) |

Logtest output for the payload that was missed before (source IP redacted):

```
**Phase 2: Completed decoding.
	name: 'web-accesslog'
	id: '200'
	protocol: 'GET'
	srcip: '<lab-gw>'
	url: '/vulnerabilities/sqli/?id=1%27+OR+%271%27%3D%271&Submit=Submit'

**Phase 3: Completed filtering (rules).
	id: '100101'
	level: '12'
	description: 'Boolean-based SQL injection returned code 200 (likely successful) from <lab-gw>.'
	groups: '['web', 'accesslog', 'attack', 'sql_injection', 'local']'
	mitre.id: '['T1190']'
	mitre.tactic: '['Initial Access']'
	mitre.technique: '['Exploit Public-Facing Application']'
	pci_dss: '['6.5', '11.4', '6.5.1']'
**Alert to be generated.
```

## Step 2: live re-test (before → after)
I backed up the manager's `etc/rules`, `etc/decoders` and `ossec.conf` to a timestamped archive, added the rule file and restarted
**only** the Wazuh manager container. The agent reconnected within seconds. Then I started the lab
(`lab up`), sent the same requests from the server through the lab proxy, and turned the lab back off (`lab down`).

| Request (Oct 26, 02:58:51 UTC) | HTTP | Before (Oct 20) | After (Oct 26), real alert |
|---|---|---|---|
| `id=1` (normal) | 200 | no alert | no alert |
| `id=1' OR '1'='1` | 200 | **no alert** | **rule 100101, level 12**, alert ID `1792983533.11509353` |
| `id=1' AND 1=2-- -` | 200 | not tested | **rule 100101, level 12**, alert ID `1792983533.11509953` |
| `id=1' AND 'a'='a` | 200 | not tested | **rule 100101, level 12**, alert ID `1792983533.11510549` |
| `id=1' UNION SELECT user,password FROM users-- -` | 200 | rule 31106 | **rule 31106, level 6** (still fires), alert ID `1792983533.11511150` |
| Juice Shop `/`, `/rest/products/search?q=apple`, `?q=apple+and+banana` | 502* | n/a | rule 31120 only (5xx), **not** 100100/100101 |

\* Juice Shop was still booting, so the proxy returned 502. Those lines still went through the ruleset, and the custom rule
ignored them as it should.

Result: **3 of 3** boolean injections now raise a level-12 alert (within 3 seconds of the request), the UNION alert is unchanged, and
the normal requests raised no SQLi alerts.

## False positives
- **Real traffic:** I ran the same regex over **29,970** request paths (1,095 unique) from my public portfolio site's nginx log.
  That log includes internet background noise and scanner probes. It matched **0** of them.
- **Lab and logtest:** no normal request matched.
- **Known edge case:** filter-style query strings that contain a comparison operator right after `and`/`or`, such as
  `?filter=year>2000 and year<2010`, will match. None of my apps use that style. On a site that does, I'd add a
  level-0 child rule for that specific path, the same way stock rule 31107 ignores search pages.
- **Limits:** this only looks at the URL in the access log. Payloads in POST bodies, headers or JSON aren't logged by nginx's
  `combined` format, so they're invisible to any log-based rule. That's why the next step is a WAF (ModSecurity + OWASP CRS).

## What I learned
- Keyword signatures miss injections that use no SQL keywords at all. Matching the structure of the payload (a quote, a boolean operator, a comparison) is more robust than adding keywords.
- Always regression-test: the new rule must not swallow alerts that the stock rules already raise (UNION → 31106 still fires).
- Test detection rules against **real** benign traffic, not just the attack.
