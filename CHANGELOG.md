# Changelog

All notable changes to this collection will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this collection adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.3.1] - 2026-09-12

### Fixed

- `it_tools` — default `it_tools_version` pointed at `2024.10.22-1`, a tag that does not exist on Docker Hub; now `2024.10.22-7ca5933` (upstream's latest release) and exposed in `argument_specs.yml`. ([#28](https://github.com/danmwallace/ansible-collection-docker/issues/28))
- `librechat` — image pins brought current: API `v0.8.7`, Meilisearch `v1.53.2`, pgvector `0.8.6-pg16-bookworm` (same Postgres major), RAG API `v0.9.0`; `librechat_version` now matches the API tag. `argument_specs.yml` defaults re-synced with `defaults/main.yml`. ([#38](https://github.com/danmwallace/ansible-collection-docker/issues/38))
- `common` — the `ansible ALL=(ALL) NOPASSWD:ALL` sudoers fragment is now gated by `common_ansible_nopasswd_sudo` (default `true`, backward-compatible); when `false` the fragment is removed. ([#39](https://github.com/danmwallace/ansible-collection-docker/issues/39))

### Added

- `librechat_meilisearch_upgrade_db` (default `true`) — sets `MEILI_UPGRADE_DB=true` so Meilisearch migrates the existing index in place across the v1.49 to v1.53 bump.

## [1.3.0] - 2026-09-03

### Security

- `traefik` — dashboard port 8081 bound to `127.0.0.1` only ([#2](https://github.com/danmwallace/ansible-collection-docker/issues/2)).
- `n8n` — app port 5678 bound to `127.0.0.1`; Traefik handles external access ([#3](https://github.com/danmwallace/ansible-collection-docker/issues/3)).
- `unifi` — admin ports 8443/8080 bound to `unifi_bind_address` (new, default `127.0.0.1`) ([#4](https://github.com/danmwallace/ansible-collection-docker/issues/4)); the same pattern extended to all optional ports ([#11](https://github.com/danmwallace/ansible-collection-docker/issues/11)).
- `adguard` — DNS/DHCP ports 53/67 bound to `adguard_bind_address` (new, default `ansible_host`) ([#5](https://github.com/danmwallace/ansible-collection-docker/issues/5)); DoT/DoH/DoQ ports bound the same way ([#13](https://github.com/danmwallace/ansible-collection-docker/issues/13)).
- `librechat` — port 3080 bound to `127.0.0.1` ([#8](https://github.com/danmwallace/ansible-collection-docker/issues/8)); `librechat_uid`/`librechat_gid` default `0` -> `3000` so the container no longer runs as root ([#34](https://github.com/danmwallace/ansible-collection-docker/issues/34)).
- `it_tools` — new `it_tools_bind_address` (default `127.0.0.1`); port 8079 is no longer published on all interfaces ([#18](https://github.com/danmwallace/ansible-collection-docker/issues/18)).
- `arcane` — Docker socket access goes through a Tecnativa `docker-socket-proxy` sidecar (`arcane_socket_proxy_version`, default `0.3.0`) to restrict the Docker API surface ([#10](https://github.com/danmwallace/ansible-collection-docker/issues/10)); image pinned `latest` -> `0.6.0` ([#27](https://github.com/danmwallace/ansible-collection-docker/issues/27)).
- `n8n`, `arcane`, `semaphore` — preflight assertions on secret length: `n8n_encryption_key` (>=16), `n8n_postgres_password` (>=12), `arcane_postgres_password` (>=12), `arcane_encryption_key` (>=16), `arcane_jwt_secret` (>=16), `semaphore_postgres_password` and `semaphore_admin_password` (>=12) ([#30](https://github.com/danmwallace/ansible-collection-docker/issues/30)).
- `:latest` image tags pinned to specific versions across all roles — new `adguard_version`, `grafana_version`, `it_tools_version`, `unifi_version` variables; `librechat_api_image` and `librechat_ragapi_image` pinned ([#6](https://github.com/danmwallace/ansible-collection-docker/issues/6)). `it_tools` pinned to `2024.10.22-1` ([#9](https://github.com/danmwallace/ansible-collection-docker/issues/9)) — that tag turned out not to exist; corrected in 1.3.1.

### Changed

- Image version bumps: `adguard` `v0.107.63` -> `v0.107.78` ([#25](https://github.com/danmwallace/ansible-collection-docker/issues/25)); `baserow` `2.2.2` -> `2.3.2` ([#26](https://github.com/danmwallace/ansible-collection-docker/issues/26)); `grafana` `11.4.0` -> `13.1.1` ([#23](https://github.com/danmwallace/ansible-collection-docker/issues/23), [#33](https://github.com/danmwallace/ansible-collection-docker/issues/33)); `librechat` API `v0.7.9` -> `v0.8.5` ([#17](https://github.com/danmwallace/ansible-collection-docker/issues/17)), Meilisearch `v1.12.3` -> `v1.49.0` ([#22](https://github.com/danmwallace/ansible-collection-docker/issues/22)), pgvector `pg15-trixie` -> `pg16-bookworm` ([#16](https://github.com/danmwallace/ansible-collection-docker/issues/16)), RAG API `v0.4.0` -> `v0.4.6` ([#20](https://github.com/danmwallace/ansible-collection-docker/issues/20)), `librechat_version` `main` -> `v0.8.5` ([#24](https://github.com/danmwallace/ansible-collection-docker/issues/24)); `n8n` `1.73.1` -> `2.32.7` ([#21](https://github.com/danmwallace/ansible-collection-docker/issues/21), [#32](https://github.com/danmwallace/ansible-collection-docker/issues/32)) and its Postgres `postgres:15` -> `postgres:17-alpine` ([#19](https://github.com/danmwallace/ansible-collection-docker/issues/19)); `semaphore` `v2.16.47` -> `v2.19.0` ([#29](https://github.com/danmwallace/ansible-collection-docker/issues/29)) and its Postgres `postgres:15` -> `postgres:17-alpine` ([#19](https://github.com/danmwallace/ansible-collection-docker/issues/19)); `unifi` `9.0.114` -> `10.4.57` ([#31](https://github.com/danmwallace/ansible-collection-docker/issues/31)) and MongoDB `7.0.26` -> `8.0.11` (EOL upgrade, [#36](https://github.com/danmwallace/ansible-collection-docker/issues/36)).
- `traefik` — `insecure-transport` serversTransport renamed `unifi-insecure-transport` to scope it to the UniFi backend ([#12](https://github.com/danmwallace/ansible-collection-docker/issues/12)).

### Added

- `traefik_https_bind_address` (default `0.0.0.0`) — explicit all-interfaces binding for the public reverse proxy ([#35](https://github.com/danmwallace/ansible-collection-docker/issues/35)).

## [1.2.0] - 2026-06-13

Fedora Server support.

### Added

- `docker` — `docker-ce-cli`, `docker-buildx-plugin`, and `container-selinux` added to `docker_fedora_packages`; `docker.service` is enabled and started after install on Fedora Server (Atomic variants skip this via the existing guard).
- `docker`, `common` — Fedora 42 Molecule scenarios (`molecule/fedora/`, image `docker.io/jrei/systemd-fedora:42`).
- `it_tools` — Molecule scenario and a `Restart it_tools compose project` handler.
- `adguard`, `arcane`, `baserow`, `grafana`, `it_tools`, `librechat`, `n8n`, `semaphore`, `traefik`, `unifi` — `Fedora` platform declared in `meta/main.yml` so Galaxy reflects actual Fedora Server support.
- CI: release-tag audit job (galaxy-tag-sync) confirming every released `galaxy.yml` version has a matching annotated `vX.Y.Z` tag ([PR #1](https://github.com/danmwallace/ansible-collection-docker/pull/1)).

### Changed

- `adguard`, `arcane`, `grafana`, `librechat`, `n8n`, `traefik`, `unifi` — `:z` SELinux relabel added to every host-path bind mount so containers start under SELinux enforcing on Fedora (no-op on Debian/Ubuntu; named volumes unaffected).
- `it_tools` — tasks rewritten to the compose-role pattern (directory -> template -> deploy); deprecated `version` key dropped from the compose template; legacy `tests/` directory removed.
- `docker` — default Molecule scenario image switched from `quay.io/ansible/molecule-ubuntu:noble` (no longer accessible) to `docker.io/jrei/systemd-ubuntu:24.04`.
- CI: `checkout`, `setup-python`, and `upload-artifact` actions bumped off Node 20 ([PR #1](https://github.com/danmwallace/ansible-collection-docker/pull/1)).

## [1.1.2] - 2026-05-28

### Removed

- `homepage` role — removed from the collection.

## [1.1.1] - 2026-05-28

### Fixed

- `librechat`: converge is now idempotent. The compose file renders to the untracked
  `docker-compose.ansible.yml` instead of upstream's git-tracked `docker-compose.yml`, which
  the `git` module reverted on every run and re-triggered a stack restart. The
  ownership-migration `find` also prunes `.git`, whose files the `git` module rewrites with
  the connecting user's primary group on every fetch.

## [1.1.0] - 2026-05-27

### Added

- `it_tools` role (formerly `ansible_docker_it_tools`) — deploys [it-tools](https://github.com/CorentinTh/it-tools).

## [1.0.0] - 2026-05-01

### Added

- Initial release of `danmwallace.docker`.
- Roles: `adguard`, `arcane`, `baserow`, `common`, `docker`, `grafana`, `librechat`, `n8n`, `semaphore`, `traefik`, `unifi`.
- Molecule scenarios + `meta/argument_specs.yml` for every role.
