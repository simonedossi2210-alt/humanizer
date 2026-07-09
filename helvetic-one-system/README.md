# Helvetic.one — Sistema SMM (VIRALIS, nicchia camper/vanlife)

Pacchetto pronto da droppare nel repo `viral-social-awesome-skills` (o equivalente).

## Struttura

```
helvetic-one-system/
├── docs/
│   └── 01_VIRALIS_ARCHITETTURA_DEFINITIVA.md   ← visione, 4 layer, roadmap
├── skills/
│   ├── s-creator.md       ← genera contenuti (nicchia camper integrata)
│   ├── s-strategist.md    ← prioritizza cosa creare (calendario stagionale integrato)
│   ├── s-editor.md        ← QA prima della pubblicazione
│   └── s-metrics.md       ← chiude il loop feedback, skill self-improving
└── workflows/
    └── VIRALIS_S-SIGNAL_router_camper-vanlife.json   ← n8n, raccoglie segnali ogni 6h
```

## Come installarlo (da telefono)

1. **Skill** → app GitHub, entra nel repo, crea cartella `.claude/skills/` (o dove Claude Code le legge nel tuo setup), incolla ogni file `.md` di `skills/`.
2. **Workflow n8n** → apri n8n via browser mobile, *Import from File*, carichi `VIRALIS_S-SIGNAL_router_camper-vanlife.json`. Poi:
   - colleghi le credenziali Supabase al posto di `PLACEHOLDER_CREDENTIAL_ID`
   - colleghi Notion API key al posto dell'altro `PLACEHOLDER_CREDENTIAL_ID`
   - inserisci l'ID del tuo database Notion al posto di `PLACEHOLDER_NOTION_DB_ID`
   - attivi il workflow
3. **Doc architetturale** → va in `docs/` del repo come riferimento, non richiede setup.

## Prerequisito su Supabase

La tabella `signals` deve esistere con almeno queste colonne: `source` (text), `title` (text), `url` (text), `category` (text), `priority` (text), `raw_score` (int, nullable), `created_at` (timestamp). Se non esiste ancora, va creata prima di attivare il workflow.

## Stato

Fase 0-1 della roadmap (fondamenta + intelligence). Fase 2 (S.VISUAL, creation upgrade) e Fase 3 (feedback loop/memoria condivisa) restano da costruire quando questa parte è validata con dati reali.
