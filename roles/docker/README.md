# danmwallace.docker.docker

Installs Docker CE and the Compose plugin on Debian, Ubuntu, and Fedora (Server and Atomic variants). On Debian-family hosts the role registers Docker's apt signing key under `/etc/apt/keyrings`, adds the Docker apt repository for the host's distro/release/arch, removes the conflicting distro-provided `docker.io`/`docker-compose`/`podman-docker` packages, then installs `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, and `docker-compose-plugin`. On Fedora it drops Docker's `docker-ce.repo` under `/etc/yum.repos.d`, detects whether the host is Atomic (IoT/CoreOS/Silverblue/Kinoite) by parsing `VARIANT_ID` from `/etc/os-release`, and installs via `rpm-ostree` on Atomic or `dnf` on Server. Finally it ensures a `docker` group exists and adds the `ansible` user to it.

## Requirements

- Ansible >= 2.16
- `community.general` collection on the controller (for `rpm_ostree_pkg` on Fedora Atomic hosts only)
- An `ansible` user on the target host — the role appends it to the `docker` group. If the user doesn't exist, the final task will create it with default settings.

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `docker_fedora_packages` | list[str] | no | `[docker-ce, docker-compose-plugin, containerd.io]` | Packages installed via `dnf` on Fedora Server. Tweak to add/remove components (e.g. `docker-buildx-plugin`). |

The Debian/Ubuntu package list is hard-coded in the tasks (mirrors Docker's upstream install guide); change it there if you need a different set.

## Dependencies

None.

## Example Playbook

```yaml
- hosts: docker_hosts
  become: true
  roles:
    - role: danmwallace.docker.docker
```

Run this once on each host before applying the application roles in this collection (`arcane`, `adguard`, `baserow`, `grafana`, `traefik`), all of which assume Docker and the `docker-compose-plugin` are already installed.

## What the Role Does

### Debian / Ubuntu
1. Ensures `/etc/apt/keyrings` exists.
2. Fetches Docker's apt signing key to `/etc/apt/keyrings/docker.asc`.
3. Registers `https://download.docker.com/linux/<distro>` as an apt repository with the correct architecture and release, signed by the key from step 2.
4. Removes any pre-existing `docker.io`, `docker-compose`, `docker-compose-v2`, `docker-doc`, `podman-docker` packages that would conflict.
5. Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`.

### Fedora
1. Parses `VARIANT_ID` from `/etc/os-release` to distinguish Atomic from Server.
2. Drops `docker-ce.repo` into `/etc/yum.repos.d/` from Docker's CDN.
3. On Atomic variants, layers `docker-ce` via `rpm-ostree`.
4. On Server, installs `docker_fedora_packages` via `dnf`.

### All hosts
1. Ensures a `docker` group exists.
2. Adds the `ansible` user to the `docker` group (creating the user if absent).

## Notes

- The role does NOT start or enable the `docker` daemon — Docker's packages do this themselves via their packaged systemd units, which take effect on the next boot or via a separate `systemctl enable --now docker` if you need it immediately.
- The role uses `dpkg` architecture inference via a small map of `ansible_facts['architecture']` → apt-arch (`x86_64`→`amd64`, `aarch64`→`arm64`, `armv7l`→`armhf`). Other architectures pass through unchanged.

## License

MIT
