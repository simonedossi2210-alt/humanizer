# Viral OS: analisi tecnica e piano di sviluppo

> Stato: proposta, nessun codice scritto. Data: 2026-10-03.

## 1. Analisi del progetto

### 1.1 Stato del repository

Questo repository (`humanizer`) contiene solo una skill Markdown per Claude Code
(`Skills/SKILL.md`) che rimuove i pattern tipici della scrittura AI. Non c'è
codice applicativo riutilizzabile per Viral OS: il progetto parte da zero.

La skill può comunque tornare utile più avanti: il suo prompt si può riusare nel
modulo che genera caption e testi, per evitare che suonino "scritti da un'AI".

**Raccomandazione:** creare un repository dedicato (`viral-os`) prima della
Milestone 0. Questo documento sta qui solo come punto di partenza.

### 1.2 Cosa deve fare il prodotto

Viral OS è un AI Social Media Manager. Il suo nucleo è un ciclo:

```
Collega account → Raccogli post e metriche → Estrai caratteristiche dei contenuti
      → Normalizza le performance → Impara cosa funziona → Spiega e consiglia
      → (più avanti) Genera, pianifica, pubblica → nuove metriche → ...
```

Il valore non sta nella dashboard (Meta Business Suite e TikTok Studio la danno
già gratis). Sta in due cose:

1. **Feature extraction sui contenuti**: capire *com'è fatto* ogni post (hook
   dei primi 3 secondi, durata, formato, testo a schermo, volti, ritmo dei tagli,
   tema, CTA, lunghezza della caption, orario).
2. **Learning per account**: legare quelle caratteristiche alle performance
   *normalizzate* dell'account stesso, e dire con un livello di confidenza
   esplicito cosa funziona e cosa no.

### 1.3 Vincoli di fondo che guidano l'architettura

| Vincolo | Conseguenza |
|---|---|
| Le API ufficiali danno metriche complete solo per account **propri** Business/Creator | Il prodotto analizza l'account dell'utente; i competitor arrivano più tardi e con dati parziali (solo like e commenti pubblici) |
| Un account tipico ha 50–500 post | Piccoli dati: servono statistica robusta e modelli semplici, non deep learning per account |
| Le metriche crescono nel tempo dopo la pubblicazione | Bisogna salvare **snapshot nel tempo** (1h, 24h, 7g, 30g), non un solo valore |
| Le performance dipendono da follower, stagionalità e algoritmo | Serve una metrica normalizzata rispetto alla baseline mobile dell'account |
| L'analisi video multimodale costa | Pipeline asincrona a code, cache dei risultati, feature versionate |
| Gli LLM possono inventare | Gli insight devono essere generati **dai numeri calcolati**, con l'evidenza allegata |

## 2. Architettura proposta

### 2.1 Vista d'insieme

```
                    ┌─────────────────────────────┐
                    │  Web App (Next.js, TS)       │
                    │  dashboard, insight, idee    │
                    └──────────────┬──────────────┘
                                   │ API (route handlers / tRPC)
                    ┌──────────────▼──────────────┐
                    │  Core API (TS)               │
                    │  auth, account, query        │
                    └───┬───────────────┬─────────┘
                        │ enqueue       │ read/write
              ┌─────────▼─────┐   ┌─────▼──────────────────┐
              │ Queue (Redis/ │   │ Postgres + pgvector     │
              │ BullMQ)       │   │ (+ partizioni time-     │
              └───┬───────┬───┘   │  series per le metriche)│
                  │       │       └─────▲──────────▲───────┘
     ┌────────────▼──┐ ┌──▼─────────────┴──┐       │
     │ Ingestion     │ │ Analysis service   │       │
     │ Worker (TS)   │ │ (Python, FastAPI)  │───────┘
     │ - connettori  │ │ - feature extract  │
     │ - backfill    │ │ - scoring          │
     │ - snapshot    │ │ - learning/stats   │
     │   scheduler   │ │ - backtest         │
     └──────┬────────┘ └──┬──────────┬──────┘
            │             │          │
   ┌────────▼─────┐ ┌─────▼────┐ ┌───▼─────────────┐
   │ Social APIs  │ │ Object   │ │ AI providers    │
   │ IG/TT/YT     │ │ storage  │ │ Claude (vision, │
   └──────────────┘ │ (R2/S3)  │ │ insight), STT   │
                    └──────────┘ └─────────────────┘
```

