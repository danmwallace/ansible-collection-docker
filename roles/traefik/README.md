# danmwallace.docker.traefik

Deploys [Traefik](https://traefik.io/) as a Docker Compose stack under `/opt/docker/traefik`, configured with the Cloudflare DNS-01 challenge so it can issue Let's Encrypt wildcard certificates without exposing port 80 to the issuer. Renders three config files (docker-compose.yml, traefik.yml static config, dynamic_conf.yml middleware/TLS options), touches `acme.json` with 0600 permissions, creates the external `proxy_network` Docker network used by all proxied services, and brings the stack up.

The Traefik dashboard is exposed through Traefik itself (router `traefik` → service `api@internal`) on the configured hostname over the standard `websecure` (443) entrypoint, plus a plain `:8081` dashboard entrypoint for direct LAN access.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An `ansible` user and `docker` group on the target host — the role chowns all rendered files to `ansible:docker`
- A Cloudflare API token with **Zone:DNS:Edit** for the zones whose certificates Traefik will issue

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `traefik_hostname` | str | yes | — | Public hostname Traefik routes to its own dashboard (api@internal). |
| `traefik_cloudflare_api_token` | str | yes | — | Cloudflare API token with Zone:DNS edit rights, used for DNS-01. **Supply from vault.** |
| `traefik_cloudflare_email` | str | yes | — | Email Let's Encrypt associates with issued certificates and Cloudflare's `CF_API_EMAIL`. |
| `traefik_version` | str | no | `v3.6.2` | Image tag for the `traefik` image. |

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker installed (e.g. via `community.docker` or a separate Docker-install role).

## Example Playbook

```yaml
- hosts: proxy_servers
  become: true
  roles:
    - role: danmwallace.docker.traefik
      vars:
        traefik_hostname: traefik.example.com
        traefik_cloudflare_api_token: "{{ vault_traefik_cloudflare_api_token }}"
        traefik_cloudflare_email: ops@example.com
```

## What the Role Does

1. Ensures `/opt/docker/traefik` exists (owned `ansible:docker`, 0755).
2. Renders `docker-compose.yml`, `traefik.yml`, and `dynamic_conf.yml` (each 0600, owned `ansible:docker`).
3. Touches `acme.json` with 0600 perms (without overwriting an existing one).
4. Ensures the external `proxy_network` Docker network exists.
5. Brings the Traefik stack up via `community.docker.docker_compose_v2`.

A `Restart traefik compose project` handler fires when any of the three rendered files changes, so config edits take effect without manual intervention.

## Notes

- Other roles in this collection (arcane, adguard, baserow, grafana) all attach to `proxy_network` and expect this role (or an equivalent) to have provisioned it.
- The static config includes an HTTP-to-HTTPS redirection on port 80, but only 443 and 8081 are published in the compose file. If you want the redirect to work over port 80 from the public internet, uncomment the `80:80` port mapping in `templates/docker-compose.yml.j2`.

## License

MIT
