# BookStack

[BookStack](https://www.bookstackapp.com/) is a self-hosted wiki and knowledge base.

## pocket-id OIDC setup

### 1. OIDC client in pocket-id

The role registers this client when you run `--tags pocket_id_clients` (see the
README): it creates the `bookstack_users` group and an OIDC client with ID `bookstack`
(`oidc_client_id`, default: the service name) for the `callback_urls`, with PKCE
enabled (`pkce_enabled`), restricted to that group. The play prints the client
secret once — store it in Ansible Vault as `vault_bookstack_oidc_secret`.

To create it by hand in the pocket-id admin UI instead, use:

| Field | Value |
|---|---|
| Name | `bookstack` |
| Client ID | `bookstack` |
| Callback URLs | `https://bookstack.example.com/oidc/callback` |
| Allowed User Groups | `bookstack_users` |
| PKCE | Enabled |

### 2. Configure BookStack via environment variables

BookStack is configured entirely through environment variables. The role already sets
`OIDC_CLIENT_ID` (from `oidc_client_id`) and injects `OIDC_CLIENT_SECRET` from a Podman
secret; add the rest to the `env:` block in your service definition:

| Variable | Value |
|---|---|
| `AUTH_METHOD` | `oidc` |
| `AUTH_AUTO_INITIATE` | `true` |
| `OIDC_NAME` | `Pocket ID` |
| `OIDC_DISPLAY_NAME_CLAIMS` | `name` |
| `OIDC_CLIENT_ID` | set by the role from `oidc_client_id` |
| `OIDC_CLIENT_SECRET` | injected by the role from `oidc_client_secret` |
| `OIDC_ISSUER` | `https://sso.example.com` |
| `OIDC_ISSUER_DISCOVER` | `true` |
| `OIDC_END_SESSION_ENDPOINT` | `true` |

**Group sync (optional):** BookStack can sync pocket-id groups as roles.
Enable group sync and set `OIDC_GROUPS_CLAIM=groups` — pocket-id includes
group names in the ID token by default.

## Ansible service definition

```yaml
quadlets_services:
  # Internal-only (no virtual_host): reachable as bookstack-db on the proxy network
  - name: bookstack-db
    image: docker.io/library/mariadb:11
    volumes:
      - "bookstack-db:/var/lib/mysql"
    secrets:                             # stored as Podman secrets, injected as env
      MARIADB_PASSWORD: "{{ vault_bookstack_db_pass }}"
    env:
      MARIADB_DATABASE: bookstack
      MARIADB_USER: bookstack
      MARIADB_RANDOM_ROOT_PASSWORD: "1"

  - name: bookstack
    image: lscr.io/linuxserver/bookstack:latest
    virtual_host: bookstack.example.com
    port: 80
    oidc_client_id: bookstack
    oidc_client_secret: "{{ vault_bookstack_oidc_secret }}"
    oidc_client_secret_env: OIDC_CLIENT_SECRET   # injected from a Podman secret
    pocket_id_group: bookstack_users
    callback_urls:
      - "https://bookstack.example.com/oidc/callback"
    volumes:
      - "%h/.local/share/bookstack:/config:Z"   # data dir created by the role
    secrets:                             # stored as Podman secrets, injected as env
      DB_PASS: "{{ vault_bookstack_db_pass }}"
    env:
      APP_URL: "https://bookstack.example.com"
      AUTH_METHOD: "oidc"
      AUTH_AUTO_INITIATE: "true"
      OIDC_NAME: "Pocket ID"
      OIDC_DISPLAY_NAME_CLAIMS: "name"
      OIDC_ISSUER: "https://sso.example.com"
      OIDC_ISSUER_DISCOVER: "true"
      OIDC_END_SESSION_ENDPOINT: "true"
      DB_HOST: "bookstack-db"
      DB_USER: bookstack
      DB_DATABASE: bookstack
    depends_on: [bookstack-db]
```

!!! tip
    BookStack requires a MariaDB/MySQL database. The `bookstack-db` entry above has no
    `virtual_host`, so it is internal-only: nginx-proxy does not expose it and no
    certificate is made, but BookStack reaches it as `DB_HOST: bookstack-db` on the
    `proxy` network. `depends_on` starts it first.
