# Open WebUI

[Open WebUI](https://openwebui.com/) is a self-hosted UI for interacting with local LLMs (Ollama, OpenAI-compatible APIs).

## pocket-id OIDC setup

### 1. Register the OIDC client in pocket-id

| Field | Value |
|---|---|
| Name | `openwebui` |
| Callback URLs | `https://openwebui.example.com/oauth/oidc/callback` |
| Allowed User Groups | `openwebui_users` |

### 2. Configure Open WebUI via environment variables

| Variable | Value |
|---|---|
| `ENABLE_OAUTH_SIGNUP` | `true` |
| `OAUTH_CLIENT_ID` | Client ID from pocket-id |
| `OAUTH_CLIENT_SECRET` | Client Secret from pocket-id |
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
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/openwebui:/app/backend/data:z"
    env:
      ENABLE_OAUTH_SIGNUP: "true"
      OAUTH_CLIENT_ID: "{{ vault_openwebui_oidc_client_id }}"
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
