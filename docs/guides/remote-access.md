---
title: 'Remote Access: SSH, xRDP, and Tailscale'
slug: remote-access
category: guides
tags:
- remote
- ssh
- tailscale
- security
- xrdp
sources:
- sessions/2026-03-16-remote-access-xrdp-ssh-filezilla.md
- sessions/2026-03-23-tailscale-security-deployment.md
last_updated: '2026-03-30'
version: 2
---

# Remote Access: SSH, xRDP, and Tailscale

If your OpenClaw server is a headless mini PC, you'll need remote access for administration. This guide covers three approaches, from basic to production-grade.

## Option 1: SSH (Recommended Baseline)

SSH is the most reliable remote access method. It works consistently and uses minimal resources.

### Setup

```bash
# On the OpenClaw server
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Connect from another machine

```bash
ssh user@server-ip
```

### File Transfer with SFTP

Use FileZilla or any SFTP client:

- **Protocol**: SFTP (not FTP — this is a common mistake)
- **Host**: your server's IP
- **Port**: 22
- **Username**: your Ubuntu username

> ⚠️ **Common error**: `Cannot establish FTP connection to an SFTP server` — change the protocol from FTP to SFTP in your client settings.

> ⚠️ **Username is case-sensitive**: `kos` and `Kos` are different users on Linux. Verify with `whoami` on the server.

### Test Connectivity

```bash
# From your local machine
nc -zv server-ip 22     # SSH
nc -zv server-ip 3389   # xRDP (if configured)
```

## Option 2: xRDP (Desktop Remote Access)

xRDP provides a graphical desktop, useful for browser-based tasks (OAuth flows, dashboard access).

### Setup

```bash
sudo apt install xrdp ubuntu-desktop -y
sudo systemctl enable xrdp

# Create session config
echo "gnome-session" > ~/.xsession
chmod +x ~/.xsession

# Add user to ssl-cert group
sudo adduser $USER ssl-cert

sudo systemctl restart xrdp
sudo reboot
```

### Connect

Use **Microsoft Remote Desktop** (macOS/Windows):
- PC name: `server-ip`
- Session type: **Xorg** (not Xvnc)

> ⚠️ **Stability warning**: xRDP on Ubuntu can be unstable — connections drop intermittently. For reliable remote access, prefer SSH for CLI work and Tailscale for everything else.

## Option 3: Tailscale (Production-Grade)

Tailscale creates a private WireGuard mesh network. This is the **single highest-impact security improvement** you can make to your OpenClaw deployment.

### Why Tailscale?

- Makes your OpenClaw server **invisible to the internet** — port scanners find nothing
- No firewall rules to manage — everything is denied by default
- Works across networks (home, office, mobile) without port forwarding
- Messaging platforms (Telegram, Discord) use outbound WebSocket connections, so no inbound ports are needed

> ⚠️ **Context**: In February 2026, a security scan found 42,665 OpenClaw instances exposed on the public internet, with 93% vulnerable to authentication bypass. Tailscale eliminates this entire attack surface.

### Setup

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

### Expose OpenClaw via Tailscale Serve

```bash
# Bind OpenClaw to loopback only (not accessible from LAN)
# Then expose via Tailscale Serve
tailscale serve --bg 8080
```

> ⚠️ **Important**: Use `tailscale serve`, **never** `tailscale funnel`. Funnel exposes your service to the public internet — the opposite of what you want.

### Access from other devices

Install Tailscale on your phone/laptop and access the server via its Tailscale IP or hostname:

```bash
ssh user@<tailnet_host>
```

## Security Architecture

For production deployments, consider layering these approaches:

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Network | Tailscale | Private mesh, zero exposed ports |
| Runtime | Docker double-network | Isolate gateway from internet |
| Egress | Squid proxy with allowlist | Block data exfiltration to unauthorized domains |
| API Keys | LiteLLM proxy | Agents never see real API keys |

### Docker Double-Network Pattern

Two Docker networks:
- `openclaw-internal`: **no internet access**, runs the gateway
- `openclaw-external`: internet access through a proxy that enforces a domain allowlist

Only approved domains (e.g., `api.anthropic.com`, `telegram.org`, `github.com`) can be reached. This directly mitigates the data exfiltration attacks documented by Cisco.

### Tool Audit Commands

```bash
openclaw security audit     # Flag insecure settings
openclaw sandbox explain    # Show effective sandbox policy per session
```

## What's Next

- [1Password Secrets Management](1password-secrets-management.md) — Protect credentials
- [Multi-Agent Architecture](../concepts/multi-agent-architecture.md) — Secure multi-agent design