### 2.2 Componenti

**Web App (Next.js + TypeScript).** Onboarding, collegamento account, dashboard
delle performance, pagina insight, idee e brief. All'inizio può ospitare anche
la Core API (route handlers), così c'è un solo deploy.

**Ingestion Worker (TypeScript).** Contiene un connettore per piattaforma dietro
un'interfaccia comune (`listPosts`, `getPostInsights`, `getAccountInsights`,
`refreshToken`). Si occupa di:
- backfill iniziale dello storico;
- sync incrementale dei nuovi post;
- **snapshot scheduler**: per ogni post rilegge le metriche a età fisse
  (1h, 6h, 24h, 72h, 7g, 30g), così si possono confrontare post di età diverse;
- download dei media su object storage, perché gli URL delle piattaforme
  scadono.

**Analysis Service (Python).** Python perché ha l'ecosistema giusto (pandas,
scikit-learn, statsmodels, ffmpeg, librerie per audio e immagini). Si occupa di:
- *Feature extraction*, a tre livelli:
  1. metadati (formato, durata, orario, lunghezza caption, hashtag, menzioni);
  2. derivate dai media (trascrizione, testo a schermo, numero di tagli, volti
     nel primo frame, presenza di musica, keyframe);
  3. semantiche via LLM multimodale (tipo di hook, tema, tono, CTA, struttura
     narrativa), in JSON con schema fisso e versionato.
- *Scoring*: calcola il Performance Index di ogni post (vedi 2.3).
- *Learning*: stima l'effetto di ogni feature sul Performance Index
  dell'account.
- *Backtest*: verifica che il modello sappia ordinare i post futuri meglio di
  una baseline.

**Insight & Recommendation layer.** Un LLM (Claude) riceve **solo** i risultati
statistici (effetti, intervalli di confidenza, esempi di post) e li traduce in
insight leggibili. Ogni insight salva l'evidenza da cui nasce e mostra un
livello di confidenza.

### 2.3 Il cuore: misurare "cosa funziona"

1. **Target normalizzato.** Per ogni post e ogni KPI (reach, views, share,
   save, follow generati):
   `PI = metrica_a_età_T / mediana_mobile_account(stesso formato, stessa età T, ultimi N post)`.
   Si lavora in log, quindi `log(PI)` vale 0 per un post nella media. Questo
   neutralizza la crescita dei follower e i cambi d'algoritmo lenti.
2. **KPI composito configurabile**, basato sull'obiettivo dell'utente (reach,
   crescita follower, engagement profondo). Per esempio
   `0.4·reach + 0.3·share + 0.2·save + 0.1·follow`, ognuno in log-PI.
3. **Modello v1, interpretabile:** regressione con regolarizzazione (ridge),
   oppure un modello gerarchico bayesiano con pooling parziale: per ogni
   feature l'effetto dell'account si "restringe" verso la media della sua
   nicchia quando i dati dell'account sono pochi. Questo risolve il cold start
   in modo onesto.
4. **Modello v2:** gradient boosting (LightGBM) con SHAP, quando ci sono
   abbastanza post o dati raggruppati per nicchia.
5. **Validazione temporale:** si allena sui post fino a una data `t` e si
   misura la correlazione di rango (Spearman) sui post successivi. Un insight
   viene mostrato solo se supera una soglia minima di campione e confidenza.
6. **Linguaggio corretto:** si parla di "associato a", non di "causa". Più
   avanti, gli esperimenti suggeriti (stesso tema con due hook diversi) danno
   evidenza quasi causale.

### 2.4 Stack consigliato

