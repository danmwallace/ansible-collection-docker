# danmwallace.docker.adguard

Deploys [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome), a network-wide DNS server and ad blocker, as a Docker Compose stack under `/opt/docker/adguard`. The container is joined to the external `proxy_network` and exposes Traefik labels so the web UI is served on the configured hostname; standard DNS, DHCP, DoT, DoQ, and DoH ports are published directly on the host.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An external Docker network named `proxy_network` (typically provisioned by the Traefik role)
- An `ansible` user and `docker` group on the target host — the role chowns the data directories to `ansible:docker`

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `adguard_hostname` | str | yes | — | Public hostname Traefik routes to the AdGuard Home web UI. |

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker and an external `proxy_network` (typically from a Traefik role).

## Example Playbook

```yaml
- hosts: dns_servers
  become: true
  roles:
    - role: danmwallace.docker.adguard
      vars:
        adguard_hostname: adguard.example.com
```

## Published Ports

| Port | Protocol | Purpose |
| --- | --- | --- |
| 53 | tcp + udp | Standard DNS |
| 67 | udp | DHCP server |
| 853 | tcp + udp | DNS-over-TLS |
| 5443 | tcp + udp | DNS-over-HTTPS |
| 8853 | udp | DNS-over-QUIC |

Web UI is reached via `https://{{ adguard_hostname }}` through Traefik (port 80 inside the container).

## License

MIT
