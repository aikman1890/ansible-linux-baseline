# hardening role

CIS-style hardening for RHEL 8/9 and Ubuntu 22.04: sshd, password policy,
auditd, host firewall, kernel sysctls, password aging, and AIDE file-integrity
monitoring.

## What it does

1. **sshd** — renders a hardened `/etc/ssh/sshd_config` (template validated
   with `sshd -t` before install): no root login, no password or
   challenge-response auth (keys only), Protocol 2, 5-minute idle timeout
   (`ClientAliveInterval 300` × 2), `MaxAuthTries 3`, no X11/TCP forwarding,
   login banner, `LogLevel VERBOSE`.
2. **Password policy** — installs `libpam-pwquality` and writes
   `/etc/security/pwquality.conf`: min length 14, 3 character classes, at
   least one digit/upper/lower/symbol, no long repeats or sequences.
3. **Password aging** — `PASS_MAX_DAYS 90`, `PASS_MIN_DAYS 7`,
   `PASS_WARN_AGE 14` in `/etc/login.defs`.
4. **auditd** — rules watching privileged command execution (setuid/setgid),
   identity files (`/etc/passwd`, `/etc/shadow`, sudoers), and sshd config
   changes. Rules live in `/etc/audit/rules.d/hardening.rules`.
5. **Firewall** — firewalld on RHEL / ufw on Ubuntu, default-deny inbound,
   only 22/80/443 TCP open (override per group, e.g. db hosts get only 22 —
   see `inventory/hosts.yml`).
6. **sysctl** — `/etc/sysctl.d/99-hardening.conf`: IP forwarding off, source
   routing and redirects off, martian logging on, SYN cookies on.
7. **AIDE** — installs AIDE; set `hardening_aide_init: true` for the first
   run to build the baseline database (slow — use a maintenance window).

## Variables

| Variable | Default | Description |
|---|---|---|
| `hardening_sshd_permit_root_login` | `no` | Root login via SSH |
| `hardening_sshd_password_authentication` | `no` | Password auth (keys only) |
| `hardening_sshd_client_alive_interval` | `300` | Idle timeout seconds |
| `hardening_sshd_allow_users` | `[]` | Restrict SSH to these users |
| `hardening_pw_minlen` / `hardening_pw_minclass` | `14` / `3` | pwquality policy |
| `hardening_pass_max_days` | `90` | Password max age |
| `hardening_firewall_allowed_tcp_ports` | `[22, 80, 443]` | Open TCP ports |
| `hardening_auditd_rules` | see defaults | auditd rule list |
| `hardening_sysctl` | see defaults | Kernel hardening map |
| `hardening_aide_init` | `false` | Build AIDE baseline DB |

## Requirements

The `ansible.posix` collection (for `firewalld` and `sysctl` modules):

```bash
ansible-galaxy collection install ansible.posix
```

## Caveats

- Disabling password auth **will lock you out** if your key isn't deployed.
  Run the `common` role first (it deploys keys) and keep a console session
  open while testing.
- ufw tasks use `command` because the `ufw` module can't express
  reset/default-policy idempotently in one pass; the reset task runs only on
  Debian hosts and re-applying the playbook converges to the same state.
