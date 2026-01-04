---
stepsCompleted: [1, 2, 3, 4, 5]
inputDocuments:
  - project_1/brief/design-harness.md
  - project_1/brief/harness-dna-graph.json
  - project_1/brief/README.md
workflowType: 'product-brief'
lastStep: 5
project_name: 'Linear-Coding-Agent-Harness'
user_name: 'User'
date: '2025-01-21'
---

# Product Brief: Linear-Coding-Agent-Harness

**Date:** 2025-01-21
**Author:** User

---

## Executive Summary

**Linear Coding Agent Harness** è un sistema di orchestrazione resiliente che trasforma gli agenti Claude da esecutori lineari vulnerabili a sistemi autonomi capaci di auto-correzione. Il sistema risolve il problema critico della perdita di contesto e degli errori distruttivi nei progetti long-running, introducendo persistenza esterna dello stato, rollback automatico e isolamento rigoroso dei compiti tramite sub-agenti specializzati.

---

## Core Vision

### Problem Statement

Gli harness attuali basati su Claude Agent SDK sono vulnerabili perché affidano lo stato del lavoro esclusivamente alla memoria volatile del contesto della chat. Questo causa:

- **Perdita di contesto** in sessioni lunghe (ore/giorni), lasciando il codice in stati incoerenti "half-implemented"
- **Errori distruttivi** senza recupero automatico, richiedendo interventi manuali o riavvii completi
- **Degradazione delle prestazioni** quando i sub-agenti falliscono senza meccanismi di rollback
- **Spreco di token e tempo** per ri-analizzare il contesto invece di progredire sul task

### Problem Impact

Il costo operativo si manifesta in:

- **Tempo perso** per diagnosticare e riparare applicazioni lasciate in stati incoerenti
- **Costo economico diretto** per token sprecati in ri-analisi invece di progresso
- **Impossibilità** di sessioni di lavoro continue su progetti complessi che richiedono giorni di sviluppo
- **Mancanza di garanzie** che il codice "completato" sia effettivamente testato e funzionante

### Why Existing Solutions Fall Short

Le soluzioni esistenti falliscono perché:

- La sola **compattazione del contesto** genera riassunti che degradano le prestazioni nel tempo
- Mancano meccanismi nativi di **"undo" a livello filesystem** per ripristinare stati sicuri
- Non integrano coesamente **gestione progetto (Linear), memoria persistente e sicurezza**
- Non separano nettamente **responsabilità tra codice harness (configurazione) e comportamento agente (prompting)**
- Mancano **quality gates rigorosi** che impediscano il merge di codice non verificato

### Proposed Solution

**Sistema Autonomo Resiliente** fondato su quattro pilastri:

1. **Persistenza**: Stato del lavoro salvato esternamente in Linear (MCP) e decisioni tecniche nel Memory Tool
2. **Sicurezza**: File checkpointing nativo SDK con rollback automatico (`rewind_files`) quando i test falliscono
3. **Isolamento**: Sub-agenti programmatici specializzati (Coding, QA, Docs) con permessi controllati
4. **Gestione Contesto**: Tool Result Clearing server-side + Memory Tool per sessioni infinite

Il momento "aha!" è lo spostamento dello stato operativo dal prompt volatile a un sistema esterno robusto (Linear MCP) e l'adozione di un ciclo di vita ciclico auto-correttivo: **Inizializzazione → Esecuzione → Verifica (Gatekeeper) → Chiusura**, con un "Safety Net" attivo che cattura errori prima che diventino permanenti.

### Key Differentiators

**Vantaggio competitivo:**
- Uso nativo e profondo delle funzionalità avanzate dell'SDK (File Checkpointing, Rewind, Context Management)
- Integrazione programmatica con logica di business esterna (Linear) gestita dal codice Python dell'harness
- Sub-agenti definiti via codice (`AgentDefinition`) per controllo dinamico e isolamento superiore
- Separazione netta tra infrastruttura (codice harness) e intelligenza (prompting agente)

