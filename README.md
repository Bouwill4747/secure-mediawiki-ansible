# Secure MediaWiki Deployment with Ansible

Two-part infrastructure automation project completed as part of a server
administration course. The goal was to contrast **manual hardening** of a
Linux server with **declarative automation** of a multi-server application
deployment.

**Part 1** hardens a Debian 12 server: Apache2 with TLS, SSH key-based
authentication, a stateful nftables firewall, Fail2ban intrusion prevention,
and ModSecurity as a Web Application Firewall.

**Part 2** uses Ansible to deploy a 2-tier MediaWiki application across two
Debian servers: one running Apache + PHP, one running MariaDB. The playbook
is idempotent and organized into reusable roles.

## Architecture

![Architecture diagram](architecture.png)

Three Debian 12 virtual machines on the `192.168.x.0/24` subnet:

- `ansible-master`: Ansible control node
- `srv-web`: Apache + PHP + MediaWiki
- `srv-db`: MariaDB hosting `wikidb`

See [ARCHITECTURE.md](ARCHITECTURE.md) for full details on network flows,
ports, and trust boundaries.

## Skills demonstrated

- Linux server hardening on Debian 12 minimal
- Apache 2.4 TLS configuration (self-signed certificate, security headers)
- SSH hardening: ed25519 keys, disabled root login, disabled password auth
- Stateful firewall (nftables) with `policy drop` and selective allowlist
- Intrusion prevention with Fail2ban on systemd backend
- Web Application Firewall: ModSecurity 3 + OWASP Core Rule Set
- Infrastructure as Code with Ansible (roles, handlers, templates)
- Jinja2 templating for host-specific configuration
- Playbook idempotence and drift detection
- Cross-host variable resolution (`hostvars`)

## Repository structure

```
.
├── ARCHITECTURE.md            Network + trust boundaries
├── architecture.png           Diagram
├── part1-hardening/           Manual hardening guide + config files
├── part2-ansible/             Ansible project (playbook, roles, inventory)
└── docs/
    ├── error-journal.md       Real errors encountered + resolutions
    └── lessons-learned.md     Reflection on what this project taught me
```

## Quickstart (Part 2 - Ansible deployment)

```bash
# On the Ansible control node
cd part2-ansible

# Copy the example files and fill in real values
cp inventory.example.ini inventory.ini
cp group_vars/all.example.yml group_vars/all.yml

# Install required collections (versions pinned for Ansible 2.14)
ansible-galaxy collection install 'community.general:<8.0.0'
ansible-galaxy collection install 'community.mysql:<4.0.0'

# Verify connectivity
ansible mediawiki -i inventory.ini -m ping

# Run the playbook
ansible-playbook -i inventory.ini site.yml
```

The target hosts are expected to be reachable via SSH key authentication,
with the Ansible user configured for `NOPASSWD` sudo.

## Security considerations

**Secrets** — The real `inventory.ini` and `group_vars/all.yml` are not
tracked (see [.gitignore](.gitignore)). Only sanitized `.example.` versions
are committed. In a production context these would be encrypted with
Ansible Vault rather than kept as plaintext files.

**Self-signed certificates** — Part 1 uses a self-signed TLS certificate
for demonstration; in production, Let's Encrypt or a commercial CA would
be used.

**No HTTPS on the MediaWiki vhost** — Part 2 deploys MediaWiki on port 80
only, to keep the focus on automation concepts. Adding a TLS vhost is a
natural extension that would reuse the patterns from Part 1.

## Documentation artefacts

The full project report (in French) including the full walkthrough,
architecture reasoning, and screenshots from the live deployment is
available on request.

## License

This project is published for educational and portfolio purposes.
