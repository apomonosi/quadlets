# Vaultwarden

[Vaultwarden](https://github.com/dani-garcia/vaultwarden) is an unofficial Bitwarden-compatible
password manager server.

## pocket-id OIDC setup

### 1. OIDC client in pocket-id

The role registers this client when you run `--tags pocket_id_clients` (see the
README): it creates the `vaultwarden_users` group and an OIDC client with ID `vaultwarden`
(`oidc_client_id`, default: the service name) for the `callback_urls`, with PKCE
enabled (`pkce_enabled`), restricted to that group. The play prints the client
secret once — store it in Ansible Vault as `vault_vaultwarden_oidc_secret`.

To create it by hand in the pocket-id admin UI instead, use:

| Field | Value |
|---|---|
| Name | `vaultwarden` |
| Client ID | `vaultwarden` |
| Callback URLs | `https://vaultwarden.example.com/identity/connect/oidc-signin` |
| Allowed User Groups | `vaultwarden_users` |

### 2. Configure Vaultwarden via environment variables

Vaultwarden SSO requires the environment variable `SSO_ENABLED=true` plus:

| Variable | Value |
|---|---|
| `SSO_ENABLED` | `true` |
| `SSO_ONLY` | `false` (set `true` to disable local login) |
| `SSO_CLIENT_ID` | `vaultwarden` (the `oidc_client_id`) |
| `SSO_CLIENT_SECRET` | injected by the role from `oidc_client_secret` (`oidc_client_secret_env`) |
| `SSO_AUTHORITY` | `https://sso.example.com` |
| `DOMAIN` | `https://vaultwarden.example.com` |

!!! note "SSO feature flag"
    SSO support in Vaultwarden requires a build with the `sso` feature flag enabled.
    Use the image tag `vaultwarden/server:latest` — SSO is included in official builds
    since v1.32.0.

## Ansible service definition

```yaml
quadlets_services:
  - name: vaultwarden
    image: vaultwarden/server:latest
    virtual_host: vaultwarden.example.com
    port: 80
    oidc_client_id: vaultwarden
    oidc_client_secret: "{{ vault_vaultwarden_oidc_secret }}"
    oidc_client_secret_env: SSO_CLIENT_SECRET   # injected from a Podman secret
    pocket_id_group: vaultwarden_users
    callback_urls:
      - "https://vaultwarden.example.com/identity/connect/oidc-signin"
    volumes:
      - "%h/.local/share/vaultwarden:/data:Z"   # data dir created by the role
    secrets:                             # stored as Podman secrets, injected as env
      ADMIN_TOKEN: "{{ vault_vaultwarden_admin_token }}"
    env:
      DOMAIN: "https://vaultwarden.example.com"
      SSO_ENABLED: "true"
      SSO_ONLY: "false"
      SSO_CLIENT_ID: vaultwarden
      SSO_AUTHORITY: "https://sso.example.com"
      SIGNUPS_ALLOWED: "false"
```

!!! tip "Signup policy"
    Set `SIGNUPS_ALLOWED=false` once your initial accounts are created to prevent
    unauthorised registration. Users can still log in via pocket-id SSO.
