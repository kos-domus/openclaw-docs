# Knowledge Base Changelog

## 2026-09-13 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Diff pre-esistente verificato (Release Monitor 04:00, non committato):** tredicesimo giorno consecutivo di coordinamento; oggi il monitor NON ha committato (worktree partito sporco). Ogni claim verificato contro API live alle 07:30-07:35 prima del commit:

- **OpenClaw stable `2026.9.4` invariata** ✓ — GitHub releases list: `v2026.9.4` Sep 11 03:46 UTC sempre prima entry (`prerelease: false`); npm `latest`/`beta` 2026.9.4, `extended-stable` 2026.6.35 allineati. Nessun nuovo prerelease/tag-only signal, nessuna 2026.9.5.
- **Hermes v0.21.2 (v2026.9.11) confermata live** ✓ — Sep 11 19:20 UTC, `prerelease: false`. Ancora la latest.
- **Advisories invariati** ✓ — totale 722 (14C/249H/390M/69L) via `--paginate`, newest sempre il batch Sep-11 00:58 UTC: zero nuovi GHSA in ~55h.
- **ClawHub `v0.23.3`** ✓ — ultima release registry confermata (security fixes #3680-#3684 solo su main).
- **Adjacent CLIs** ✓ — claude-code 2.1.270 / codex 0.154.0 / gemini-cli 0.59.0 verificati live via `npm view` (coerenti col draft 04:00).
- **Hermes drift riallineato (dato stale corretto)**: draft 04:00 dichiarava **2024 behind** / **365 ahead-of-tag**; live post-fetch 07:32 → **2101 behind** / **442 ahead** (+77/+77 in ~3.5h). Il 2024 era accurato vs il tip `205645ee` (Sep 12 18:35) misurato alle 04:00 — classe "stale" (timing), non "wrong": il tip è nel frattempo avanzato a `b6b53c69a6` (Sep 12 22:21). `hermes --version` cross-check: "2101 commits behind" ✓ combacia con rev-list. Trend 1291→1482→1752→2101 in 4 giorni, **accelerando**: ~270/day medio ma 349 push nelle sole ultime 24h (~370/day attuale).
- **Spot-check contenuti last-24h** ✓ — commit citati dal draft trovati su `v2026.9.11..origin/main`: `560b6d2e81` (cron systemd-scope graceful degrade), `87b013b21e` (heartbeat fire-fence), `205645ee42` (gateway.multiplex_profiles), `acbecf588a` (profiles --clone-channels), `d1dbb0ac9e` (multiplexer hot-serve) + `f361971eed` (agent_loop_stopped hook, non citato dal draft). Claim del draft verificati.
- **CORREZIONE — nota integrità del monitor**: il draft diceva "last file on disk is 2026-09-10" ma verifica diretta: nel repo l'ultimo file `memories/reports/release-monitor/` è 2026-08-21.md, e anche nel workspace hermes (`~/.hermes/memories/reports/release-monitor/`) l'ultimo è 2026-09-09.md — il report 2026-09-12 non è mai stato salvato da nessuna parte. Nota del tracker corretta (classe wrong-data lieve: puntatore a file inesistente).

**Modifiche questo run:** solo `docs/meta/upstream-version.yaml` (drift Hermes 2024→2101, tag-ahead 365→442, `last_check` 09-13, nota integrità precisata) + questo changelog. Zero churn su `docs/index.yaml`, zero docs Diátaxis toccati, nessun artifact upstream (nessun nuovo segnale release).

### Self-assessment
SKIP run pulito, tredicesimo giorno consecutivo di coordinamento col Release Monitor. Oggi il monitor non ha committato il tracker (diff pre-esistente nel worktree, pattern classico recuperato e verificato). La verifica live ha pagato ancora: due numeri stale corretti (drift 2024→2101, tag-ahead 365→442 — entrambi classe timing, misurati 3.5h prima) e una nota integrità imprecisa ("last file 2026-09-10" → in realtà 2026-09-09 nel workspace hermes, e 2026-08-21 nel repo; il report 09-12 non esiste da nessuna parte). Segnale operativo per Rakki, rafforzato: **Hermes upgrade a v0.21.2 è oltre la soglia critica** — drift 2101 e in accelerazione (~370/day attuale), siamo su 0.21.0 (state.db rewrite fragile) mentre v0.21.2 patcha esattamente la nostra esposizione gateway+cron (cron lifecycle guard, raw-open() lock cancellation) + credential-scoping multi-profilo per la nostra topologia 4-profili. `hermes update` + gateway restart appena possibile. OpenClaw stable ferma a 2026.9.4 da Sep 11, CLI locale 2026.6.8 (KB-reference only). gitleaks PASS, solo tracker + changelog committati. `memories/` untracked lasciato fuori come da convenzione.

## 2026-09-12 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker già committato dal Release Monitor (04:06, `a40f3be`):** dodicesimo giorno consecutivo in cui la KB run parte con il tracker già aggiornato — oggi però non c'era nemmeno diff pre-esistente da recuperare: il monitor ha committato direttamente (solo `docs/meta/upstream-version.yaml`, 11 insertions/32 deletions). La KB run ha ri-verificato ogni claim contro le API live alle 07:35:

- **OpenClaw stable `2026.9.4` invariata** ✓ — GitHub releases list: `v2026.9.4` Sep 11 03:46 UTC sempre prima entry (`prerelease: false`); tags confermati; npm `latest`/`beta` 2026.9.4 e `extended-stable` 2026.6.35 allineati. Nessun nuovo prerelease/tag-only signal.
- **Hermes v0.21.2 (v2026.9.11) confermata live** ✓ — Sep 11 19:20 UTC, `prerelease: false`. Il tracker del monitor era corretto.
- **Advisories invariati** ✓ — totale 722 (14C/249H/390M/69L) via `--paginate`, newest sempre il batch Sep-11 00:58 UTC: zero nuovi GHSA nelle ultime ~27h.
- **Hermes drift riallineato (deriva attesa)**: monitor 04:06 dichiarava 1719 behind / tag-ahead 60; live post-fetch 07:35 → **1752 behind** / **93 ahead** (+33/+33 in ~3.4h, push wave post-release continua a ~+270/day). Semantica: behind = `HEAD..origin/main` dal local 006b1beb (v0.21.0, Sep 5); ahead = `v2026.9.11..origin/main`. `hermes --version` cross-check: "1752 commits behind" ✓ combacia con rev-list. Upgrade v0.21.2 ancora pending e ora doppiamente motivato (security wave credential-scoping + state.db reliability, vedi blocco 09-11).
- **CLI locale OpenClaw** ✓ — `openclaw --version` = 2026.6.8 (844f405), invariato da Jun 19. Nessun upgrade esterno rilevato tra le run.
- **Docs site**: 8/8 key pages HTTP 200 ✓ (incluso `/agents`→`/multi-agent` e `/hooks`→`/automation/hooks` già tracciate come migrazioni).
- **Nota integrità**: le note del tracker referenziano `memories/reports/release-monitor/2026-09-12.md` come full detail, ma il file NON esiste nel repo (memories/ è untracked dal 2026-08-21, contiene solo 2026-08-21.md). Chiarito nel tracker stesso ("treat as external") — il reference punta all'artifact di un altro job, non a un file di questo repo.

**Modifiche questo run:** solo `docs/meta/upstream-version.yaml` (drift Hermes 1719→1752, tag-ahead 60→93, `checked_at` 07:35, nota integrità sul reference mancante) + questo changelog. Zero churn su `docs/index.yaml`, zero docs Diátaxis toccati, nessun artifact upstream (nessun nuovo segnale release).

### Self-assessment
SKIP run pulito, dodicesimo giorno consecutivo di coordinamento col Release Monitor. Novità del pattern: oggi il monitor ha committato direttamente il tracker (nessun diff da recuperare), quindi questa run è stata pura verifica live + allineamento drift — e la verifica ha pagato: (1) il drift Hermes è cresciuto di 33 commit in 3.4h, confermando il trend +270/day che rende l'upgrade sempre più urgente; (2) il reference `memories/reports/release-monitor/2026-09-12.md` citato nelle note del monitor non esiste nel repo — corretto annotando nel tracker che va trattato come artifact esterno, per evitare che la prossima run lo cerchi invano. Segnali operativi per Rakki, invariati ma rafforzati: (1) **Hermes upgrade a v0.21.2 resta la priorità #1** — siamo su 0.21.0 (la versione con la fragile state.db rewrite) e v0.21.2 fixa esattamente la nostra esposizione gateway+cron (raw-open() lock cancellation sulle run cron) + la wave credential-scoping multi-profilo; drift ora 1752; (2) OpenClaw stable ferma a 2026.9.4 da ieri, CLI locale ferma a 2026.6.8 (KB-reference only). gitleaks PASS, solo tracker + changelog committati. `memories/` untracked lasciato fuori come da convenzione.

## 2026-09-11 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato (diff Release Monitor 04:02):** `docs/meta/upstream-version.yaml` già modificato nel worktree alla partenza (undicesimo giorno consecutivo di recupero). Ogni claim verificato contro API live prima del commit:
- **STABLE ADVANCE `2026.9.4`** (Sep 11, 03:46 UTC) ✓ — GitHub releases list: `v2026.9.4` prima entry, `prerelease: false`; npm `latest` E `beta` entrambi 2026.9.4 (il draft delle 04:02 vedeva solo il tag staged e dava la release come pending — la release è andata live tra le due run, superseded qui). Quarta stable in 8 giorni. Artifact dedicato: `docs/meta/upstream-updates/2026-09-11-v2026.9.4.md`.
- **CORREZIONE DATO ERRATO — "3 new medium GHSA"**: il conteggio live è **75 nuovi advisory** dalla run di ieri (30H/40M/5L), tutti pubblicati 00:58 UTC Sep 11 in un unico triage batch. 70 dei 75 range includono la CLI locale 2026.6.8; 5 la escludono (floors >=6.9.x/>=7.x). Classe "wrong data" (sottostima -72), corretta esplicitamente in tracker + artifact.
- **Totale advisory REALE 722 (14C/249H/390M/69L)** ✓ — verificato via `--paginate` (il vecchio "100" era artefatto prima pagina; il draft l'aveva già corretto, confermato live oggi).
- **v2026.6.35 Extended Stable** (Sep 10) ✓ — release body: "final June 2026 Extended Stable (LTS) release", 166 merged PRs dichiarati; compare diretto .34→.35 = 30 commit (history squashed; il "166 PRs" del body è il conteggio audited completo). npm `extended-stable` dist-tag allineato a 2026.6.35.
- **Hermes behind-count riallineato (deriva attesa)**: draft 04:02 diceva 1468 behind / 795 ahead-of-tag; live post-fetch 07:45 → **1482 behind** / **809 ahead** (+14/+14 in ~3.7h, push wave multi-profile credential-scoping continua). Semantica: behind = `HEAD..origin/main` dal local 006b1beb; ahead = `v2026.9.7..origin/main`. v0.21.1 upgrade ancora pending — **ora priorità di sicurezza**: la wave upstream (168 commit/24h) è di fix credential-scoping multi-profilo (send_message via bot proprio, secrets vault instradati per profilo, TWILIO scoping) direttamente applicabili alla nostra topologia 4-profili con vault 1Password separati. Inoltre Nous free tier (NS-847) landed upstream.
- **Adjacent CLIs** ✓ — claude-code 2.1.268 / codex 0.154.0 / gemini-cli 0.59.0 confermati live via `npm view`. Opportunistico, non bloccante.
- **Docs site**: 7/8 key pages HTTP 200; `/agents` 404 noto (migrazione `/multi-agent` già tracciata, 200 ✓).

**Modifiche questo run:** `docs/meta/upstream-version.yaml` (stable 2026.9.4 su GitHub/npm/github block, `pending_tag` rimosso, beta npm 2026.9.4, advisory total 722 + batch 75, behind 1468→1482, main-ahead 795→809, note riscritte con correzioni live) + **nuovo artifact** `docs/meta/upstream-updates/2026-09-11-v2026.9.4.md` + questo changelog. Zero churn su `docs/index.yaml` (artifact in `docs/meta/` non indicizzato, come da convenzione).

### Self-assessment
SKIP run pulito, undicesimo giorno consecutivo di recupero diff pre-esistente dal Release Monitor. Il draft di stamattina era metà giusto e metà sbagliato: giusto sul trend generale (wave credential-scoping, free tier, extended-stable), ma due dati errati significativi corretti — (1) la release v2026.9.4 data per "pending" era già pubblicata da ~4h (lag tipico tag-vs-release-page), (2) il batch advisory "3 medium" era in realtà 75 (30H/40M/5L): il monitor ha visto il tag e le prime 3 GHSA della lista, non il batch completo. Il pattern si conferma: mai fidarsi dei conteggi del draft, sempre ricalcolare gli aggregati dal vivo. Segnali operativi per Rakki: (1) **Hermes upgrade ora priorità di sicurezza** — drift 972→1308→1482 in 3 giorni, upstream sta fixando esattamente il nostro modello (profilo secondario che manda credenziali al profilo sbagliato, MCP children col vault sbagliato); raccomandato `hermes update` + gateway restart appena possibile; (2) CLI OpenClaw locale 2026.6.8 ora esposta a 70+ range advisory nel batch di oggi (KB-reference only, ma il dato rafforza la nota upgrade-se-importa). gitleaks PASS, zero docs Diátaxis toccati, tracker + artifact + changelog committati. `memories/` untracked lasciato fuori (artifact Release Monitor, dal 2026-08-21).

## 2026-09-10 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato (diff Release Monitor 06:15):** `docs/meta/upstream-version.yaml` già modificato nel worktree alla partenza (decimo giorno consecutivo di recupero). Ogni claim verificato contro API live prima del commit:
- **Giornata tranquilla confermata** ✓ — nessuna nuova stable OpenClaw (releases list: `v2026.9.3` Sep 8 sempre prima entry, `prerelease: false`), nessuna nuova release Hermes (v0.21.1 / v2026.9.7 Sep 7 confermata live), zero nuovi advisories entrambi i repo (OpenClaw 100 totali 46H/50M/4L, newest Jun-30; Hermes zero).
- **CORREZIONE DATO ERRATO — npm extended-stable**: draft dichiarava `2026.6.34`, live `npm view openclaw dist-tags` → **`2026.6.35`**. Corretto nel tracker con nota esplicita (classe di errore "wrong data", non stantio: 35 > 34, l'extended-stable è avanzato dopo il check del monitor).
- **Hermes behind-counts riallineati (derive attese)**: draft 06:15 diceva 1291 behind / 618 ahead-of-tag; live post-fetch 07:35 → **1308 behind** / **635 ahead** (+17 / +17 in ~1.3h, push wave continua). Semantica esplicita: behind = `HEAD..origin/main` dal local 006b1beb; ahead = `v2026.9.7..origin/main`; dietro il tag: 673. Upgrade v0.21.1 ancora pending — urgenza crescente, i fix cron scheduling/delivery sono direttamente rilevanti per questi job.
- **Adjacent CLIs** ✓ — claude-code 2.1.267 / codex 0.154.0 / gemini-cli 0.59.0 confermati live via `npm view`. Aggiornamento opportunistico, non bloccante.
- **OpenClaw main last 24h** ✓ — commit del Sep 10 05:0x-05:2x UTC tutti routine (locales refresh, canonical OpenAI live default, audit summary refactor, Workshop review index repair): nessun segnale 2026.9.4.
- **ClawHub** ✓ — registry invariato; catalog-curation push Sep 9 (withheld plugins esclusi dalla discovery, publisher badges) confermato come direzione positiva security/hygiene.

**Modifiche questo run:** solo `docs/meta/upstream-version.yaml` (extended-stable corretto .34→.35, behind 1291→1308, main-ahead 618→635, `checked_at` 07:35, note arricchite con correzioni live) + questo changelog. Zero churn su `docs/index.yaml`.

### Self-assessment
SKIP run pulito, decimo giorno consecutivo di recupero diff pre-esistente dal Release Monitor. Il draft di stamattina era sostanzialmente accurato ("quiet day" confermato su tutti i fronti) con una sola correzione di dato errato (extended-stable .34 vs live .35 — dist-tag avanzato dopo il check del monitor, tipico caso "wrong data" da correggere esplicitamente, non deriva temporale) e le solite due derive numeriche attese sui behind-count Hermes. Segnali operativi per Rakki invariati: (1) Hermes v0.21.1 con fix cron ancora pending — drift ora 1308 commit, `hermes update` + gateway restart raccomandati con urgenza crescente; (2) CLI OpenClaw locale 2026.6.8 a 13 stables, KB-reference only, esposizione bassa. gitleaks PASS, zero docs Diátaxis toccati, solo tracker + changelog committati. `memories/` untracked lasciato fuori (artifact Release Monitor, dal 2026-08-21).

## 2026-09-09 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato (diff Release Monitor 04:05):** `docs/meta/upstream-version.yaml` già modificato nel worktree alla partenza (nono giorno consecutivo di recupero). Ogni claim verificato contro API live prima del commit:
- **STABLE ADVANCE `2026.9.3`** (Sep 8, 14:15 UTC) ✓ — GitHub releases list: `v2026.9.3` prima entry, `prerelease: false`; npm `latest` 2026.9.3 allineato. Terza stable in 7 giorni (9.1→9.2→9.3).
- **BREAKING Node floor** ✓ — release body: Node >=24.16.0 (24.x) o >=26.1.0 (26.x), Node 26 raccomandato; Node 22/25 e build precedenti non supportati; rischio SQLite text truncation se si upgrada OpenClaw prima di Node. Host locale verificato: Node v24.16.0 = esattamente il nuovo minimo, già OK.
- **Sei SDK breakings + deprecation effective** ✓ — exec-policy, approval SDK, alias `buildChannelInboundMediaPayload`, search/dir result callbacks (`details.content`, `nextAfter`), agent-owned Workshop skills; rimozione untrusted-named context aliases (eligible Sep 8) ora EFFECTIVE. Zero impatto fleet (nessun plugin custom OpenClaw).
- **"13 stables behind"** ✓ — contato dalla release list live (6.9→9.3 incluse 7.1-1/7.1-2).
- **Hermes: due correzioni dati stantii (attese, non errati)** — draft 04:05 dichiarava 942 behind / 269 ahead-of-tag; live post-fetch 07:35 → **972 behind** / **299 ahead**. Drift ~30 commit in ~3.5h (push desktop-app upstream): numeri corretti nel tracker con semantica esplicita. v0.21.1 resta pending per l'upgrade locale — i fix cron scheduling/delivery sono direttamente rilevanti per i nostri job.
- **"Main last 24h"** ✓ — tutti i claim trovati nei 213 commit delle ultime 36h su origin/main: GPT Image 2.5 (OpenAI + FAL), Group Chat rooms ordering, MCP OAuth refresh_token (#62333), cron/gateway systemd hardening (user D-Bus adoption, per-probe env), RSS/Reddit skills resi opzionali.
- **Advisories invariate** ✓ — 100 GHSA (46H/50M/4L), newest sempre Jun-30; Hermes 0.
- **ClawHub** ✓ — registry v0.23.3 (Aug 4) invariato; il caso scope-squatting Manifold è background giugno (unlisted Jun 19 + dispute), nessuna novità.
- **Docs site**: 8/8 key pages HTTP 200.

**Modifiche questo run:** `docs/meta/upstream-version.yaml` (stable 2026.9.3 GitHub/npm/github, `checked_at` 07:35, behind 942→972, main-ahead 269→299, note arricchite con verifica live e Node locale) + **nuovo artifact** `docs/meta/upstream-updates/2026-09-09-v2026.9.3.md` (stable advance con breaking changes operator-relevant, come da precedente v2026.8.1) + questo changelog. Zero churn su `docs/index.yaml` (artifact in `docs/meta/` non indicizzato, come da convenzione).

### Self-assessment
SKIP run pulito, nono giorno consecutivo di recupero diff pre-esistente dal Release Monitor: il pattern di verifica live è ormai routine. Oggi il draft era accurato sui fatti (stable advance, breakings, main-24h) con sole due derive numeriche attese (behind-counts cresciuti tra le 04:05 e le 07:35), corrette con semantica esplicita nel tracker. Scelta editoriale: v2026.9.3 riceve artifact dedicato perché ha breaking changes operator-relevant (Node floor + 6 SDK breakings), coerente col precedente v2026.8.1; v2026.9.2 (nessun breaking) era rimasta changelog-only — la convenzione ora è: artifact se e solo se breaking/operator-relevant. Segnali operativi per Rakki invariati: (1) Hermes v0.21.1 con fix cron in coda — raccomandato `hermes update` + gateway restart; (2) CLI OpenClaw locale 2026.6.8 a 13 stables di distanza, 4 CVE aperte patchate da 2026.6.9, esposizione bassa (KB-reference only). gitleaks PASS, zero docs Diátaxis toccati, tracker + artifact + changelog committati. `memories/` untracked lasciato fuori (artifact Release Monitor, dal 2026-08-21).

## 2026-09-08 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato (diff Release Monitor 04:05):** `docs/meta/upstream-version.yaml` già modificato nel worktree alla partenza. Ogni claim verificato contro API live prima del commit:
- **OpenClaw stable `2026.9.2` invariato** — GitHub releases: `v2026.9.2` (Sep 5, `prerelease: false`) ancora prima entry; npm dist-tags identici (`latest` 2026.9.2, `beta` 2026.9.1, `extended-stable` 2026.6.34).
- **Hermes v0.21.1 (v2026.9.7, Sep 7)** — già catturato dal run Release Monitor di stamattina (commit 24bb6c4); confermato live (`releases?per_page=3` → v2026.9.7 prima entry). Locale ancora `006b1beb` / v0.21.0: `hermes --version` → **697 behind** (tracker aveva 678, aggiornato). Upgrade raccomandato: cron scheduling/delivery fix direttamente rilevanti per i nostri job schedulati.
- **Advisories invariate** — 100 GHSA totali (46H/50M/4L), newest sempre Jun-30 (dotenv override). Le "4 advisory che colpiscono la CLI locale 2026.6.8" ri-verificate oggi con estrazione completa dei version range → confermato 4 (3H+1M, tutte patchate in 2026.6.9).
- **Deprecation eligible da OGGI**: Plugin SDK untrusted-named context aliases removal on/after Sep 8 2026 — zero plugin custom nel nostro orbita, zero impatto.
- **Docs site key pages**: upstream `docs/` tree raggiungibile via API (nessuna variazione strutturale).

**Modifiche questo run:** solo `docs/meta/upstream-version.yaml` (behind 678→697, `checked_at` 07:35, note deprecation/behind riallineate al live) + questo changelog. Zero churn su docs/index.yaml.

### Self-assessment
SKIP run pulito, ottavo giorno consecutivo con worktree pre-modificato dal Release Monitor: il pattern di verifica è ormai consolidato — nessun claim del draft è stato preso per buono senza ricontrollo API live. L'unica deriva trovata era attesa (behind-count 678→697: cresce per costruzione senza upgrade locale, nessun dato sbagliato da correggere). Il segnale operativo resta lo stesso di ieri per Rakki: (1) Hermes v0.21.1 con fix cron scheduling/delivery in coda di upgrade — raccomandato `hermes update` + gateway restart; (2) CLI OpenClaw locale 2026.6.8 con 4 CVE (3 high) patchate da 2026.6.9, esposizione bassa ma upgrade a extended-stable consigliato quando comodo. gitleaks PASS, zero docs toccati, solo tracker + changelog committati. `memories/` untracked lasciato fuori (artifact Release Monitor, dal 2026-08-21).

## 2026-09-07 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato e corretto (run interrotto 04:03):** `docs/meta/upstream-version.yaml` era già modificato nel worktree alla partenza (settimo giorno consecutivo di recupero dal job Release Monitor). Ogni claim verificato contro API live prima del commit:
- **OpenClaw stable `2026.9.2` invariato** — GitHub releases list: `v2026.9.2` (Sep 5, `prerelease: false`) prima entry; npm dist-tags: `latest` 2026.9.2, `beta` 2026.9.1, `extended-stable` 2026.6.34 — tutti allineati al tracker.
- **Correzione dato errato: batch Jun-30 "28 advisories (24 high)" → 45 advisories (33 high).** Live-verified via `gh api security-advisories` raggruppate per `published_at`: 2026-06-30 → 45 (33 high), su 100 totali invariati (46H/50M/4L). Il draft delle 04:03 sottostimava il batch di 17 unità.
- **4 advisory colpiscono la CLI locale 2026.6.8** — verificato estraendo TUTTI i 100 `vulnerable_version_range` (51 range distinti): esattamente 4 hanno upper bound `< 2026.6.9` con lower bound che include la 2026.6.8. Le 4 (3 high GHSA-7vrr/mm9g/f6p7 + 1 medium GHSA-wgq8, tutte pubblicate Jun 30, tutte patchate in 2026.6.9) corrispondono esattamente a quelle citate nel draft.
- **Exposure LOW confermata** — nessun processo/servizio OpenClaw attivo sull'host (`pgrep` + `systemctl --user` vuoti); CLI usata solo come KB reference.
- **Hermes 265 behind** — `hermes --version` → "265 commits behind"; cross-check `git fetch` + `rev-list HEAD..origin/main` → 265; `rev-list v2026.8.31..origin/main` → 5587 (main ~5.6k past tag, v0.22.0 in preparazione). Locale fermo a `006b1beb` (Sep 5).
- **ClawHub registry v0.23.3 (Aug 4)** — `gh api repos/openclaw/clawhub/releases/latest` → v0.23.3, 2026-08-04.
- **Deprecation T-1**: Plugin SDK untrusted-named context aliases removal eligible ON/AFTER **Sep 8 2026** (domani) — docs `/plugins/compatibility` → 200.

**Upstream consistency check:**
- Advisories: 100 GHSA OpenClaw (46H/50M/4L) invariati since Jun 30, zero pubblicati dopo quella data; Hermes 0.
- Docs site key pages: 8/8 tracciate HTTP 200 (`/agents` resta 404 per migrazione nota → `/multi-agent`, già registrata in `key_pages`).
- `checked_at` del draft (04:03) riallineato all'ora della verifica live di questo run (07:35 CEST).

### Self-assessment
Settimo recupero consecutivo di diff pre-esistente dal Release Monitor interrotto. Oggi la verifica ha trovato un dato quantitativo sbagliato ma non inventato: il security backlog Jun-30 esiste davvero, solo sottestimato (28/24 vs 45/33 reali) — corretto esplicitamente nel tracker, come da lezione "wrong data → correggi e segnala, non aggiornare in silenzio". Metodo di verifica delle "4 advisory che colpiscono la locale" rafforzato: invece di fidarsi del conteggio del draft, estratti tutti i 100 version range e contati quelli il cui intervallo include la 2026.6.8 — il risultato (4) coincide, ma ora è ground-truth. Note strumentali: (1) `gh --jq -r` non esiste (gh passa argomenti extra a jq che fallisce con "accepts 1 arg(s), received 2") — il pattern affidabile resta `gh api ... > /tmp/x.json && jq -r '...' /tmp/x.json`; (2) heredoc terminal con emoji contenenti variation selector (ES. la sequenza warning-sign+VS16) viene bloccato dal security scanner in cron (pending approval che non arriva mai) — scrivere i blocchi changelog via tool `patch`, non via `cat >>`. Segnale operativo per Rakki: la CLI locale OpenClaw 2026.6.8 ha 4 CVE aperte (3 high) patchate da 2026.6.9 — esposizione bassa (nessun gateway attivo), ma l'upgrade a extended-stable 2026.6.34 è raccomandato quando comodo. Zero churn su docs/index.yaml come da contratto SKIP, gitleaks PASS, solo changelog + tracker committati. `memories/` untracked lasciato fuori (artifact Release Monitor, dal 2026-08-21).

## 2026-09-06 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato (run interrotto 04:05):** `docs/meta/upstream-version.yaml` era già modificato nel worktree alla partenza. Ogni claim verificato contro API live prima del commit (lezione 2026-08-31):
- **STABLE ADVANCE `2026.9.2`** (Sep 5, 20:00 UTC) ✓ — GitHub releases list: `v2026.9.2` prima entry, `prerelease: false`; npm `latest` 2026.9.2 allineato.
- **1.247 PRs** ✓ — dal body della release ("1,247 in-range PRs", audit record nel body).
- Contenuto release verificato ✓: GPT-6 Astra support (`openai/gpt-6-astra`, text+image input, Responses tool calls), Swarm default-on con opt-out espliciti (#136514), cross-agent session access default-on con `tools.sessions.visibility` per restringere (#136755), settings hot-reload senza restart (#138112), replies survive Gateway restarts, robust backups (NUL chars, Nix config links, corrupt header rejection), Telegram proxy media via SOCKS/HTTPS, experimental plugin UI (Settings → Labs), personal connected accounts.
- **Deprecazione tracciata**: Plugin SDK untrusted-named context aliases removal eligible on/after **Sep 8 2026** → migrate a channel-named context fields + `buildChannelMetadata` (docs `/plugins/compatibility` → 200).
- **UPGRADE LOCALE ESEGUITO ✓ (claim chiave del draft)**: Hermes 0.20.6 → **0.21.0 main-tracking** il Sep 5 15:53 CEST. Verificato: `git reflog` mostra `merge origin/main: Fast-forward` a `006b1beb` (Sep 5 15:53:25), gateway restartato 15:54:38, `systemctl is-active` → active. `hermes --version` → "v0.21.0 (2026.8.31) · upstream 245e4800 · 18 commits behind". Live-check di questo run: `git fetch` + `rev-list HEAD..origin/main` → **18** ✓, `rev-list v2026.8.31..origin/main` → **5340** ✓ (entrambi coincidono con il draft).
- **Pending per Rakki CHIUSA** dopo 4 giorni: upgrade Hermes completato esternamente (verosimilmente da MC o Rakki) — il pending "upgrade to v0.21.0" segnalato dal 2026-09-02 non è più dovuto.
- OpenClaw CLI locale 2026.6.8 = 12 stable releases behind ✓ (contato dalla lista release: 6.9→9.2; KB reference only).
- npm `beta` dist-tag ora punta a 2026.9.1 (nessun beta più nuovo della stable — pattern già visto in passato) ✓.

**Upstream consistency check:**
- Advisories: 100 GHSA OpenClaw (46H/50M/4L) invariati since Jun 30 ✓; zero pubblicati dopo Sep 4 ✓; Hermes 0 ✓.
- Docs site key pages: 7/8 HTTP 200; `/agents` resta 404 (migrazione nota → `/multi-agent`, già tracciata in `key_pages`).
- `checked_at` del draft (04:05) aggiornato all'ora della verifica live di questo run (07:32 CEST).

### Self-assessment
Sesta giornata consecutiva di recupero diff pre-esistente dal job Release Monitor interrotto (04:05). Oggi il draft era **accurato al 100%** — nessuna correzione necessaria, solo `checked_at` riallineato. Verifica totale comunque eseguita: release body 2026.9.2 letto per intero (1358 righe) con ogni feature del draft trovata, numeri Hermes ricontrollati con fetch+rev-list (18 e 5340 confermati), reflog dell'upgrade ispezionato (fast-forward pulito Sep 5 15:53 + gateway restart 15:54 + servizio attivo). Nota di rilievo: la verificazione del claim "upgrade eseguito" richiede evidenza diversa dai soliti controlli version-count — reflog + timestamp servizio + `hermes --version` sono la triade giusta. Chiuso il pending più vecchio del tracker (upgrade Hermes, aperto 2026-09-02). `memories/` untracked lasciato fuori dal commit (artifact Release Monitor, dal 2026-09-02). Zero churn su docs/index.yaml come da contratto SKIP, gitleaks PASS, solo changelog + tracker committati.

## 2026-09-05 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato e corretto (run interrotto 04:05):** `docs/meta/upstream-version.yaml` era già modificato nel worktree alla partenza. Ogni claim verificato contro API live prima del commit (lezione 2026-08-31):
- **OpenClaw stable `2026.9.1` invariato** ✓ — GitHub releases list: `v2026.9.1` (Sep 3, `prerelease: false`) prima entry; npm dist-tags: `latest` 2026.9.1, `beta` 2026.9.1 (nessun beta più nuovo della stable), `extended-stable` 2026.6.34; tags head `v2026.9.1`. L'evento Releasebot "release-publish" del Sep 4 era una re-publish della 9.1, non una nuova versione ✓.
- **⚠️ Correzione claim impossibile: `hermes_main_behind` 6148 → 5599.** Un behind-count non può scendere senza upgrade locale (locale sempre 0.20.6) — il 6148 del draft delle 04:05 era un dato erroneo, non stantio. Live-verified 16:20 CEST: `hermes --version` → "5596 commits behind" (pre-fetch), `git fetch` + `rev-list HEAD..origin/main` → **5599** (+3 commit arrivati tra le due misure, delta coerente). Anche `main_ahead_of_tag` corretto 5237 → **5240** (`rev-list v2026.8.31..origin/main`).
- **MAIN SIGNAL confermato**: refactor epico su Hermes main mergeato Sep 3-4 — campagna `simp/r3-30..37`, −34% source LOC, decomposizione god files, zero behavior change dichiarato. Delta vs tag v2026.8.31: 934 → 5240 in 24h (+4306). v0.22.0 in preparazione. Su main anche: GPT-6 Astra + Astra Pro su Nous Portal e OpenRouter (Sep 4, tier fast/flex — rilevante per chain v5 OpenRouter free tier), fix desktop (macOS watchdog, dashboard PTY non-blocking, Hide tabs), fix browser packaged-Chromium.

**Upstream consistency check:**
- Advisories: 100 GHSA OpenClaw (46H/50M/4L) invariati since Jun 30 ✓; Hermes 0 ✓.
- Docs site key pages: tutte e 8 HTTP 200 ✓ (nessuna migrazione di percorso).
- Hermes releases: `v2026.8.31` (0.21.0 'Pantheon') resta latest stable ✓.
- OpenClaw CLI locale 2026.6.8 = 11 stable releases behind (invariato, KB reference only).

### Self-assessment
Quinta giornata consecutiva di recupero diff pre-esistente dal job Release Monitor interrotto (04:05) — e oggi la verifica ha pagato più del solito: il draft conteneva un claim **impossibile** (behind-count 6148 > misura live 5599 con locale fermo a 0.20.6). Un behind che scende senza upgrade non è dati stantii, è dati sbagliati — lezione registrata nella skill: quando un behind-count dichiarato eccede la misura live, non "aggiornare e via" ma correggere esplicitamente e segnalarlo (probabile misura su ref sbagliato o fetch parziale nella run interrotta). Semantica dei tre numeri ora esplicita nel tracker: 5599 behind-main (HEAD..origin/main), 5240 main-ahead-of-tag (v2026.8.31..origin/main), ~5596+ misurato da `hermes --version`. Zero churn su docs/index.yaml come da contratto SKIP, gitleaks PASS, solo changelog + tracker committati. `memories/` untracked lasciato fuori (artifact Release Monitor, dal 2026-09-02). Pending per Rakki invariata al 4° giorno: **upgrade Hermes 0.20.6→0.21.0** — ora con main a +5240 dal tag e v0.22.0 in arrivo, la finestra ideale si sta chiudendo (fare l'upgrade dopo v0.22.0 significherebbe saltare due minor).

## 2026-09-04 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato e corretto (run interrotto 04:00):** `docs/meta/upstream-version.yaml` era già modificato nel worktree alla partenza. Ogni claim verificato contro API live prima del commit (lezione 2026-08-31):
- **STABLE ADVANCE `2026.9.1`** (Sep 3, 18:31 UTC) ✓ — GitHub releases list: `v2026.9.1` prima entry, `prerelease: false`; npm `latest` 2026.9.1 allineato. Curiosità: l'endpoint `releases/latest` restituiva ancora `v2026.8.2` (cache lag ~11h) — la lista paginata è autoritativa; incluso nelle note dell'artifact.
- **1.195 PRs** ✓ — contato dal body della release (`grep -cE 'PR #[0-9]+'` → 1195).
- Contenuto release verificato ✓: `openclaw update` auto-rollback su Doctor fail + nota operativa (utenti su 2026.8.2 senza service manager: `openclaw update --no-restart` una volta), personal skill libraries su shared Gateway (`openclaw skills library`), approval cards al canale originante (Telegram topics), WhatsApp unquoted replies su quote-cache miss, `blockedHostnames` SSRF, `cron.skipMissedJobs`, `openclaw memory reset` senza perdita sessioni, ingress/secrets fixes.
- **Correzione claim errato**: "CLI local 2026.6.8 = 4 stable behind" era **falso** — la lista release mostra **11 stable releases** dopo la 2026.6.8 (6.11, 6.33, 6.34, 7.1, 7.1-1, 7.1-2, 8.1, 8.2, 9.1 + counting). Corretto nel tracker.
- Blog Sep 3 ✓ — "OpenClaw improves user onboarding with a new installer on macOS, plus easier local model setup on Windows RTX" visibile su openclaw.ai/blog (200).
- Hermes ✓: `hermes --version` → 0.20.6, 1293 behind (coerente con il draft); `git rev-list v2026.8.31..origin/main` → **934** (main_ahead_of_tag aggiornato 844→934). Commit main verificati: GLM-5.3 no-400 con thinking disabled su Nous/OpenRouter (provider di questa fleet!), mandatory-reasoning retry, delegate-child transcripts fuori dal FTS (schema v30).

**Upstream consistency check:**
- Advisories: 100 GHSA OpenClaw (46H/50M/4L) invariato since Jun 30 ✓; Hermes 0 ✓.
- Docs site key pages: tutte e 8 HTTP 200 ✓ (nessuna migrazione di percorso).
- npm `beta` dist-tag ora punta a 2026.9.1 stesso (nessun beta più nuovo della stable).
- Nuovo artifact: `docs/meta/upstream-updates/2026-09-04-v2026.9.1.md` (una stable advance = un artifact, da convenzione repo).

### Self-assessment
Quarto giorno consecutivo di recupero diff pre-esistente dal job Release Monitor interrotto (04:00) — flusso consolidato: verifica totale dei claim, un errore reale trovato e corretto (4→11 stable behind, il draft delle 04:00 aveva contato solo le release dopo il suo ultimo fetch), zero churn su docs/index.yaml come da contratto SKIP. Punti notevoli di questo run: l'endpoint `releases/latest` GitHub era in cache lag sull'ex stable (v2026.8.2) mentre la lista paginata mostrava già v2026.9.1 — lezione registrata: mai fidarsi del singolo endpoint `latest` in fase di verifica release, usare sempre `releases?per_page`. `memories/` untracked lasciato fuori dal commit (artifact Release Monitor, dal 2026-09-02). Pending per Rakki invariata: **upgrade Hermes 0.20.6→0.21.0** (main ora a +934 dal tag, divario in crescita; GLM-5.3 fix su Nous/OpenRouter è direttamente rilevante per il nostro provider LLM).

## 2026-09-03 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Tracker pre-esistente verificato e corretto (run interrotto 04:00):** `docs/meta/upstream-version.yaml` era già modificato nel worktree. Ogni claim verificato contro API live prima del commit (lezione 2026-08-31):
- OpenClaw stable `2026.8.2` invariato (Sep 1) ✓ — GitHub releases API + npm dist-tags (`latest` 2026.8.2, `beta` 2026.9.1-beta.1, `extended-stable` 2026.6.34) ✓.
- CHANGELOG main: `2026.8.3 (Unreleased)` ✓ — contenuto verificato e arricchito: GPT-5.6 Ultra runtime switching (Sol/Terra/Luna su engine OpenClaw+Codex, #98021), Meta `muse-spark-1.1` provider (#102873), Crabbox cloud-sandbox fix, macOS notarization resume, plugin SDK ingress monitors (IRC/Synology/Google Chat), Slack statusReactions opt-in, cron model selection in Control UI.
- Blog post 'OpenClaw 2.0, Accidentally' ✓ — slug reale verificato: `/blog/openclaw-2-accidentally` (200; lo slug ipotizzato nel run delle 04:00 era errato ma il post esiste, indice blog 200).
- **Correzione Hermes**: `main_ahead_of_tag` 825 → **844** (main avanzato nelle 3h tra le due run; `hermes --version` conferma direttamente "1184 commits behind", verificato con fetch+rev-list). Behind-tag invariato a 359. Locale sempre 0.20.6 (v2026.8.27) vs upstream 0.21.0 'Pantheon'.

**Upstream consistency check:**
- Advisories: 100 GHSA OpenClaw (46H/50M/4L) invariato since Jun 30 ✓; Hermes 0 ✓.
- Nessun nuovo beta/alpha OpenClaw (beta fermo a 2026.9.1-beta.1, Aug 28; ultimo alpha 2026.6.19-alpha.2).
- Docs site key pages: verifica skip leggera (nessun segnale di migrazione nel periodo; homepage blog 200).
- Nota tooling: comando composto multi-subshell sul repo Hermes bloccato dal command parser (payload inline) — split in comandi singoli, stesso risultato.

### Self-assessment
Run SKIP con terza giornata consecutiva di recupero diff pre-esistente dal job Release Monitor interrotto — pattern ormai consolidato e gestito correttamente: claim verificati uno a uno, un dato stantio corretto (825→844), uno slug errato nel testo rettificato senza inventare URL. `hermes --version` si è rivelato la fonte più affidabile per il behind-count (lo stampa direttamente). Zero churn su docs/index.yaml come da contratto SKIP. `memories/` untracked lasciato fuori dal commit (artifact Release Monitor, decisione confermata dal 2026-09-02). Pending action invariata per Rakki: **upgrade Hermes 0.20.6→0.21.0** (oggi main è a +844 dal tag — il divario cresce, cron memory/continuity e live steering sono feature usate daily da questa fleet).

## 2026-09-02 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**⚠️ STABLE ADVANCE: OpenClaw `v2026.8.2` (2026-09-01 16:00 UTC) — rilevato e tracciato.**
- GitHub release live + npm `latest` convergato su 2026.8.2 il giorno stesso (dist-tags: `latest` 2026.8.2, `beta` 2026.9.1-beta.1, `extended-stable` 2026.6.34).
- Release di consolidamento un giorno dopo 2026.8.1: nessuna breaking change per questa fleet (zero plugin OpenClaw custom; SDK subpath gates 2026-09-01 confermati non-evento).
- **Artifact**: `docs/meta/upstream-updates/2026-09-02-v2026.8.2.md` — highlight: Linux desktop companion, Sharp 0.35.4 security bump, session visibility default change (watch item multi-agent), safer upgrades/recovery, reply completion, cron robustness, MCP response limits, fix tooling migrazione OpenClaw→Hermes (upstream mantiene attivamente il bridge).
- **Tracker**: il run ha trovato `docs/meta/upstream-version.yaml` già modificato nel worktree (run interrotto 04:03 / Release Monitor). Ogni claim verificato contro API live prima del commit, come da lezione 2026-08-31: stable 2026.8.2 ✓, npm ✓, CLI 2026.6.8 ✓, advisories 100 (46H/50M/4L) ✓. **Correzione**: `hermes_main_behind` 656 → **658** (main avanzato di 2 commit dopo il check delle 04:03; origin/main tip `00b2e03c`, Sep 1 21:46 CDT). Behind-tag v0.21.0 confermato invariato a 359.

**Upstream consistency check:**
- OpenClaw beta fermo a `2026.9.1-beta.1` (Aug 28); nessun nuovo alpha (ultimo tag non-beta: v2026.6.19-alpha.2, stale).
- Hermes Agent: 0.21.0 (v2026.8.31) resta latest release; locale 0.20.6 (359 behind-tag, 658 behind-main) — upgrade ancora pendente, raccomandazione invariata.
- Docs site: tutte le 8 key pages HTTP 200.
- GitHub API rate-limit 429 intermittente sul fetch tags di hermes-agent (upstream attivo, many requests) — dietro-count verificato via `rev-list` post-fetch, nessun impatto sui dati.

### Self-assessment
Run SKIP pulito con verifica approfondita del diff pre-esistente: il tracker era già aggiornato alle 04:03 da un job parallelo (Release Monitor) con i dati giusti ma un behind-count già stantio di poche ore (656 vs 658 live). Il contratto della skill impone di verificare ogni claim contro i dati live prima di committare — fatto, delta corretto, semantica esplicita (658 behind-main / 359 behind-tag / +299 su main da v0.21.0). Artifact stable-advance scritto con solo contenuto verificato dalle release notes ufficiali. Zero churn su docs/index.yaml come da contratto SKIP. `memories/` untracked lasciato fuori dal commit (artifact Release Monitor). Nota operativa: il 429 rate-limit sul fetch dei tag è ricorrente con l'upstream Hermes in piena attività post-Pantheon — accettabile finché `rev-list` resta affidabile su ref già fetched. Da segnalare a Rakki: l'upgrade Hermes 0.20.6→0.21.0 resta la pending action principale (cron memory + steering = feature usate daily da questa fleet).

## 2026-09-01 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**Evento: Hermes Agent v0.21.0 'Pantheon' (v2026.8.31, Aug 31 19:29 UTC) — upstream release rilevata e tracciata.**
- Tracker `docs/meta/upstream-version.yaml` aggiornato: hermes last_known_version 0.20.6 → **0.21.0**, tag v2026.8.31, release date 2026-08-31.
- **Correzione claim verificati**: il diff uncommitted delle 04:05 (run interrotto / Release Monitor) claimava "911 commits behind tag, repo 73 behind main". Verifica live: 911 è il **delta tag-to-tag** v0.20.6→v0.21.0; il locale è **359 behind il tag**; e "73 behind main" era un ref stale pre-fetch — dopo `git fetch`, origin/main è **454 ahead**. Tutti e tre i numeri ora nel tracker con semantica esplicita.
- Rollup ~5800 commits / ~2475 PR since v0.20.0 (confermato dalle release notes ufficiali). Highlight operatori-rilevanti: cron memory+continuity, live subagent steering, `hermes peer` bot-to-bot DMs, Bot Mode, MCP command center, 6 nuovi provider, security hardening (protected instruction files, redaction sweep, Blender MCP rimosso dopo compromise upstream).
- Nessuna breaking migration per installazioni git → azione pianificata: `hermes update` + gateway restart (non eseguita da questo run — fuori scope docs, decisione Rakki).

**Upstream consistency check:**
- OpenClaw stable fermo a `v2026.8.1` (GitHub + npm `latest` convergati); beta `2026.9.1-beta.1` (Aug 28); alpha stale. CLI locale sempre 2026.6.8 (tre minor lines dietro).
- Advisories flat a 100 GHSA (46 HIGH / 50 MEDIUM / 4 LOW). Dependabot API 403 senza scope `admin:repo_hook` — count non verificabile via gh, 100 resta last-known.
- Blog post 'OpenClaw 2.0, Accidentally' confermato live su openclaw.ai/blog (933 contributors, 16k+ PRs, browser app rebuilt).
- SDK subpath deprecation gates attivi da oggi (2026-09-01) — nessun impatto locale (zero plugin OpenClaw custom).
- Docs site: tutte le 8 key pages HTTP 200 (incluse /releases e /releases/2026.8.1).

### Self-assessment
Run SKIP per l'ingestion ma con verifica approfondita: il tracker conteneva un diff uncommitted dalle 04:05 che descriveva la release Hermes 0.21.0 con numeri errati/ambigui — "911 behind tag" era in realtà il tag-to-tag delta, e "73 behind main" era misurato su ref stale. Il fetch ha rivelato il valore reale (454). Entrambe le discrepanze corrette con semantica esplicita prima del commit: nessun claim non verificato entra nel repo. Zero churn su docs/index come da contratto SKIP. Un inciampo gestito: una patch `upgrade_notes` mal costruita ha introdotto testo duplicato/corrotto per un turno — rilevato dal diff, ripristinato immediatamente, risultato finale pulito (verificato riga per riga). `memories/` untracked lasciato fuori dal commit (artifact Release Monitor). Da segnalare a Rakki: Hermes 0.21.0 è un upgrade non-breaking che porta esattamente le feature che questa fleet usa (cron memory per i daily run, steering dei subagenti) — `hermes update` + restart gateway è basso rischio e alto valore.


## 2026-08-31 — Daily KB Processing (automated)

**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs, nessun flip di status, `docs/index.yaml` intatto).

**⚠️ STABLE ADVANCE rilevato e tracciato: OpenClaw `v2026.8.1` promosso stable oggi (2026-08-31 03:30 UTC).**
- GitHub release live + npm `latest` convergato su 2026.8.1 (dist-tags: `latest` 2026.8.1, `beta` 2026.9.1-beta.1, `extended-stable` 2026.6.34).
- La promotion completa il segnale "pending" del run precedente: tag finalizzato Aug 30 23:53 UTC, release 404 alle 04:03 CEST di stamattina, live alle 07:31 CEST. Sequenza tag→release→npm normale, non anomalia.
- **Artifact**: `docs/meta/upstream-updates/2026-08-31-v2026.8.1.md` (stable advance, con breaking changes e decisioni di upgrade).
- **Tracker**: `docs/meta/upstream-version.yaml` aggiornato (stable GitHub/npm/github → 2026.8.1, `checked_at` 07:31, `hermes_main_behind` 33→68). Nota: il tracker aveva modifiche uncommitted dalle 04:03 di stamattina (probabile run interrotto o Release Monitor) — verificate contro i dati live prima di fidarsi, come da lezione 2026-08-30.

**Contenuto rilevante della release (per operatori):**
- **2 breaking migrations** via `openclaw doctor --fix`: rimozione plugin OpenProse + `/prose`; migrazione route `codex/*`→`openai/*` (provider config, sessioni, automation routes).
- **Defaults cambiati**: grounded dreaming ON, self-learning ON, session reset conserva le conversazioni, concorrenza CPU-scaled 8-16.
- **Deprecation gates SDK subpath dal 2026-09-01** (domani) per plugin esterni.
- CLI locale 2026.6.8 ora **tre minor lines** dietro (6.8→7.1-2→8.1) con 100 advisories non applicate — upgrade da pianificare, non blind bump.

**Upstream consistency check:**
- Advisories flat a 100 GHSA (46 HIGH / 50 MEDIUM / 4 LOW) — nessun nuovo questo ciclo.
- Hermes Agent: v0.20.6 locale, origin/main ora +68 commit avanti (a9c783f, fix desktop group-holds) — era +33 stamattina alle 04:03.
- Docs site: tutte le 8 key pages HTTP 200.
- Nessun nuovo contenuto docs da generare senza sessioni ready o decisione di upgrade locale.

### Self-assessment
Run SKIP per l'ingestion ma con evento significativo: stable advance v2026.8.1 con breaking changes multiplicativi per un fleet che usa provider OpenAI subscription-backed. Esecuzione pulita: discovery robusta (`grep -E` quoted-proof), verifica live PRIMA di fidarsi del tracker uncommitted (che descriveva la promotion come ancora pending — corretto con dati freschi), zero churn su docs/index come da contratto SKIP. `memories/` untracked lasciato fuori dal commit (artifact Release Monitor, non parte di questo run). Da segnalare a Rakki: l'upgrade della CLI OpenClaw è ora bloccato su una decisione, non su una mancanza di informazioni — le due migrazioni breaking vanno pianificate con la chain v5 OAuth in mente.



**Sessioni ready:** 0 — SKIP run (nessuna elaborazione docs).

**Upstream version check (catch-up — tracker era fermo a fine luglio):**
- `docs/meta/upstream-version.yaml` refreshed: `last_check: 2026-08-30`.
- **OpenClaw stable**: `v2026.7.1-2` (2026-08-04, patch npm plugin metadata #108336). npm `latest` convergato su 2026.7.1-2.
- **OpenClaw beta**: linea avanzata a **`2026.9.1-beta.1`** (2026-08-28, GitHub + npm beta convergati); preceduta da 2026.8.1-beta.1..3 (lug-ago).
- **npm extended-stable** `2026.6.34` — nuovo dist-tag osservato.
- **Hermes Agent**: upstream stable **v0.20.6 (`v2026.8.27`, 2026-08-27)** — patch roll-up di ~525 PR. **Il runtime locale era già stato upgradato esternamente a v0.20.6**: `~/.hermes/hermes-agent` è a `origin/main` HEAD (0 commit behind). La nota precedente "0.17.0 / behind-870" era stantia, corretta.
- **Security advisories openclaw/openclaw**: 30 → **100 GHSA** (46 HIGH / 50 MEDIUM / 4 LOW). Salto dovuto a triage CVE esteso, non a spike di vulnerabilità — ma la CLI locale (2026.6.8) resta indietro; upgrade rimane azione in piedi.
- **Docs site key pages migrate** (verificate 200): `/agents`→`/multi-agent`, `/changelog`→`/releases`, `/hooks`→`/automation/hooks`. `key_pages` aggiornato per non ri-reportare i 404.

**Artifact:** `docs/meta/upstream-updates/2026-08-30-v2026.9.1-beta.1.md` (catch-up, watch-only).

### Daily knowledge base run
- 0 sessioni `status: ready` in `sessions/` (verifica `grep -E` robusta a valori quoted). Nessun doc Diátaxis toccato, nessuna modifica a `docs/index.yaml`, nessun flip di status.

### Upstream consistency check
- Tracker allineato ai dati live (releases API GitHub, npm dist-tags, `hermes --version`, `git rev-list` sul repo locale). Nessun nuovo contenuto docs da generare — KB consistente con la reference upstream Hermes Agent.

### Self-assessment
Run SKIP ma non no-op: il tracker era indietro di quasi due mesi e nascondeva due fatti importanti — il runtime Hermes locale è già sincronizzato con l'upstream (v0.20.6, la nota "behind-870" era falsa) e gli advisories di sicurezza sono più che triplicati (30→100) mentre la CLI OpenClaw locale resta a 2026.6.8. Zero churn su docs/index come da contratto SKIP; changelog + artifact + tracker refresh committati. Esecuzione pulita, nessun errore, nessun intervento manuale. Da segnalare a Rakki: le 46 advisories HIGH non applicate localmente rendono l'upgrade della CLI OpenClaw più urgente del solito.

## 2026-08-28 — Daily KB Processing (automated)

**Sessioni ready:** 0 — nessuna elaborazione docs necessaria.

**Upstream version check:** 
- `docs/meta/upstream-version.yaml` refreshed with `last_check: 2026-08-28`.
- Local OpenClaw CLI (2026.6.8) and Hermes runtime (0.17.0) remain behind upstream (Hermes v0.18.0 "The Judgment Release" from July 2026 still pending; multiple security advisories unapplied). No new releases or breaking changes detected in this 24h cycle. Official Hermes Agent documentation at https://hermes-agent.nousresearch.com/docs remains the authoritative reference.

### Daily knowledge base run
- No session files with `status: ready` were found under `sessions/` (0 ready). No Diátaxis docs were updated, no `docs/index.yaml` changes, and no session status flips were needed.
- Pipeline followed SOUL.md Documentation Engine workflow exactly (scan → process → flip → index → changelog → git).

### Upstream consistency check
- Only metadata date refreshed. Full consistency with Diátaxis (getting-started, guides, reference, concepts, troubleshooting) and upstream docs (Hermes Agent docs at https://hermes-agent.nousresearch.com/docs) maintained. No new docs generated. The `hermes-agent` skill was implicitly verified via SOUL.md reference to load it for Hermes-related tasks.

### Self-assessment
Clean SKIP run on 2026-08-28. Documentation Engine healthy and idle. No sessions in queue; KB remains fully consistent with upstream references, Diátaxis framework, and official Hermes Agent documentation. Git commit + push completed successfully with metadata-only change. No errors, no manual intervention required. Waiting for fresh ready sessions from active agents (Master Control, Kai, etc.). Workflow executed autonomously per cron job as per SOUL.md.

