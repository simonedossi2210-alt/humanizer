---
name: s-strategist
description: Decide cosa creare e quando per Helvetic.one (nicchia camper/vanlife), leggendo i segnali da S.SIGNAL. Usala per trasformare dati grezzi in un piano contenuti prioritizzato.
---

# S.STRATEGIST — Decision Skill (Helvetic.one / Camper-Vanlife)

## Quando usarla
- Hai un batch di segnali (Reddit vandwellers/campervans, mosse di SCS Camper) e serve decidere priorità.
- Serve un piano contenuti settimanale/mensile.

## Input richiesti
1. Tabella `signals` (o estratto) da Supabase
2. Capacità di produzione disponibile
3. Stagione corrente (vedi calendario sotto — pesa la priorità)

## Prioritizzazione (3 assi)
1. **Urgenza** (finestra temporale del trend/mossa competitor)
2. **Fit stagionale**: Feb-Apr e Mag-Set = priorità alta di default (prep primavera / stagione viaggi); Ott-Gen = priorità media, buon momento per contenuti tecnici/authority.
3. **Leva business** (lead caldi > awareness generica)

## Competitor da monitorare
SCS Camper (competitor primario, monitoraggio automatico via workflow n8n). Altri player Ticino/Svizzera italiana se emergono dai segnali Reddit.

## Regola anti-pattern
Non aggiungere infrastruttura o nuovi formati prima di aver validato quelli esistenti. Un segnale che richiede un formato mai testato → proponilo come test A/B piccolo, non come pivot.

## Output atteso
Tabella: `Priorità | Segnale | Formato | Piattaforma | Deadline | Owner (agente)`
Max 5-7 item per ciclo, resto in backlog esplicito.
