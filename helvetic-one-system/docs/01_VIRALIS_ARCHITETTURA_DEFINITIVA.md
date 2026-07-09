# VIRALIS — Architettura Definitiva
### Sistema SMM Multi-Agente — Fusione smm.kill + awesome-llm-apps + JAVIS

---

## 0. Principio guida

Non esiste un sistema "finito". Questo documento fissa lo **stato dell'arte oggi** (luglio 2026) integrando:
- lo stack VIRALIS/smm.kill esistente (agenti S.ANALISTA, S.STRATEGIST, S.CREATOR, S.EDITOR, S.METRICS su Supabase + n8n)
- i pattern migliori estratti da `awesome-llm-apps`
- l'infrastruttura JAVIS (Ollama locale + MCP + Qdrant)

Filosofia invariata: **zero-cost/local-first, 30-50 CHF/mese**, Helvetic.one come pilota.

---

## 1. Architettura a 4 layer (Intelligence → Creation → Distribution → Feedback)

```
┌─────────────────────────────────────────────────────────────────┐
│                         JAVIS (control center)                    │
│         Ollama locale + MCP router + Qdrant + n8n orchestrator    │
└─────────────────────────────────────────────────────────────────┘
        │              │               │               │
   INTELLIGENCE    CREATION       DISTRIBUTION      FEEDBACK
```

### 1.1 INTELLIGENCE (potenziamento del tuo layer più debole)

| Agente | Ruolo | Fonte pattern | Sostituisce/integra |
|---|---|---|---|
| S.ANALISTA (esiste) | analisi trend/competitor | — | mantieni |
| **S.SIGNAL** (nuovo) | signal intelligence multi-fonte | `devpulse_ai` | evolve il tuo AI Signal Parser (arXiv/HN/GitHub) aggiungendo social signals |
| **S.COMPETITOR** (nuovo) | competitor tracking automatico | `ai_competitor_intelligence_agent_team` | monitoraggio SCS Camper + concorrenti Helvetic.one in continuo |
| **S.LEAD** (nuovo) | lead scoring | `ai_sales_intelligence_agent_team` | qualifica automatica SMB target |

Output: tutti scrivono su Supabase in una tabella `signals` condivisa, taggata per priorità e categoria — S.STRATEGIST la legge per decidere i contenuti.

### 1.2 CREATION (il layer con più leva immediata)

| Agente | Ruolo | Fonte pattern | Note |
|---|---|---|---|
| S.CREATOR (esiste) | genera script/copy | — | fonde con skill `content-creator` del repo |
| **S.VISUAL** (nuovo) | redesign foto prima/dopo | `ai_home_renovation_agent` (Nano Banana Pro) | applicato a conversioni camper: prima/dopo cantiere |
| S.EDITOR (esiste) | revisione qualità | — | integra pattern `multimodal_uiux_feedback_agent_team` per QA visivo automatico |
| **S.MEME** (nuovo, opzionale) | contenuto rapido low-effort | `ai_meme_generator_agent_browseruse` | riempitivo per cadenza pubblicazione |

Motore video: **OpenShorts resta il core**, non sostituito — `ai_movie_production_agent` va solo studiato come benchmark pipeline, non integrato (ridondante).

### 1.3 DISTRIBUTION (orchestrazione e routing)

- **MCP Router centrale**: pattern `multi_mcp_agent_router` — un solo punto che smista le richieste tra Notion MCP (Secondo Cervello), GitHub MCP (repo VIRALIS), e i connettori social. Questo è il pezzo che oggi in JAVIS manca come layer esplicito: va costruito prima di aggiungere altri MCP.
- Pubblicazione multi-piattaforma resta su n8n, invariato.

### 1.4 FEEDBACK (memoria e apprendimento)

- **S.METRICS (esiste)** → alimenta Secondo Cervello.
- **Memoria multi-LLM condivisa**: pattern `multi_llm_memory` — un unico store (Qdrant, già in JAVIS) letto sia dai modelli locali (qwen2.5-coder, llama3.1) sia da Claude in cloud, così S.STRATEGIST "ricorda" cosa ha funzionato indipendentemente dal modello usato quel giorno.
- **Ottimizzazione token**: `headroom_context_optimization` + `toonify_token_optimization` — applicati ai prompt di S.ANALISTA/S.SIGNAL che processano grandi volumi di segnali, per restare dentro budget.
- **Self-improving skills**: pattern `self-improving-agent-skills` — i prompt degli agenti si auto-correggono in base ai risultati di S.METRICS, chiudendo il loop.

---

## 2. Cosa NON integrare (per disciplina, non per pigrizia)

- CrewAI vs Google ADK: non migrare adesso. Il crash course ADK va solo studiato come opzione futura se CrewAI diventa un collo di bottiglia.
- `ai_movie_production_agent`: ridondante con OpenShorts, rischio di reintrodurre il pattern "infrastruttura prima delle vendite" già segnalato in passato.
- Qualsiasi agente enterprise-scale (multi-tenant, RBAC complesso): fuori scope per un pilota mono-cliente.

---

## 3. Roadmap (4 fasi)

**Fase 0 — Fondamenta MCP (1-2 settimane)**
Costruire il MCP Router centrale in JAVIS. Senza questo, ogni integrazione successiva è debito tecnico.

**Fase 1 — Intelligence upgrade (2-3 settimane)**
S.SIGNAL + S.COMPETITOR + tabella `signals` su Supabase. Validare con dati reali su Helvetic.one prima di procedere.

**Fase 2 — Creation upgrade (2-3 settimane)**
S.VISUAL (redesign prima/dopo) + fusione skill `content-creator`. Output: primo batch di contenuti generati col nuovo stack.

**Fase 3 — Feedback loop (1-2 settimane)**
Memoria condivisa Qdrant + ottimizzazione token + self-improving skills. Qui il sistema comincia a migliorarsi da solo.

**Fase 4 — Cannibalizzazione skill pack**
19 skill di `awesome_agent_skills/` riscritte in versione VIRALIS (deliverable successivo, vedi sotto).

---

## 4. Prossimi deliverable (in sequenza, come richiesto)

1. ✅ Questo documento
2. ⏭️ Skill pack Claude Code (.md) — riscrittura mirata delle skill del repo per VIRALIS
3. ⏭️ Workflow n8n (JSON) — il MCP Router + tabella signals come primo workflow eseguibile

---

## Nota di onestà

Questo è il miglior sistema costruibile **con le informazioni e i pattern disponibili oggi**. Andrà rivisto quando: (a) validi Fase 1 con dati reali, (b) il repo awesome-llm-apps si aggiorna, (c) i costi token cambiano. Nessun sistema SMM è "finito" — quello vincente è quello che chiude il loop feedback più veloce dei concorrenti.