**Difficoltà di replica:**
- Orchestrazione precisa di feature beta avanzate (`context-management`, `mcp-client`) che devono lavorare in concerto
- Logica di "Prioritizzazione Assoluta" per issue In-Progress gestita programmaticamente, non dall'agente
- Design architetturale che separa responsabilità tra configurazione tecnica e comportamento operativo

**Timing:**
- SDK Claude ha reso disponibili nativamente le primitive necessarie (`rewind_files`, `enable_file_checkpointing`, Tool Result Clearing)
- Modelli Claude 4.5 Sonnet/Opus con capacità di ragionamento estese abilitano sub-agenti specializzati affidabili come QA Specialist

---

## Target Users

### Primary Users

#### Alex - Senior Software Engineer / Tech Lead

**Contesto:**
Alex è un Senior Software Engineer o Tech Lead in un team agile. Ha un background tecnico solido (Python/Node.js) e lavora su progetti complessi che richiedono refactoring, nuove feature e manutenzione continua. Usa già strumenti come Linear per il project management e git per il versionamento.

**Esperienza del Problema:**
Alex usa strumenti AI ma è frustrato dalla loro fragilità. Vive il problema della "memoria volatile": se assegna all'agente un compito lungo (es. "rifai l'autenticazione"), l'agente spesso esaurisce il contesto a metà, lasciando il codice in uno stato rotto ("half-implemented") o allucinando di aver finito.

I suoi workaround attuali sono costosi: deve fare "babysitting" all'agente, spezzettare i task in micro-prompt manuali e intervenire fisicamente per revertire i file quando l'agente commette errori distruttivi.

**Visione del Successo:**
Per Alex, il successo non è un agente che scrive codice più velocemente, ma un agente che **non rompe la build**.

- **Il momento "Aha!":** Avviene quando Alex vede che un test è fallito durante la notte, ma invece di trovare il progetto rotto al mattino, scopre che l'harness ha eseguito automaticamente un `rewind_files()`, ha ripristinato lo stato sicuro e ha provato una strategia alternativa o ha chiesto aiuto senza distruggere il lavoro fatto.
- **Valore:** La capacità di delegare un ticket Linear complesso sapendo che il sistema ha una "Safety Net" attiva.

**Trasformazione del Ruolo:**
Il passaggio dall'harness attuale a quello proposto trasforma il lavoro di Alex da **"Operatore di Chatbot"** (scrivere prompt, correggere output, incollare file) a **"Gestore di Flusso"** (definire requisiti in Linear, revisionare PR, gestire eccezioni di alto livello). L'harness diventa un "impiegato virtuale" che lavora su turni, dotato di memoria a lungo termine e capacità di auto-correzione.

### Secondary Users

#### Product Owner / Engineering Manager

**Beneficio:**
Visibilità trasparente. Non devono chiedere "a che punto è l'agente?"; vedono lo stato dei ticket aggiornarsi in tempo reale su Linear (In Progress → Done) con commenti e link ai commit.

**Ruolo:**
Oversight. Possono bloccare un task o cambiare priorità semplicemente spostando un ticket in Linear, senza dover interagire via terminale con l'agente.

#### QA Engineer / Reviewer

**Beneficio:**
Ricevono Pull Request più pulite. Grazie al "QA Specialist" (sub-agente isolato) che esegue i test prima di dichiarare il lavoro finito, il codice che arriva alla revisione umana ha già passato una barriera di qualità automatizzata.

### User Journey

#### Fase 1: Onboarding e Configurazione (Setup)

Alex clona la repo dell'harness.

1. **Configurazione Ambientale:** Imposta `LINEAR_API_KEY` e le variabili per `file_checkpointing`.
2. **Preflight Check:** Lancia l'harness. Il sistema esegue immediatamente una sequenza di controlli ("Preflight Checklist"): verifica la presenza di Python, Node.js, git e la connessione a Linear.
   - **Valore immediato:** Se manca qualcosa, riceve un errore deterministico e azionabile (es. "Manca Claude Code CLI"), invece di un crash oscuro a metà esecuzione.

