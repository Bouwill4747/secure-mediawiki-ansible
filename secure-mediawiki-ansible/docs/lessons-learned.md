# Lessons learned

A short reflection on the concepts this project drove home — not the
commands, but the shifts in thinking.

## Declarative beats imperative once the project outgrows a single file

Part 1 was a sequence of commands applied once. It works, but it is
fragile: the only record of what was done lives in bash history and in
the current state of config files. If a file is edited manually later,
the original intent is lost.

Part 2 inverts the logic. The playbook describes *what the server
should look like*, not *what commands to run*. Re-running it is safe by
design because tasks check state before acting. This means the playbook
itself is the documentation of intent. Editing
`roles/mediawiki/tasks/main.yml` is equivalent to editing the
specification of the server, not scripting a one-shot change.

## Idempotence is a design constraint, not an afterthought

Writing "an Ansible task" is easy. Writing "an Ansible task that can be
run a thousand times in a row without making any state change after the
first" is a different exercise. It forces explicit answers to:

- How do I know whether the work is already done?
- If I re-run with different inputs, should I converge or fail loudly?
- What is the authoritative source of truth — the playbook or the
  current state of the host?

The `creates:` argument on `command` tasks, the `state: present`
semantic on package modules, and `notify:` for handlers are not
conveniences; they are what makes the playbook trustworthy for
re-execution.

## Minimal installs expose hidden dependencies

Two of the stickier bugs in this project (Fail2ban's missing
`/var/log/auth.log`, Ansible's need for the `acl` package) shared the
same root cause: a production-oriented tool assuming a feature of the
base OS that Debian 12 minimal does not install by default.

The lesson is not "install more packages on minimal servers." The
lesson is "when something works on one machine and fails on another,
the first hypothesis should be a difference in the OS baseline, not a
configuration bug." The fix in both cases was a one-line package
install, but finding it required questioning whether the tool's error
message was pointing at the real cause.

## Cross-host state needs explicit modelling

The `mediawiki` role runs on `srv-web` but must know the address of
`srv-db`. The naive approach is to hardcode an IP. The robust approach
is `hostvars['srv-db'].ansible_host` — a single source of truth, the
inventory, owns that fact.

This is a small pattern in Ansible, but it scales. In larger
infrastructures, "who owns this fact" is the central design question.
Every hardcoded value is a dependency that can fall out of sync.

## Security-as-a-playbook changes the threat model

A manually hardened server is only as good as the last admin who
touched it. A playbook-applied server is as good as its playbook, and
the playbook is a reviewable artefact. Running it periodically (or
after incidents) re-asserts the desired state — drift detection
becomes a trivial side-effect.

Testing this was instructive: I manually edited a managed config file
on the target, then re-ran the playbook. Ansible detected the drift
and corrected it in a single task. That same mechanism, in a real
environment, would flag an unauthorized change within minutes.
