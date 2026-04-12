# quadlets

Ansible role for deploying rootless Podman quadlet services behind
[nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) with
[pocket-id](https://pocket-id.org/) OIDC/SSO on RHEL/CentOS Stream 9+.

## Requirements

- RHEL / CentOS Stream 9+ on the target host
- Ansible 2.15+
- A dedicated OS user for rootless Podman (default: `podman`)
- Collections: `ansible.posix`, `community.general` (see `requirements.yml`)

Install collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Architecture

```
Internet → nginx-proxy (80/443)
             ↑ docker-gen watches Podman socket → auto-generates vhosts
             │
     ┌───────┴──────────────────────────┐
     │  pocket-id  │  service-a  │ ...  │   ← All on shared proxy-net
     └──────────────────────────────────┘
           Rootless Podman (user systemd)
```

- **nginx-proxy** terminates SSL using certs in `~/.config/containers/certs/`
- **pocket-id** acts as the OIDC provider; each service has its own client registration
- **All containers** join a shared `proxy-net` Podman network
- **docker-gen** discovers containers by reading `VIRTUAL_HOST` environment variables via the Podman socket

## Role Variables

### Identity

| Variable | Default | Description |
|---|---|---|
| `quadlets_user` | `podman` | OS user owning all rootless units |
| `quadlets_group` | `podman` | Primary group of the quadlet user |
| `quadlets_uid` | `""` | UID — auto-detected if empty |

### Paths (auto-derived from `quadlets_user` home)

| Variable | Default | Description |
|---|---|---|
| `quadlets_systemd_dir` | `~/.config/containers/systemd` | Where quadlet unit files are deployed |
| `quadlets_certs_dir` | `~/.config/containers/certs` | TLS certificates for nginx-proxy |
| `quadlets_data_base_dir` | `~/.local/share` | Base directory for service data volumes |

### Network

| Variable | Default | Description |
|---|---|---|
| `quadlets_network_name` | `proxy-net` | Podman network name |
| `quadlets_network_subnet` | `10.89.0.0/24` | Network CIDR |
| `quadlets_network_gateway` | `10.89.0.1` | Network gateway |

### nginx-proxy

| Variable | Default | Description |
|---|---|---|
| `quadlets_nginx_proxy_enabled` | `true` | Deploy nginx-proxy |
| `quadlets_nginx_proxy_image` | `nginxproxy/nginx-proxy:latest` | Container image |
| `quadlets_nginx_proxy_http_port` | `80` | Host HTTP port |
| `quadlets_nginx_proxy_https_port` | `443` | Host HTTPS port |
| `quadlets_nginx_proxy_extra_env` | `{}` | Extra env vars for the container |
| `quadlets_nginx_proxy_custom_conf` | `false` | Deploy custom nginx snippet |

### pocket-id

| Variable | Default | Description |
|---|---|---|
| `quadlets_pocket_id_enabled` | `true` | Deploy pocket-id |
| `quadlets_pocket_id_image` | `ghcr.io/stonith404/pocket-id:latest` | Container image |
| `quadlets_pocket_id_virtual_host` | `sso.example.com` | FQDN for pocket-id |
| `quadlets_pocket_id_issuer_url` | `https://{{ pocket_id_virtual_host }}` | OIDC issuer URL |
| `quadlets_pocket_id_api_key` | `""` | API key (from bootstrap, store in vault) |
| `quadlets_pocket_id_provision_clients` | `false` | Auto-register OIDC clients |
| `quadlets_pocket_id_bootstrap` | `false` | Run first-time admin setup |

### Certificates

| Variable | Default | Description |
|---|---|---|
| `quadlets_cert_mode` | `mkcert` | `mkcert` or `byo` |
| `quadlets_mkcert_version` | `v1.4.4` | mkcert release to download |
| `quadlets_byo_cert_src` | `""` | Global BYO cert path (controller-side) |
| `quadlets_byo_key_src` | `""` | Global BYO key path (controller-side) |

### Services

`quadlets_services` is a list of dicts. Full schema:

```yaml
quadlets_services:
  - name: myapp                          # required — used as the systemd unit name
    image: myapp:latest                  # required — container image
    virtual_host: myapp.example.com      # required — FQDN routed by nginx-proxy
    port: 8080                           # required — container port
    oidc_client_id: myapp                # OIDC client ID in pocket-id
    oidc_client_secret: "{{ vault }}"    # OIDC client secret (Vault-encrypt!)
    oidc_issuer_url: ""                  # override issuer (default: pocket-id URL)
    pocket_id_group: myapp_users         # pocket-id user group name
    callback_urls:                       # redirect URIs for OIDC client registration
      - "https://myapp.example.com/auth/callback"
    volumes:
      - "/data/myapp:/data:z"
    env:
      MY_ENV_VAR: value
    cert_mode: mkcert                    # per-service cert mode override
    byo_cert_src: ""                     # per-service BYO cert path
    byo_key_src: ""                      # per-service BYO key path
    depends_on: []                       # other unit names → After=/Requires=
    extra_labels: {}                     # additional container labels
```

## Quick start

### 1. Install requirements

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Create your inventory and playbook

```yaml
# inventory/hosts.yml
all:
  hosts:
    myserver:
      ansible_host: 192.168.1.10
      ansible_user: root
```

```yaml
# site.yml
- hosts: all
  roles:
    - quadlets
```

### 3. Configure variables

```yaml
# group_vars/all.yml
quadlets_user: podman
quadlets_pocket_id_virtual_host: sso.mydomain.com
quadlets_cert_mode: mkcert   # use 'byo' for production

quadlets_services:
  - name: grafana
    image: grafana/grafana:latest
    virtual_host: grafana.mydomain.com
    port: 3000
    oidc_client_id: grafana
    oidc_client_secret: "{{ vault_grafana_oidc_secret }}"
    pocket_id_group: grafana_users
    callback_urls:
      - "https://grafana.mydomain.com/login/generic_oauth"
    env:
      GF_SERVER_ROOT_URL: "https://grafana.mydomain.com"
      GF_AUTH_GENERIC_OAUTH_ENABLED: "true"
      GF_AUTH_GENERIC_OAUTH_NAME: "Pocket ID"
      GF_AUTH_GENERIC_OAUTH_CLIENT_ID: grafana
      GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET: "{{ vault_grafana_oidc_secret }}"
      GF_AUTH_GENERIC_OAUTH_SCOPES: "openid profile email"
      GF_AUTH_GENERIC_OAUTH_AUTH_URL: "https://sso.mydomain.com/authorize"
      GF_AUTH_GENERIC_OAUTH_TOKEN_URL: "https://sso.mydomain.com/api/oidc/token"
      GF_AUTH_GENERIC_OAUTH_API_URL: "https://sso.mydomain.com/api/oidc/userinfo"
      GF_AUTH_GENERIC_OAUTH_USE_PKCE: "true"
```

### 4. Deploy the stack

```bash
ansible-playbook site.yml -i inventory/hosts.yml
```

### 5. Bootstrap pocket-id (first run only)

```bash
ansible-playbook site.yml -i inventory/hosts.yml \
  --tags pocket_id_bootstrap \
  -e quadlets_pocket_id_bootstrap=true \
  -e quadlets_pocket_id_admin_email=admin@mydomain.com \
  -e @vault.yml --ask-vault-pass
```

Save the printed API token in your vault as `quadlets_pocket_id_api_key`.

### 6. Register OIDC clients (optional automation)

```bash
ansible-playbook site.yml -i inventory/hosts.yml \
  --tags pocket_id_clients \
  -e quadlets_pocket_id_provision_clients=true \
  -e @vault.yml --ask-vault-pass
```

## Ansible tags

| Tag | Description |
|---|---|
| `infrastructure` | Deploy network, nginx-proxy, pocket-id, and certificates |
| `services` | Deploy service quadlets only |
| `nginx_proxy` | Deploy/update nginx-proxy only |
| `pocket_id` | Deploy/update pocket-id only |
| `certificates` | Provision/copy TLS certificates only |
| `pocket_id_bootstrap` | First-run admin account setup (use with `--tags`) |
| `pocket_id_clients` | Register OIDC clients in pocket-id (use with `--tags`) |

## Supported services

See the `docs/services/` directory for per-service OIDC configuration guides:

- [Audiobookshelf](docs/services/audiobookshelf.md)
- [BookStack](docs/services/bookstack.md)
- [GitLab](docs/services/gitlab.md)
- [Grafana](docs/services/grafana.md)
- [Immich](docs/services/immich.md)
- [Komodo](docs/services/komodo.md)
- [Open WebUI](docs/services/openwebui.md)
- [Vaultwarden](docs/services/vaultwarden.md)

## Dependencies

None. Required Ansible collections are in `requirements.yml`.

## License

GPL-2.0-or-later

## Author Information

Lars Melwyn Jensen
