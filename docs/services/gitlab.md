# GitLab

[GitLab CE](https://about.gitlab.com/) is a self-hosted Git platform with CI/CD.

## pocket-id OIDC setup

### 1. Register the OIDC client in pocket-id

| Field | Value |
|---|---|
| Name | `gitlab` |
| Callback URLs | `https://gitlab.example.com/users/auth/openid_connect/callback` |
| Allowed User Groups | `gitlab_users` |

### 2. Configure GitLab

GitLab OIDC is configured via `gitlab.rb` (Omnibus) or `gitlab.yml` (source).
Using the container, pass configuration via `GITLAB_OMNIBUS_CONFIG`:

```ruby
gitlab_rails['omniauth_enabled'] = true
gitlab_rails['omniauth_allow_single_sign_on'] = ['openid_connect']
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_providers'] = [
  {
    name: "openid_connect",
    label: "Pocket ID",
    args: {
      name: "openid_connect",
      scope: ["openid", "profile", "email"],
      response_type: "code",
      issuer: "https://sso.example.com",
      discovery: true,
      client_auth_method: "query",
      uid_field: "preferred_username",
      client_options: {
        identifier: "<CLIENT_ID>",
        secret: "<CLIENT_SECRET>",
        redirect_uri: "https://gitlab.example.com/users/auth/openid_connect/callback"
      }
    }
  }
]
```

## Ansible service definition

```yaml
quadlets_services:
  - name: gitlab
    image: gitlab/gitlab-ce:latest
    virtual_host: gitlab.example.com
    port: 80
    oidc_client_id: gitlab
    oidc_client_secret: "{{ vault_gitlab_oidc_secret }}"
    pocket_id_group: gitlab_users
    callback_urls:
      - "https://gitlab.example.com/users/auth/openid_connect/callback"
    volumes:
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/gitlab/config:/etc/gitlab:z"
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/gitlab/logs:/var/log/gitlab:z"
      - "{{ quadlets_data_base_dir | default('/home/podman/.local/share') }}/gitlab/data:/var/opt/gitlab:z"
    env:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.example.com'
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        nginx['proxy_set_headers'] = {
          'X-Forwarded-Proto' => 'https',
          'X-Forwarded-Ssl' => 'on'
        }
        gitlab_rails['omniauth_enabled'] = true
        gitlab_rails['omniauth_allow_single_sign_on'] = ['openid_connect']
        gitlab_rails['omniauth_block_auto_created_users'] = false
```

!!! warning "Resource requirements"
    GitLab requires at least 4 GB of RAM. Ensure your host has sufficient memory.

!!! note "SSL termination"
    nginx-proxy handles TLS. GitLab must be configured to listen on HTTP internally
    (`nginx['listen_https'] = false`) and trust the `X-Forwarded-Proto: https` header.
