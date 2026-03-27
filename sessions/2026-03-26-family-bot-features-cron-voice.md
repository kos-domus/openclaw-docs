---
title: "Feature exploration: cron jobs, sub-agents, Drive/PDF, social fetching, voice TTS/STT per family bot"
date: "2026-03-26"
author: "kos-domus"
status: "ready"
tags: ["skills", "automation", "cron", "configuration", "multi-agent"]
openclaw_version: "2026.3.23-2"
environment:
  os: "Ubuntu 24 (AceMagic mini PC, 16GB RAM)"
  ide: "CLI"
  model: "claude-opus-4-6"
---

## Objective

Esplorare tutte le funzionalità implementabili per il family assistant Kai: cron jobs, sub-agents, gestione documenti/PDF/immagini, fetching promozioni e eventi, social media, messaggi vocali con customizzazione voce.

## Context

- Kai operativo su WhatsApp gruppo "Family stuff"
- gws skills linkate (calendar, docs, drive, drive-upload, gmail, gmail-send)
- Necessità di automatizzare routine familiari e espandere le capacità del bot

## Steps Taken

### 1. Mappatura cron jobs possibili

**Routine giornaliere**:
- Briefing mattutino (07:00): email urgenti, agenda del giorno, meteo
- Controllo mezzogiorno: promemoria farmaci, appuntamenti, lista spesa
- Riepilogo serale (21:00): recap giornata, task aperte, agenda domani

**Routine settimanali**:
- Revisione settimanale (lunedì): riepilogo, pianificazione, budget, task
- Pianificazione pasti (domenica): menu settimanale + lista spesa
- Backup workspace (domenica notte): commit git
- Analisi spese (venerdì): riassunto uscite settimanali

**Routine mensili**:
- Promemoria bollette (primo del mese)
- Compleanni e anniversari (con annuncio anticipato 3 giorni)
- Rinnovo abbonamenti (avviso 7 giorni prima)

**Watcher periodici**:
- Monitor email urgenti (ogni 20 min)
- Controllo calendario prossime 2h (ogni ora)

Formato cron OpenClaw:
```bash
openclaw cron add \
  --name "Nome" \
  --cron "0 7 * * 1" \
  --tz "Europe/Rome" \
  --session isolated \
  --message "Istruzioni" \
  --model "opus" \
  --announce \
  --channel whatsapp
```

### 2. Sub-agents possibili

**On-demand**: research-agent, email-triage-agent, budget-agent, shopping-agent, schedule-optimizer-agent, document-analyzer-agent, travel-planner-agent

**Pattern combo**: cron job che spawna sub-agent per task pesanti (es. analisi settimanale con opus + thinking high)

### 3. Gestione documenti e media

| Funzionalità | Supporto | Note |
|---|---|---|
| Archiviazione documenti su Drive | ✅ Nativo | Via gws-drive |
| Lettura/analisi PDF | ✅ Nativo | Il modello legge PDF direttamente |
| Editing PDF | ⚠️ Con setup | Richiede pdftk nel sandbox Docker |
| Conversione immagini → PDF | ⚠️ Con setup | Richiede img2pdf |
| Editing immagini | ⚠️ Con setup | Richiede ImageMagick |
| Password vault | ⚠️ Tramite 1Password | MAI salvare credenziali in chiaro nel workspace |
| Backup Git | ✅ Nativo | Via exec + cron |

**Nota sicurezza password**: usare 1Password come secrets provider nativo di OpenClaw. Il bot recupera credenziali al volo senza conservarle in file.

### 4. Fetching promozioni e volantini

**Approccio diretto**: web_search + web.fetch + browser (Playwright) per siti come lidl.it, eurospin.it.

**Approccio aggregatore** (più robusto): fetch su PromoQui o VolantinoFacile per CAP → tutte le catene aggregate, risolve il problema del raggio in un colpo.

