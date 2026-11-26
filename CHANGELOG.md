# Changelog

All notable changes to this collection will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this collection adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [2.0.2] - 2026-11-26

### Fixed

- `arcane`: a version bump or config change no longer restarts the stack twice (danmwallace/ansible-collection-docker#77). The `Restart arcane compose project` handler ran `docker compose restart` right after `docker compose up` had already recreated the changed services. That restarted Postgres under the new Arcane, which then served about 2.5 minutes of 502s while its stale actor-host registration expired. The handler is gone. The role now brings up `arcane-postgres` and `socket-proxy`, waits for `pg_isready` over TCP, then runs `up` for the project, so each changed container is recreated once, after the database is ready. The rendered `docker-compose.yml` is unchanged. The readiness check uses `community.docker.docker_container_exec`, which needs Python `requests` on the host (`traefik` already needs it).

### Added

- `arcane`: an `upgrade` molecule scenario that runs the stack under Docker. It bumps the Arcane version, re-runs unchanged, then changes all three services at once. Each round asserts the number of `start` events per container (1 for each changed container, 0 otherwise), that Arcane started after its dependencies, and that `/api/health` reports `UP` within 60 seconds.

## [2.0.1] - 2026-11-26

### Fixed

- `traefik`: the ACME `storage` was the relative path `acme.json`, which resolves inside the container rather than on the `./letsencrypt` mount, so certificates were lost and re-requested from Let's Encrypt every time the container was recreated. It is now `/letsencrypt/acme.json`, and the role touches `/opt/docker/traefik/letsencrypt/acme.json` (root, 0600) instead of the unmounted `/opt/docker/traefik/acme.json`. Molecule asserts the path.

### Added

- `traefik`: `traefik_extra_env` (dict, default `{}`) for extra container environment, such as lego's `LEGO_DISABLE_CNAME_SUPPORT`.
- `traefik`: `traefik_dashboard_local_port` (default `true`). `false` drops the `127.0.0.1:8081` dashboard port and its command-line flags.
- `traefik`: `traefik_dashboard_via_hostname` (default `true`). `false` leaves the `traefik_hostname` router off `api@internal`; the role adds no auth or allowlist middleware to the dashboard.
- `traefik`: an empty `traefik_https_bind_address` publishes 443 without an address (Docker's default: every IPv4 and IPv6 interface). The default stays `0.0.0.0`.

## [2.0.0] - 2026-11-26

### Changed (breaking)

- `arcane`: move to Arcane 3.x, default `v3.1.1` (#64). 2.x installs must run `v2.17.3` once before upgrading and back up first; see the role README. The image is now `ghcr.io/getarcaneapp/manager` and the Arcane variables carry the `ARCANE_` prefix. The role refuses an `arcane_version` older than `v3.0.0`, and an `arcane_encryption_key` shorter than 32 characters (3.x refuses it too). `arcane_jwt_secret` is optional and ignored, and the `POSTGRES_*` variables are no longer passed to the Arcane container.

### Added

- `adguard`, `baserow`, `grafana`, `librechat`, `n8n`, `semaphore`, `traefik`, `unifi`: `<role>_deploy` (default `true`), as `arcane_deploy` in 1.4.0. When `false`, the role renders its files without creating the Docker network or starting or restarting the stack.

### Security

- `traefik`: the default image moves from `v3.6.2` to `v3.7.19`, picking up the v3.7.14–v3.7.18 fixes for the SNI check bypass and the unauthenticated Compress-middleware memory DoS (#46). This matches `danmwallace.podman.traefik`.

### Changed

- Image bumps to the current upstream release of each pinned line. Back up first where noted; each app migrates its data on start and downgrades are unsupported.
  - `adguard` `v0.107.78` → `v0.107.80` (#52). Upstream's latest is v1.0.0, a new major, not taken here.
  - `baserow` `2.3.2` → `2.4.3` (#50). Back up `baserow_data`.
  - `grafana` `13.1.1` → `13.3.1` (#43). Back up `grafana_data`.
  - `librechat` API `v0.8.7` → `v0.8.9` (#55), now from `ghcr.io/librechat-ai/librechat-api` because upstream moved organisations and `ghcr.io/danny-avila/librechat-api` stops at `v0.8.8-rc4`; the checkout clones `LibreChat-AI/LibreChat`. Meilisearch `v1.53.2` → `v1.57.0` (#51), pgvector `0.8.6-pg16-bookworm` → `0.8.7-pg16-bookworm` (#56). Matches `danmwallace.podman`. A blank `librechat_mongo_auto_index` / `librechat_mongo_auto_create` now means enabled; see the role README.
  - `n8n` `2.32.7` → `2.48.6` (#44). Back up Postgres. From 2.42.6, workflows can no longer add or edit the deprecated Function, Function Item and LangChain Code nodes (GHSA-9p7v); from 2.45.0 the durable scheduler is the default.
  - `semaphore` `v2.19.0` → `v2.20.23` (#45): no published 2.19 image has the GHSA-ffp6-w57q-758q / GHSA-q9cq-8j9q-r93m fixes. `pg_dump` first.
  - `unifi` `10.4.57` → `10.6.109` (#48), MongoDB `8.0.11` → `8.0.38` (#49, CVE-2025-14847 "MongoBleed", same 8.0 line).
  - `n8n` and `semaphore`: Postgres `17-alpine` (floating) → `17.12-alpine` (#59).
- `adguard`, `grafana`, `unifi`: drop the stale `| default(...)` image-tag fallbacks in the compose templates and declare `adguard_version`, `grafana_version` and `unifi_version` in `argument_specs.yml`.
- `traefik`: replace the ACME `dnsChallenge.delayBeforeCheck`, deprecated since Traefik v3.3, with `dnsChallenge.propagation.delayBeforeChecks`. The value is the same (`0`).
- `arcane`: socket-proxy moves from `v0.5.0` to the `3.4` tag pinned by its multi-arch index digest (`arcane_socket_proxy_version: "3.4@sha256:027f9fa5…"`) (#47). Upstream publishes no versioned tags after `v0.5.0`; `3.4` is the same proxy config on a newer HAProxy 3.4 base, with the same environment variables and port.

### Fixed

- Molecule: the `default` scenarios of `adguard`, `baserow`, `common`, `docker`, `grafana`, `librechat`, `n8n`, `semaphore`, `traefik` and `unifi` never passed their inventory, so converge and verify ran against no hosts and passed without testing anything (#63).
  - They now pass the inventory, use `docker.io/jrei/systemd-ubuntu:noble` (`quay.io/ansible/molecule-ubuntu:noble` is gone), use `su` for become, and render with `<role>_deploy: false`.
  - Image assertions that had never run were stale (`adguard`, `baserow`, `grafana`, `semaphore`, `unifi` mongo) and now match the pinned defaults.
  - `common` sets the container hostname and installs `sudo`, as on a stock Ubuntu Server. `librechat` connects as `ansible` with passwordless sudo, the owner the role expects, installs `git`, and stubs every `vault_librechat_*` name `env.j2` references.
- `traefik` molecule: `verify.yml` compares the image tag with `traefik_version` instead of a hard-coded `v3.6.2`.
- `n8n`: the compose template fell back to `n8nio/n8n:1.73.1` when `n8n_version` was unset, bypassing the pin in `defaults/main.yml` (#60). The image now comes from `n8n_version` only. `n8n_version` is now in `argument_specs.yml` and the README.

## [1.4.0] - 2026-11-15

### Security

- `arcane`: publish no host ports. Postgres 5432 and the Arcane UI 3552 were published, with Docker short syntax, on random host ports on every interface, bypassing Traefik (#54, #61). Traefik and Arcane reach them over their compose networks.
- `arcane`: pin Postgres to `17.11-alpine` (new `arcane_postgres_version`) instead of the floating `17-alpine` (#58).

### Changed

- `arcane`: the default image moves from `0.6.0` to `v2.10.2`, and socket-proxy from `0.3.0` to `v0.5.0`, matching `danmwallace.podman.arcane` (#53). Arcane migrates its database on start: back up the `postgres-data` volume, or pin `arcane_version`.
- `arcane`: drop `OIDC_ENABLED`/`OIDC_SCOPES`, which were on with no issuer or client configured. Fix `APP_URL`, whose quotes were part of the value in compose list syntax.

### Added

- `arcane`: `arcane_deploy` (default `true`). When `false`, the role renders the compose file without starting or restarting the stack.

### Fixed

- `arcane` molecule: the scenario never passed its inventory, so converge and verify ran against no hosts and passed without testing anything.
  - It now passes the inventory, uses `docker.io/jrei/systemd-ubuntu:noble` (`quay.io/ansible/molecule-ubuntu:noble` is gone), uses `su` for become, and renders with `arcane_deploy: false`.
  - Its old image assertion (`arcane:latest`) had never actually run.

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
