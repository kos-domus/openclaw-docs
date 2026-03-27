---
title: "Architettura deployment sicuro: Tailscale + Docker sandbox + egress controllato"
date: "2026-03-23"
author: "kos-domus"
status: "ready"
tags: ["security", "configuration", "tailscale", "setup"]
openclaw_version: "2026.3.13"
environment:
  os: "Ubuntu (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Configurare un'architettura di deployment sicuro per OpenClaw con Tailscale (rete privata), Docker double-network (isolamento runtime) e egress controllato (allowlist domini).

## Context

- OpenClaw operativo su AceMagic Ubuntu con framework multi-agent
- Necessità di sicurezza enterprise-grade mantenendo permessi ampi per gli agenti
- 42.665 istanze OpenClaw esposte pubblicamente in uno scan di febbraio 2026, 93% vulnerabili a bypass auth
- Cisco ha documentato data exfiltration via skill di terze parti

## Steps Taken

### 1. Analisi della superficie d'attacco

**Rischi documentati**:
- Prompt injection e log poisoning
- Skill malevole su ClawHub (~80% di scarsa qualità o malevole)
- Istanze esposte su internet senza autenticazione
- Data exfiltration via domini arbitrari
- tools.elevated è globale e basato sul mittente, non configurabile per agente

**Principio guida**: sicurezza e permessi non si contraddicono se applichi il **blast radius minimo per agente**.

### 2. Layer 0 — Tailscale tailnet (WireGuard mesh)

Tailscale è il cambiamento di sicurezza con il più alto impatto singolo. Rende OpenClaw **irraggiungibile per default** — un port scanner non trova nulla.

```bash
# Installare Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Configurare OpenClaw per ascoltare solo su loopback
# Tailscale Serve espone sulla tailnet
tailscale serve --bg 8080
```

**Raccomandazione ufficiale**: usare Tailscale Serve, **MAI** Tailscale Funnel (che esporrebbe al pubblico).

Le piattaforme di messaggistica (Telegram, Discord) entrano via websocket outbound — non serve aprire porte.

### 3. Layer 1 — Docker double-network

Due reti Docker separate:
- `openclaw-internal`: **senza accesso internet**, dove gira il Gateway
- `openclaw-external`: con proxy Squid che applica allowlist domini

**LiteLLM proxy**: l'agente non vede mai le API key reali — il proxy le inietta al momento della richiesta con rate limiting e cost control.

### 4. Layer 2 — Egress controllato

Solo domini in allowlist possono uscire:
- `github.com`, `npmjs.org`, `telegram.org`, `api.anthropic.com`
- Tutto il resto bloccato

Questo mitiga direttamente l'attacco Cisco: skill che tentano exfiltration verso domini arbitrari vengono bloccate.

### 5. Permessi multi-agent stratificati

| Agente | Canale | Sandbox | Tool set | Blast radius |
|--------|--------|---------|----------|--------------|
| Admin/owner | DM privato Tailscale | off | tutto | host |
| Team interno | Slack/Discord | non-main (container) | exec, read, sessions | container |
| Bot pubblico | Telegram/canale aperto | all (sessione isolata) | solo read + sessions_list | sessione singola |

**exec_ask**: prompt di conferma interattiva prima di ogni comando shell — elimina il rischio di prompt injection che triggerano comandi distruttivi.

### 6. Tool di audit integrati

```bash
openclaw security audit     # flag insicuri abilitati
openclaw sandbox explain    # policy sandbox effettiva per sessione
```

## Key Discoveries

- **Tailscale elimina la superficie d'attacco di rete**: OpenClaw non deve MAI essere esposto su internet pubblico
- **Docker double-network**: rete interna senza internet + proxy egress = isolamento completo
- **LiteLLM proxy**: l'agente non vede API key reali → impedisce exfiltration credenziali
- **Ogni dominio nell'allowlist è una potenziale via di exfiltration**: il contenimento egress è solo buono quanto i buchi che deliberatamente apri
- **tools.elevated è globale**: non configurabile per agente, serve negare `exec` esplicitamente in `agents.list[].tools`
- **Regole hard in tool policy, non in AGENTS.md**: AGENTS.md è advisory, la policy nel config è enforcement
- **Comunicazione inter-agente**: sandbox permette `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`
- **42.665 istanze esposte (febbraio 2026)**: 93% vulnerabili — la configurazione sicura NON è opzionale

## Open Questions

- Come automatizzare l'aggiornamento dell'allowlist egress quando si aggiungono nuove integrazioni?
- Performance di Tailscale Serve su mini PC con molte connessioni concorrenti?
- Come monitorare tentativi di exfiltration bloccati dal proxy?
