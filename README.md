<div align="center">

# 🏠 Self-Hosted Home Lab

**A modular, private, HTTPS-enabled home lab running on Debian Headless.**
Docker Compose · Caddy · AdGuard Home · Tailscale

![Debian](https://img.shields.io/badge/Debian-Headless-A81D33?logo=debian&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Caddy](https://img.shields.io/badge/Caddy-Reverse%20Proxy-1F88C0?logo=caddy&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-Mesh%20VPN-242424?logo=tailscale&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

</div>

---

## 📖 Overview

This repository contains the container orchestration files, reverse proxy rules, and local DNS configuration for a personal home lab. It provides workflow automation, stack management, system monitoring, file management, and a central dashboard, all reachable through clean HTTPS domains such as `n8n.lab.local`.

**Nothing is exposed to the public internet.** Access is restricted to devices on your Tailscale network.

### ✨ Features

- 🔐 **Private by design**: accessible only through a Tailscale mesh VPN
- 🌐 **Custom domains**: `*.lab.local` resolved locally by AdGuard Home
- 🔒 **Automatic HTTPS**: Caddy terminates TLS with its internal Certificate Authority
- 🛡️ **Network-wide ad blocking**: DNS-level filtering for every connected device
- 🧩 **Modular**: add a service with one Compose block and one Caddy entry
- 📊 **Observable**: live system metrics and a central launcher dashboard

---

## 🧰 Tech Stack

| Service | Category | Purpose | Address |
| :--- | :--- | :--- | :--- |
| [Tailscale](https://tailscale.com/) | Networking | Secure mesh VPN for remote access | Mesh IP |
| [AdGuard Home](https://adguard.com/en/adguard-home/overview.html) | DNS / Security | Ad blocking and local DNS rewrites | `adguard.lab.local` · port `53` |
| [Caddy](https://caddyserver.com/) | Reverse proxy | HTTPS termination and domain routing | ports `80`, `443` |
| [Homarr](https://homarr.dev/) | Dashboard | Central homepage and app launcher | `dashboard.lab.local` |
| [Dockge](https://dockge.kuma.pet/) | Stack manager | Web UI for Docker Compose stacks | `dockge.lab.local` |
| [n8n](https://n8n.io/) | Automation | Workflow engine and Telegram bot host | `n8n.lab.local` |
| [Glances](https://nicolargo.github.io/glances/) | Monitoring | CPU, RAM, disk and network metrics | `glances.lab.local` |
| [File Browser](https://filebrowser.org/) | Storage | Web interface for server files | `files.lab.local` |

---

## 🧭 Architecture

Name resolution and encryption are both handled locally:

1. **DNS rewrite (AdGuard Home):** any query for `*.lab.local` is answered with the Debian host's Tailscale IP.
2. **TLS termination (Caddy):** Caddy serves HTTPS using its internal CA (`tls internal`) and routes each hostname to the matching container over the Docker network.

```text
 ┌────────────────┐
 │  Client Device │
 └───────┬────────┘
         │  Tailscale VPN
         ▼
 ┌────────────────┐   resolves *.lab.local → Tailscale IP
 │  AdGuard Home  │
 └───────┬────────┘
         ▼
 ┌────────────────┐   HTTPS / TLS termination
 │     Caddy      │
 └───────┬────────┘
         │
   ┌─────┴───────┬────────────┬────────────┬─────────────┐
   ▼             ▼            ▼            ▼             ▼
 Homarr        Dockge        n8n        Glances    File Browser
```

---

## 📂 Repository Structure

```text
.
├── docker-compose.yml       # Primary stack definition
├── caddy/
│   └── Caddyfile            # Reverse proxy and domain routing rules
├── adguard_data/            # AdGuard Home configuration (persistent)
├── adguard_work/            # AdGuard query logs and filter data
├── dashboard_data/          # Homarr configuration and icons
├── dockge_data/             # Dockge database and stacks
├── filebrowser_data/        # File Browser configuration and database
└── n8n_data/                # n8n workflows and credentials
```

---

## ✅ Prerequisites

- A Debian (or compatible) server with **Docker** and **Docker Compose v2**
- A **Tailscale** account, with Tailscale installed on the server and on every client
- Free ports `53` (DNS), `80` and `443` on the host

> **Note:** if port `53` is already taken (for example by `systemd-resolved`), free it before starting AdGuard Home.

---

## 🚀 Quick Start

**1. Clone the repository**

```bash
git clone https://github.com/Fares-dev-Bit/home-lab-.git
cd home-lab-
```

**2. Start the stack**

```bash
docker compose up -d
```

**3. Verify the containers**

```bash
docker compose ps
```

**4. Complete the configuration** in the sections below (DNS rewrite, then trust the CA).

---

## ⚙️ Configuration

### 1. AdGuard Home: DNS rewrite

In the AdGuard Home interface, open **Filters → DNS rewrites** and add:

| Domain | Answer |
| :--- | :--- |
| `*.lab.local` | `100.x.y.z` *(the Debian host's Tailscale IP)* |

Then set **AdGuard Home as the DNS server in your Tailscale admin console** (DNS → Nameservers → *Override local DNS*), so every client uses it automatically.

### 2. Caddy: reverse proxy rules

`caddy/Caddyfile`:

```caddyfile
dashboard.lab.local {
    tls internal
    reverse_proxy dashboard:7575
}

dockge.lab.local {
    tls internal
    reverse_proxy dockge:5001
}

n8n.lab.local {
    tls internal
    reverse_proxy n8n:5678
}

glances.lab.local {
    tls internal
    reverse_proxy glances:61208
}

files.lab.local {
    tls internal
    reverse_proxy filebrowser:80
}

adguard.lab.local {
    tls internal
    reverse_proxy adguardhome:80
}
```

> **AdGuard port:** port `3000` is only used by the first-run setup wizard. After setup, the web interface listens on the port you selected (port `80` by default). Adjust the last block if you chose differently.

Reload Caddy after any change, without downtime:

```bash
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### 3. Trust Caddy's root certificate

Because `tls internal` uses a private CA, browsers show a warning until each device trusts its root certificate.

```bash
docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt ./caddy-root.crt
```

Import `caddy-root.crt` as a trusted root CA on each device (system keychain, browser, or phone certificate settings).

---

## 🛠️ Troubleshooting

| Symptom | Likely cause | Fix |
| :--- | :--- | :--- |
| `lab.local` names do not resolve | Client is not using AdGuard | Check Tailscale DNS settings and that the client is connected |
| AdGuard container fails to start | Port `53` already in use | Disable the DNS stub listener of `systemd-resolved` |
| Browser shows a certificate warning | Caddy root CA not trusted | Install `caddy-root.crt` on the device |
| `502 Bad Gateway` | Wrong container name or port in the Caddyfile | Verify with `docker compose ps` and the service's internal port |

---

## 🔐 Security Notes

- Do **not** forward ports `53`, `80` or `443` on your router. Access should stay Tailscale-only.
- Never commit secrets (`.env` files, API keys, n8n credentials). Add `*_data/` directories to `.gitignore` if they contain sensitive data.
- Restrict access with Tailscale ACLs if the network is shared with other people.
- The `.local` suffix is reserved for mDNS (RFC 6762) and can occasionally conflict with it. If you encounter odd resolution behaviour, switch to `home.arpa`, the suffix reserved for home networks (RFC 8375).

---

## 🗺️ Roadmap

- [ ] Automated backups of the `*_data/` directories
- [ ] Uptime monitoring and alerting
- [ ] Centralised log aggregation
- [ ] Tailscale ACL hardening

---

## 🤝 Contributing

Suggestions and improvements are welcome. Open an issue or submit a pull request.

---

## 📜 License

Released under the [MIT License](LICENSE). Free to use and adapt for personal home labs.
