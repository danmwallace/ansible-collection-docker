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

## [1.1.2] - 2026-05-28

### Removed

- `homepage` role — removed from the collection.

## [1.1.0] - 2026-05-27

### Added

- `it_tools` role (formerly `ansible_docker_it_tools`) — deploys [it-tools](https://github.com/CorentinTh/it-tools).

## [1.0.0] - 2026-05-01

### Added

- Initial release of `danmwallace.docker`.
- Roles: `adguard`, `arcane`, `baserow`, `common`, `docker`, `grafana`, `librechat`, `n8n`, `semaphore`, `traefik`, `unifi`.
- Molecule scenarios + `meta/argument_specs.yml` for every role.
