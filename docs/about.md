# About

This Ansible role deploys a complete self-hosted service stack using rootless
[Podman quadlets](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
on RHEL/CentOS Stream 9+.

## Architecture

```
                  ┌─────────────────────────────────────────────────────┐
Internet ─────────► nginx-proxy (port 80/443)                           │
                  │   ↑ docker-gen auto-discovers containers via Podman │
                  │   ↑ socket and generates vhost configs              │
                  │                                                     │
                  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │
                  │  │  pocket-id   │  │  nextcloud   │  │ grafana  │  │
                  │  │  (SSO/OIDC)  │  │              │  │          │  │
                  │  └──────────────┘  └──────────────┘  └──────────┘  │
                  │                                                     │
                  │         All containers on shared proxy-net          │
                  └─────────────────────────────────────────────────────┘
                                Rootless Podman (user systemd)
```

## Components

| Component | Description |
|---|---|
| **nginx-proxy** | Reverse proxy with automatic vhost generation via docker-gen |
| **pocket-id** | Lightweight OIDC provider — the SSO frontend for all services |
| **Services** | Any container with `VIRTUAL_HOST` set is auto-proxied and can use pocket-id for SSO |

## Security model

- All containers run as a non-root `podman` user (rootless Podman)
- SSL terminates at nginx-proxy; inter-container traffic is HTTP on the private `proxy-net` network
- Each service has its own OIDC client registration and user group in pocket-id
- Secrets (OIDC client secrets, admin passwords) are stored in Ansible Vault

## Technology stack

- [Podman](https://podman.io/) + [quadlets](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) — systemd-native container management
- [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) — automatic reverse proxy
- [pocket-id](https://pocket-id.org/) — OIDC identity provider
- [mkcert](https://github.com/FiloSottile/mkcert) — locally-trusted certificates for development
- [Ansible](https://docs.ansible.com/) — orchestration and configuration management