| Area | Scelta | Alternativa |
|---|---|---|
| Monorepo | pnpm + Turborepo | Nx |
| Frontend | Next.js (App Router), Tailwind, shadcn/ui | Remix |
| ORM / migrazioni | Drizzle | Prisma |
| DB | Postgres (Supabase o Neon) + pgvector | Timescale per le metriche, se servisse |
| Code | BullMQ + Redis (Upstash) | Inngest / Trigger.dev, che sono gestiti |
| Analisi | Python 3.12, FastAPI, pandas, scikit-learn, statsmodels/PyMC, LightGBM | — |
| Media | ffmpeg, PySceneDetect, OCR (PaddleOCR) | Rekognition / Video Intelligence |
| LLM | Claude API (vision, structured output, prompt caching) | — |
| Speech-to-text | Whisper (self-hosted) o un servizio gestito | — |
| Storage | Cloudflare R2 | S3 |
| Auth | Clerk o Supabase Auth | Auth.js |
| Billing | Stripe | — |
| Osservabilità | Sentry + OpenTelemetry + log strutturati | — |
| Deploy | Vercel (web) + Fly.io/Render (worker e Python) | Tutto su container (Railway, ECS) |

## 3. Struttura del database

Postgres, multi-tenant con `organization_id` su tutte le tabelle di dominio e
Row Level Security. Timestamp in UTC.

### 3.1 Tenancy e utenti

```
organizations      (id, name, plan, stripe_customer_id, created_at)
users              (id, email, name, auth_provider_id, created_at)
memberships        (organization_id, user_id, role[owner|admin|member|viewer])
subscriptions      (id, organization_id, stripe_subscription_id, status, plan, period_end)
```

### 3.2 Account social

```
social_accounts    (id, organization_id, platform[instagram|tiktok|youtube|...],
                    external_id, username, display_name, account_type,
                    niche, goal[reach|followers|engagement|sales],
                    status[active|token_expired|revoked|error],
                    connected_at, last_synced_at)
oauth_tokens       (social_account_id, access_token_enc, refresh_token_enc,
                    scopes[], expires_at, rotated_at)          -- cifrati (KMS/pgsodium)
account_snapshots  (social_account_id, captured_on DATE, followers, following,
                    media_count, reach_day, views_day, profile_views,
                    follower_delta, raw JSONB)                 -- PK (account, date)
audience_snapshots (social_account_id, captured_on, breakdown[age|gender|country|city|online_hours],
                    data JSONB)
```

### 3.3 Contenuti

```
posts              (id, social_account_id, platform_post_id UNIQUE per account,
                    format[reel|carousel|image|story|short|video|live],
                    caption, permalink, published_at, duration_sec,
                    is_paid_boosted, collab, audio_id, audio_title,
                    thumbnail_asset_id, raw JSONB, deleted_at)
post_assets        (id, post_id, kind[video|image|thumbnail|audio|frame],
                    position, storage_key, mime, width, height, duration_sec,
                    checksum, fetched_at)
hashtags           (id, tag)            post_hashtags (post_id, hashtag_id)
```

### 3.4 Metriche (time series: il volume più grande)

```
post_metric_snapshots (post_id, captured_at, age_hours,
                       views, reach, likes, comments, shares, saves,
                       profile_visits, follows, total_interactions,
                       avg_watch_time_ms, total_watch_time_ms, completion_rate,
                       replays, skip_rate, raw JSONB)
   -- partizionata per mese su captured_at; indice (post_id, age_hours)
post_metric_at_age   (post_id, age_bucket[1h|24h|72h|7d|30d], ...metriche)
   -- vista materializzata: il valore interpolato a età fisse, usato dal modello
retention_curves     (post_id, captured_at, points JSONB)  -- dove disponibile
```

### 3.5 Feature, punteggi, modelli

