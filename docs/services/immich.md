# Immich

[Immich](https://immich.app/) is a self-hosted photo and video management solution.

## pocket-id OIDC setup

### 1. OIDC client in pocket-id

The role registers this client when you run `--tags pocket_id_clients` (see the
README): it creates the `immich_users` group and an OIDC client with ID `immich`
(`oidc_client_id`, default: the service name) for the `callback_urls`, with PKCE
enabled (`pkce_enabled`), restricted to that group. The play prints the client
secret once; you enter it in the Immich UI below.

To create it by hand in the pocket-id admin UI instead, use:

| Field | Value |
|---|---|
| Name | `immich` |
| Client ID | `immich` |
| Callback URLs | `https://immich.example.com/auth/login` |
| Callback URLs (settings) | `https://immich.example.com/user-settings` |
| Callback URLs (mobile) | `app.immich:///oauth-callback` |
| Allowed User Groups | `immich_users` |

### 2. Configure Immich

In the Immich web UI go to **Administration → Settings → Authentication Settings → OAuth**:

- Enable **OAuth Login**
- **Issuer URL**: `https://sso.example.com` (pocket-id URL)
- **Client ID**: `immich`
- **Client Secret**: the one printed by `--tags pocket_id_clients`
- **Scope**: `openid profile email`
- Enable **Auto Register** if you want new pocket-id users to be automatically created in Immich

## Ansible service definition

```yaml
quadlets_services:
  - name: immich-server
    image: ghcr.io/immich-app/immich-server:release
    virtual_host: immich.example.com
    port: 2283
    oidc_client_id: immich
    pocket_id_group: immich_users
    callback_urls:
      - "https://immich.example.com/auth/login"
      - "https://immich.example.com/user-settings"
      - "app.immich:///oauth-callback"
    volumes:
      - "%h/.local/share/immich-server:/usr/src/app/upload:Z"   # data dir created by the role
      - "/etc/localtime:/etc/localtime:ro"
    env:
      DB_HOSTNAME: "immich-db"
      DB_USERNAME: immich
      DB_PASSWORD: "{{ vault_immich_db_pass }}"
      DB_DATABASE_NAME: immich
      REDIS_HOSTNAME: "immich-redis"
      IMMICH_SERVER_URL: "https://immich.example.com"
```

!!! note "Additional services"
    Immich requires a PostgreSQL database with the `pgvecto-rs` extension and a Redis
    instance. Add them as service entries named `immich-db` and `immich-redis`
    **without** `virtual_host`: such entries are internal-only (not exposed by
    nginx-proxy, no certificate) but reachable by name on the `proxy` network as
    `DB_HOSTNAME` / `REDIS_HOSTNAME`. Add `depends_on: [immich-db, immich-redis]` here.
    OAuth is configured in the Immich UI, so no `oidc_client_secret` is needed.

!!! tip "Machine learning"
    Add `ghcr.io/immich-app/immich-machine-learning:release` as another internal
    service entry (e.g. `immich-ml`, no `virtual_host`) for smart search and face
    recognition, and set `MACHINE_LEARNING_URL: "http://immich-ml:3003"` in the
    server's env.
