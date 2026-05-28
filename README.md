# Ansible Collection — danmwallace.docker

Roles for deploying useful Docker containers as Docker Compose stacks on a target
host. Each service role renders a `docker-compose.yml` under `/opt/docker/<service>/`
and brings the stack up via `community.docker.docker_compose_v2`. Proxied services
attach to a shared external `proxy_network` and expose themselves only through the
`traefik` role, which terminates TLS using Let's Encrypt certificates issued over the
Cloudflare DNS-01 challenge — so the whole set can be composed together on one host
without port conflicts. Two foundation roles (`common`, `docker`) prepare the host
itself.

## Requirements

- Ansible >= 2.16
- Collection dependencies (from `galaxy.yml`):
  - `community.docker >= 3.0.0`
  - `community.general >= 8.0.0`
  - `ansible.posix >= 1.5.0`
- Target host with Docker installed and an `ansible` user plus `docker` group
  (service roles chown rendered files to `ansible:docker`). The `docker` role can
  provide the runtime; most service roles also expect the external `proxy_network`
  from `traefik`.

## Installation

```bash
ansible-galaxy collection install danmwallace.docker
```

Or pin it in `requirements.yml`:

```yaml
collections:
  - name: danmwallace.docker
    version: ">=1.1.0"
```

## Roles

| Role | Description |
| --- | --- |
| [`danmwallace.docker.adguard`](roles/adguard/README.md) | Deploy AdGuard Home DNS server in Docker behind Traefik via Docker Compose. |
| [`danmwallace.docker.arcane`](roles/arcane/README.md) | Deploy Arcane, a modern web UI for managing Docker containers, via Docker Compose. |
| [`danmwallace.docker.baserow`](roles/baserow/README.md) | Deploy Baserow, an open-source no-code database, in Docker behind Traefik via Docker Compose. |
| [`danmwallace.docker.common`](roles/common/README.md) | Baseline system configuration — hostname, packages, timezone, users, sudoers — for homelab hosts. |
| [`danmwallace.docker.docker`](roles/docker/README.md) | Install Docker CE and the compose plugin on Debian, Ubuntu, and Fedora (Server and Atomic). |
| [`danmwallace.docker.grafana`](roles/grafana/README.md) | Deploy Grafana in Docker behind Traefik via Docker Compose. |
| [`danmwallace.docker.it_tools`](roles/it_tools/README.md) | Deploy IT-Tools, a web collection of developer/IT utilities, in Docker behind Traefik via Docker Compose. |
| [`danmwallace.docker.librechat`](roles/librechat/README.md) | Deploy LibreChat (multi-LLM chat UI) with Meilisearch, pgvector, and a RAG sidecar behind Traefik via Docker Compose. |
| [`danmwallace.docker.n8n`](roles/n8n/README.md) | Deploy n8n workflow automation with a PostgreSQL backend behind Traefik via Docker Compose. |
| [`danmwallace.docker.semaphore`](roles/semaphore/README.md) | Deploy Semaphore UI for Ansible automation with a PostgreSQL backend behind Traefik via Docker Compose. |
| [`danmwallace.docker.traefik`](roles/traefik/README.md) | Deploy Traefik reverse proxy with automatic SSL via Let's Encrypt and Cloudflare DNS-01. |
| [`danmwallace.docker.unifi`](roles/unifi/README.md) | Deploy UniFi Network Application (linuxserver image) with a MongoDB backend behind Traefik via Docker Compose. |

## Example Playbook

```yaml
- hosts: docker_hosts
  become: true
  roles:
    - danmwallace.docker.common
    - danmwallace.docker.docker
    - danmwallace.docker.traefik
```

## License

MIT
