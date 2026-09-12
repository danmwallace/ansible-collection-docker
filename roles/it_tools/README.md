# danmwallace.docker.it_tools

Deploys [IT-Tools](https://it-tools.tech/) — a browser-based collection of developer
and sysadmin utilities (encoders/decoders, hash and crypto helpers, network
calculators, converters, generators) — as a Docker Compose stack under
`/opt/docker/it-tools`. The container (`corentinth/it-tools:{{ it_tools_version }}`)
joins the external `proxy_network` and is exposed through Traefik on the configured
hostname over the `websecure` (443) entrypoint with the `cloudflare` cert resolver.
The compose file also publishes the UI directly on host port `8079`, bound to
`it_tools_bind_address` (loopback by default) so the unproxied port is not reachable
from the LAN unless you opt in.

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
| `it_tools_bind_address` | str | no | `127.0.0.1` | IP address the direct host port 8079 is bound to. Set to `0.0.0.0` only if Traefik-bypass LAN access is intentionally required. |
| `it_tools_version` | str | no | `2024.10.22-7ca5933` | Tag of the `corentinth/it-tools` image to deploy. Upstream tags are `<date>-<short sha>`; see the [Docker Hub tag list](https://hub.docker.com/r/corentinth/it-tools/tags). |

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

1. Ensures `/opt/docker/it-tools` exists (0755).
2. Renders `docker-compose.yml` to `/opt/docker/it-tools/` (0600).
3. Brings the IT-Tools stack up via `community.docker.docker_compose_v2`
   (`state: present`).

A `Restart it-tools compose project` handler fires when the rendered compose file
changes, so a hostname, bind-address, or image-tag change rolls out without manual
intervention.

## Notes

- IT-Tools processes everything client-side in the browser; no data leaves the host.
- Upstream does not publish a `-1` style tag: `2024.10.22-1` (the pre-1.3.1 default)
  never existed on Docker Hub, which made a fresh deploy fail at image pull. The
  default now tracks the real `2024.10.22-7ca5933` tag, which is also upstream's
  latest release as of 1.3.1.

## License

MIT
