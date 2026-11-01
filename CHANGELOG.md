# Changelog

All notable changes to this role are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Breaking changes

- `quadlets_pocket_id_encryption_key` is now **required** when pocket-id is enabled
  (at least 16 characters). pocket-id v2 refuses to start without it. Generate it with
  `openssl rand -base64 32`, keep it in Ansible Vault and never change it afterwards.
- New collection dependency `containers.podman >= 1.18.0`. Run
  `ansible-galaxy collection install -r requirements.yml` again.
- Secrets passed through `env` stay in the unit file in plain text. Move OIDC client
  secrets to `oidc_client_secret` + `oidc_client_secret_env`, and other passwords and
  tokens to the new `secrets:` map. The generic `OIDC_CLIENT_SECRET` variable is now set
  only when `oidc_client_secret_env` is left at its default.
- OIDC clients are created with the client ID `oidc_client_id` (default: the service
  name). Clients created earlier with a generated ID are not migrated.
- Units are installed with `WantedBy=default.target` instead of `multi-user.target`.
- `tests/test.yml` refers to the role by name; run it with `ANSIBLE_ROLES_PATH=..` from
  a checkout named `quadlets`.

### Added

- Podman secrets: the pocket-id encryption key and static API key, each service's
  `oidc_client_secret`, and the new `secrets:` map are stored in the quadlet user's
  Podman secret store and injected with `Secret=…,type=env`. They no longer appear in
  unit files or in `podman inspect`. A secret is recreated, and its container
  restarted, only when the value changes (podman >= 4.7).
- Per-service `secrets:` map (`VAR: value`) for database passwords, admin tokens and
  other sensitive variables. Each entry becomes the Podman secret `<name>-env-<var>`;
  entries removed from the map have their secret deleted.
- Per-service `oidc_client_secret_env`: the variable the app reads its client secret
  from (default `OIDC_CLIENT_SECRET`).
- Per-service `pkce_enabled` (default `true`) for the registered OIDC client.
- Internal-only services: `virtual_host` is optional. An entry without it gets no
  nginx-proxy routing, no certificate and no `OIDC_ISSUER_URL`, but stays on the `proxy`
  network where other containers reach it by name. Use this for databases and caches,
  together with `depends_on`.
- Molecule `verify.yml` for the `el9` scenario. It checks:
  - systemd: lingering is on, the units are active and wanted by `default.target`, and
    the unprivileged-port setting allows the proxy ports;
  - secrets: they exist, reach the containers, and are absent from unit files and
    `podman inspect`;
  - environment: values with spaces arrive intact;
  - internal services: they carry no proxy routing and are reachable by name;
  - HTTPS: trusted through nginx-proxy with the mkcert CA, and plain HTTP redirects;
  - provisioning: the Grafana OIDC client exists with the expected settings and group.

### Changed

- The default pocket-id image is `ghcr.io/pocket-id/pocket-id:v2`.
- When the proxy ports are below 1024, the role lowers
  `net.ipv4.ip_unprivileged_port_start` to the lowest proxy port (persisted in
  `/etc/sysctl.d/90-quadlets.conf`). It never raises a value that is already lower.
- OIDC clients are registered only for services that define `callback_urls`.
- `port` is required only together with `virtual_host`.
- SELinux booleans are set only when SELinux is enabled, instead of ignoring errors.
- `update-ca-trust` runs as a handler. Lingering is enabled with `creates:` on the
  linger file instead of a separate status check.
- The service docs describe automated client registration. Their examples use fixed
  client IDs, the role-created data directory (`%h/.local/share/<name>`) or named
  volumes, `secrets:` for passwords, and internal entries for databases.

### Fixed

- pocket-id v2 failed to start because `ENCRYPTION_KEY` was never set.
- Rootless nginx-proxy could not publish ports 80/443.
- Client provisioning failed on pocket-id's paginated list responses. It now reads
  every page, 100 items at a time.
- The client ID registered in pocket-id did not match the one the service was
  configured with.
- `Environment=` values containing spaces were split by systemd. For example, OIDC
  scopes `openid profile email` became just `openid`.
- Multi-line `env` values, such as `GITLAB_OMNIBUS_CONFIG`, broke the unit file.
- Units did not start at boot (`multi-user.target` does not exist in the user manager).
- Running with `--tags` (`pocket_id_clients`, `services`, `certificates`, …) ran none of
  the tasks inside the selected files.
- The volume paths in the service docs rendered as `/<name>`.
- Molecule `prepare.yml` failed:
  - its `replace` task used an unsupported option;
  - it looked up repo files on the controller instead of the VM;
  - it did not run as root;
  - it skipped `rocky.repo`.
- A failed service-entry validation printed the whole entry, including secret values.
- All `ansible-lint` findings; the role passes the `production` profile.

### Removed

- The unused `quadlets_pocket_id_bootstrap` variable.