**Limiti**:
- Catene nazionali (Lidl, Eurospin): fetch diretto affidabile
- Catene regionali: dipende se hanno sito con volantino web
- CAPTCHA/anti-scraping: il browser aiuta ma non garantisce
- Volantino solo su app mobile: non raggiungibile

### 5. Social media e eventi

**Facebook/Instagram**: tecnicamente possibile con browser + profilo autenticato persistente (`browser create-profile`), ma Meta rileva automazione → rischio ban account. Usare solo per lettura, mai pubblicare.

**Approccio consigliato per eventi**: pipeline su fonti aperte:
1. web_search freschi
2. Eventbrite locale
3. Sito del comune sezione eventi
4. RSS giornale locale

### 6. Messaggi vocali e TTS/STT

**Ricezione vocali (STT)**: trascrizione automatica con OpenAI Whisper (gpt-4o-mini-transcribe). Zero config lato utente.

**Risposta vocale (TTS)**: tre provider con livelli diversi:

| Provider | Qualità | Costo | Customizzazione | Italiano |
|---|---|---|---|---|
| ElevenLabs | Eccellente | A pagamento | Cloning, 1000+ voci | ✅ multilingual_v2 |
| OpenAI TTS | Buona | A pagamento | 6 voci predefinite | ✅ |
| Edge TTS | Buona | Gratuito | 400+ voci | ✅ (ElsaNeural, DiegoNeural) |

**Voice cloning**: ElevenLabs supporta instant voice cloning con 1-5 minuti di audio pulito. ATTENZIONE: clonare voci di persone reali (es. doppiatori) senza consenso è problematico per GDPR (voce = dato biometrico).

**Feature avanzata**: directive inline nel testo per cambiare voce a metà risposta.

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs",
      elevenlabs: {
        voiceId: "ID",
        modelId: "eleven_multilingual_v2",
        languageCode: "it",
        voiceSettings: { stability: 0.6, similarityBoost: 0.75, speed: 1.0 }
      }
    }
  }
}
```

### 7. Skills ClawHub più scaricate

| Skill | Download | Funzione |
|---|---|---|
| GOG (Google Workspace) | 14.000+ | Gmail, Calendar, Drive, Docs, Sheets |
| Summarize | 10.000+ | Riassumi URL, video, podcast, file |
| Composio | Alto | 860+ strumenti esterni senza gestire auth |
| Exa | Alto | Ricerca tecnica (repo, docs, forum) |
| n8n Workflow | Alto | Bridge OpenClaw ↔ n8n automazione |
| Obsidian | Alto | RAG su vault Obsidian |
| ElevenLabs | Alto | TTS, STT, voice cloning, failsafe telefonico |

**Skill bundled**: gemini (coding + ricerca Google), peekaboo (screenshot macOS)

**Attenzione**: ~80% delle skill su ClawHub è di scarsa qualità o malevola. Leggere sempre il codice.

## Key Discoveries

- **Cron + sub-agent = pattern potente**: il cron spawna un sub-agent con modello opus per task pesanti (analisi settimanale, report budget)
- **PromoQui > scraping diretto**: per volantini supermercati, usare aggregatori è più affidabile che fare scraping sito per sito
- **Browser profili persistenti**: `browser create-profile` permette login una-tantum su siti, ma Facebook rileva automazione
- **Password MAI in chiaro**: usare 1Password come secrets provider nativo, non VAULT_INDEX.md con credenziali
- **Edge TTS è gratuito**: ottimo fallback per voci italiane (ElsaNeural, DiegoNeural) senza costi
- **Voice cloning + GDPR**: la voce è dato biometrico, clonare senza consenso è problematico
- **Directive inline TTS**: il modello può cambiare voce/velocità/tono a metà risposta

## Open Questions

- Quale provider TTS usare per il family bot? (Edge gratuito vs ElevenLabs qualità)
- Come gestire il rate limiting con 8 agenti su gpt-5.4-mini concorrenti?
- Vale la pena integrare n8n per automazioni più complesse?
- Come automatizzare il backup git del workspace?
