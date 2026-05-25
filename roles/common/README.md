# danmwallace.docker.common

Baseline system configuration for homelab hosts. Sets the system hostname from `inventory_hostname`, installs a baseline package set (apt on Debian/Ubuntu, `rpm-ostree` on Fedora Atomic, `dnf` on Fedora Server), conditionally installs `qemu-guest-agent` on KVM guests, sets the timezone, creates user accounts and grants them `sudo`/`wheel` membership plus authorized SSH keys, and drops a `/etc/sudoers.d/ansible` fragment that grants the `ansible` user passwordless sudo.

> **Note on collection placement.** This role is not Docker-specific. It lives in `danmwallace.docker` for convenience, but conceptually belongs in a baseline collection (`danmwallace.common`). The FQCN `danmwallace.docker.common` reads a bit odd; that's intentional and unavoidable for now.

## Requirements

- Ansible >= 2.16
- `community.general` collection on the controller (for `timezone` and `rpm_ostree_pkg`)
- `ansible.posix` collection on the controller (for `authorized_key`)
- Root/sudo access on the target host

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `common_packages_debian` | list[str] | no | apt-transport-https, ca-certificates, cockpit, curl, git, gnupg, lsb-release, python3, python3-venv | Package list installed via apt on Debian-family hosts. |
| `common_packages_fedora` | list[str] | no | ca-certificates, certbot, curl, git, gnupg2, python3, python3-virtualenv | Package list installed via dnf (Server) or rpm-ostree (Atomic) on Fedora hosts. |
| `common_timezone` | str | no | `America/New_York` | Timezone applied via `community.general.timezone`. |
| `common_users` | list[dict] | no | `[]` | User accounts to create. Each entry is `{ name: <login>, ssh_key: "<openssh public key>" }`. |

The `ansible` user that gets the passwordless-sudo grant is hard-coded in `tasks/main.yml`. Edit the task there if you want a different user.

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  become: true
  roles:
    - role: danmwallace.docker.common
      vars:
        common_timezone: "America/Los_Angeles"
        common_users:
          - name: dwallace
            ssh_key: "{{ vault_dwallace_ssh_key }}"
```

## What the Role Does

1. Sets the system hostname to `inventory_hostname`.
2. Installs the baseline package set per OS family.
3. On KVM virtual machines (`ansible_facts['virtualization_type'] == 'kvm'`), installs `qemu-guest-agent`.
4. On Fedora, parses `VARIANT_ID` from `/etc/os-release` to distinguish Atomic (IoT/CoreOS/Silverblue/Kinoite) from Server, and picks `rpm-ostree` or `dnf` accordingly.
5. Sets the system timezone.
6. Creates user accounts in `common_users`, adding them to `sudo` (Debian-family) or `wheel` (Fedora), and installs each user's SSH key into `authorized_keys`.
7. Drops `/etc/sudoers.d/ansible` granting `ansible ALL=(ALL) NOPASSWD:ALL` (mode 0440, validated with `visudo -cf`).

## License

MIT
