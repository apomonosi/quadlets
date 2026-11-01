# GitLab

[GitLab CE](https://about.gitlab.com/) is a self-hosted Git platform with CI/CD.

## pocket-id OIDC setup

### 1. OIDC client in pocket-id

The role registers this client when you run `--tags pocket_id_clients` (see the
README): it creates the `gitlab_users` group and an OIDC client with ID `gitlab`
(`oidc_client_id`, default: the service name) for the `callback_urls`, with PKCE
enabled (`pkce_enabled`), restricted to that group. The play prints the client
secret once — store it in Ansible Vault as `vault_gitlab_oidc_secret`.

To create it by hand in the pocket-id admin UI instead, use:

| Field | Value |
|---|---|
| Name | `gitlab` |
| Client ID | `gitlab` |
| Callback URLs | `https://gitlab.example.com/users/auth/openid_connect/callback` |
| Allowed User Groups | `gitlab_users` |

### 2. Configure GitLab

GitLab OIDC is configured via `gitlab.rb` (Omnibus) or `gitlab.yml` (source).
Using the container, pass configuration via `GITLAB_OMNIBUS_CONFIG`. It is Ruby
evaluated inside the container, so it can read `OIDC_CLIENT_ID` (set by the role from
`oidc_client_id`) and `OIDC_CLIENT_SECRET` (injected from a Podman secret) from the
environment:

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
        identifier: ENV["OIDC_CLIENT_ID"],
        secret: ENV["OIDC_CLIENT_SECRET"],  # injected from the Podman secret
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
      - "gitlab-config:/etc/gitlab"          # named volumes: Podman creates them
      - "gitlab-logs:/var/log/gitlab"
      - "gitlab-data:/var/opt/gitlab"
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
