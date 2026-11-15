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
| `arcane_hostname` | str | yes | — | Public hostname the Arcane web UI is served on; used for `APP_URL` and the Traefik `Host` rule. |
| `arcane_encryption_key` | str | yes | — | Encryption key (`ENCRYPTION_KEY`). Supply from vault. |
| `arcane_jwt_secret` | str | yes | — | JWT signing secret (`JWT_SECRET`). Supply from vault. |
| `arcane_postgres_password` | str | yes | — | Password for the Arcane PostgreSQL user. Supply from vault. |
| `arcane_version` | str | no | `v2.10.2` | Image tag for `ghcr.io/getarcaneapp/arcane`; the same pin as `danmwallace.podman.arcane`. |
| `arcane_socket_proxy_version` | str | no | `v0.5.0` | Image tag for `ghcr.io/tecnativa/docker-socket-proxy`, which exposes a filtered Docker socket to Arcane. |
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
        arcane_jwt_secret: "{{ vault_arcane_jwt_secret }}"
        arcane_postgres_password: "{{ vault_arcane_postgres_password }}"
```

## Notes

- **No host ports.** Traefik reaches Arcane over `proxy_network`, and Arcane reaches Postgres over the internal `arcane_network`. Nothing is published on the host. Before 1.4.0 the role published 3552 and 5432 on random host ports on every interface.
- **Upgrading from 0.6.0:** 1.4.0 moves the default to Arcane `v2.10.2`, the version `danmwallace.podman.arcane` runs in production. Arcane migrates its database on start. Back up the `postgres-data` volume first, or pin `arcane_version` to stay put.
- **OIDC:** the role no longer sets `OIDC_ENABLED`. It was on without any issuer or client configured.

## License

MIT
