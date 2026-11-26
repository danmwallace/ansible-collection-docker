# danmwallace.docker.traefik

Deploys [Traefik](https://traefik.io/) as a Docker Compose stack under `/opt/docker/traefik`, configured with the Cloudflare DNS-01 challenge so it can issue Let's Encrypt wildcard certificates without exposing port 80 to the issuer. Renders three config files (docker-compose.yml, traefik.yml static config, dynamic_conf.yml middleware/TLS options), touches `letsencrypt/acme.json` (the ACME store, on the container's `/letsencrypt` mount) with 0600 permissions, creates the external `proxy_network` Docker network used by all proxied services, and brings the stack up.

The Traefik dashboard is exposed through Traefik itself (router `traefik` → service `api@internal`) on the configured hostname over the standard `websecure` (443) entrypoint (`traefik_dashboard_via_hostname`; no auth middleware is added), plus, by default, a plain `:8081` dashboard entrypoint published on `127.0.0.1:8081` (`traefik_dashboard_local_port`).

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
| `traefik_version` | str | no | `v3.7.19` | Image tag for the `traefik` image. |
| `traefik_https_bind_address` | str | no | `0.0.0.0` | Address 443 is published on. An empty string publishes without an address: Docker's default of every IPv4 and IPv6 interface. |
| `traefik_dashboard_local_port` | bool | no | `true` | Publish the `:8081` dashboard entrypoint on `127.0.0.1:8081`. `false` drops the port and the matching command-line flags. |
| `traefik_dashboard_via_hostname` | bool | no | `true` | Attach the router for `traefik_hostname` to the dashboard (`api@internal`). The role adds no auth or IP allowlist middleware, so set `false` wherever that hostname is reachable by anyone who should not see the dashboard. |
| `traefik_extra_env` | dict | no | `{}` | Extra container environment, `NAME: value`, rendered sorted and verbatim (do not quote values). For lego provider options such as `LEGO_DISABLE_CNAME_SUPPORT: "true"`. |
| `traefik_deploy` | bool | no | `true` | When `false`, renders the compose project under `/opt/docker/traefik/` without starting or restarting the stack (molecule uses this). |

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
3. Ensures `/opt/docker/traefik/letsencrypt/` exists and touches `letsencrypt/acme.json` (root, 0600) without overwriting an existing one. Traefik's ACME `storage` is `/letsencrypt/acme.json`, on that mount.
4. Ensures the external `proxy_network` Docker network exists.
5. Brings the Traefik stack up via `community.docker.docker_compose_v2`.

A `Restart traefik compose project` handler fires when any of the three rendered files changes, so config edits take effect without manual intervention.

## Notes

- **Upgrading from 2.0.0 or earlier:** the ACME `storage` was the relative path `acme.json`, which Traefik resolves inside the container's own filesystem, not on the `./letsencrypt` mount. Certificates were therefore lost, and re-requested from Let's Encrypt, every time the container was recreated; the `/opt/docker/traefik/acme.json` the role touched was never mounted. From 2.0.1 the store is `/opt/docker/traefik/letsencrypt/acme.json`. The first start after upgrading requests certificates once more; an existing `letsencrypt/acme.json` is kept. The old top-level `acme.json` is unused and can be deleted.

- Other roles in this collection (arcane, adguard, baserow, grafana) all attach to `proxy_network` and expect this role (or an equivalent) to have provisioned it.
- The static config includes an HTTP-to-HTTPS redirection on port 80, but only 443 and 8081 are published in the compose file. If you want the redirect to work over port 80 from the public internet, uncomment the `80:80` port mapping in `templates/docker-compose.yml.j2`.

## License

MIT
