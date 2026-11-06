# Tasks: Hardening and exposure

**Phase:** tasks. Written 2026-11-06. Checked items are already implemented in the linked files. Unchecked items are the README next steps that belong to this project. Spec: [spec.md](spec.md). Plan: [plan.md](plan.md).

## Done

- [x] Run a Greenbone baseline and a same-day rescan with one task, one target, config "Full and fast", QoD ≥ 70, and no overrides. Medium and Low findings moved from 9 to 1. Critical and High stayed at 0. [Write-up](../../docs/hardening-greenbone.md).
- [x] Back up each changed file and change one control at a time, keeping an SSH session open and confirming the panel still loaded. [Write-up, Method and safety](../../docs/hardening-greenbone.md).
- [x] Install an sshd drop-in that removes `umac-64` and SHA-1 MACs, prefers `sntrup761x25519`, and limits ciphers to AEAD and CTR. Validated with `sshd -t`, applied with a reload, and checked with a new login and with `ssh -m umac-64@openssh.com`. [40-crypto-hardening.conf](../../host-hardening/ssh/40-crypto-hardening.conf).
- [x] Set ufw default deny inbound. Keep SSH, Webmin, and the HTTPS panel on the LAN. Allow the tailnet. Drop ICMP timestamp requests. Accept AMP SFTP 2223/2224 only on `tailscale0`, and drop 8080 except on Tailscale (loopback stays accepted by stock ufw). [ufw-rules.md](../../host-hardening/ufw/ufw-rules.md), [before.rules.snippet](../../host-hardening/ufw/before.rules.snippet).
- [x] Disable TCP timestamps so the host stops advertising uptime. [99-homelab-tcp-timestamps.conf](../../host-hardening/sysctl/99-homelab-tcp-timestamps.conf).
- [x] Put the AMP panel behind the HTTPS reverse proxy on 8443 and replace the LAN allow for 8080 with an allow for 8443. [amp-https/compose.yaml](../../amp-https/compose.yaml), [amp-https/nginx/amp.conf](../../amp-https/nginx/amp.conf). The proxy is shared infrastructure; this task is the hardening control that used it.
- [x] Confirm, from a container on the scanner Docker network, that 22, 8443, and 10000 were open and that 8080, 2223, and 2224 were closed. Remove the temporary scanner ufw allow when the scan finished. [Write-up](../../docs/hardening-greenbone.md).
- [x] Document the remaining Medium as a false positive: AMP returns its login page (HTTP 200) for unknown paths, including `/.ssh/known_hosts`, and the finding followed the panel from 8080 to 8443. [Write-up, section C](../../docs/hardening-greenbone.md).
- [x] Measure outside-in exposure with public checkers, after discarding a proxied `nmap` that failed the `scanme.nmap.org` sanity check. Result: 0 of 1,007 IPv4 TCP ports open; tested IPv6 ports all timed out. [Write-up](../../docs/external-exposure-check.md).

## Next

- [ ] **Trusted certificate for the panel via `tailscale serve`.**
  - **Done when:** Serve/HTTPS is enabled on the tailnet, the panel is presented with a tailnet-trusted certificate through `tailscale serve`, and a browser on the tailnet no longer stops on the self-signed warning described in [amp-https/certs/README.md](../../amp-https/certs/README.md). The hardening write-up (or a short note linked from it) records the change. No new inbound port is opened, and no private key is committed.
  - **Why it is open:** section B of the hardening write-up leaves the certificate self-signed and names `tailscale serve` as the upgrade. The README lists the same item.

- [ ] **Greenbone override for the documented false positive.**
  - **Done when:** an override covers only "Linux Home Folder Accessible" and its text cites the section C check (HTTP 200 is the login page; `known_hosts` content is not returned). A rescan uses the same task, target, "Full and fast" config, and QoD ≥ 70 filter as the 2026-10-20 scans. The write-up records the severity totals from that report, including that the override is in place. No other finding is overridden to make the count move.
  - **Why it is open:** the published reports were filtered with no overrides, so the false positive still counts as the 1 remaining Medium.

## Implement

Checked tasks point at files that already exist. The two unchecked tasks have no config in this repo yet.
