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
| `arcane_version` | str | no | `latest` | Image tag for `ghcr.io/getarcaneapp/arcane`. |
| `arcane_postgres_db` | str | no | `arcane` | PostgreSQL database name. |
| `arcane_postgres_user` | str | no | `arcane` | PostgreSQL user name. |
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

## License

MIT
