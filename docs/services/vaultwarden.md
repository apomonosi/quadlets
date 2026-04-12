# Vaultwarden

[Vaultwarden](https://github.com/dani-garcia/vaultwarden) is an unofficial Bitwarden-compatible
password manager server.

## pocket-id OIDC setup

### 1. Register the OIDC client in pocket-id

| Field | Value |
|---|---|
| Name | `vaultwarden` |
| Callback URLs | `https://vaultwarden.example.com/identity/connect/oidc-signin` |
| Allowed User Groups | `vaultwarden_users` |

### 2. Configure Vaultwarden via environment variables

Vaultwarden SSO requires the environment variable `SSO_ENABLED=true` plus:

| Variable | Value |
|---|---|
| `SSO_ENABLED` | `true` |
| `SSO_ONLY` | `false` (set `true` to disable local login) |
| `SSO_CLIENT_ID` | Client ID from pocket-id |
| `SSO_CLIENT_SECRET` | Client Secret from pocket-id |
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
    pocket_id_group: vaultwarden_users
    callback_urls:
      - "https://vaultwarden.example.com/identity/connect/oidc-signin"
    volumes:
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/vaultwarden/data:/data:z"
    env:
      DOMAIN: "https://vaultwarden.example.com"
      SSO_ENABLED: "true"
      SSO_ONLY: "false"
      SSO_CLIENT_ID: "{{ vault_vaultwarden_oidc_client_id }}"
      SSO_CLIENT_SECRET: "{{ vault_vaultwarden_oidc_secret }}"
      SSO_AUTHORITY: "https://sso.example.com"
      SIGNUPS_ALLOWED: "false"
      ADMIN_TOKEN: "{{ vault_vaultwarden_admin_token }}"
```

!!! tip "Signup policy"
    Set `SIGNUPS_ALLOWED=false` once your initial accounts are created to prevent
    unauthorised registration. Users can still log in via pocket-id SSO.
