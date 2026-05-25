# danmwallace.docker.it_tools

Deploys [IT-Tools](https://it-tools.tech/) — a browser-based collection of developer
and sysadmin utilities (encoders/decoders, hash and crypto helpers, network
calculators, converters, generators) — as a Docker Compose stack under
`/opt/docker/it-tools`. The container (`corentinth/it-tools:latest`) joins the
external `proxy_network` and is exposed through Traefik on the configured hostname
over the `websecure` (443) entrypoint with the `cloudflare` cert resolver. The
compose file also publishes the UI directly on host port `8079` for LAN access.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An external Docker network named `proxy_network` (typically provisioned by the
  Traefik role)

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `it_tools_hostname` | str | no | `it-tools.example.com` | Public hostname Traefik routes to the IT-Tools web UI. Override per host. |

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker and an
external `proxy_network` (typically from the Traefik role).

## Example Playbook

```yaml
- hosts: utility_servers
  become: true
  roles:
    - role: danmwallace.docker.it_tools
      vars:
        it_tools_hostname: tools.example.com
```

## What the Role Does

1. Ensures `/opt/docker/it-tools/data` exists (0755).
2. Renders `docker-compose.yml` to `/opt/docker/it-tools/` (0600).
3. Stops the compose project if it is already running (`state: absent`).
4. Brings the IT-Tools stack up via `community.docker.docker_compose_v2`.

## Notes

- The role has no `handlers/main.yml`; instead of notifying a restart handler on a
  config change, every run tears the stack down (step 3) and brings it back up
  (step 4). The teardown task uses `ignore_errors: true` and is not idempotent — it
  reports a change on each run and causes a brief restart even when nothing changed.
  This deviates from the collection's notify-on-change compose pattern.
- IT-Tools processes everything client-side in the browser; no data leaves the host.
- The compose template hardcodes the `corentinth/it-tools:latest` image tag and the
  `8079:80` host port mapping — neither is currently exposed as a role variable.

## License

MIT
