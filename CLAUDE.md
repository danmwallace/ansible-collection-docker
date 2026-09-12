# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

`danmwallace.docker` is an Ansible Collection (FQCN namespace `danmwallace`, name `docker`). Each role under `roles/` deploys one Docker Compose stack on a target host. Roles are designed to share a single `traefik` reverse proxy for routing and TLS, so they can be composed together on the same host without conflict.

This checkout lives at `~/Code/Infrastructure/dev-collections/ansible_collections/danmwallace/docker/`. The homelab playbooks that consume it live in the control repo at `~/Code/Infrastructure/ansible-homelab-cfg/` (see `~/Code/Infrastructure/CLAUDE.md` for the workspace layout). The legacy standalone `ansible_docker_*` roles were migrated **into** this collection during the Phase 3 restructure; the `ansible-homelab-roles/` repo they came from **no longer exists**. The `/migrate-role` slash command in `.claude/commands/` is kept for the pattern only.

## Common Commands

The shared lint/test toolchain lives in the control repo's venv (there is no venv inside this collection):

```
~/Code/Infrastructure/ansible-homelab-cfg/.venv/bin/
```

Either put that directory on `PATH`, or prefix commands with `uv run --project ~/Code/Infrastructure/ansible-homelab-cfg`.

> **Warning — do not set `ANSIBLE_COLLECTIONS_PATH` to the dev-collections tree.**
> ansible-lint and molecule both run an ansible-compat "prerun" that builds this
> collection and installs it into the **first** entry of the collections path.
> When that entry is `~/Code/Infrastructure/dev-collections`, the install target
> is this very checkout, and `ansible-galaxy collection install --force` deletes
> the source tree before unpacking the tarball over it. This wiped a sibling
> collection's checkout on 2026-09-08. Run the tools with no collections-path
> override at all; nothing below needs one.

Run from the collection root unless noted:

```bash
# Lint a single role. --offline stops ansible-lint from trying to fetch
# dependencies from Galaxy; no env override is needed.
ansible-lint --offline roles/<role>

# Lint everything
ansible-lint --offline

# Build the collection tarball
ansible-galaxy collection build
```

Molecule — never run it inside this checkout with a collections-path override (see the warning above). Two safe options:

1. Run in place with **no** `ANSIBLE_COLLECTIONS_PATH` set. Must `cd` into the role directory first:

   ```bash
   cd roles/<role> && molecule test            # full sequence
   cd roles/<role> && molecule converge        # just apply, leave instance up for debugging
   cd roles/<role> && molecule verify          # re-run verify.yml against running instance
   cd roles/<role> && molecule destroy         # tear down
   ```

2. Copy the tree to a scratch directory that mirrors the Galaxy layout and run there, so the prerun can never touch the source:

   ```bash
   rsync -a --exclude .git ./ /tmp/scratch/ansible_collections/danmwallace/docker/
   cd /tmp/scratch/ansible_collections/danmwallace/docker/roles/<role> && molecule test
   ```

The Molecule scenarios use Podman as the driver via the **ansible-native delegated** pattern (`create.yml`/`destroy.yml` in each scenario manage the container directly via `containers.podman.podman_container`). There is no `driver:` block in `molecule.yml` — do not add one, and do not regress to the deprecated `driver: name: docker` shape.

## Architecture and Conventions

### The compose-role pattern

Every service role in this collection follows the same shape; deviating from it costs more than it gains:

1. `tasks/main.yml` creates `/opt/docker/<service>/` (and any subdirs) owned `ansible:docker`, mode `0755`.
2. Renders `templates/docker-compose.yml.j2` to `/opt/docker/<service>/docker-compose.yml`, mode `0600`, owned `ansible:docker`, with `notify: Restart <service> compose project`.
3. Renders any additional config files (e.g. traefik's `traefik.yml`, `dynamic_conf.yml`) with the same mode/owner and the same `notify:`.
4. Calls `community.docker.docker_compose_v2` with `project_src: /opt/docker/<service>/`, `state: present`.
5. `handlers/main.yml` defines `Restart <service> compose project` — a `docker_compose_v2` call with `state: restarted` so config edits roll out without manual intervention.

When adding a new service role, copy this skeleton from `roles/n8n/` (simple) or `roles/traefik/` (multi-file config). Do not introduce `docker_container` calls, shell `docker compose` invocations, or systemd unit templates.

### Networking model

`traefik` creates an external Docker network named `proxy_network`. Every other proxied service attaches its compose stack to that network and exposes itself **only** via Traefik labels — no host port publishes for app containers. If a new role needs proxying, it depends on `traefik` having run first; add `proxy_network: external: true` in its compose template and the appropriate `traefik.http.routers.*` labels.

### Role metadata is mandatory

Every role must carry both `meta/main.yml` (galaxy_info + dependencies) and `meta/argument_specs.yml` (typed argument validation). The `ansible-role-meta` skill owns these files — invoke it when adding/removing role variables, changing platforms, or fixing `schema[meta]` lint errors. Hard rules enforced by that skill:

- `min_ansible_version` must be quoted (`"2.16"`, not `2.16`).
- `role_name` matches the directory name, underscores only.
- Cross-role references in `dependencies:` and in tasks use FQCN: `danmwallace.docker.<role>`, never the bare name.
- Every variable in `defaults/main.yml` must have a corresponding entry in `argument_specs.yml`. Every variable referenced in tasks without a default must be `required: true`.
- Variables are prefixed with the role name (`traefik_hostname`, `n8n_domain`) to avoid collisions when multiple roles run in the same play.

### Multi-OS detection

The `common` and `docker` roles support Debian/Ubuntu and Fedora (including Atomic variants: IoT, CoreOS, Silverblue, Kinoite). The Atomic detection pattern is to shell out to `awk` on `/etc/os-release` for `VARIANT_ID` (since `ansible_facts` doesn't expose it cleanly), register the result, then branch on `<role>_variant_id.stdout in ['iot', 'coreos', 'silverblue', 'kinoite']`. Use `community.general.rpm_ostree_pkg` for Atomic, `ansible.builtin.dnf` for Server, `ansible.builtin.apt`/`package` for Debian family. Service roles (everything other than `common` and `docker`) declare only Debian/Ubuntu platforms — they have not been validated on Fedora.

### Molecule scenario shape

Each role's `molecule/default/` directory contains exactly: `molecule.yml`, `create.yml`, `destroy.yml`, `converge.yml`, `verify.yml`, `inventory/hosts.yml`. The instance is a privileged systemd-enabled Ubuntu noble container (`quay.io/ansible/molecule-ubuntu:noble`) with cgroup v2 host namespace. `verify.yml` must make **role-specific** assertions (rendered file content, container state, expected labels) — never `assert: that: true` placeholders. See `roles/traefik/molecule/default/verify.yml` for the reference template that slurps rendered files and asserts on their content.

## Repository-Specific Workflows

- **Migrating a legacy `ansible_docker_*` role into the collection**: run `/migrate-role <new_role_name> <path_to_old_role_repo>`. The skill copies files, rewrites modules to FQCN, generates `argument_specs.yml`, modernizes the Molecule scenario, runs ansible-lint, and produces a commit message. It does NOT run `git add`/`git commit` — the user reviews and commits manually. The `ansible-homelab-roles/` repo it was written against is gone; point it at any standalone role checkout.
- **Commit-message convention**: imperative subject line referencing the role, e.g. `Add unifi role migrated from ansible_docker_unifi`. Recent commits follow this exactly; match the style.
- **`.gitignore`** excludes `.ansible/` and `.claude/`. Anything you put under `.claude/` (skills, commands, settings) stays local to the working copy.
