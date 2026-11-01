# Grafana

[Grafana](https://grafana.com/) is an observability and metrics visualization platform.

## pocket-id OIDC setup

### 1. OIDC client in pocket-id

The role registers this client when you run `--tags pocket_id_clients` (see the
README): it creates the `grafana_users` group and an OIDC client with ID `grafana`
(`oidc_client_id`, default: the service name) for the `callback_urls`, with PKCE
enabled (`pkce_enabled`), restricted to that group. The play prints the client
secret once — store it in Ansible Vault as `vault_grafana_oidc_secret`.

To create it by hand in the pocket-id admin UI instead, use:

| Field | Value |
|---|---|
| Name | `grafana` |
| Client ID | `grafana` |
| Callback URLs | `https://grafana.example.com/login/generic_oauth` |
| Allowed User Groups | `grafana_users` |

### 2. Configure Grafana via environment variables

Grafana uses its Generic OAuth integration to connect to pocket-id:

| Variable | Value |
|---|---|
| `GF_AUTH_GENERIC_OAUTH_ENABLED` | `true` |
| `GF_AUTH_GENERIC_OAUTH_NAME` | `Pocket ID` |
| `GF_AUTH_GENERIC_OAUTH_CLIENT_ID` | `grafana` (the `oidc_client_id`) |
| `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET` | injected by the role from `oidc_client_secret` (`oidc_client_secret_env`) |
| `GF_AUTH_GENERIC_OAUTH_SCOPES` | `openid profile email` |
| `GF_AUTH_GENERIC_OAUTH_AUTH_URL` | `https://sso.example.com/authorize` |
| `GF_AUTH_GENERIC_OAUTH_TOKEN_URL` | `https://sso.example.com/api/oidc/token` |
| `GF_AUTH_GENERIC_OAUTH_API_URL` | `https://sso.example.com/api/oidc/userinfo` |
| `GF_AUTH_GENERIC_OAUTH_USE_PKCE` | `true` |
| `GF_SERVER_ROOT_URL` | `https://grafana.example.com` |

**Troubleshooting:** If login fails with "wrong callback URL", ensure `GF_SERVER_ROOT_URL`
is set correctly. If "Sign up is disabled" appears, set `GF_AUTH_GENERIC_OAUTH_AUTO_LOGIN=true`.

## Ansible service definition

```yaml
quadlets_services:
  - name: grafana
    image: grafana/grafana:latest
    virtual_host: grafana.example.com
    port: 3000
    oidc_client_id: grafana
    oidc_client_secret: "{{ vault_grafana_oidc_secret }}"
    oidc_client_secret_env: GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET   # injected from a Podman secret
    pocket_id_group: grafana_users
    callback_urls:
      - "https://grafana.example.com/login/generic_oauth"
    volumes:
      # Role-created data dir; U chowns it to the grafana user (UID 472) in the container
      - "%h/.local/share/grafana:/var/lib/grafana:Z,U"
    env:
      GF_SERVER_ROOT_URL: "https://grafana.example.com"
      GF_AUTH_GENERIC_OAUTH_ENABLED: "true"
      GF_AUTH_GENERIC_OAUTH_NAME: "Pocket ID"
      GF_AUTH_GENERIC_OAUTH_CLIENT_ID: grafana
      GF_AUTH_GENERIC_OAUTH_SCOPES: "openid profile email"
      GF_AUTH_GENERIC_OAUTH_AUTH_URL: "https://sso.example.com/authorize"
      GF_AUTH_GENERIC_OAUTH_TOKEN_URL: "https://sso.example.com/api/oidc/token"
      GF_AUTH_GENERIC_OAUTH_API_URL: "https://sso.example.com/api/oidc/userinfo"
      GF_AUTH_GENERIC_OAUTH_USE_PKCE: "true"
      GF_AUTH_GENERIC_OAUTH_AUTO_LOGIN: "false"
```
