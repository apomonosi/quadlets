# Immich

[Immich](https://immich.app/) is a self-hosted photo and video management solution.

## pocket-id OIDC setup

### 1. Register the OIDC client in pocket-id

| Field | Value |
|---|---|
| Name | `immich` |
| Callback URLs | `https://immich.example.com/auth/login` |
| Callback URLs (settings) | `https://immich.example.com/user-settings` |
| Callback URLs (mobile) | `app.immich:///oauth-callback` |
| Allowed User Groups | `immich_users` |

### 2. Configure Immich

In the Immich web UI go to **Administration → Settings → Authentication Settings → OAuth**:

- Enable **OAuth Login**
- **Issuer URL**: `https://sso.example.com` (pocket-id URL)
- **Client ID**: from pocket-id
- **Client Secret**: from pocket-id
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
    oidc_client_secret: "{{ vault_immich_oidc_secret }}"
    pocket_id_group: immich_users
    callback_urls:
      - "https://immich.example.com/auth/login"
      - "https://immich.example.com/user-settings"
      - "app.immich:///oauth-callback"
    volumes:
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/immich/upload:/usr/src/app/upload:z"
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
    instance. Deploy these as additional service entries without `VIRTUAL_HOST`
    (add a dedicated network or use `depends_on`).

!!! tip "Machine learning"
    Deploy `ghcr.io/immich-app/immich-machine-learning:release` as a separate quadlet
    for smart search and face recognition. Connect it to the same network and set
    `MACHINE_LEARNING_URL` in the server's env.
