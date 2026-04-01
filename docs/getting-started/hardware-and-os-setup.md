---
title: Hardware and OS Setup for OpenClaw
slug: hardware-and-os-setup
category: getting-started
tags:
- setup
- hardware
- ubuntu
- installation
sources:
- sessions/2026-03-12-ubuntu-usb-setup-for-openclaw.md
- sessions/2026-03-13-acemagic-ubuntu-openclaw-install.md
last_updated: '2026-03-30'
version: 2
---

# Hardware and OS Setup for OpenClaw

This guide covers setting up a dedicated mini PC with Ubuntu as an always-on OpenClaw server. While OpenClaw runs on any Linux/macOS machine, a dedicated device ensures 24/7 availability for messaging bots and scheduled tasks.

## Hardware Requirements

| Spec | Minimum | Recommended |
|------|---------|-------------|
| RAM | 4 GB (1 GB for gateway alone) | 8–16 GB |
| Storage | 32 GB SSD | 128 GB+ SSD |
| Network | Ethernet or WiFi | Ethernet (more reliable) |
| CPU | Any x86_64 | Intel N-series or AMD Ryzen embedded |

**Cloud-only mode** (no local models): Even an entry-level mini PC with 8 GB RAM is sufficient — the heavy computation happens on Anthropic/OpenAI servers.

**Local models with Ollama**: You need 16–32 GB RAM depending on model size.

> ⚠️ **Tested hardware**: AceMagic mini PC with 16 GB RAM, Intel processor. The MediaTek MT7902 WiFi chip has **no Linux driver** until kernel 7.1 — use Ethernet or a USB WiFi dongle (TP-Link TL-WN725N or Archer T3U).

## Step 1: Create a Bootable USB Drive

1. Download [Ubuntu 24.04 LTS](https://ubuntu.com/download/desktop) ISO (~5–6 GB)
2. Download [balenaEtcher](https://etcher.balena.io) (works on macOS, Windows, Linux)
3. Insert a USB drive (8 GB minimum — **contents will be erased**)
4. In balenaEtcher: **Flash from file** → select ISO → select USB → **Flash!**

This takes about 10 minutes.

## Step 2: Install Ubuntu

1. Insert the USB into the mini PC and power on
2. Press the boot menu key repeatedly during startup:

| Brand | Common Keys |
|-------|------------|
| AceMagic | **F7** (also try F11, F12, Esc) |
| Generic | F2, F10, F12, Del, Esc |

3. Select the USB drive as boot device
4. Choose **"Install Ubuntu"**
5. Recommended options:
   - **Normal installation** (not minimal)
   - Check **"Install third-party software"** — this installs necessary hardware drivers
   - **"Erase disk and install Ubuntu"** — safe for a dedicated OpenClaw machine
6. Set username, password, and timezone

> ⚠️ **AceMagic gotcha**: Windows Autopilot may appear instead of the boot menu. If this happens, press `Shift+F10` to open a command prompt, then type `wpeutil shutdown` to force shutdown and retry.

> ⚠️ **Install target**: Verify the installation targets the internal SSD (`sda` or `nvme0n1`), **not** the USB drive you booted from.

Installation takes about 15 minutes.

## Step 3: Post-Install System Setup

```bash
# Update everything
sudo apt update && sudo apt upgrade -y

# Install base dependencies
sudo apt install -y curl git build-essential

# Verify internet connectivity
ping -c 4 google.com
```

If WiFi is not available (e.g., unsupported chipset), connect via Ethernet — it works immediately with no driver installation.

## Step 4: Install Node.js 22+

Node.js 22+ is a **hard requirement** for the OpenClaw gateway — not optional.

```bash
# Install NVM (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Reload shell profile
source ~/.bashrc

# Install Node.js 22 LTS
nvm install 22

# Verify
node --version   # must show v22.x.x
npm --version
```

**Why NVM?** It lets you manage multiple Node.js versions without conflicts. If OpenClaw later requires a newer version, `nvm install <version>` handles it cleanly.

## Step 5: Install OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

The installation script auto-detects your environment and launches an onboarding wizard that asks for:

- **AI provider**: Anthropic (Claude), OpenAI, or local models via Ollama
- **Chat channel**: WhatsApp, Telegram, Discord, Signal
- **Skills**: Optional — can be added later
- **Daemon/Gateway**: systemd service for 24/7 operation

## Step 6: Verify Installation

```bash
openclaw status          # System status overview
openclaw dashboard       # Web dashboard (http://localhost:18789)
openclaw doctor          # Health check
openclaw logs --follow   # Real-time log stream
```

If all commands return without errors, OpenClaw is operational.

## Security Notes

- **Never install OpenClaw as root** — run it under a regular user account
- **Use a dedicated device** — avoid running OpenClaw on your personal workstation
- **Start with minimal permissions** — add skills and integrations incrementally
- **Avoid unverified third-party skills** — approximately 80% of ClawHub skills are low-quality or malicious (see the "ClawHavoc" campaign of early 2026)

## What's Next

- [Authentication Setup](authentication.md) — Configure your AI provider credentials
- [Remote Access](../guides/remote-access.md) — Access your mini PC from another machine
