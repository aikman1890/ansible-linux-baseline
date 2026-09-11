# common role

Base Linux configuration applied to every host before hardening: packages,
admin users with managed SSH keys, MOTD banner, chrony NTP, and pruning of
unneeded services.

## What it does

1. Installs a base package set (`vim`, `htop`, `chrony`, `rsyslog`, …) via
   dnf on RHEL / apt on Ubuntu.
2. Creates admin users from `common_admin_users`, adds them to the sudo group
   (`wheel` on RHEL, `sudo` on Ubuntu), and deploys their SSH keys with
   `exclusive: true` so orphaned keys are removed.
3. Drops a passwordless-sudo file for the admin group (validated with
   `visudo` before install).
4. Renders `/etc/motd` from `templates/motd.j2`.
5. Configures chrony with the servers in `common_ntp_servers` and ensures
   `chronyd`/`chrony` and `rsyslog` are running.
6. Stops and disables unneeded services (avahi, cups, rpcbind, …). Missing
   services are skipped, not failed.

## Variables

| Variable | Default | Description |
|---|---|---|
| `common_base_packages` | vim, htop, chrony, rsyslog, curl, tar, unzip | Packages on every host |
| `common_disabled_services` | avahi-daemon, cups, rpcbind, … | Services to stop/disable |
| `common_admin_users` | `[]` | List of `{name, comment, groups, shell, ssh_keys}` |
| `common_sudo_group` | `wheel` / `sudo` | Sudo group, auto-selected by OS family |
| `common_motd_banner` | authorized-use banner | MOTD text |
| `common_ntp_servers` | pool.ntp.org ×4 | Chrony servers |

## Example

```yaml
common_admin_users:
  - name: sstjohn
    comment: "Shaun St. John"
    groups: ["wheel"]
    ssh_keys:
      - "ssh-ed25519 AAAA... sstjohn@laptop"
```

Store real keys in `group_vars/all/vault.yml` (ansible-vault), not in the
inventory.
