# danmwallace.docker.unifi

Deploys the [UniFi Network Application](https://github.com/linuxserver/docker-unifi-network-application) (LinuxServer image) and a MongoDB backing store as a Docker Compose stack under `/opt/docker/unifi`. The UniFi container publishes the standard controller ports (8443/8080/3478/10001/…) directly on the host so APs and switches can adopt to it; the web UI is also routed through Traefik on the configured hostname. The included `init-mongo.sh` runs on first MongoDB boot to create the unprivileged user and the three databases the controller needs.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An external `proxy_network` Docker network (typically provisioned by the `traefik` role)
- Traefik's `dynamic_conf.yml` must define the `insecure-transport@file` serversTransport (UniFi serves HTTPS with a self-signed cert that Traefik must not verify) — the `traefik` role in this collection ships this transport.

## Role Variables

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `unifi_hostname` | str | yes | — | Public hostname Traefik routes to the UniFi web UI. |
| `unifi_mongo_pass` | str | yes | — | Password for the unprivileged MongoDB user (`MONGO_PASS`). **Supply from vault.** |
| `unifi_mongo_initdb_root_password` | str | yes | — | Password for the MongoDB root user, used by `init-mongo.sh` to create the unprivileged user. **Supply from vault.** |
| `unifi_puid` | int | no | `1000` | PUID inside the linuxserver container. |
| `unifi_pgid` | int | no | `1000` | PGID inside the linuxserver container. |
| `unifi_timezone` | str | no | `America/New_York` | Container TZ. |
| `unifi_mongo_user` | str | no | `unifi` | MongoDB user the UniFi app authenticates as. |
| `unifi_mongo_host` | str | no | `unifi-db` | MongoDB host (inside the `unifi_network`). |
| `unifi_mongo_port` | int | no | `27017` | MongoDB port. |
| `unifi_mongo_dbname` | str | no | `unifi` | MongoDB database name. |
| `unifi_mongo_version` | str | no | `7.0.26` | Image tag for `docker.io/mongo`. |
| `unifi_mongo_authsource` | str | no | `admin` | MongoDB auth source. |
| `unifi_mongo_initdb_root_username` | str | no | `root` | MongoDB root username. |
| `unifi_mongo_tls` | str | no | `"false"` | Whether the UniFi app should connect to MongoDB over TLS. String, not bool. |
| `unifi_mem_limit` | int | no | `1024` | JVM max heap (MB). |
| `unifi_mem_startup` | int | no | `1024` | JVM initial heap (MB). |

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker, the external `proxy_network`, and Traefik with the `insecure-transport@file` serversTransport configured.

## Example Playbook

```yaml
- hosts: network_hosts
  become: true
  roles:
    - role: danmwallace.docker.unifi
      vars:
        unifi_hostname: unifi.example.com
        unifi_mongo_pass: "{{ vault_unifi_mongo_pass }}"
        unifi_mongo_initdb_root_password: "{{ vault_unifi_mongo_initdb_root_password }}"
```

## Published Ports

| Port | Protocol | Purpose |
| --- | --- | --- |
| 8080 | tcp | Device communication / inform |
| 8443 | tcp | Web UI (HTTPS) |
| 3478 | udp | STUN |
| 10001 | udp | Device discovery |
| 8843 | tcp | Guest portal HTTPS (optional) |
| 8880 | tcp | Guest portal HTTP (optional) |
| 6789 | tcp | Speed test (optional) |
| 1900 | udp | UPnP discovery (optional) |
| 5514 | udp | Remote syslog (optional) |

## First-Time Setup

Once the stack is up:

1. Navigate to `https://{{ unifi_hostname }}` (Traefik routes you to the controller).
2. Complete the controller setup wizard.
3. Adopt UniFi devices — devices on the same L2 will auto-discover; otherwise inform them with `set-inform http://<controller-host-or-ip>:8080/inform` via SSH (legacy creds `ubnt/ubnt`).

## License

MIT
