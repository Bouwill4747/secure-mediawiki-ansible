# Part 1 — Manual hardening of a Debian 12 server

This part covers the hardening of a single Debian 12 server that serves an
Apache site over HTTPS with a self-signed certificate. All steps are done
manually to contrast with Part 2's automated approach.

## Scope

1. System baseline and user hardening
2. Apache 2.4 with TLS, security headers, and `ServerTokens ProductOnly`
3. SSH hardened with ed25519 keys, no root login, no password auth
4. Stateful `nftables` firewall with `policy drop`
5. Fail2ban with systemd backend (Debian 12 has no `/var/log/auth.log`
   by default)
6. ModSecurity 3 + OWASP Core Rule Set as a Web Application Firewall

## Config files in this directory

| File                            | Purpose                                   |
|---------------------------------|-------------------------------------------|
| `configs/nftables.conf`         | Firewall ruleset (policy drop, allowlist) |
| `configs/sshd_config.hardened`  | SSH daemon config                         |
| `configs/jail.local`            | Fail2ban with `backend = systemd`         |
| `configs/mediawiki-ssl.conf`    | Apache SSL VirtualHost example            |

## Key commands used (for reference)

```bash
# Generate a self-signed certificate (CN must match server hostname)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/server.key \
  -out    /etc/ssl/certs/server.crt

# Enable required Apache modules
sudo a2enmod ssl headers rewrite

# Apply nftables rules
sudo nft -f /etc/nftables.conf
sudo systemctl enable nftables

# Verify firewall
sudo nft list ruleset

# Fail2ban with systemd backend (Debian 12 minimal has no rsyslog)
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd

# ModSecurity — switch from DetectionOnly to On
sudo sed -i 's/^SecRuleEngine DetectionOnly/SecRuleEngine On/' \
  /etc/modsecurity/modsecurity.conf
sudo systemctl restart apache2
```

## Hardening choices

**TLS with a self-signed certificate** — for a lab environment. In
production this becomes a Let's Encrypt or commercial CA certificate
without changing the rest of the configuration.

**SSH key-only authentication** — `PasswordAuthentication no` and
`PubkeyAuthentication yes`. Password brute-force is structurally
impossible once enforced.

**`PermitRootLogin no`** — defense against credential-stuffing attacks
that target the `root` account; attackers cannot escalate to admin in a
single step.

**`policy drop` on nftables** — default-deny posture. Only explicitly
allowed traffic passes. Incoming SSH/HTTP/HTTPS are allowed; everything
else is dropped silently.

**Stateful filtering** — `ct state established,related accept` allows
return traffic from legitimate outbound connections without opening
source ports explicitly.

**Fail2ban on systemd backend** — Debian 12 minimal ships without
`rsyslog`, so the default Fail2ban config referencing
`/var/log/auth.log` does not work. A `jail.local` forcing
`backend = systemd` reads from `journalctl` directly.

**ModSecurity + OWASP CRS** — a general-purpose WAF catching common
injection, XSS, scanner, and anomaly patterns before the request
reaches PHP.
