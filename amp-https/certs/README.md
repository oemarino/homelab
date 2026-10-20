Put `amp.crt` and `amp.key` here (root-owned, key mode 600). They are NOT included in this repo.
A self-signed pair works for LAN use, e.g.:

    openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -days 825 \
      -subj "/CN=homelab-server" -keyout amp.key -out amp.crt
