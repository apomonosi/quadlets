# Audiobookshelf

[Audiobookshelf](https://www.audiobookshelf.org/) is a self-hosted audiobook and podcast server.

## pocket-id OIDC setup

### 1. OIDC client in pocket-id

The role registers this client when you run `--tags pocket_id_clients` (see the
README): it creates the `audiobookshelf_users` group and an OIDC client with ID `audiobookshelf`
(`oidc_client_id`, default: the service name) for the `callback_urls`, with PKCE
enabled (`pkce_enabled`), restricted to that group. The play prints the client
secret once; you enter it in the Audiobookshelf UI below.

To create it by hand in the pocket-id admin UI instead, use:

| Field | Value |
|---|---|
| Name | `audiobookshelf` |
| Client ID | `audiobookshelf` |
| Callback URLs | `https://audiobookshelf.example.com/auth/openid/callback` |
| Callback URLs (mobile) | `https://audiobookshelf.example.com/auth/openid/mobile-redirect` |
| Allowed User Groups | `audiobookshelf_users` |

### 2. Configure Audiobookshelf

In the Audiobookshelf web UI go to **Settings → Authentication**:

- Enable **OpenID Connect Authentication**
- **Issuer URL**: `https://sso.example.com` (the pocket-id URL — **do not** append `/.well-known/openid-configuration`)
- Click **Auto-Populate** to fill in the remaining endpoint URLs automatically
- **Client ID**: `audiobookshelf`; **Client Secret**: the one printed by `--tags pocket_id_clients`
- Save and **restart the Audiobookshelf container** for changes to take effect

!!! warning "Issuer URL format"
    Audiobookshelf requires the bare issuer URL (e.g. `https://sso.example.com`).
    Remove the `/.well-known/openid-configuration` path if it appears after auto-populate.

## Ansible service definition

```yaml
quadlets_services:
  - name: audiobookshelf
    image: ghcr.io/advplyr/audiobookshelf:latest
    virtual_host: audiobookshelf.example.com
    port: 80
    oidc_client_id: audiobookshelf
    pocket_id_group: audiobookshelf_users
    callback_urls:
      - "https://audiobookshelf.example.com/auth/openid/callback"
      - "https://audiobookshelf.example.com/auth/openid/mobile-redirect"
    volumes:
      - "audiobookshelf-config:/config"        # named volumes: Podman creates them
      - "audiobookshelf-metadata:/metadata"
      - "/media/audiobooks:/audiobooks:z"
    env:
      PORT: "80"
```

!!! note
    OIDC is configured through the Audiobookshelf UI, not via environment variables,
    so the service definition needs no `oidc_client_secret`. `oidc_client_id`,
    `pocket_id_group` and `callback_urls` are used to register the client in pocket-id.
