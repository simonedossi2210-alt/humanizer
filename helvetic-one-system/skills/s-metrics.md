---
name: s-metrics
description: Analizza le performance dei contenuti pubblicati e aggiorna le regole delle altre skill (S.CREATOR, S.STRATEGIST) in base a cosa ha funzionato davvero. Usala dopo ogni ciclo di pubblicazione per chiudere il loop feedback.
---

# S.METRICS — Self-Improving Feedback Skill

## Quando usarla
- Dopo ogni ciclo di pubblicazione (settimanale consigliato).
- Quando un contenuto ha performance anomale (molto sopra o molto sotto media) e serve capire perché.

## Input richiesti
1. Metriche contenuti pubblicati (views, retention, engagement, conversioni se tracciabili)
2. Contenuto originale + versione approvata da S.EDITOR
3. Segnale/brief di origine (da S.STRATEGIST)

## Processo
1. **Isolare la variabile**: confronta contenuti simili per formato/piattaforma, identifica cosa cambia tra i top e bottom performer (hook? timing? claim? visual?).
2. **Non correlazione spuria**: se il campione è < 5 contenuti simili, segnala esplicitamente "dato insufficiente" invece di generare una regola.
3. **Scrivere la regola**: se il pattern è confermato, formula una regola concreta e verificabile (es. "hook con domanda diretta +40% retention primi 3s su TikTok Helvetic.one").
4. **Propagazione**: la regola va aggiunta come nota alla skill S.CREATOR (sezione "Regole apprese") — non sovrascrive le regole esistenti, si accumula.

## Regola anti-drift
Ogni regola appresa deve avere: data, campione (N contenuti), metrica di riferimento. Regole senza questi 3 dati non vengono propagate — evita che il sistema si convinca di pattern falsi.

## Output atteso
1. Report performance ciclo (tabella)
2. Lista regole nuove/aggiornate da aggiungere a S.CREATOR, con evidenza a supporto
3. Flag esplicito se serve intervento umano (Simo) prima di propagare — es. se la regola contraddice il tono di brand
