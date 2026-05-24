# danmwallace.docker.grafana

Deploys [Grafana](https://grafana.com/oss/grafana/) as a Docker Compose stack under `/opt/docker/grafana`, joined to the external `proxy_network` and exposed through Traefik on the configured hostname. The role surfaces the most common Grafana settings (admin password, server root URL, login-form rendering, plugin preinstall, CPU/memory limits) as role variables.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An external Docker network named `proxy_network` (typically provisioned by the Traefik role)

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `grafana_hostname` | str | yes | — | Public hostname Traefik routes to Grafana. |
| `grafana_admin_password` | str | yes | — | Grafana admin password (`GF_SECURITY_ADMIN_PASSWORD`). Supply from vault. |
| `grafana_server_root_url` | str | no | `https://{{ grafana_hostname }}` | Public URL Grafana uses for links and callbacks (`GF_SERVER_ROOT_URL`). |
| `grafana_disable_login_form` | str | no | `"false"` | Whether to disable the Grafana login form (`GF_SECURITY_DISABLE_GRAFANA_LOGIN_FORM_RENDERING`). String, not bool. |
| `grafana_install_plugins` | str | no | `""` | Comma-separated plugin list (`GF_PLUGINS_PREINSTALL`). |
| `grafana_cpu_limit` | str | no | `"2"` | Compose `deploy.resources.limits.cpus` value. |
| `grafana_memory_limit` | str | no | `"2gb"` | Compose `deploy.resources.limits.memory` value. |

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker and an external `proxy_network` (typically from a Traefik role).

## Example Playbook

```yaml
- hosts: grafana_hosts
  become: true
  roles:
    - role: danmwallace.docker.grafana
      vars:
        grafana_hostname: grafana.example.com
        grafana_admin_password: "{{ vault_grafana_admin_password }}"
```

## Notes

- The template uses `grafana/grafana:latest`. Pin a specific tag for reproducible deployments by editing the template (or, if useful enough, a follow-up could expose `grafana_version`).
- The role creates `/opt/docker/grafana/data/provisioning` (mounted into the container at `/etc/grafana/provisioning`); drop datasource/dashboard provisioning YAML there to have Grafana pick it up on start.

## License

MIT
