# Spec: Hardening and exposure

**Phase:** spec. Written 2026-11-06 to document work already built. Every number below is taken from [docs/hardening-greenbone.md](../../docs/hardening-greenbone.md), [docs/external-exposure-check.md](../../docs/external-exposure-check.md), or the README. Plan: [plan.md](plan.md). Tasks: [tasks.md](tasks.md).

## Problem

The homelab server (Ubuntu Server 24.04 on LAN `192.168.1.0/24`) was running the AMP game panel, OpenSSH, Webmin, and Docker. A Greenbone Community Edition scan on 2026-10-20 found weak SSH algorithms, a cleartext panel login, TCP and ICMP timestamp leaks, and two AMP SFTP ports whose software cannot drop weak ciphers. Separately, I needed an outside-in answer to what the public internet can actually connect to.

## Goals

- Remove the Medium and Low findings that a config change or a firewall limit can remove, and show the result with a second scan of the same task and target.
- Where the software cannot change its algorithms (AMP SFTP), reduce exposure and document the residual risk.
- Keep an SSH session alive across each change.
- Measure inbound exposure from outside the network, and discard any scan path that fails a sanity check.
- Leave the one finding that is not a real file disclosure documented, with the manual check that supports that conclusion.

## Non-goals

- Changing AMP's SFTP cipher or key-exchange settings. The vendor file only sets enable, port, bind address, and idle timeout. Restarting the Minecraft instance to rebind or disable SFTP was out of scope.
- Treating the Tailscale limit on ports 2223 and 2224 as a cryptographic fix.
- A full 65,535-port external scan, or an external UDP scan. The exposure check states both limits.
- Re-running the external scan after the website was added later the same day. The site uses an outbound-only Cloudflare Tunnel and adds no inbound port; the write-up records that the external scan was not repeated.
- A trusted certificate for the panel. The proxy certificate is self-signed today. `tailscale serve` is an open task.
- A Greenbone override for the false positive. The 2026-10-20 reports were filtered at QoD ≥ 70 with no overrides.
- Hardening `website/` or `amp-https/` as their own projects. Both are shared infrastructure. The proxy is the control that closed the cleartext-login finding; the site is relevant only to the exposure result.

## Success metrics

Measured on the Greenbone scans (config "Full and fast", port list "All TCP and Nmap top 100 UDP", QoD ≥ 70, no overrides). Baseline 2026-10-20 06:37 UTC, rescan 2026-10-20 23:48 UTC.

| Severity | Before | After |
|---|---|---|
| Critical | 0 | 0 |
| High | 0 | 0 |
| Medium | 6 | 1 (documented false positive) |
| Low | 3 | 0 |
| Log (info) | 43 | 51 |
| CVEs referenced | 0 | 0 |

Medium and Low findings went from **9 to 1**. The remaining Medium is "Linux Home Folder Accessible." Log entries rose because the new HTTPS listener on 8443 produced informational TLS and HTTP notes. None of those notes carry a severity.

Findings closed, as recorded in the write-up:

| Finding | Severity | What changed |
|---|---|---|
| Weak key exchange on AMP SFTP (2223 and 2224) | Medium 5.3, two instances | Tailscale-only in `before.rules`. Ports closed from the LAN. Absent from the rescan. |
| Weak ciphers (CBC, 3DES, Twofish) on the same ports | Medium 4.3, two instances | Same exposure limit. Absent from the rescan. |
| Cleartext panel login | Medium 4.8 on port 8080 | HTTPS proxy on 8443. Port 8080 closed from the LAN. No cleartext finding in the rescan. |
| Weak SSH MACs `umac-64@openssh.com` and `umac-64-etm@openssh.com` | Low 2.6 on port 22 | sshd drop-in. `ssh -m umac-64@openssh.com` reports no matching MAC. Absent from the rescan. |
| TCP timestamps | Low 2.6 | `net.ipv4.tcp_timestamps = 0`. Absent from the rescan. |
| ICMP timestamp replies | Low 2.1 | Drop ICMP type 13. Echo still works. Absent from the rescan. |

Before the rescan, a reachability check from a container on the scanner's Docker network showed ports 22, 8443, and 10000 open, and ports 8080, 2223, and 2224 closed.

Outside-in check, 2026-10-20, after a direct `nmap` from the scanning machine was discarded (its proxy marked known-open ports on `scanme.nmap.org` as filtered):

| Check | Result |
|---|---|
| IPv4, 1,007 TCP ports via portchecker.io, including 22, 80, 443, 3306, 8080, 8443, and 10000 | **0 open** |
| IPv6, both server addresses, 16 common ports each, via pingie.net | **all timed out** |

check-host.net was used to separate "timed out" from "refused." Each checker was validated against a host with known open ports.