#### Fase 2: Attivazione del Workflow (Day-to-Day)

Invece di scrivere un lungo prompt in chat, Alex va su Linear.

1. **Input:** Crea un ticket Linear: "Implementare endpoint API per login utenti" con i Criteri di Accettazione. Lo lascia in stato "Todo".
2. **Esecuzione:** Lancia l'harness (o questo gira su un server/container persistente).
3. **Prioritizzazione Automatica:** L'harness interroga Linear. Nota che non ci sono issue "In Progress" interrotte (che avrebbero precedenza assoluta) e seleziona il ticket "Todo" appena creato.

#### Fase 3: Monitoraggio e Intervento (Il Loop Autonomo)

Mentre Alex fa altro, l'harness lavora ciclicamente:

1. **Inizializzazione:** L'Agente Principale legge il ticket e pianifica il lavoro, salvando il piano nel Memory Tool per persistenza.
2. **Coding & Safety Net:** Il "Coding Specialist" scrive il codice. L'SDK crea checkpoint automatici dei file prima delle modifiche.
3. **Gatekeeping:** Il "QA Specialist" esegue i test.
   - **Scenario Errore:** I test falliscono. L'harness intercetta l'errore, esegue il rollback dei file al checkpoint precedente e l'Agente Principale riprova con un approccio diverso.
4. **Chiusura:** Se i test passano, il "Docs Specialist" aggiorna la documentazione e l'harness marca il ticket Linear come "Done".

#### Fase 4: Percezione del Valore

Alex riceve una notifica da Linear: "Ticket completato". Controlla il codice:

- I file sono puliti (grazie al rollback degli errori intermedi).
- C'è un file `feature_list.json` o una memoria persistente aggiornata che traccia il progresso.
- Non deve passare ore a "riparare" ciò che l'agente ha rotto.

---

## Success Metrics

### User Success Metrics

Per **Alex (Senior Software Engineer / Tech Lead)**, il successo non è la velocità di scrittura del codice, ma la **fiducia** nel delegare.

**Momenti di valore che portano a dire "ne è valsa la pena":**

- **Il "Zero-Repair Morning":** Alex arriva al lavoro e trova che l'agente ha tentato un task complesso durante la notte, ha fallito un test, ha eseguito automaticamente un `rewind_files()` per annullare le modifiche rotte, e ha lasciato un commento su Linear spiegando il blocco, mantenendo il resto del progetto intatto.

- **La PR "Green":** Ricevere una Pull Request dove il *QA Specialist* (sub-agente) ha già eseguito e validato i test, garantendo che il codice sia conforme alla "Definition of Done" prima ancora che Alex lo guardi.

**Il momento "aha!" di risoluzione del problema:**
È il momento in cui l'harness intercetta un errore distruttivo (es. cancellazione accidentale di config) e lo corregge autonomamente usando il checkpointing, senza che Alex debba intervenire manualmente via git per ripristinare lo stato.

**Sintesi del successo per Alex:**
Il successo è raggiunto quando Alex può assegnare un ticket Linear taggato `agent-task`, andare a dormire, e la mattina successiva trovare o una PR verde pronta per il merge, o il ticket ancora "In Progress" con un commento dettagliato dell'agente che spiega un blocco tecnico, senza che nessuna riga di codice sia stata corrotta nel processo.

### Business Objectives

#### Successo a 3 Mesi (Fase: Stabilità & Fiducia)

**Obiettivo:** L'harness è "safe". Non rompe mai il progetto.

**Metriche chiave:**
- Rollback automatico funzionante nel 100% dei casi di test falliti
- Integrazione Linear MCP stabile: lo stato dei ticket è sempre sincronizzato con la realtà
- Alex usa l'harness quotidianamente per task di refactoring e manutenzione