```
feature_extractors (id, name, version, kind[metadata|media|llm], schema JSONB, active)
post_features      (post_id, extractor_id, extracted_at, status, cost_cents,
                    values JSONB)                     -- PK (post_id, extractor_id)
post_feature_flat  (post_id, feature_key, value_num, value_text)
                                                      -- formato lungo per l'analisi
post_embeddings    (post_id, kind[caption|transcript|visual], model, vector VECTOR(n))
transcripts        (post_id, language, text, segments JSONB, provider)

performance_scores (post_id, kpi, age_bucket, baseline_value, raw_value,
                    log_pi, percentile, baseline_version, computed_at)
model_runs         (id, social_account_id | niche_id, model_type, feature_set_version,
                    trained_on_from, trained_on_to, n_posts,
                    metrics JSONB {spearman, mae, baseline_spearman},
                    artifact_key, status, created_at)
feature_effects    (model_run_id, feature_key, level, effect, ci_low, ci_high,
                    support_n, direction)
niches             (id, name, parent_id)      -- per il pooling tra account
```

### 3.6 Output verso l'utente

```
insights           (id, social_account_id, model_run_id, type[pattern|anomaly|trend|timing],
                    title, body, evidence JSONB {feature_effects, example_post_ids},
                    confidence, status[new|seen|pinned|dismissed], feedback, created_at)
recommendations    (id, social_account_id, insight_id, kind[content_idea|experiment|timing|format],
                    payload JSONB, status, created_at)
content_briefs     (id, social_account_id, title, hook, script, caption, format,
                    predicted_pi, predicted_ci, status[draft|approved|published], post_id)
experiments        (id, social_account_id, hypothesis, variant_a, variant_b,
                    post_ids[], result JSONB, status)
```

### 3.7 Operatività

```
sync_runs          (id, social_account_id, kind[backfill|incremental|snapshot],
                    started_at, finished_at, status, items, error, api_calls)
api_usage          (platform, social_account_id, window_start, calls, throttled)
llm_usage          (organization_id, feature, model, input_tokens, output_tokens, cost_cents)
audit_log          (id, organization_id, actor_user_id, action, target, at, meta)
data_deletion_requests (id, platform, external_user_id, received_at, completed_at)
                                         -- obbligatorio per le callback di Meta
```

## 4. Struttura delle cartelle

```
viral-os/
├── apps/
│   ├── web/                     # Next.js: UI + API route handlers
│   │   ├── app/
│   │   │   ├── (marketing)/
│   │   │   ├── (app)/dashboard/  accounts/  insights/  ideas/  settings/
│   │   │   └── api/             # oauth callbacks, webhooks (Meta, Stripe), tRPC
│   │   ├── components/
│   │   └── lib/
│   └── worker/                  # job runner TS (BullMQ)
│       └── src/
│           ├── jobs/            # backfill.ts, incremental-sync.ts, snapshot.ts,
│           │                    # refresh-token.ts, fetch-media.ts
│           ├── scheduler/       # pianificazione degli snapshot per età
│           └── index.ts
├── services/
│   └── analysis/                # Python
│       ├── viral_analysis/
│       │   ├── api/             # FastAPI (trigger, health)
│       │   ├── features/
│       │   │   ├── metadata.py
│       │   │   ├── media/       # scenes.py, ocr.py, faces.py, audio.py
│       │   │   ├── transcript.py
│       │   │   └── llm/         # prompts versionati + schema JSON delle feature
│       │   ├── scoring/         # baseline.py, performance_index.py
│       │   ├── learning/        # hierarchical.py, gbm.py, shap_explain.py
│       │   ├── evaluation/      # backtest.py
│       │   └── insights/        # costruzione dell'evidenza da passare all'LLM
│       ├── tests/
│       └── pyproject.toml
├── packages/
│   ├── db/                      # schema Drizzle, migrazioni, seed, RLS policy
│   ├── connectors/              # interfaccia comune + adapter
│   │   ├── src/core/            # types.ts, rate-limiter.ts, errors.ts
│   │   ├── src/instagram/
│   │   ├── src/tiktok/
│   │   └── src/youtube/
│   ├── domain/                  # tipi e regole di business condivisi (KPI, goal, piani)
│   ├── ai/                      # client Claude, prompt di insight e idee, guardrail
│   ├── queue/                   # definizioni tipizzate dei job, condivise tra web e worker
│   ├── ui/                      # design system
│   └── config/                  # eslint, tsconfig, env schema (zod)
├── infra/                       # docker-compose dev, IaC, Dockerfile
├── docs/                        # ADR, questa analisi, runbook
├── .github/workflows/           # CI: lint, typecheck, test TS e Python, migrazioni
├── turbo.json
└── pnpm-workspace.yaml
```

