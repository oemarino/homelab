# Outside-in exposure check (October 20, 2026)

**Question:** what can the internet actually reach on my home connection?

## Method
- My scanning machine's outbound traffic is proxied, so a direct `nmap` from it gave
  misleading results (a sanity check against `scanme.nmap.org` showed known-open ports as
  filtered). I discarded those results.
- Instead I used public outside-in checkers, each validated against a host with known open ports:
  - **portchecker.io:** IPv4, 1,007 TCP ports (validated against scanme.nmap.org)
  - **check-host.net:** multi-country TCP connects; tells "timed out" apart from "refused"
  - **pingie.net:** IPv6 TCP (validated against a public IPv6 DNS server on 443)

## Results
| Check | Result |
|---|---|
| IPv4, 1,007 TCP ports (incl. 22, 80, 443, 3306, 8080, 8443, 10000) | **0 open** |
| IPv6, both server addresses, 16 common ports each | **all timed out** (the ISP gateway's IPv6 firewall blocks unsolicited inbound) |

## Caveats
- Not a full 65,535-port scan, and UDP was not tested from outside.
- The website was added later the same day. It uses an outbound-only Cloudflare Tunnel,
  so it adds no inbound port. That is by design; I did not re-run the external scan afterwards.
