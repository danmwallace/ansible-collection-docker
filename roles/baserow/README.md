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
| `baserow_version` | str | no | `'2.06'` | Image tag for `baserow/baserow`. |

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
        baserow_version: '2.2.2'
```

## License

MIT
