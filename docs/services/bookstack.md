# BookStack

[BookStack](https://www.bookstackapp.com/) is a self-hosted wiki and knowledge base.

## pocket-id OIDC setup

### 1. Register the OIDC client in pocket-id

| Field | Value |
|---|---|
| Name | `bookstack` |
| Callback URLs | `https://bookstack.example.com/oidc/callback` |
| Allowed User Groups | `bookstack_users` |
| PKCE | Recommended (enable in pocket-id) |

### 2. Configure BookStack via environment variables

BookStack is configured entirely through environment variables. Add the following
to the `env:` block in your service definition:

| Variable | Value |
|---|---|
| `AUTH_METHOD` | `oidc` |
| `AUTH_AUTO_INITIATE` | `true` |
| `OIDC_NAME` | `Pocket ID` |
| `OIDC_DISPLAY_NAME_CLAIMS` | `name` |
| `OIDC_CLIENT_ID` | Client ID from pocket-id |
| `OIDC_CLIENT_SECRET` | Client Secret from pocket-id |
| `OIDC_ISSUER` | `https://sso.example.com` |
| `OIDC_ISSUER_DISCOVER` | `true` |
| `OIDC_END_SESSION_ENDPOINT` | `true` |

**Group sync (optional):** BookStack can sync pocket-id groups as roles.
Enable group sync and set `OIDC_GROUPS_CLAIM=groups` — pocket-id includes
group names in the ID token by default.

## Ansible service definition

```yaml
quadlets_services:
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
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/bookstack/config:/config:z"
    env:
      APP_URL: "https://bookstack.example.com"
      AUTH_METHOD: "oidc"
      AUTH_AUTO_INITIATE: "true"
      OIDC_NAME: "Pocket ID"
      OIDC_DISPLAY_NAME_CLAIMS: "name"
      OIDC_CLIENT_ID: bookstack          # replaced at runtime via oidc_client_id
      OIDC_ISSUER: "https://sso.example.com"
      OIDC_ISSUER_DISCOVER: "true"
      OIDC_END_SESSION_ENDPOINT: "true"
      DB_HOST: "bookstack-db"
      DB_USER: bookstack
      DB_PASS: "{{ vault_bookstack_db_pass }}"
      DB_DATABASE: bookstack
```

!!! tip
    BookStack requires a MariaDB/MySQL database. Deploy it as a separate service entry
    with `depends_on: []` and no `VIRTUAL_HOST` (omit `virtual_host` or route internally).
