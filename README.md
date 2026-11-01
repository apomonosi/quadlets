# quadlets

Ansible role for deploying rootless Podman quadlet services behind
[nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) with
[pocket-id](https://pocket-id.org/) OIDC/SSO on RHEL/CentOS Stream 9+.

## Requirements

- RHEL / CentOS Stream 9+ on the target host
- Ansible 2.15+
- A dedicated OS user for rootless Podman (default: `podman`)
- Collections: `ansible.posix`, `community.general`, `containers.podman` (see `requirements.yml`)
- Podman >= 4.7 on the target (EL9 ships 5.x) for idempotent secret updates

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
     │  pocket-id  │  service-a  │ ...  │   ← All on shared `proxy` network
     └──────────────────────────────────┘
           Rootless Podman (user systemd)
```

- **nginx-proxy** terminates SSL using certs in `~/.config/nginx-proxy/certs/`
- **pocket-id** acts as the OIDC provider; each service has its own OIDC client,
  restricted to its own pocket-id user group
- **All containers** join a shared `proxy` Podman network
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
| `quadlets_certs_dir` | `~/.config/nginx-proxy/certs` | TLS certificates for nginx-proxy |
| `quadlets_data_base_dir` | `~/.local/share` | Base directory for service data volumes |

### Network

| Variable | Default | Description |
|---|---|---|
| `quadlets_network_name` | `proxy` | Podman network name |
| `quadlets_network_subnet` | `10.89.0.0/24` | Network CIDR |
| `quadlets_network_gateway` | `10.89.0.1` | Network gateway |

### nginx-proxy

| Variable | Default | Description |
|---|---|---|
| `quadlets_nginx_proxy_enabled` | `true` | Deploy nginx-proxy |
| `quadlets_nginx_proxy_image` | `docker.io/jwilder/nginx-proxy:latest` | Container image |
| `quadlets_nginx_proxy_http_port` | `80` | Host HTTP port |
| `quadlets_nginx_proxy_https_port` | `443` | Host HTTPS port |
| `quadlets_nginx_proxy_https_method` | `redirect` | nginx-proxy `HTTPS_METHOD` |
| `quadlets_nginx_proxy_read_timeout` | `600` | `PROXY_READ_TIMEOUT` (seconds) |
| `quadlets_nginx_proxy_connect_timeout` | `600` | `PROXY_CONNECT_TIMEOUT` (seconds) |
| `quadlets_nginx_proxy_extra_env` | `{}` | Extra env vars for the container |
| `quadlets_nginx_proxy_custom_conf` | `false` | Deploy custom nginx snippet |

Rootless Podman cannot publish ports below `net.ipv4.ip_unprivileged_port_start`
(1024 by default). When the proxy ports are lower, the role lowers that sysctl to the
lowest proxy port (persisted in `/etc/sysctl.d/90-quadlets.conf`). This lets *any*
local user bind ports from that value upwards; use ports >= 1024 and forward 80/443
in the firewall if that is not acceptable. A host that already allows lower ports is
left unchanged.

### pocket-id

| Variable | Default | Description |
|---|---|---|
| `quadlets_pocket_id_enabled` | `true` | Deploy pocket-id |
| `quadlets_pocket_id_image` | `ghcr.io/pocket-id/pocket-id:v2` | Container image |
| `quadlets_pocket_id_virtual_host` | `sso.example.com` | FQDN for pocket-id |
| `quadlets_pocket_id_issuer_url` | `https://{{ quadlets_pocket_id_virtual_host }}` | OIDC issuer URL |
| `quadlets_pocket_id_extra_env` | `{}` | Extra env vars for the container |
| `quadlets_pocket_id_encryption_key` | `""` | **Required.** `ENCRYPTION_KEY`, >= 16 chars (Vault). Never change after first start |
| `quadlets_pocket_id_static_api_key` | `""` | `STATIC_API_KEY`, >= 16 chars (Vault). Enables API automation |
| `quadlets_pocket_id_api_key` | `""` | API key used by Ansible; defaults to the static key when empty |
| `quadlets_pocket_id_provision_clients` | `false` | Allow `--tags pocket_id_clients` to register clients |

pocket-id has no username/password login (passkeys only), so Ansible cannot create
an admin account or mint an API key itself. Instead it injects
`quadlets_pocket_id_static_api_key` as `STATIC_API_KEY`, which pocket-id accepts in
the `X-API-Key` header as an admin credential.

Both keys are stored as Podman secrets (`pocket-id-encryption-key`,
`pocket-id-static-api-key`) in the quadlet user's secret store and injected with
quadlet `Secret=…,type=env`, so they never appear in the unit file or in
`podman inspect`. A changed value updates the secret and restarts pocket-id;
clearing `quadlets_pocket_id_static_api_key` removes its secret.

Each service's `oidc_client_secret` is handled the same way: it is stored as the
Podman secret `<name>-oidc-client-secret` and injected as the environment variable
named by `oidc_client_secret_env` (default `OIDC_CLIENT_SECRET`). Set that to the
variable the app reads (e.g. `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET`) and do **not**
repeat the secret in `env`, which is written to the unit file in plain text.
Apps configured through a single config variable can usually read it from the
environment instead (GitLab: `secret: ENV["OIDC_CLIENT_SECRET"]`).

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
    oidc_client_id: myapp                # OIDC client ID in pocket-id (default: name)
    oidc_client_secret: "{{ vault }}"    # OIDC client secret (Vault-encrypt!), stored
                                         # as a Podman secret, not in the unit file
    oidc_client_secret_env: OIDC_CLIENT_SECRET   # env var the app reads it from
    oidc_issuer_url: ""                  # override issuer (default: pocket-id URL)
    pocket_id_group: myapp_users         # pocket-id group allowed to use this client
    callback_urls:                       # redirect URIs; an OIDC client is registered
      - "https://myapp.example.com/auth/callback"   # only if this is set
    pkce_enabled: true                   # PKCE on the OIDC client (default: true)
    volumes:
      - "/data/myapp:/data:z"
    env:                                 # values may contain spaces; they are quoted
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

# Generate each with: openssl rand -base64 32   (keep them in vault.yml)
quadlets_pocket_id_encryption_key: "{{ vault_pocket_id_encryption_key }}"
quadlets_pocket_id_static_api_key: "{{ vault_pocket_id_static_api_key }}"
quadlets_pocket_id_provision_clients: true

quadlets_services:
  - name: grafana
    image: grafana/grafana:latest
    virtual_host: grafana.mydomain.com
    port: 3000
    oidc_client_id: grafana
    oidc_client_secret: "{{ vault_grafana_oidc_secret }}"
    oidc_client_secret_env: GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET
    pocket_id_group: grafana_users
    callback_urls:
      - "https://grafana.mydomain.com/login/generic_oauth"
    env:
      GF_SERVER_ROOT_URL: "https://grafana.mydomain.com"
      GF_AUTH_GENERIC_OAUTH_ENABLED: "true"
      GF_AUTH_GENERIC_OAUTH_NAME: "Pocket ID"
      GF_AUTH_GENERIC_OAUTH_CLIENT_ID: grafana
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

### 5. Create the pocket-id admin (first run only)

Open `https://sso.mydomain.com/setup` in a browser and register the admin passkey.
Then check that Ansible can reach the API with the static key:

```bash
ansible-playbook site.yml -i inventory/hosts.yml \
  --tags pocket_id_bootstrap -e @vault.yml --ask-vault-pass
```

### 6. Register OIDC clients and user groups

```bash
ansible-playbook site.yml -i inventory/hosts.yml \
  --tags pocket_id_clients -e @vault.yml --ask-vault-pass
```

For every service with `callback_urls`, this creates (if missing) the
`pocket_id_group` user group and an OIDC client whose ID is `oidc_client_id`
(default: the service name), then restricts the client to that group. Re-running is
safe: existing groups and clients are reused and the group link is re-applied.

pocket-id generates the client secret and returns it **only once**, so the play
prints it for each newly created client. Store it in your vault (e.g.
`vault_grafana_oidc_secret`) and re-run step 4 so the service picks it up. Add users
to the group in the pocket-id UI to grant them access.

## Ansible tags

| Tag | Description |
|---|---|
| `infrastructure` | Deploy network, nginx-proxy, pocket-id, and certificates |
| `services` | Deploy service quadlets only |
| `nginx_proxy` | Deploy/update nginx-proxy only |
| `pocket_id` | Deploy/update pocket-id only |
| `certificates` | Provision/copy TLS certificates only |
| `pocket_id_bootstrap` | Verify pocket-id API access with the configured key (only runs when tagged) |
| `pocket_id_clients` | Register OIDC clients and user groups in pocket-id (only runs when tagged) |

Units are installed with `WantedBy=default.target` in the quadlet user's systemd
instance; with lingering enabled they start at boot without anyone logging in.

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

MIT

## Author Information

L. M. Jensen (melwyn@dtu.dk)
