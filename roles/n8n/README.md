# danmwallace.docker.n8n

Deploys [n8n](https://n8n.io/), a self-hosted workflow automation platform, as a Docker Compose stack under `/opt/docker/n8n`. Brings up two containers: `n8n` (the app, image `n8nio/n8n:latest`) and `n8n-postgres` (the backing store, image `postgres:15`). Joins the external `proxy_network` Docker network and emits Traefik labels so the n8n web UI is served on the configured hostname.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An external `proxy_network` Docker network (typically provisioned by the `traefik` role)
- An `ansible` user and `docker` group on the target host (the role chowns rendered files to `ansible:docker`)

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `n8n_hostname` | str | yes | — | Public hostname Traefik routes to the n8n web UI; also used for `N8N_HOST` and `WEBHOOK_URL`. |
| `n8n_encryption_key` | str | yes | — | n8n credentials encryption key (`N8N_ENCRYPTION_KEY`). **Supply from vault.** |
| `n8n_postgres_password` | str | yes | — | Password for the n8n PostgreSQL user. **Supply from vault.** |

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker and the external `proxy_network`.

## Example Playbook

```yaml
- hosts: automation_hosts
  become: true
  roles:
    - role: danmwallace.docker.n8n
      vars:
        n8n_hostname: n8n.example.com
        n8n_encryption_key: "{{ vault_n8n_encryption_key }}"
        n8n_postgres_password: "{{ vault_n8n_postgres_password }}"
```

## Notes

- The postgres backing store binds `127.0.0.1:5432` on the host for direct DB access. Drop the `ports:` block on `n8n-postgres` in `templates/docker-compose.yml.j2` if you don't want that.
- The role creates the bind-mount source directories `/opt/docker/n8n/volumes/postgres-db` and `/opt/docker/n8n/volumes/n8n` so the named volumes can mount them.

## License

MIT
