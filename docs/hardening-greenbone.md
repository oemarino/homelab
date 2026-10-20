# Homelab server hardening: Greenbone before/after

**System:** homelab server (Ubuntu 24.04 LTS, on the home LAN 192.168.1.0/24), running the AMP game-server panel, OpenSSH, Webmin and Docker.
**Scanner:** Greenbone Community Edition (OpenVAS) in Docker on the same host. Scan config "Full and fast", port list "All TCP and Nmap top 100 UDP". Same task and target for both runs. Reports filtered at QoD ≥ 70 with no overrides.
**Dates:** baseline scan 2026-10-20 06:37 UTC, rescan 2026-10-20 23:48 UTC.

## Results

| Severity | Before | After |
|---|---|---|
| Critical | 0 | 0 |
| High | 0 | 0 |
| Medium | 6 | 1 (false positive, explained below) |
| Low | 3 | 0 |
| Log (info) | 43 | 51 |
| CVEs referenced | 0 | 0 |

**Medium and Low findings went from 9 to 1, and that one is a documented false positive. There are 0 real findings left.**
Log entries went up because a new HTTPS endpoint (port 8443) produces informational TLS and HTTP detections, such as the cipher list and a self-signed certificate notice. None of them carry a severity.

## Findings and fixes

| # | Finding (Greenbone) | Sev. | Port | Fix | Verified |
|---|---|---|---|---|---|
| 1 | Weak Key Exchange: `diffie-hellman-group-exchange-sha1` | Medium 5.3 | 2223 | AMP's built-in SFTP server can't be configured to drop it, so the port is firewalled to Tailscale only (see A) | Port closed from LAN/Docker; not in rescan |
| 2 | Same as #1 (second AMP instance) | Medium 5.3 | 2224 | Same | Same |
| 3 | Weak Encryption: CBC, 3DES and Twofish ciphers | Medium 4.3 | 2223 | Same | Same |
| 4 | Same as #3 (second AMP instance) | Medium 4.3 | 2224 | Same | Same |
| 5 | Cleartext Transmission of Sensitive Information via HTTP (panel login form) | Medium 4.8 | 8080 | Panel moved behind an HTTPS reverse proxy on 8443; plain HTTP on 8080 now only reachable from localhost or the encrypted Tailscale VPN (see B) | 8080 closed from LAN; no cleartext finding in rescan |
| 6 | "Linux Home Folder Accessible (HTTP)" | Medium 5.0 | 8080, now 8443 | **False positive, accepted** (see C) | Manually confirmed |
| 7 | Weak MAC: `umac-64@openssh.com`, `umac-64-etm@openssh.com` | Low 2.6 | 22 | Explicit strong `MACs` / `KexAlgorithms` / `Ciphers` list in an sshd drop-in (see D) | `ssh -m umac-64@openssh.com` gets "no matching MAC found"; not in rescan |
| 8 | TCP Timestamps Information Disclosure (leaks uptime) | Low 2.6 | general/tcp | `net.ipv4.tcp_timestamps = 0` (see E) | Not in rescan |
| 9 | ICMP Timestamp Reply Information Disclosure | Low 2.1 | general/icmp | Firewall drops ICMP type 13 (timestamp request) (see F) | Not in rescan |

### A. AMP SFTP (2223/2224): exposure reduction, not a crypto fix
AMP ships its own SFTP server. Its config (`FileManagerPlugin.kvp`) only sets enable, port, bind address and idle timeout. There is no setting for ciphers or key exchange. Rebinding or disabling it would have meant restarting the Minecraft instance, which was out of scope. Files are normally managed through the panel's web file manager instead.
So I added two rules to `/etc/ufw/before.rules`: ACCEPT 2223 and 2224 on `tailscale0`, DROP them on every other interface. They come before any user-defined ufw rule, so even a broad "allow" added later can't reopen the ports.
**Residual risk:** the weak algorithms still exist, but only devices that are authenticated on the WireGuard-encrypted tailnet can reach them. The real fix depends on the AMP vendor.

### B. AMP panel over HTTPS
- I added a hardened nginx container (`/opt/amp-https`) as a TLS reverse proxy on port 8443. It runs non-root with a read-only filesystem, all capabilities dropped and memory/CPU/PID limits. It allows TLS 1.2/1.3 with ECDHE plus AES-GCM or ChaCha20 ciphers only, sends HSTS, redirects plain HTTP to HTTPS and supports WebSocket upgrades.
- The firewall drops 8080 on all interfaces except loopback (used by the proxy) and `tailscale0`, where traffic is already WireGuard-encrypted.
- The ufw rule "8080 from LAN" was replaced with "8443 from LAN".
- The certificate is currently self-signed, so browsers warn once. The planned upgrade is `tailscale serve` with a trusted tailnet certificate, which needs Serve/HTTPS enabled in the tailnet admin console.

### C. False positive: "Linux Home Folder Accessible"
Greenbone requested `/.ssh/known_hosts` (and `/api/...`, `/Api/...`, `/API/...`) and got HTTP 200. AMP is a single-page app that returns its login page for every unknown path, so a 200 doesn't mean a file was served. No `known_hosts` content is ever returned. The finding moved from 8080 to 8443 along with the panel, which confirms it comes from AMP's routing and not from the filesystem. Accepted and documented. A Greenbone override can suppress it in future scans.

### D. OpenSSH algorithms
I added `/etc/ssh/sshd_config.d/40-crypto-hardening.conf` with:
- MACs `hmac-sha2-256-etm, hmac-sha2-512-etm, umac-128-etm, hmac-sha2-256, hmac-sha2-512`
- KEX `sntrup761x25519 (post-quantum hybrid), curve25519, DH group16/18, DH-GEX-SHA256`
- Ciphers `chacha20-poly1305, AES-GCM, AES-CTR`

I validated with `sshd -t` and applied with a reload instead of a restart, so existing sessions stayed up. I then confirmed a new login worked before closing the old session. `sshd -T` shows the new lists.

### E. TCP timestamps: disabled
Timestamps let an attacker estimate uptime, and with it how far behind on reboots and kernel patches the host is. Turning them off removes the PAWS and RTT measurement extensions. That matters on very high-bandwidth or long-latency links, not on a home gigabit LAN. Setting: `/etc/sysctl.d/99-homelab-tcp-timestamps.conf`.

### F. ICMP timestamps: dropped
I added `-A ufw-before-input -p icmp --icmp-type timestamp-request -j DROP` to `before.rules`. Echo (ping) still works.

## Method and safety
- I backed up every changed file first (`*.bak-20261020`) and changed one control at a time. After each change I checked that the SSH lifeline worked and that the panel still loaded.
- Before rescanning I checked reachability from inside a container on the scanner's Docker network: 22, 8443 and 10000 open; 8080, 2223 and 2224 closed.
- The scanner needs a temporary ufw allow for its Docker subnet. I removed it automatically when the scan finished.

## Evidence
- Baseline: `greenbone-server-scan-20261020-0637.{pdf,csv,txt,xml}`
- Rescan: `greenbone-server-rescan-20261020-2348.{pdf,csv,txt,xml}`

## Next steps
- Turn on tailnet HTTPS and switch the panel to `tailscale serve` to get a trusted certificate.
- Review the router's port-forwarding table so the firewall isn't the only layer.
- Add a Greenbone override for the false positive.