I contratti tra TS e Python passano dal DB e dalla coda. Gli schemi JSON
delle feature stanno in `services/analysis/.../llm/` e vengono esportati come
tipi TS durante la build.

## 5. API necessarie

### 5.1 Piattaforme social

| Piattaforma | API | Cosa dà | Note critiche |
|---|---|---|---|
| **Instagram** | Instagram API with Instagram Login *oppure* Instagram Graph API via Facebook Login | media, insight per post (views, reach, likes, comments, shares, saved, total_interactions, follows, profile_visits, metriche di watch time per i reel), insight dell'account, demografia del pubblico, `business_discovery` per i dati pubblici di altri account business | Solo account Business/Creator. Serve la **App Review di Meta** (es. `instagram_business_basic`, `instagram_business_manage_insights`). Rate limit per utente. Token long-lived di 60 giorni da rinnovare. Meta cambia e depreca metriche spesso (es. `impressions` → `views`) |
| **Facebook Pages** | Graph API | post e insight delle Pagine | Stessa App Review; utile in un secondo momento |
| **TikTok** | Login Kit + Display API; Business/Marketing API per account business | video, views, like, commenti, share; insight più ricchi solo per account business | Display API povera di metriche (niente reach e retention dettagliate). Audit app obbligatorio. Research API riservata agli accademici |
| **YouTube** | Data API v3 + YouTube Analytics API (+ Reporting API) | video, views, watch time, durata media di visione, retention, sorgenti di traffico, iscritti guadagnati | L'API più completa. Quota giornaliera in unità; serve la verifica OAuth di Google per gli scope sensibili |
| LinkedIn / X | Community Management API / X API | — | Fuori MVP: LinkedIn richiede un programma partner, X è a pagamento e costoso |
| Pubblicazione (fase avanzata) | IG Content Publishing, TikTok Content Posting API, YouTube upload | pubblicazione programmata | Altri permessi e altra review |

### 5.2 AI e media

- **Claude API**: analisi multimodale di keyframe e copertine, estrazione di
  feature semantiche in JSON strutturato, generazione degli insight a partire
  dall'evidenza, idee e brief. Il prompt caching serve per il contesto
  ricorrente dell'account.
- **Speech-to-text**: Whisper self-hosted o un servizio gestito con timestamp
  per parola, necessari per isolare il parlato dei primi 3 secondi.
- **OCR** per il testo a schermo, **scene detection** per i tagli, ffmpeg.
- **Embeddings** (testo e immagine) per raggruppare i temi e trovare post
  simili.

### 5.3 Infrastruttura e SaaS

Auth (Clerk/Supabase), Stripe (abbonamenti, webhook), email transazionali
(Resend), object storage (R2/S3), Redis, Sentry, gestione dei segreti e KMS
per cifrare i token.

### 5.4 API interne (prima bozza)

```
POST /api/accounts/connect/:platform        → avvia OAuth
GET  /api/oauth/:platform/callback
POST /api/meta/deauthorize | /api/meta/data-deletion   (callback obbligatorie)
GET  /api/accounts/:id/overview?range=
GET  /api/accounts/:id/posts?sort=pi&format=
GET  /api/posts/:id                          → metriche, curva, feature, PI
GET  /api/accounts/:id/insights
POST /api/insights/:id/feedback
GET  /api/accounts/:id/recommendations
POST /api/accounts/:id/briefs                → genera un brief guidato dagli insight
POST /api/accounts/:id/sync                  → resync manuale (con rate limit)
POST /api/webhooks/stripe
```

## 6. Rischi tecnici

