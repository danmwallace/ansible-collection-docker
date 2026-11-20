# danmwallace.docker.baserow

Deploys [Baserow](https://baserow.io/), an open-source no-code database (Airtable alternative), as a Docker Compose stack under `/opt/docker/baserow`. Uses the all-in-one `baserow/baserow` image, which bundles PostgreSQL and Redis. The container is joined to the external `proxy_network` and emits Traefik labels so the web UI is served on the configured hostname.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An external Docker network named `proxy_network` (typically provisioned by the Traefik role)

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `baserow_hostname` | str | yes | — | Public hostname Traefik routes to Baserow; also fed to the container as `BASEROW_PUBLIC_URL`. |
| `baserow_version` | str | no | `"2.4.3"` | Image tag for `baserow/baserow`. |
| `baserow_deploy` | bool | no | `true` | When `false`, renders the compose project under `/opt/docker/baserow/` without starting or restarting the stack (molecule uses this). |

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker and an external `proxy_network` (typically from a Traefik role).

## Example Playbook

```yaml
- hosts: docker_hosts
  become: true
  roles:
    - role: danmwallace.docker.baserow
      vars:
        baserow_hostname: baserow.example.com
        baserow_version: "2.4.3"
```

## Notes

- Baserow runs its Django migrations on container start and does not support downgrades. Back up the `baserow_data` volume before raising `baserow_version`.
- From 2.4.0, AI provider environment variables (`BASEROW_OPENAI_API_KEY` and friends) are imported into the database once on upgrade and ignored afterwards. The role sets none of them; if you add them through an override, manage providers in the admin UI after upgrading.

## License

MIT
