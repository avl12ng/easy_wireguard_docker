# WireGuard Easy (wg-easy) via Docker Compose

Quick and simple deployment of a **WireGuard** VPN server with a web admin interface, powered by [wg-easy](https://github.com/wg-easy/wg-easy).

![Docker](https://img.shields.io/badge/docker-compose-blue?logo=docker)
![WireGuard](https://img.shields.io/badge/WireGuard-VPN-88171A?logo=wireguard)
![License](https://img.shields.io/badge/license-MIT-green)

🇫🇷 [Version française disponible ici](./README.md)

## 📋 Table of Contents

- [Overview](#-overview)
- [Prerequisites](#-prerequisites)
- [Installation](#-installation)
- [Configuration](#-configuration)
- [Opening Ports](#-opening-ports)
- [Usage](#-usage)
- [Security](#-security)
- [Useful Commands](#-useful-commands)
- [Troubleshooting](#-troubleshooting)
- [License](#-license)

## 🎯 Overview

This repository provides a ready-to-use `docker-compose` configuration to deploy **wg-easy**, a web-based layer on top of WireGuard that lets you:

- Create and manage VPN clients through a graphical interface
- Generate configuration QR codes for mobile devices
- Monitor active connections
- Administer everything without touching the command line

## ✅ Prerequisites

- A Linux server with [Docker](https://docs.docker.com/engine/install/) and [Docker Compose](https://docs.docker.com/compose/install/) installed
- A fixed public IP address or a domain name (FQDN) pointing to the server
- Administrator access to your router/box for port forwarding
- The `wireguard` kernel module available on the host (usually included in Linux kernels ≥ 5.6)

## 🚀 Installation

### 1. Clone the repository

```bash
git clone https://github.com/avl12ng/easy_wireguard_docker.git
cd easy_wireguard_docker
```

### 2. Generate the password hash

The web interface is protected by a password, which must be provided as a **bcrypt** hash, generated with the official image:

```bash
docker run --rm ghcr.io/wg-easy/wg-easy wgpw 'Your_Strong_Password'
```

The command returns a line like:

```
PASSWORD_HASH='$2a$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```

> ⚠️ Remember to escape the `$` characters if you copy this hash outside of single quotes in your `.env` file.

### 3. Configure the `.env` file

Create a `.env` file at the root of the project:

```env
WG_PUBLIC_IP=your_public_ip_or_fqdn
WG_PASSWORD_HASH='$2a$12$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
```

| Variable | Description |
|---|---|
| `WG_PUBLIC_IP` | Public IP address or domain name that VPN clients will use to reach the server |
| `WG_PASSWORD_HASH` | Bcrypt hash of the password used to access the web interface (generated in the previous step) |

### 4. Start the containers

```bash
docker compose up -d
```

Check that the container starts correctly:

```bash
docker compose logs -f wg-easy
```

## ⚙️ Configuration

Excerpt from the `docker-compose.yml` file:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    environment:
      - WG_HOST=${WG_PUBLIC_IP}
      - PASSWORD_HASH=${WG_PASSWORD_HASH}
      - PORT=51821
      - WG_PORT=51820
    volumes:
      - ./wg-easy-data:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

### Parameter details

- **`WG_HOST`**: public address announced to clients (from `WG_PUBLIC_IP`)
- **`PASSWORD_HASH`**: bcrypt hash used for web interface authentication
- **`PORT`**: TCP port for the web admin interface (default `51821`)
- **`WG_PORT`**: UDP port used by the WireGuard protocol (default `51820`)
- **`volumes`**: persists WireGuard configuration and client data in `./wg-easy-data`
- **`cap_add`**: required Linux capabilities (`NET_ADMIN` for network management, `SYS_MODULE` to load the WireGuard kernel module if needed)
- **`sysctls`**: enables IP forwarding, required to route VPN traffic

## 🌐 Opening Ports

On your router/internet box, forward the following port to the local IP of the server running Docker:

| Port | Protocol | Direction |
|---|---|---|
| `51820` | UDP | Inbound (required, WireGuard traffic) |
| `51821` | TCP | Inbound (optional, only if you administer the interface from outside) |

> 🔒 It is recommended **not to expose the web interface port (51821) directly to the internet**, but rather to restrict it to local access or place it behind a reverse proxy with additional authentication / a management VPN.

## 🖥️ Usage

1. Go to `http://<ip_or_domain>:51821`
2. Log in with the password you set earlier
3. Click **New Client** to create a VPN profile
4. Scan the QR code with the WireGuard mobile app, or download the `.conf` file for a desktop client

## 🔐 Security

- Use a strong, unique password for `PASSWORD_HASH`
- **Never** commit your `.env` file to GitHub (add it to `.gitignore`)
- Restrict exposure of the web interface (port 51821) to a trusted network when possible
- Keep the Docker image regularly updated
- Back up the `wg-easy-data` folder regularly (it contains clients' private keys)

Recommended `.gitignore`:

```gitignore
.env
wg-easy-data/
```

## 🛠️ Useful Commands

```bash
# Start the services
docker compose up -d

# Stop the services
docker compose down

# View live logs
docker compose logs -f

# Update the image
docker compose pull
docker compose up -d

# Regenerate a password hash
docker run --rm ghcr.io/wg-easy/wg-easy wgpw 'NewPassword'
```

## 🩺 Troubleshooting

| Issue | Suggested fix |
|---|---|
| Web interface unreachable | Check that port `51821/tcp` is properly forwarded and the container is running (`docker compose ps`) |
| Clients can't connect | Check port `51820/udp` forwarding and that `WG_HOST` matches the correct public IP/domain |
| No internet traffic through the VPN | Verify that `net.ipv4.ip_forward=1` is active and that the host's NAT/iptables rules aren't blocking traffic |
| WireGuard kernel module error | Make sure the host kernel natively supports WireGuard, or install `wireguard-tools` on the host |

## 📄 License

This configuration project is distributed under the MIT license. wg-easy itself is developed by the [wg-easy](https://github.com/wg-easy/wg-easy) community under its own license.

---

*Based on the official [ghcr.io/wg-easy/wg-easy](https://github.com/wg-easy/wg-easy) image*
