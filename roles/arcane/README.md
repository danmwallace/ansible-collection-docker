# danmwallace.docker.arcane

Deploys [Arcane](https://getarcane.app/), a modern web UI for managing Docker containers, alongside a bundled PostgreSQL backing store. The role renders a `docker-compose.yml` under `/opt/docker/arcane` and brings the stack up with `community.docker.docker_compose_v2`. Traefik labels are emitted on the `proxy_network` external network so a Traefik instance on the same host can route `https://<arcane_hostname>` to the container.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or Podman compatible with `docker_compose_v2`) installed
- An external Docker network named `proxy_network` on the target host (typically provisioned by the Traefik role)

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `arcane_hostname` | str | yes | — | Public hostname the Arcane web UI is served on; used for `ARCANE_APP_URL` and the Traefik `Host` rule. |
| `arcane_encryption_key` | str | yes | — | Encryption key (`ARCANE_ENCRYPTION_KEY`). At least 32 characters, or a `hex:`/`base64:` encoded 32-byte key. Supply from vault, and keep the value 2.x used. |
| `arcane_jwt_secret` | str | no | — | **Deprecated, ignored.** Arcane 3.x signs sessions with its own key. Still accepted so existing inventories pass. |
| `arcane_postgres_password` | str | yes | — | Password for the Arcane PostgreSQL user. Supply from vault. |
| `arcane_version` | str | no | `v3.1.1` | Image tag for `ghcr.io/getarcaneapp/manager`. Must be `v3.0.0` or newer. |
| `arcane_socket_proxy_version` | str | no | `3.4@sha256:027f9fa5ee70be5836eeaa10ce889d7a83f34a8db16a7f412818b92997e0beab` | What follows `ghcr.io/tecnativa/docker-socket-proxy:`, a tag or `tag@sha256:<digest>`. The proxy exposes a filtered Docker socket to Arcane. Upstream stopped versioned tags at `v0.5.0` and rebuilds `3.4` (HAProxy 3.4) weekly, so the default pins `3.4` to its multi-arch index digest (amd64, arm64, arm/v7 and others). |
| `arcane_postgres_version` | str | no | `17.11-alpine` | Image tag for `postgres`, pinned to a patch release. |
| `arcane_postgres_db` | str | no | `arcane` | PostgreSQL database name. |
| `arcane_postgres_user` | str | no | `arcane` | PostgreSQL user name. |
| `arcane_deploy` | bool | no | `true` | When `false`, renders `/opt/docker/arcane/docker-compose.yml` without starting or restarting the stack (molecule uses this). |
| `arcane_database_url` | str | no | `postgresql://{{ arcane_postgres_user }}:{{ arcane_postgres_password }}@arcane-postgres:5432/{{ arcane_postgres_db }}` | Full Postgres connection URL Arcane uses. |

## Dependencies

None declared in `meta/main.yml`. In practice you will want Docker and Traefik already provisioned on the host.

## Example Playbook

```yaml
- hosts: docker_servers
  become: true
  roles:
    - role: danmwallace.docker.arcane
      vars:
        arcane_hostname: arcane.example.com
        arcane_encryption_key: "{{ vault_arcane_encryption_key }}"
        arcane_postgres_password: "{{ vault_arcane_postgres_password }}"
```

## Upgrading from Arcane 2.x

The role renders a 3.x-only compose file: the image is `ghcr.io/getarcaneapp/manager` (3.x removed the GHCR `arcane` alias) and every Arcane variable carries the `ARCANE_` prefix. 3.x refuses to start if an unprefixed 2.x name is set, or if the database was not first migrated by 2.17.x. Upstream guide: <https://getarcane.app/docs/get-started/migrate-v3>.

1. **Run v2.17.3 once.** With collection 1.4.x (the 2.x template), set `arcane_version: v2.17.3`, deploy, and wait until Arcane is up. Older databases stop 3.x with `upgrade to Arcane v2.17.1 first`.
2. **Resolve job runs marked "Needs attention".** 3.x removes "Resolve after review".
3. **Back up.** 3.x replaces its SQL migrations with one-way Goose migrations, so a restore is the only way back to 2.x.

   ```sh
   docker exec arcane-postgres pg_dump -U arcane arcane > arcane-$(date +%F).sql
   docker run --rm -v arcane_arcane-data:/data -v "$PWD":/backup alpine tar czf /backup/arcane-data-$(date +%F).tgz -C /data .
   cp /opt/docker/arcane/docker-compose.yml docker-compose.yml.2x
   ```

4. **Check `arcane_encryption_key`.** Keep the value 2.x used (it decrypts stored credentials). If it is shorter than 32 characters, 3.x refuses to start and the role fails first with an assert. Changing the key means Arcane can no longer decrypt what 2.x stored with the old one, so plan to re-enter stored registry, Git and OIDC credentials, and do not upgrade until you have decided.
5. **Deploy this role version.** `arcane_jwt_secret` can stay in the inventory; the role ignores it.
6. **Afterwards:** set up notification providers again under System → Notifications (they are now global to the Manager), review custom roles (permissions were renamed; the built-in Deployer role loses `containers:update`), and update any scripts that call `/api/environments` (now `/api/agents`) or six-field cron schedules (now five-field, converted automatically).

## Notes

- **No host ports.** Traefik reaches Arcane over `proxy_network`, and Arcane reaches Postgres over the internal `arcane_network`. Nothing is published on the host. Before 1.4.0 the role published 3552 and 5432 on random host ports on every interface.
- **Upgrading from 0.6.0:** 1.4.0 moves the default to Arcane `v2.10.2`, the version `danmwallace.podman.arcane` runs in production. Arcane migrates its database on start. Back up the `postgres-data` volume first, or pin `arcane_version` to stay put.
- **OIDC:** the role no longer sets `OIDC_ENABLED`. It was on without any issuer or client configured.

## License

MIT