| # | Rischio | Impatto | Mitigazione |
|---|---|---|---|
| R1 | **App Review Meta/TikTok** rifiutata o lenta (settimane) | Blocca il lancio | Avviarla presto (M1), con privacy policy, video demo e un caso d'uso chiaro. Sviluppare con account di test in Development mode |
| R2 | **Metriche limitate o cambiate** dalle piattaforme (deprecazioni, dati non disponibili per post vecchi o prima della conversione a Business) | Modello instabile, dashboard rotte | Salvare `raw JSONB`, mappare le metriche in un livello di normalizzazione versionato, monitorare le deprecazioni, test contrattuali sui connettori |
| R3 | **Pochi dati per account** e rumore elevato | Insight falsi o banali | Modello gerarchico con pooling per nicchia, soglie minime di supporto, intervalli di confidenza visibili, backtest obbligatorio prima di mostrare insight |
| R4 | **Correlazione ≠ causalità** (confondenti: trend, collab, sponsorizzate, orario) | Consigli sbagliati e perdita di fiducia | Controllare i confondenti noti (`is_paid_boosted`, collab, formato), linguaggio prudente, esperimenti A/B guidati |
| R5 | **Costi di analisi video e LLM** che scalano con i post | Margini negativi | Estrazione a livelli (prima i metadati gratuiti, poi media, poi LLM), keyframe invece del video intero, cache per `(post, extractor_version)`, budget per piano, `llm_usage` tracciato |
| R6 | **Allucinazioni negli insight** | Perdita di fiducia | L'LLM riceve solo statistiche calcolate, output con schema, l'evidenza è linkata e verificabile, eval automatiche sui prompt |
| R7 | **Rate limit e quote** | Sync lenti o bloccati | Rate limiter per token e piattaforma, backoff, priorità (prima i post giovani), snapshot a età fisse invece del polling continuo |
| R8 | **Token scaduti o revocati** | Dati mancanti senza che nessuno se ne accorga | Job di refresh, stato `token_expired` e notifica all'utente, gestione delle callback di deauthorize |
| R9 | **GDPR / privacy** (utente UE, dati di terzi nei commenti, demografia) | Legale | Minimizzazione dei dati, DPA con i fornitori (LLM incluso), data residency UE se possibile, cancellazione end-to-end, niente training su dati cross-tenant senza consenso |
| R10 | **Copyright e ToS sui media salvati** | Legale | Conservare solo il necessario (keyframe, trascrizione), retention limitata, **niente scraping** di account terzi |
| R11 | **Dipendenza dalle piattaforme** | Esistenziale | Multi-piattaforma dopo l'MVP, valore nell'analisi e non nell'accesso ai dati |
| R12 | **Drift dell'algoritmo** | Il modello invecchia | Baseline mobile, retraining periodico, finestre temporali pesate |
| R13 | **Coerenza time-series** (snapshot mancati, età non allineate) | PI non confrontabili | Interpolazione a età fisse e flag di qualità del dato per post |

## 7. Piano di sviluppo per milestone

Principio: **una piattaforma, fatta bene, fino al valore**. L'MVP è solo
Instagram, perché ha le metriche migliori per reel e caroselli e il pubblico
target più ampio. Le altre piattaforme arrivano dopo, sopra l'interfaccia dei
connettori. Le durate indicate valgono per 1–2 sviluppatori.

### M0 — Fondamenta (1 settimana)
- Monorepo, CI (lint, typecheck, test TS e Python), env schema, docker-compose
  di sviluppo (Postgres e Redis).
- Auth, organizzazioni, membership, RLS di base.
- Schema DB iniziale con le tabelle delle sezioni 3.1–3.4.
- Privacy policy, termini, avvio della **registrazione dell'app Meta**.
- **Done quando:** login funzionante, deploy di staging, CI verde.

### M1 — Connettore Instagram e ingestion (2 settimane)
- OAuth Instagram, token cifrati, refresh automatico, callback di deauthorize
  e data deletion.
- Backfill dello storico post, insight per post, snapshot dell'account e
  demografia.
- Snapshot scheduler per età, download dei media su R2.
- `sync_runs`, rate limiter, gestione degli errori.
- **Invio della App Review di Meta.**
- **Done quando:** un account di test ha lo storico completo e snapshot che
  si aggiornano da soli per 7 giorni senza errori.

