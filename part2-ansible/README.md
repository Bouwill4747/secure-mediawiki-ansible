# Part 2 — Ansible deployment of MediaWiki

Declarative, idempotent deployment of MediaWiki across two Debian 12 hosts:
a web/application tier (Apache + PHP) and a database tier (MariaDB).

## Project layout

```
part2-ansible/
├── site.yml                    Top-level playbook (orchestrates all roles)
├── inventory.example.ini       Inventory template (sanitized)
├── group_vars/
│   └── all.example.yml         Global variables template (sanitized)
└── roles/
    ├── common/                 Shared setup (packages, acl, base tools)
    ├── apache_php/             Apache 2.4 + PHP + MediaWiki vhost
    ├── mariadb/                MariaDB 10.11 + wikidb + wikiuser
    └── mediawiki/              Download, extract, composer, install.php
```

## Design choices

**Roles over monolithic playbooks.** Each role encapsulates one concern
(OS baseline, web tier, DB tier, application install). `site.yml` stays
readable at a glance.

**Idempotence via native module semantics.** Every task uses declarative
modules (`apt`, `service`, `file`, `template`, `lineinfile`) or includes
explicit guards (`creates:`) on imperative modules like `command`. Running
the playbook twice yields `changed=0` on the second run.

**Handlers for service reloads.** The Apache `VirtualHost` deployment
uses `notify: Reload Apache` rather than an unconditional restart. If the
template produces no diff, the handler does not fire.

**Cross-host variable resolution.** The `mediawiki` role runs on the web
host but needs the database IP. Rather than hardcoding it, the install
command uses `hostvars['srv-db'].ansible_host` to pull the value from the
inventory.

**Principle of least privilege on the DB.** `wikiuser` is granted access
only to `wikidb.*`, not global privileges.

## Prerequisites on target hosts

- Debian 12 minimal installation
- An SSH-reachable unprivileged user with `NOPASSWD` sudo
- Network connectivity from `ansible-master` on port 22
- Network connectivity between `srv-web` and `srv-db` on port 3306

## Usage

```bash
# From the Ansible control node
cd part2-ansible
cp inventory.example.ini inventory.ini
cp group_vars/all.example.yml group_vars/all.yml
# Edit both files to match your environment

# Install compatible collections for Ansible 2.14 (Debian 12 default)
ansible-galaxy collection install 'community.general:<8.0.0'
ansible-galaxy collection install 'community.mysql:<4.0.0'

# Syntax check + connectivity
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible mediawiki -i inventory.ini -m ping

# Dry run (recommended first time)
ansible-playbook -i inventory.ini site.yml --check

# Real run
ansible-playbook -i inventory.ini site.yml
```

After a successful run, MediaWiki is available at
`http://<srv-web-ip>/mediawiki`.

## Testing idempotence

Re-run the playbook immediately after a successful deployment. The
`PLAY RECAP` should show `changed=0` on both hosts — this is the
canonical demonstration that the playbook converges to a desired state
rather than re-applying work.
