# Audiobookshelf

[Audiobookshelf](https://www.audiobookshelf.org/) is a self-hosted audiobook and podcast server.

## pocket-id OIDC setup

### 1. Register the OIDC client in pocket-id

In the pocket-id admin UI, create a new OIDC application with:

| Field | Value |
|---|---|
| Name | `audiobookshelf` |
| Callback URLs | `https://audiobookshelf.example.com/auth/openid/callback` |
| Callback URLs (mobile) | `https://audiobookshelf.example.com/auth/openid/mobile-redirect` |
| Allowed User Groups | `audiobookshelf_users` |

### 2. Configure Audiobookshelf

In the Audiobookshelf web UI go to **Settings → Authentication**:

- Enable **OpenID Connect Authentication**
- **Issuer URL**: `https://sso.example.com` (the pocket-id URL — **do not** append `/.well-known/openid-configuration`)
- Click **Auto-Populate** to fill in the remaining endpoint URLs automatically
- Enter the **Client ID** and **Client Secret** from pocket-id
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
    oidc_client_secret: "{{ vault_audiobookshelf_oidc_secret }}"
    pocket_id_group: audiobookshelf_users
    callback_urls:
      - "https://audiobookshelf.example.com/auth/openid/callback"
      - "https://audiobookshelf.example.com/auth/openid/mobile-redirect"
    volumes:
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/audiobookshelf/config:/config:z"
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/audiobookshelf/metadata:/metadata:z"
      - "/media/audiobooks:/audiobooks:z"
    env:
      PORT: "80"
```

!!! note
    OIDC is configured through the Audiobookshelf UI, not via environment variables.
    The `oidc_client_id` and `oidc_client_secret` fields in the service definition are
    used for pocket-id client registration only (when `quadlets_pocket_id_provision_clients: true`).
