# Error journal

Real errors and warnings encountered during the project, with the
diagnosis and the fix that resolved each one. Documented as-is — these
are what actually happened, not what was anticipated up front.

---

## Part 1 — Hardening

### `ssh-copy-id` not recognized under Windows

**Symptom.** PowerShell returns
`ssh-copy-id : The term is not recognized as the name of a cmdlet`.

**Cause.** The OpenSSH Client shipped with Windows 10/11 does not include
`ssh-copy-id`; that utility is Unix-specific.

**Fix.** Use a PowerShell one-liner that pipes the public key over SSH and
appends it to `authorized_keys` on the server:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh admin@SERVER_IP `
    "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

### Fail2ban refuses to start on Debian 12 minimal

**Symptom.** `systemctl start fail2ban` fails with an error referencing
`/var/log/auth.log: No such file or directory`.

**Cause.** Debian 12 minimal does not install `rsyslog`. SSH
authentication events land in the systemd journal (`journalctl`) rather
than in the traditional `/var/log/auth.log` file that the default jail
configuration expects.

**Fix.** Create `/etc/fail2ban/jail.local` forcing the systemd backend
and restart the service. See
[`part1-hardening/configs/jail.local`](../part1-hardening/configs/jail.local).

---

### Firewall test returns "Connection refused" instead of timing out

**Symptom.** Running `nc -vz 127.0.0.1 25` on the server itself returns
an immediate RST (connection refused) rather than being silently dropped
by the firewall.

**Cause.** The nftables ruleset includes `iif "lo" accept`, which is
deliberate — many local services rely on loopback traffic. Testing from
the host itself does not exercise the firewall at all.

**Fix.** Run the test from an external machine (e.g. the client PC)
using `Test-NetConnection <ip> -Port 25` on PowerShell. A silent timeout
confirms the `policy drop` is effective on incoming external traffic.

---

## Part 2 — Ansible

### Ansible cannot parse the inventory

**Symptom.**

```
[WARNING]: Unable to parse /home/deploy/inventory.ini as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
```

**Cause.** The file did not exist in the current working directory when
the command was issued.

**Fix.** Create the file with the correct structure in the project
directory, and run `ansible` from that directory (or pass the full path
with `-i`).

---

### "Missing sudo password"

**Symptom.**

```
srv-web | FAILED! => { "msg": "Missing sudo password" }
srv-db  | FAILED! => { "msg": "Missing sudo password" }
```

**Cause.** The playbook uses `become: yes` to run tasks as root via
`sudo`. Without `NOPASSWD` in `/etc/sudoers`, Ansible cannot supply the
password non-interactively.

**Fix.** Add a line to the target hosts' sudoers:

```
deploy ALL=(ALL) NOPASSWD:ALL
```

Alternatively, run Ansible with `--ask-become-pass` for interactive use.

---

### Inventory groups not matched by the playbook

**Symptom.**

```
[WARNING]: Could not match supplied host pattern, ignoring: mediawiki
[WARNING]: Could not match supplied host pattern, ignoring: db
[WARNING]: Could not match supplied host pattern, ignoring: web
```

**Cause.** Group names in `inventory.ini` did not match the ones
referenced in `site.yml`. The inventory used `[webservers]` /
`[dbservers]`; the playbook expected `[web]` / `[db]` /
`[mediawiki:children]`.

**Fix.** Align both files. A child group
(`[mediawiki:children]` containing `web` and `db`) lets the first play
target both hosts with one pattern.

---

### `community.general` collection incompatible with Ansible 2.14

**Symptom.**

```
[WARNING]: Collection community.general does not support Ansible version 2.14.18
```

**Cause.** Ansible collections 8.x+ (community.general) and 4.x+
(community.mysql) dropped support for Ansible 2.14, which is the
version shipped by Debian 12's APT.

**Fix.** Pin compatible versions:

```bash
ansible-galaxy collection install 'community.general:<8.0.0' --force
ansible-galaxy collection install 'community.mysql:<4.0.0'  --force
```

This is the most important fix to document for anyone reproducing the
project from APT-installed Ansible.

---

### `chmod: invalid mode 'A+user:www-data:rx:allow'` (missing ACL support)

**Symptom.**

```
Failed to set permissions on the temporary files Ansible needs to
create when becoming an unprivileged user
(rc: 1, err: chmod: invalid mode: 'A+user:www-data:rx:allow')
```

**Cause.** The `mediawiki` role uses `become_user: www-data` to run
`composer install` and `install.php` under the Apache identity. For
sharing temp files between two non-root users, Ansible relies on POSIX
ACLs (`setfacl`). The `acl` package is not installed on Debian 12
minimal by default.

**Fix.** Add `acl` to the package list of the `common` role. This keeps
the playbook fully self-contained and reproducible without manual
pre-steps.

```yaml
- name: Install base packages
  apt:
    name:
      - ...existing packages...
      - acl
    state: present
```

This is the error I learned the most from. It revealed an unspoken
dependency between Ansible's privilege-escalation model and Linux
kernel-level features. The documentation mentions ACLs briefly; the
practical impact on minimal installs is where theory meets reality.

---

### Warning: `/var/www/.ansible Permission denied`

**Symptom.**

```
[WARNING]: Unable to use /var/www/.ansible/tmp as temporary directory,
falling back to system
```

**Cause.** Ansible tries to create its temp directory under the
`become_user`'s home (`/var/www/` for `www-data`), which is not writable.

**Fix.** None required — Ansible falls back to `/tmp` automatically.
Informational only.

---

### Warning: `column_case_sensitive is not provided`

**Symptom.** Emitted by `community.mysql.mysql_user` when creating the
wiki user.

**Cause.** Deprecation notice announcing a behavior change in the
community.mysql 4.0 release.

**Fix.** None required for this project. Worth reviewing if the
playbook is upgraded to a future major version of the collection.
