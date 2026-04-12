# Komodo

[Komodo](https://komo.do/) is a container and server management platform.

## pocket-id OIDC setup

### 1. Register the OIDC client in pocket-id

| Field | Value |
|---|---|
| Name | `komodo` |
| Callback URLs | `https://komodo.example.com/auth/oidc/callback` |
| Allowed User Groups | `komodo_users` |

### 2. Configure Komodo via environment variables

| Variable | Value |
|---|---|
| `KOMODO_OIDC_ENABLED` | `true` |
| `KOMODO_OIDC_PROVIDER` | `https://sso.example.com` |
| `KOMODO_OIDC_CLIENT_ID` | Client ID from pocket-id |
| `KOMODO_OIDC_CLIENT_SECRET` | Client Secret from pocket-id |

## Ansible service definition

```yaml
quadlets_services:
  - name: komodo
    image: ghcr.io/moghtech/komodo:latest
    virtual_host: komodo.example.com
    port: 9120
    oidc_client_id: komodo
    oidc_client_secret: "{{ vault_komodo_oidc_secret }}"
    pocket_id_group: komodo_users
    callback_urls:
      - "https://komodo.example.com/auth/oidc/callback"
    volumes:
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/komodo/data:/data:z"
    env:
      KOMODO_OIDC_ENABLED: "true"
      KOMODO_OIDC_PROVIDER: "https://sso.example.com"
      KOMODO_OIDC_CLIENT_ID: "{{ vault_komodo_oidc_client_id }}"
      KOMODO_OIDC_CLIENT_SECRET: "{{ vault_komodo_oidc_secret }}"
      KOMODO_HOST: "https://komodo.example.com"
```
