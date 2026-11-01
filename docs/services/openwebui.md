# Open WebUI

[Open WebUI](https://openwebui.com/) is a self-hosted UI for interacting with local LLMs (Ollama, OpenAI-compatible APIs).

## pocket-id OIDC setup

### 1. OIDC client in pocket-id

The role registers this client when you run `--tags pocket_id_clients` (see the
README): it creates the `openwebui_users` group and an OIDC client with ID `openwebui`
(`oidc_client_id`, default: the service name) for the `callback_urls`, with PKCE
enabled (`pkce_enabled`), restricted to that group. The play prints the client
secret once — store it in Ansible Vault as `vault_openwebui_oidc_secret`.

To create it by hand in the pocket-id admin UI instead, use:

| Field | Value |
|---|---|
| Name | `openwebui` |
| Client ID | `openwebui` |
| Callback URLs | `https://openwebui.example.com/oauth/oidc/callback` |
| Allowed User Groups | `openwebui_users` |

### 2. Configure Open WebUI via environment variables

| Variable | Value |
|---|---|
| `ENABLE_OAUTH_SIGNUP` | `true` |
| `OAUTH_CLIENT_ID` | `openwebui` (the `oidc_client_id`) |
| `OAUTH_CLIENT_SECRET` | injected by the role from `oidc_client_secret` (`oidc_client_secret_env`) |
| `OAUTH_PROVIDER_NAME` | `Pocket ID` |
| `OPENID_PROVIDER_URL` | `https://sso.example.com/.well-known/openid-configuration` |
| `OAUTH_MERGE_ACCOUNTS_BY_EMAIL` | `true` |
| `OPENID_REDIRECT_URI` | `https://openwebui.example.com/oauth/oidc/callback` |

!!! warning "Callback URL mismatch"
    If pocket-id throws an **Invalid Callback URL** error, explicitly set `OPENID_REDIRECT_URI`
    to ensure HTTPS is used. This is required when Open WebUI sits behind a reverse proxy.

## Ansible service definition

```yaml
quadlets_services:
  - name: openwebui
    image: ghcr.io/open-webui/open-webui:main
    virtual_host: openwebui.example.com
    port: 8080
    oidc_client_id: openwebui
    oidc_client_secret: "{{ vault_openwebui_oidc_secret }}"
    oidc_client_secret_env: OAUTH_CLIENT_SECRET   # injected from a Podman secret
    pocket_id_group: openwebui_users
    callback_urls:
      - "https://openwebui.example.com/oauth/oidc/callback"
    volumes:
      - "%h/.local/share/openwebui:/app/backend/data:Z"   # data dir created by the role
    env:
      ENABLE_OAUTH_SIGNUP: "true"
      OAUTH_CLIENT_ID: openwebui
      OAUTH_PROVIDER_NAME: "Pocket ID"
      OPENID_PROVIDER_URL: "https://sso.example.com/.well-known/openid-configuration"
      OAUTH_MERGE_ACCOUNTS_BY_EMAIL: "true"
      OPENID_REDIRECT_URI: "https://openwebui.example.com/oauth/oidc/callback"
      # Connect to an Ollama instance on the host (adjust as needed)
      OLLAMA_BASE_URL: "http://host.containers.internal:11434"
```

!!! tip
    To connect to Ollama running on the host, use `host.containers.internal` as the hostname.
    This special hostname resolves to the host machine from inside a rootless Podman container.
