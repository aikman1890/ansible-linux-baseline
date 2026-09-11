# Ansible Linux Baseline

Fleet-wide baseline for RHEL 8/9 and Ubuntu 22.04: common configuration
(packages, admin users, NTP, MOTD) followed by CIS-style hardening (sshd,
password policy, auditd, firewall, sysctls, AIDE). Idempotent, FQCN'd, and
built to be safe to re-run on a schedule — this is the same shape I used at
NTT Data and IBM to bring heterogeneous fleets under one baseline and cut
new-host provisioning from a day of snowflake builds to a single playbook
run.

## Layout

```
ansible.cfg            # inventory, roles_path, pipelining, no retry files
inventory/hosts.yml    # example groups: web, db, bastion
site.yml               # applies common, then hardening
roles/common/          # packages, admin users + SSH keys, MOTD, chrony
roles/hardening/       # sshd, pwquality, auditd, firewalld/ufw, sysctl, AIDE
```

## How to run

```bash
# Install the one external collection the hardening role needs
ansible-galaxy collection install ansible.posix

# Syntax-check everything
ansible-playbook -i inventory/hosts.yml site.yml --syntax-check

# Dry run against one host first
ansible-playbook -i inventory/hosts.yml site.yml --limit web01.example.com --check --diff

# Full run
ansible-playbook -i inventory/hosts.yml site.yml

# Only one role
ansible-playbook -i inventory/hosts.yml site.yml --tags hardening
```

## Variable overrides

All tunables live in each role's `defaults/main.yml`. Override per group or
host in `group_vars/` / `host_vars/`, or ad-hoc:

```bash
# Extra admin user with a key (better: ansible-vault in group_vars/all/vault.yml)
ansible-playbook -i inventory/hosts.yml site.yml -e \
  '{"common_admin_users":[{"name":"sstjohn","groups":["wheel"],"ssh_keys":["ssh-ed25519 AAAA... sstjohn@laptop"]}]}'

# Lock SSH down to named users on the db group (group_vars/db.yml)
hardening_sshd_allow_users: ["deploy", "dba"]
hardening_firewall_allowed_tcp_ports: [22]

# Build the AIDE baseline on first run (maintenance window — it's slow)
ansible-playbook -i inventory/hosts.yml site.yml --tags hardening -e hardening_aide_init=true
```

## Idempotency notes

- Every task uses a named module (`package`, `template`, `lineinfile`,
  `sysctl`, …) and is safe to re-run; re-runs report `ok` with no changes.
- `sshd_config` is validated with `sshd -t` before it's installed, so a bad
  template can never lock you out at the config-file level.
- SSH keys are deployed with `exclusive: true`: keys removed from your
  variables are removed from the hosts.
- The ufw tasks shell out because the `ufw` module can't express
  reset + default-policy idempotently; the reset runs each time on Debian
  hosts but converges to the same rule set.

## Tested on

- RHEL 8 / 9 (dnf, firewalld, chronyd)
- Ubuntu 22.04 (apt, ufw, chrony)

Ansible 2.14+ on the control node. `become` defaults to sudo-as-root in
`ansible.cfg`; hosts need Python 3 and an `ansible_user` with sudo.

## Safety

Read the hardening role README before running against production: disabling
password authentication while your key is missing means a locked host. Run
`common` first, verify key login works, keep a console session open.
