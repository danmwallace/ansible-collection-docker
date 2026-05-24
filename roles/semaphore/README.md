# danmwallace.docker.semaphore

Deploys [Semaphore UI](https://www.semaphoreui.com/), a web frontend for Ansible (run playbooks, manage inventories, schedule tasks), as a Docker Compose stack under `/opt/docker/semaphore`. Brings up two containers: `semaphore` (the app, image `semaphoreui/semaphore`) and `semaphore_db` (the backing PostgreSQL store). Joins the external `proxy_network` Docker network and emits Traefik labels so the UI is served on the configured hostname.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An external `proxy_network` Docker network (typically provisioned by the `traefik` role)

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `semaphore_hostname` | str | yes | — | Public hostname Traefik routes to the Semaphore web UI. |
| `semaphore_postgres_password` | str | yes | — | Password for the Semaphore PostgreSQL user. **Supply from vault.** |
| `semaphore_admin_password` | str | yes | — | Initial Semaphore admin password. **Supply from vault.** |
| `semaphore_version` | str | no | `v2.16.47` | Image tag for `semaphoreui/semaphore`. |
| `semaphore_postgres_user` | str | no | `semaphore` | PostgreSQL user used by Semaphore. |
| `semaphore_postgres_db` | str | no | `semaphore` | PostgreSQL database used by Semaphore. |
| `semaphore_admin` | str | no | `admin` | Semaphore admin login name. |
| `semaphore_admin_name` | str | no | `Admin` | Display name for the Semaphore admin account. |
| `semaphore_admin_email` | str | no | `admin@localhost` | Email address for the Semaphore admin account. |
| `semaphore_ansible_host_key_checking` | str | no | `"false"` | Passed through to the container's `ANSIBLE_HOST_KEY_CHECKING`. String, not bool. |

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker and the external `proxy_network`.

## Example Playbook

```yaml
- hosts: automation_hosts
  become: true
  roles:
    - role: danmwallace.docker.semaphore
      vars:
        semaphore_hostname: semaphore.example.com
        semaphore_postgres_password: "{{ vault_semaphore_postgres_password }}"
        semaphore_admin_password: "{{ vault_semaphore_admin_password }}"
        semaphore_admin_email: ops@example.com
```

## License

MIT
