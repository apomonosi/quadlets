# Komodo

[Komodo](https://komo.do/) is a container and server management platform.

## pocket-id OIDC setup

### 1. OIDC client in pocket-id

The role registers this client when you run `--tags pocket_id_clients` (see the
README): it creates the `komodo_users` group and an OIDC client with ID `komodo`
(`oidc_client_id`, default: the service name) for the `callback_urls`, with PKCE
enabled (`pkce_enabled`), restricted to that group. The play prints the client
secret once — store it in Ansible Vault as `vault_komodo_oidc_secret`.

To create it by hand in the pocket-id admin UI instead, use:

| Field | Value |
|---|---|
| Name | `komodo` |
| Client ID | `komodo` |
| Callback URLs | `https://komodo.example.com/auth/oidc/callback` |
| Allowed User Groups | `komodo_users` |

### 2. Configure Komodo via environment variables

| Variable | Value |
|---|---|
| `KOMODO_OIDC_ENABLED` | `true` |
| `KOMODO_OIDC_PROVIDER` | `https://sso.example.com` |
| `KOMODO_OIDC_CLIENT_ID` | `komodo` (the `oidc_client_id`) |
| `KOMODO_OIDC_CLIENT_SECRET` | injected by the role from `oidc_client_secret` (`oidc_client_secret_env`) |

## Ansible service definition

```yaml
quadlets_services:
  - name: komodo
    image: ghcr.io/moghtech/komodo:latest
    virtual_host: komodo.example.com
    port: 9120
    oidc_client_id: komodo
    oidc_client_secret: "{{ vault_komodo_oidc_secret }}"
    oidc_client_secret_env: KOMODO_OIDC_CLIENT_SECRET   # injected from a Podman secret
    pocket_id_group: komodo_users
    callback_urls:
      - "https://komodo.example.com/auth/oidc/callback"
    volumes:
      - "%h/.local/share/komodo:/data:Z"   # data dir created by the role
    env:
      KOMODO_OIDC_ENABLED: "true"
      KOMODO_OIDC_PROVIDER: "https://sso.example.com"
      KOMODO_OIDC_CLIENT_ID: komodo
      KOMODO_HOST: "https://komodo.example.com"
```