#### Successo a 12 Mesi (Fase: Scalabilità & Autonomia)

**Obiettivo:** L'harness è un "collaboratore autonomo". Gestisce intere epic.

**Metriche chiave:**
- Capacità di gestire sessioni multi-giorno (grazie a Memory Tool e persistenza) senza degradazione delle performance
- Riduzione del 50% del tempo speso dal team su bug fix triviali e setup di boilerplate
- Adozione da parte di utenti secondari (es. Product Owner che aprono ticket sapendo che verranno gestiti)

### Key Performance Indicators

#### A. Metriche di Resilienza e Qualità (Strategic - Il "Safety Net")

Queste metriche validano direttamente l'architettura di *File Checkpointing* e *Rollback* definita nel design.

1. **Tasso di Ripristino Automatico (Self-Correction Rate):**
   - **Definizione:** Percentuale di errori (es. test falliti, eccezioni) gestiti autonomamente dall'harness tramite `client.rewind_files()` rispetto a quelli che hanno richiesto intervento umano.
   - **Obiettivo:** > 90% degli errori tecnici durante il coding loop vengono revertiti automaticamente senza rompere l'ambiente.

2. **Integrità della Build (Build Stability):**
   - **Definizione:** Percentuale di ticket marcati come "Done" dall'agente che passano effettivamente la CI/CD pipeline principale al primo colpo.
   - **Obiettivo:** 100%. Il *QA Specialist* deve agire da gatekeeper; se la build è rossa, il ticket non deve mai passare a "Done".