### M2 — Dashboard e Performance Index (1,5 settimane)
- Baseline mobile e calcolo di PI e percentile per i KPI principali.
- KPI composito basato sull'obiettivo scelto in onboarding.
- UI: panoramica dell'account, lista dei post ordinabile per PI, dettaglio del
  post con la curva di crescita.
- **Done quando:** l'utente vede quali post hanno sovra- o sotto-performato
  rispetto alla *sua* media, a parità di età del post.

### M3 — Feature extraction (2,5 settimane)
- Livello 1 (metadati), livello 2 (trascrizione, OCR, scene, volti, keyframe),
  livello 3 (feature semantiche LLM con schema JSON versionato).
- Registro degli extractor, versioning, tracciamento dei costi, riesecuzione
  selettiva.
- Set di valutazione: circa 100 post etichettati a mano per misurare
  l'accuratezza delle feature LLM.
- **Done quando:** ogni post ha le feature complete, l'accuratezza delle
  feature LLM sul set di valutazione supera una soglia concordata, e il costo
  per post è misurato e sotto budget.

### M4 — Learning engine v1 (2,5 settimane)
- Dataset per account (feature + log-PI a età fissa).
- Modello interpretabile (ridge, poi gerarchico con pooling per nicchia).
- Backtest temporale e confronto con una baseline ingenua.
- Tabella `feature_effects` con intervalli di confidenza e supporto.
- **Done quando:** su account reali di beta, la Spearman nel backtest batte la
  baseline in modo consistente, e gli effetti sono stabili tra un retraining e
  l'altro.

### M5 — Insight e raccomandazioni (2 settimane)
- Generazione degli insight dall'evidenza statistica (Claude, output
  strutturato), filtrati per confidenza e supporto.
- UI degli insight con l'evidenza cliccabile (i post d'esempio), feedback
  👍/👎.
- Raccomandazioni: formato, durata, tipo di hook, orari, temi; 3 idee di
  contenuto con brief.
- Eval automatiche dei prompt (fedeltà all'evidenza).
- **Done quando:** ogni insight è tracciabile ai numeri che lo sostengono, e i
  beta tester lo giudicano utile in almeno il 60% dei casi.

### M6 — Beta privata pronta al pagamento (1,5 settimane)
- Stripe (piani e limiti per numero di account e post analizzati), onboarding
  guidato, email (report settimanale).
- Hardening GDPR: export e cancellazione dati, DPA, retention.
- Osservabilità, alerting, runbook.
- **Done quando:** App Review approvata, 10–20 account beta attivi, costo per
  account monitorato.

### M7 — Multi-piattaforma (3 settimane)
- Connettori YouTube (Shorts e video lunghi, con retention) e TikTok.
- Normalizzazione delle metriche tra piattaforme; modelli per piattaforma.
- **Done quando:** stesso flusso M1–M5 funzionante su YouTube e TikTok.

### M8 — Da analista a manager (in seguito)
- Punteggio predittivo pre-pubblicazione: l'utente carica una bozza e riceve
  il PI previsto con intervallo.
- Esperimenti A/B guidati, calendario editoriale, generazione di script e
  caption (riusando le regole della skill humanizer).
- Pubblicazione programmata, competitor benchmark via `business_discovery`,
  analisi dei commenti.

**Totale MVP (M0–M6), solo Instagram:** circa 13 settimane.

## 8. Decisioni aperte (da chiudere prima di M0)

1. **Target primario:** singoli creator, PMI o agenzie (multi-account)? Cambia
   onboarding, pricing e tenancy.
2. **Piattaforma di partenza:** Instagram (proposta) oppure TikTok o YouTube?
3. **Lingua dei contenuti:** solo italiano all'inizio, o multilingua? Incide
   su STT e prompt.
4. **Stack:** ok con TS + Python, o preferisci un solo linguaggio (tutto TS,
   con ML più limitato)?
5. **Hosting e data residency:** UE obbligatoria?
6. **Budget di costo variabile per account al mese** (vincola la profondità
   dell'analisi video).
