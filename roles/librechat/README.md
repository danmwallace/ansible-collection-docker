# danmwallace.docker.librechat

Deploys [LibreChat](https://www.librechat.ai/), a self-hosted multi-LLM chat interface, as a Docker Compose stack under `/opt/docker/librechat`. Clones the upstream LibreChat repository at a pinned ref, renders `.env`, `docker-compose.ansible.yml`, and `librechat.yaml` from templates on top of the clone, and brings up four containers behind Traefik:

- `librechat` — the API and UI (default port `3080`)
- `meilisearch` — full-text search backend
- `vectordb` — pgvector PostgreSQL for RAG embeddings
- `rag_api` — sidecar that serves the RAG pipeline (default port `8000`)

The role joins the external `proxy_network` Docker network for Traefik routing and (optionally) bind-mounts an Obsidian vault into the API container so a configured `mcp-obsidian` server can read it.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Target host with Docker (or a Docker-compatible runtime) installed
- An external `proxy_network` Docker network (typically provisioned by the `traefik` role)
- An `ansible` user in the `docker` group on the target host (override via `librechat_owner` / `librechat_group`)
- A vault file defining the `vault_librechat_*` keys the templates expect (see [Vault Keys](#vault-keys))

## Role Variables

The role exposes ~200 settings via `defaults/main.yml` — every LLM endpoint, every UI toggle, every speech/file/memory/email/OAuth/RAG/redis/MCP option that LibreChat's `.env` supports. The full list is documented inline in `defaults/main.yml` and serves as the canonical reference; the table below covers only the core required + commonly-overridden settings.

> **LibreChat's `.env` options change frequently** — upstream adds, renames, and removes settings release to release. This role's `defaults/main.yml` and `templates/env.j2` capture a *snapshot* of that surface, not a live mirror, so treat them as a starting point rather than an authoritative contract. Before relying on or changing any setting, review the official [LibreChat `.env` reference](https://www.librechat.ai/docs/configuration/dotenv). If you need an option this role doesn't expose yet, or upstream has changed one, the intended path is to **fork the collection and edit `defaults/main.yml` and `templates/env.j2`** to match the version of LibreChat you're running.

| Variable | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `librechat_hostname` | str | yes | — | Public hostname Traefik routes to the LibreChat UI. |
| `librechat_version` | str | no | `v0.8.7` | Git ref of `danny-avila/LibreChat` to clone. Kept in step with `librechat_api_image`; `main` tracks upstream HEAD. |
| `librechat_install_path` | path | no | `/opt/docker/librechat` | Filesystem location of the LibreChat checkout. |
| `librechat_data_path` | path | no | `/opt/docker/librechat/data` | Subdirectory for persistent container volumes. Excluded from ownership-migration. |
| `librechat_owner` | str | no | `ansible` | User that owns the checkout and config files. Must be in the docker group. |
| `librechat_group` | str | no | `docker` | Group that owns the checkout and config files. |
| `librechat_port` | int | no | `3080` | API/UI container port. |
| `librechat_api_image` | str | no | `ghcr.io/danny-avila/librechat-api:v0.8.7` | API container image. |
| `librechat_meilisearch_image` | str | no | `getmeili/meilisearch:v1.53.2` | Meilisearch container image. |
| `librechat_vectordb_image` | str | no | `pgvector/pgvector:0.8.6-pg16-bookworm` | pgvector container image (Postgres 16 — changing the PG major requires a manual `pg_upgrade`/dump-restore of `librechat_data_path/vectordb`). |
| `librechat_ragapi_image` | str | no | `ghcr.io/danny-avila/librechat-rag-api-dev-lite:v0.9.0` | RAG API container image. |
| `librechat_vectordb_postgres_db` | str | no | `librechat` | pgvector database name. |
| `librechat_vectordb_postgres_user` | str | no | `librechat` | pgvector database user. |
| `librechat_meilisearch_upgrade_db` | bool | no | `true` | Set `MEILI_UPGRADE_DB=true` on the Meilisearch container so it migrates an existing index in place when the image moves to a newer minor version. Set `false` to fail loudly on a version mismatch instead. |
| `librechat_obsidian_mount_enabled` | bool | no | `true` | Bind-mount an Obsidian vault and register the mcp-obsidian server. |
| `librechat_obsidian_mount_path` | path | no | `/opt/syncthing/obsidian` | Host path of the Obsidian vault when the mount is enabled. |

Everything else in `defaults/main.yml` is overridable from inventory the usual way; consult that file for the comprehensive list.

## Vault Keys

The `.env` and `librechat.yaml` templates reference secrets under the `vault_librechat_*` namespace. These are **caller-supplied** — they are intentionally not declared in `defaults/main.yml` so they cannot accidentally land in plaintext. Define them in a vault file (e.g. `group_vars/librechat/vault.yml`) before running the role.

Required:

- `vault_librechat_mongo_uri` — full MongoDB connection string the API uses for its primary store
- `vault_librechat_vector_postgres_password` — password for `librechat_vectordb_postgres_user`

Conditionally required (depending on which providers / features you enable):

- `vault_librechat_anthropic_api_key`, `vault_librechat_openai_api_key`, `vault_librechat_google_*`, `vault_librechat_azure_*`, `vault_librechat_bedrock_*` — per LLM endpoint
- `vault_librechat_serper_api_key`, `vault_librechat_firecrawl_api_key`, etc. — per web-search provider
- `vault_librechat_openid_client_secret`, `vault_librechat_google_client_secret`, etc. — per OAuth provider
- `vault_librechat_aws_secret_access_key`, `vault_librechat_azure_storage_connection_string` — per file-storage backend
- `vault_librechat_email_password`, `vault_librechat_mailgun_api_key` — per email backend

See `templates/env.j2` for the authoritative list (search for `vault_librechat_`).

## Dependencies

None declared in `meta/main.yml`. In practice the target host needs Docker, the external `proxy_network`, and the `ansible:docker` user/group precondition.

## Example Playbook

```yaml
- hosts: librechat_hosts
  become: true
  roles:
    - role: danmwallace.docker.librechat
      vars:
        librechat_hostname: chat.example.com
        librechat_version: v0.8.7
        librechat_endpoints: anthropic,openai
        librechat_obsidian_mount_enabled: false
```

With a `vault.yml` providing at minimum:

```yaml
vault_librechat_mongo_uri: "mongodb://mongo:27017/LibreChat"
vault_librechat_vector_postgres_password: "..."
vault_librechat_anthropic_api_key: "..."
vault_librechat_openai_api_key: "..."
```

## Task Tags

The role's tasks are tagged so partial runs are possible:

- `librechat-setup` — install dir, ownership migration, git clone, data dir, docker_network
- `librechat-config` — render `.env`, `docker-compose.ansible.yml`, `librechat.yaml`
- `librechat-deploy` — docker_network and docker_compose_v2 deploy

Example: `ansible-playbook ... --tags librechat-config` re-renders the config files without re-cloning or restarting containers (template changes still trigger the `Restart librechat` handler at end of play).

## Notes

- `librechat_version` defaults to the same release as `librechat_api_image`; keep the two aligned when overriding, since the checkout supplies `librechat.yaml` schema and client assets the API image expects. Setting it to `main` tracks upstream HEAD, which is convenient but fragile.
- The compose file is rendered to `docker-compose.ansible.yml`, not upstream's tracked `docker-compose.yml`, so the `force: true` git update never reverts it and the stack is not restarted on every converge.
- Meilisearch will not start on a data directory written by an older version unless `MEILI_UPGRADE_DB` is set. `librechat_meilisearch_upgrade_db` defaults to `true` so the 1.3.1 bump from v1.49 to v1.53 upgrades `librechat_data_path/meilisearch` in place. Take a copy of that directory first if the index matters to you; the upgrade is one-way.
- The role runs the file/template/git/docker_compose tasks as `librechat_owner` (`become: false`) rather than root, which requires the connecting user to actually be that user (or to have ssh agent forwarding / become_user wired up). Only the install-dir create, ownership migration, data-dir create, and docker_network tasks run as root.
- The ownership-migration task at the start exists for upgrades from earlier deployments that cloned LibreChat as root; on fresh installs it's a no-op.

## License

MIT
