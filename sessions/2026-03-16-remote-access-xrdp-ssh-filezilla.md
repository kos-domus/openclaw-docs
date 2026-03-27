---
title: "Configurazione accesso remoto al mini PC: xrdp, SSH e FileZilla"
date: "2026-03-16"
author: "kos-domus"
status: "ready"
tags: ["remote", "setup", "configuration"]
openclaw_version: "latest"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Configurare l'accesso remoto completo al mini PC AceMagic (Ubuntu) dal Mac M4: desktop remoto (xrdp), SSH e trasferimento file (FileZilla/SFTP).

## Context

- Mini PC AceMagic con Ubuntu, connesso via ethernet (192.168.1.87)
- Mac M4 connesso via WiFi alla stessa rete locale
- Necessità di controllare il mini PC da remoto e trasferire file (script OpenClaw da Telegram)
- Mac e AceMagic su stessa rete → ethernet e WiFi non fanno differenza

## Steps Taken

### 1. Configurazione xrdp (Desktop Remoto)

```bash
# Sul mini PC Ubuntu
sudo apt install xrdp -y
sudo systemctl enable xrdp
sudo systemctl start xrdp
```

Client su Mac: **Microsoft Remote Desktop**
- PC name: `192.168.1.87`
- Credentials: utente e password Ubuntu
- Port: 3389 (default)

**Result**: Connessione stabilita ma login falliva inizialmente.

### 2. Risoluzione problemi xrdp

Problema 1: "Login failed for user Kos"
- Causa: nome utente case-sensitive (`kos` non `Kos`)
- Verifica: `whoami` sul terminale Ubuntu

Problema 2: VNC connection failed
- Fix applicato:

```bash
# Installare desktop environment completo
sudo apt install ubuntu-desktop -y

# Creare file di configurazione sessione
echo "gnome-session" > ~/.xsession
chmod +x ~/.xsession

# Aggiungere utente al gruppo ssl-cert
sudo adduser $USER ssl-cert

# Riavviare xrdp
sudo systemctl restart xrdp
sudo reboot
```

- Nella schermata login xrdp: usare **Session: Xorg** (non Xvnc)

**Result**: Problemi persistenti — xrdp funziona a tratti.

### 3. Configurazione SSH e SFTP

```bash
# Sul mini PC Ubuntu
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

Verifica raggiungibilità da Mac:
```bash
nc -zv 192.168.1.87 3389  # Per xrdp
nc -zv 192.168.1.87 22     # Per SSH
```

**Result**: SSH funzionante e stabile.

### 4. Configurazione FileZilla per trasferimento file

FileZilla Site Manager:
- **Protocol: SFTP** (NON FTP — errore comune)
- Host: `192.168.1.87`
- Port: `22`
- Logon Type: Normal
- User: `kos`
- Password: password Ubuntu

Directory remota per script OpenClaw: `/home/kos/openclaw/workspace`
Directory locale consigliata: `/Users/rakki/Desktop` (evitare `/Users/rakki/` per permission denied)

**Result**: Trasferimento file funzionante via SFTP.

## Configuration Changes

- xrdp installato e configurato (porta 3389)
- SSH server installato e abilitato (porta 22)
- ubuntu-desktop installato per supporto sessioni grafiche remote

## Key Discoveries

- **Ethernet + WiFi sulla stessa rete funzionano**: non serve che entrambi i device siano sulla stessa interfaccia
- **xrdp è instabile su Ubuntu**: la connessione desktop remoto ha problemi ricorrenti; SSH è molto più affidabile
- **SFTP ≠ FTP**: FileZilla deve essere configurato con protocollo SFTP, non FTP. Errore tipico: `Cannot establish FTP connection to an SFTP server`
- **Username case-sensitive**: `kos` e `Kos` sono utenti diversi su Linux
- **Permission denied su macOS**: non scrivere nella home root `/Users/rakki/`, usare sottocartelle come Desktop o Documents
- **Verifica connettività**: `nc -zv IP PORTA` è il modo rapido per testare se un servizio è raggiungibile

## Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| Login failed for user Kos | Username case-sensitive | Usare `kos` minuscolo (verificare con `whoami`) |
| VNC connection failed | Desktop environment mancante o sessione sbagliata | Installare `ubuntu-desktop`, creare `.xsession`, usare Xorg |
| Cannot establish FTP connection to SFTP server | Protocollo sbagliato in FileZilla | Cambiare da FTP a SFTP |
| Permission denied su `/Users/rakki/` | macOS non permette scrittura in home root | Usare `/Users/rakki/Desktop/` |

## Final State

Accesso remoto funzionante via SSH e SFTP. xrdp configurato ma instabile. File transferibili da Mac a Ubuntu via FileZilla SFTP.

## Open Questions

- Valutare Tailscale come alternativa più robusta per accesso remoto (non limitato alla stessa rete locale)
- xrdp vs VNC vs altri protocolli per desktop remoto più stabile?
- Automatizzare il trasferimento file da Telegram → Ubuntu senza FileZilla manuale?
