# Security notes

**Everything in this repository is a sanitized example, not a live copy.**

- Secrets (passwords, API keys, tunnel credentials, TLS keys/certificates, the
  Wazuh enrollment password) are **not** included. They are replaced by
  `${VARIABLES}`, `<PLACEHOLDERS>` or `.env.example` files.
- Public IP addresses, Tailscale addresses, tunnel IDs and individual device IPs
  are removed or generalized (for example the LAN is shown as `192.168.1.0/24`).
- The attack lab (`attack-lab/`) runs **deliberately vulnerable** applications
  (DVWA, OWASP Juice Shop). Run it only on an isolated network, keep it off when
  not in use, and never expose it to the internet.
- These configs fit my hardware and network. Review them before reusing them.

Found something sensitive I missed, or a security problem in these configs?
Please email **contact@osvaldomarino.dev** instead of opening a public issue.