3. **Tasso di Regressione:**
   - **Definizione:** Numero di funzionalità esistenti rotte dall'implementazione di una nuova feature.
   - **Obiettivo:** 0 (Grazie all'isolamento del contesto e ai test di regressione del QA Specialist).

#### B. Metriche di Engagement e Autonomia (Operational)

Misurano quanto l'harness libera Alex dal "babysitting".

4. **Intervention Ratio (Rapporto Interventi):**
   - **Definizione:** Numero di input umani richiesti per completare un ticket Linear di media complessità.
   - **Obiettivo:** < 1 per ticket (idealmente solo la definizione iniziale e la review finale).

5. **Context Efficiency (Efficienza della Memoria):**
   - **Definizione:** Efficacia del *Memory Tool* e del *Context Clearing* nel preservare le informazioni critiche. Misurabile tramite il numero di volte che l'agente deve "ri-chiedere" informazioni già fornite o presenti nel progetto.
   - **Obiettivo:** L'agente non chiede mai due volte la stessa specifica architetturale presente nel Memory Tool.

#### C. Metriche Finanziarie (Efficiency)

6. **Costo per Story Point (Cost per Unit of Work):**
   - **Definizione:** Costo API (input/output tokens) medio per completare un ticket, normalizzato per complessità.
   - **Insight:** Monitorare se l'uso di *Context Compaction* e *Result Clearing* mantiene i costi stabili anche in sessioni lunghe ("Long-running agents"), evitando l'esplosione esponenziale dei costi dovuta alla saturazione del contesto.

---

## MVP Scope

### Core Features

**Filosofia MVP: "Safety First, Autonomy Second"**

L'MVP mira alla robustezza, non a sessioni "infinite" o intelligenza suprema. Criterio decisionale: *Se l'agente sbaglia, il progetto sopravvive?*

#### A. Il "Safety Net" (Checkpointing & Rollback) - CRITICA

Funzionalità core. L'harness deve poter annullare le modifiche se i test falliscono.

- **Requisito Tecnico:** Abilitare `enable_file_checkpointing=True` e `permission_mode="acceptEdits"` (per evitare blocchi su ogni file).
- **Logica:** Prima di ogni iterazione del *Coding Agent*, salvare il `message.uuid`. Se il *QA Agent* riporta fallimento, eseguire `client.rewind_files(uuid)`.
- **Valore Utente:** L'utente vede i file cambiare e poi tornare indietro magicamente se il codice è rotto.

#### B. Il "Gatekeeper" (QA Sub-agent) - ALTA

Non basta scrivere codice; serve qualcuno che dica "NO".

- **Requisito Tecnico:** Definire programmaticamente un sub-agente `QA Specialist` tramite `AgentDefinition`.
- **Configurazione:** Questo agente deve avere accesso ai tool di test (`bash` per pytest/npm test) ma, crucialmente, istruzioni rigide di **non modificare** il codice di produzione, solo i file di test se necessario.

#### C. Integrazione Linear "Read-Only + Status Update" - ALTA

Per evitare il "babysitting", l'agente deve sapere cosa fare da solo.

- **Requisito Tecnico:** Configurare `mcp_servers` per connettersi a Linear.
- **Logica Semplificata:**
  1. L'Harness (Python) interroga Linear per ticket con tag "Ready".
  2. Passa il contesto del ticket al *Main Agent*.
  3. A fine lavoro, l'agente usa il tool per marcare il ticket come "In Review" o commentare.
- **Esclusione MVP:** Non permettere all'agente di *creare* nuovi ticket o modificare epiche complesse.

#### D. Preflight Checks Rigorosi - CRITICA

Per evitare crash a metà lavoro.

- **Requisito Tecnico:** Prima di lanciare `client.query()`, eseguire controlli su API Key, presenza di `git`, e connessione a Linear.
- **Valore:** Trasformare errori oscuri di runtime in messaggi chiari all'avvio (es. "Manca Claude Code CLI").

### Out of Scope for MVP

Per consegnare in 3 mesi, esclusi:

- **Gestione "Infinita" del Contesto:** Non implementare `Tool Result Clearing` o `Compaction` complessa. Se il task è troppo lungo e il contesto finisce, l'agente fallisce "gracefully" e chiede aiuto, piuttosto che cercare di gestire una memoria infinita.
- **Docs Specialist:** La documentazione può essere aggiornata dall'umano o dal Coding Agent per ora.
- **Apprendimento a Lungo Termine:** Il *Memory Tool* verrà usato solo per salvare `project_id` e decisioni vitali, non come knowledge base complessa.
- **Interfaccia Grafica:** L'output CLI è sufficiente.

### MVP Success Criteria

Come sapremo di aver finito?

1. **Test del Disastro:** Iniettiamo volontariamente un bug nel codice generato. L'harness deve:
   - Rilevarlo tramite QA.
   - Eseguire il rollback automatico.
   - Lasciare il repo pulito come prima.

2. **Test dell'Autonomia:** Assegniamo un ticket Linear "Creare un file hello.py". L'utente lancia lo script e va a prendere il caffè. Al ritorno, il ticket su Linear deve essere "Done" e il file deve esistere.

**Sintesi per gli Stakeholder:**
L'MVP sarà un **"Developer Junior Cauto"**. Non sarà velocissimo e non gestirà progetti enormi, ma avrà una "rete di sicurezza" infallibile: o consegna codice funzionante e testato, o non tocca nulla. Questo risolve direttamente la paura principale dell'utente ("l'agente mi rompe il progetto").

### Future Vision

**Evoluzione post-MVP verso il Sistema Autonomo Resiliente completo:**

- **Fase 2 (6-9 mesi):** Implementazione completa del pilastro "Gestione Contesto" con Tool Result Clearing e Context Compaction per sessioni multi-giorno
- **Fase 3 (9-12 mesi):** Docs Specialist autonomo e Memory Tool avanzato come knowledge base persistente
- **Fase 4 (12+ mesi):** Gestione di intere epic, apprendimento incrementale, e adozione da parte di utenti secondari (Product Owner, Engineering Manager)

**Visione a lungo termine:**
L'harness evolve da "Developer Junior Cauto" a "Collaboratore Autonomo" capace di gestire progetti complessi multi-giorno senza degradazione delle performance, con memoria persistente avanzata e capacità di auto-apprendimento.

---

