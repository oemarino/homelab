# ufw rule summary (homelab server)

Default policy: **deny incoming**, allow outgoing.

| Port / interface | From | Purpose |
|---|---|---|
| 22/tcp | 192.168.1.0/24 (home LAN) | SSH |
| 10000/tcp | 192.168.1.0/24 | Webmin |
| 8443/tcp | 192.168.1.0/24 | AMP panel over HTTPS (TLS reverse proxy) |
| all traffic on `tailscale0` | tailnet devices | Private remote access (WireGuard) |
| 41641/udp | anywhere | Tailscale direct (peer-to-peer) connections |
| 25566/tcp | anywhere | Minecraft server (not port-forwarded on the router; reached through a relay) |

Plus the `before.rules` drops in [`before.rules.snippet`](before.rules.snippet):
ICMP timestamp requests, AMP SFTP 2223/2224 except on Tailscale, and plain-HTTP 8080
except on loopback/Tailscale.

The website needs **no** rule at all: nginx listens on 127.0.0.1 only and Cloudflare
Tunnel connects outbound.

> Docker-published ports bypass ufw. That's why every container in this repo binds
> to an explicit address (127.0.0.1, the LAN IP or the Tailscale IP), never 0.0.0.0.
