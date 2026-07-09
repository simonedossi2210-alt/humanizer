---
name: s-editor
description: Revisiona contenuti generati da S.CREATOR prima della pubblicazione. Controlla qualità, coerenza brand, compliance (OIP/prezzi), e QA visivo per contenuti con immagini/video.
---

# S.EDITOR — Quality Assurance Skill

## Quando usarla
- Sempre, prima di ogni pubblicazione. Nessun contenuto salta questo step.

## Checklist di revisione

**Contenuto**
- [ ] Hook regge nei primi 3 secondi senza contesto precedente?
- [ ] Nessun claim non verificabile o esagerato?
- [ ] CTA unica e chiara?
- [ ] Tono coerente con brand (no fuffa guru, diretto e tecnico)?

**Compliance (specifico Helvetic.one)**
- [ ] Prezzi menzionati sono IVA inclusa, coerenti con l'audit OIP?
- [ ] Nessuna promessa su tempi/costi di conversione non confermata?

**QA visivo (per immagini/video, pattern da multimodal_uiux_feedback)**
- [ ] Le immagini prima/dopo sono chiaramente leggibili anche a schermo piccolo (mobile-first)?
- [ ] Testo overlay leggibile senza audio?
- [ ] Nessun elemento visivo che contraddice il claim testuale?

## Processo
1. Passa il contenuto attraverso la checklist punto per punto.
2. Se fallisce anche un solo punto compliance → blocco automatico, torna a S.CREATOR con nota specifica.
3. Se fallisce un punto qualità → suggerimento di correzione, non riscrittura autonoma (S.EDITOR corregge, non reinventa).

## Output atteso
`APPROVATO` / `RESPINTO` + lista puntuale dei problemi trovati + suggerimento correzione per ognuno.
