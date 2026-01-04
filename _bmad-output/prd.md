---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
inputDocuments:
  - analysis/product-brief-Linear-Coding-Agent-Harness-2025-01-21.md
  - analysis/risposta1.md
  - analysis/risposta2.md
  - analysis/risposta3.md
  - analysis/risposta4.md
documentCounts:
  briefs: 1
  research: 0
  brainstorming: 4
  projectDocs: 0
workflowType: 'prd'
lastStep: 11
project_name: 'Linear-Coding-Agent-Harness'
user_name: 'User'
date: '2025-01-21'
completedAt: '2025-01-21'
---

# Product Requirements Document - Linear-Coding-Agent-Harness

**Author:** User
**Date:** 2025-01-21

## Executive Summary

**Linear Coding Agent Harness** è un sistema di orchestrazione resiliente che trasforma gli agenti Claude da esecutori lineari vulnerabili a sistemi autonomi capaci di auto-correzione. Il sistema risolve il problema critico della perdita di contesto e degli errori distruttivi nei progetti long-running, introducendo persistenza esterna dello stato, rollback automatico e isolamento rigoroso dei compiti tramite sub-agenti specializzati.

Il sistema si basa su quattro pilastri fondamentali: **Persistenza** (stato del lavoro salvato esternamente in Linear MCP e decisioni tecniche nel Memory Tool), **Sicurezza** (file checkpointing nativo SDK con rollback automatico quando i test falliscono), **Isolamento** (sub-agenti programmatici specializzati con permessi controllati), e **Gestione Contesto** (per l'MVP limitata all'uso intelligente del prompt e al partizionamento dei task tramite Sub-agenti; il Tool Result Clearing server-side è differito alla Fase 2). Il Memory Tool è centrale nel flusso operativo sia nel ciclo di lavoro normale che in caso di emergenza: **Nel workflow standard**, l'agente deve consultare il Memory Tool all'avvio ("Inizializzazione") per recuperare decisioni architetturali pregresse; **alla conclusione di Epic**, l'agente salva decisioni chiave (piano di implementazione aggiornato ad alto livello, ID progetto, scelte architetturali significative) per garantire persistenza tecnica oltre Linear e prevenire amnesia su decisioni architetturali in caso di riavvio sessione. Il Memory Tool non è solo un meccanismo di crash recovery, ma parte attiva del workflow standard che garantisce continuità tra sessioni attraverso uso strategico e moderato alla conclusione di Epic. Il tracciamento granulare dello stato operativo rimane delegato a Linear e Git.

L'integrazione con **git** è parte integrante del sistema, fornendo tracciabilità, audit trail e supporto al workflow di sviluppo attraverso commit automatici, branch management e collegamento con i ticket Linear. **Sequenza operativa:** 1) Checkpoint SDK automatico pre-modifica, 2) Coding Loop, 3) QA Check, 4) Se QA fallisce: `rewind_files()` (SDK) eseguito dall'Harness, 5) Se QA passa: **l'Agente** esegue `git commit` usando tool `bash` o `git` dedicato per garantire messaggi di commit semantici e descrittivi. Il Checkpointing SDK è effimero e legato alla sessione, utilizzato per il rollback tattico immediato durante il loop di auto-correzione "Try/Catch". Git è utilizzato esclusivamente per lo stato consolidato quando il QA passa, evitando di sporcare la history con commit intermedi falliti. **Nota:** L'Harness gestisce solo il rollback in caso di fallimento (`rewind_files`), mentre l'Agente è responsabile della creazione di commit strutturati con messaggi significativi che descrivono le modifiche implementate.

### What Makes This Special

Il vantaggio competitivo risiede nell'uso nativo e profondo delle funzionalità avanzate dell'SDK Claude (File Checkpointing, Rewind) integrate con logica di business esterna (Linear, git) gestita programmaticamente dal codice Python dell'harness. I sub-agenti sono definiti via codice (`AgentDefinition`) per controllo dinamico e isolamento superiore, con separazione netta tra infrastruttura (codice harness) e intelligenza (prompting agente). **Nota:** Le definizioni dettagliate dei sub-agenti (identità, principi core, activation sequence, skills, tools, workflows e pattern di decision-making) sono documentate nei grafi DNA strutturati disponibili in `project_1/personas_per_agents/` (grafi per Developer, Scrum Master, Technical Expert/Architect). Questi grafi forniscono il framework completo per le capacità specifiche dei sub-agenti che saranno formalizzate nei Requisiti Funzionali (Step 9).

La difficoltà di replica deriva dall'orchestrazione precisa di feature beta avanzate (`mcp-client`) che devono lavorare in concerto, e dalla logica di "Prioritizzazione Assoluta" per issue In-Progress gestita **deterministicamente e hard-coded in Python** prima dell'istanziazione di `ClaudeSDKClient`. L'harness esegue le query MCP (`mcp__linear__list_issues`) **prima** di istanziare il client e passa all'agente solo il ticket preselezionato, evitando che l'agente possa "allucinare" la priorità o decidere autonomamente su cosa lavorare.

Il timing è favorevole: l'SDK Claude ha reso disponibili nativamente le primitive necessarie (`rewind_files`, `enable_file_checkpointing`), e i modelli Claude 4.5 Sonnet/Opus con capacità di ragionamento estese abilitano sub-agenti specializzati affidabili come QA Specialist.

## Project Classification

**Technical Type:** developer_tool  
**Domain:** general  
**Complexity:** low-medium  
**Project Context:** Greenfield - new project

Il progetto è classificato come strumento per sviluppatori (`developer_tool`) che opera nel dominio generale (`general`) con complessità bassa-media. Non richiede conformità a regolamenti specifici di settore, ma richiede una comprensione approfondita delle capacità avanzate dell'SDK Claude Agent e delle best practices per l'orchestrazione di agenti autonomi.

### Requisiti Funzionali Critici MVP

**Preflight Checks Rigorosi:** L'harness deve validare l'ambiente **prima** di spendere il primo token. Controlli mandatori: API Keys (Claude, Linear), versioni Python/Node.js, presenza di Git, connessione a Linear, presenza di Claude Code CLI, variabile d'ambiente `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1`. **Requisiti Puppeteer (Condizionali):** Per supportare browser automation e test end-to-end del QA Specialist **solo per progetti che richiedono UI testing** (Web App frontend), il sistema deve verificare la presenza di Chrome/Chromium installato e le dipendenze di sistema necessarie per Puppeteer (browser headless). Per progetti CLI/backend (Python, Rust, Go senza interfaccia grafica), il controllo Puppeteer è **opzionale e non bloccante**. Il sistema deve rilevare automaticamente il tipo di progetto (presenza di file HTML/CSS/JS frontend vs solo codice backend) e adattare i preflight checks di conseguenza. Il sistema deve catturare e gestire tipi di errore specifici dell'SDK (`CLINotFoundError`, `CLIConnectionError`) per fornire feedback azionabile e fallimento graceful. I controlli base (API Keys, Python/Node.js, Git, Linear, Claude Code CLI) sono sempre bloccanti e costituiscono un requisito funzionale essenziale per l'esperienza utente.

**Halt Conditions e Rollback:** Il sistema deve definire esplicitamente le condizioni che innescano il rollback (`rewind_files`), tra cui: fallimento preflight, test rossi dal QA Specialist, tentativo di modifica file fuori scope (es. `.env`), e operazioni distruttive (es. `rm -rf`). Gli Hook SDK (`PreToolUse` con decisione `deny`) sono il meccanismo di enforcement per bloccare operazioni pericolose prima che vengano eseguite.

**Vincoli del QA Specialist:** Il QA Specialist deve avere istruzioni rigide di **non modificare il codice di produzione**. L'oggetto `AgentDefinition` del QA deve avere i tool `Edit` e `Write` **fisicamente rimossi** dalla configurazione, non solo vietati nel prompt. Il set di `tools` è **dinamico** e dipende dal tipo di progetto target. Il server MCP `puppeteer` (o `playwright`) sarà iniettato nel contesto del QA Agent **se e solo se** l'Harness rileva che il progetto target include un'interfaccia utente (Web/UI). Per progetti puramente backend/CLI (Python, Rust, Go senza interfaccia grafica), il QA Agent non deve avere accesso né invocare mai Puppeteer/Playwright. Il set di tool base include sempre: 1) **Tool di test CLI** (`bash` per pytest/npm test/cargo test/go test), 2) **Tool di lettura** (`read_file` per analisi codice). Il tool di **Browser automation** (`mcp__puppeteer__...` o `mcp__playwright__...`) è aggiunto condizionalmente solo per progetti che richiedono test end-to-end e validazione visiva/UI. Questo garantisce tecnicamente che il QA agisca come "Gatekeeper" imparziale con capacità di validazione end-to-end quando appropriato. **Nota Importante:** Il requisito non vincola l'architettura esclusivamente a Puppeteer; Playwright è un'alternativa valida, mantenendo l'obiettivo della validazione end-to-end senza limitare lo stack tecnologico.

**Scope di Linear (MVP):** L'integrazione Linear per l'MVP è limitata a **Read-Only + Status Update + Comment** per il **Coding Agent** (day-to-day operations). Il Coding Agent può leggere ticket esistenti, aggiornare lo stato (es. "In Progress" → "Done") e aggiungere commenti. La creazione di nuovi ticket complessi o la modifica di epiche è esplicitamente esclusa dall'MVP per il Coding Agent per evitare che un agente fuori controllo inondi il backlog di ticket spazzatura. **Nota Importante:** L'**Initializer Agent** (Session 1 - bootstrap progetto) deve avere permessi di scrittura completi su Linear (`mcp__linear__create_issue`) per popolare il backlog iniziale del progetto. La restrizione sulla creazione ticket si applica solo al Coding Agent durante le operazioni day-to-day, non all'Initializer Agent durante la fase di inizializzazione.

**Context Exhaustion (Halt Condition) - Rotazione Sub-Agente:** Il sistema deve definire una halt condition esplicita per "Context Exhaustion" con una **soglia di token conservativa dell'85%**. Quando questa soglia viene raggiunta, il sistema deve attivare il meccanismo di **rotazione del Sub-Agente** invece di terminare semplicemente il lavoro. **Flusso Tecnico Esatto:**

1. L'Agente Principale (Harness) monitora il consumo del *Coding Sub-Agent* tramite `usage` nei messaggi.
2. Al raggiungimento dell'85% del limite fisico del modello, il *Coding Sub-Agent* viene istruito a scrivere un "passaggio di consegne" (stato lavori, file modificati, prossimi step) nei documenti di sviluppo (o come commento/file di log).
3. L'Agente Principale termina il *Coding Sub-Agent* corrente (liberando contesto).
4. L'Agente Principale istanzia un **nuovo Coding Sub-Agent** pulito con finestra di contesto fresca.
5. Il nuovo sub-agente legge il passaggio di consegne e continua lo sviluppo in modo informato e coerente.

**Obiettivo Operativo:** La Halt Condition serve a prevenire allucinazioni da contesto pieno, garantendo "memoria fresca" tramite rotazione degli agenti. Quando l'agente raggiunge l'85% del contesto, **non termina il lavoro** in senso assoluto. Si ferma solo per: 1) Salvare lo stato corrente (progressi e roadmap) nel Memory Tool, 2) Effettuare una pulizia del contesto (Context Clearing) o avviare un nuovo sub-agente con finestra pulita, 3) Riprendere immediatamente il lavoro autonomamente. L'obiettivo rimane che Alex, al risveglio, trovi il ticket su Linear come "Done" o con una PR pronta, e non semplicemente un lavoro "pausato" per salvezza. La halt condition è un meccanismo di gestione memoria che permette continuità operativa, non una terminazione del lavoro. **Ottimizzazione Passiva:** Per l'MVP, è raccomandato abilitare la strategia di clearing passivo `clear_tool_uses_20250919` come ottimizzazione server-side automatica (se disponibile nell'SDK con header `context-management-2025-06-27`), mantenendo la Halt Condition come sicurezza finale. Questo allunga la vita operativa dell'agente senza aggiungere complessità di codice lato harness, riducendo la frequenza di riavvii costosi. La Halt Condition rimane il meccanismo di sicurezza principale per garantire salvataggio di emergenza affidabile.

## Success Criteria

### User Success

Per **Alex (Senior Software Engineer / Tech Lead)**, il successo non è la velocità di scrittura del codice, ma la **fiducia** nel delegare compiti complessi senza "babysitting".

**Momenti di valore che portano a dire "ne è valsa la pena":**

- **Il "Zero-Repair Morning":** Alex arriva al lavoro e trova che l'agente ha tentato un task complesso durante la notte, ha fallito un test, ha eseguito automaticamente un `rewind_files()` per annullare le modifiche rotte, e ha lasciato un commento su Linear spiegando il blocco, mantenendo il resto del progetto intatto.

- **La PR "Green":** Ricevere una Pull Request dove il *QA Specialist* (sub-agente) ha già eseguito e validato i test, garantendo che il codice sia conforme alla "Definition of Done" prima ancora che Alex lo guardi.

**Il momento "aha!" di risoluzione del problema:**
È il momento in cui l'harness intercetta un errore distruttivo (es. cancellazione accidentale di config) e lo corregge autonomamente usando il checkpointing, senza che Alex debba intervenire manualmente via git per ripristinare lo stato.

**Sintesi del successo per Alex:**
Il successo è raggiunto quando Alex può assegnare un ticket Linear taggato `agent-task`, andare a dormire, e la mattina successiva trovare o una PR verde pronta per il merge, o il ticket ancora "In Progress" con un commento dettagliato dell'agente che spiega un blocco tecnico, senza che nessuna riga di codice sia stata corrotta nel processo.

**Successo per utenti secondari:**

- **Product Owner / Engineering Manager:** Visibilità trasparente in tempo reale su Linear senza dover interagire via terminale. Possono bloccare task o cambiare priorità semplicemente spostando ticket in Linear.

- **QA Engineer / Reviewer:** Ricevono Pull Request più pulite con codice già validato dal QA Specialist, riducendo il tempo di review e aumentando la fiducia nella qualità del codice.

### Business Success

#### Successo a 3 Mesi (Fase: Stabilità & Fiducia)

**Obiettivo:** L'harness è "safe". Non rompe mai il progetto.

**Metriche chiave:**
- Rollback automatico funzionante nel 100% dei casi di test falliti
- Integrazione Linear MCP stabile: lo stato dei ticket è sempre sincronizzato con la realtà
- Alex usa l'harness quotidianamente per task di refactoring e manutenzione
- Zero regressioni introdotte dall'harness durante l'implementazione di nuove feature

#### Successo a 12 Mesi (Fase: Scalabilità & Autonomia)

**Obiettivo:** L'harness è un "collaboratore autonomo". Gestisce intere epic.

**Metriche chiave:**
- Capacità di gestire sessioni multi-giorno (grazie a Memory Tool e persistenza) senza degradazione delle performance
- Riduzione del 50% del tempo speso dal team su bug fix triviali e setup di boilerplate
- Adozione da parte di utenti secondari (es. Product Owner che aprono ticket sapendo che verranno gestiti)
- Gestione autonoma di epic complesse senza intervento umano intermedio

### Technical Success

#### Metriche di Resilienza e Qualità (Strategic - Il "Safety Net")

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

#### Metriche di Engagement e Autonomia (Operational)

Misurano quanto l'harness libera Alex dal "babysitting".

4. **Intervention Ratio (Rapporto Interventi):**
   - **Definizione:** Numero di input umani richiesti per completare un ticket Linear di media complessità.
   - **Obiettivo:** < 1 per ticket (idealmente solo la definizione iniziale e la review finale).

5. **Context Efficiency (Efficienza della Memoria):**
   - **Definizione:** Efficacia del *Memory Tool* e del *Context Clearing* nel preservare le informazioni critiche. Misurabile tramite il numero di volte che l'agente deve "ri-chiedere" informazioni già fornite o presenti nel progetto.
   - **Obiettivo:** L'agente non chiede mai due volte la stessa specifica architetturale presente nel Memory Tool.

#### Metriche di Efficienza Token (Efficiency)

6. **Consumo Token per Story Point (Token Consumption per Unit of Work):**
   - **Definizione:** Consumo di token (input/output) medio per completare un ticket, normalizzato per complessità.
   - **Contesto:** La harness può essere utilizzata sia con API key Anthropic (pay-per-use) che con token OAuth per abbonamento Claude (prezzo fisso con limite token su finestra temporale, tipicamente tot token/5 ore). Con l'abbonamento, il limite è su finestra oraria e modelli come Opus 4.5 consumano token rapidamente.
   - **Misurazione:** 
     - Consumo token per finestra oraria (se Anthropic mette a disposizione strumenti di monitoraggio)
     - Consumo token per finestra di contesto (per dedurre efficienza del contesto)
     - Pattern di consumo per identificare ottimizzazioni
   - **Insight:** Monitorare se l'uso di *Context Compaction* e *Result Clearing* mantiene il consumo token stabile anche in sessioni lunghe ("Long-running agents"), evitando l'esaurimento rapido del limite orario dell'abbonamento dovuto alla saturazione del contesto. Per utenti con abbonamento, questa metrica è critica per evitare di raggiungere il limite orario prima del completamento del task.

### Measurable Outcomes

**Outcome primario:** Trasformazione del ruolo di Alex da "Operatore di Chatbot" (scrivere prompt, correggere output, incollare file) a "Gestore di Flusso" (definire requisiti in Linear, revisionare PR, gestire eccezioni di alto livello).

**Outcome secondario:** L'harness diventa un "impiegato virtuale" che lavora su turni, dotato di memoria a lungo termine e capacità di auto-correzione, permettendo sessioni di lavoro continue su progetti complessi che richiedono giorni di sviluppo.

**Outcome tecnico:** Sistema resiliente che non rompe mai il progetto, con capacità di auto-correzione che previene errori distruttivi prima che diventino permanenti.

## Product Scope

### MVP - Minimum Viable Product

**Filosofia MVP: "Safety First, Autonomy Second"**

L'MVP mira alla robustezza, non a sessioni "infinite" o intelligenza suprema. Criterio decisionale: *Se l'agente sbaglia, il progetto sopravvive?*

#### Core Features MVP

**A. Il "Safety Net" (Checkpointing & Rollback) - CRITICA**

Funzionalità core. L'harness deve poter annullare le modifiche se i test falliscono.

- **Requisito Tecnico:** 
  - Abilitare `enable_file_checkpointing=True` e `permission_mode="acceptEdits"` (per evitare blocchi su ogni file).
  - **Configurazione Obbligatoria:** Per far funzionare correttamente il checkpointing e ricevere i `checkpoint_uuid` necessari per il rollback, è **obbligatorio**:
    1. Passare `extra_args={"replay-user-messages": None}` durante l'istanziazione del `ClaudeSDKClient`
    2. Impostare la variabile d'ambiente `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1` prima di avviare l'harness
  - Senza queste configurazioni specifiche, la funzione `rewind_files()` fallirà o non sarà disponibile.
- **Logica:** Prima di ogni iterazione del *Coding Agent*, salvare il `message.uuid` e il `checkpoint_uuid`. Se il *QA Agent* riporta fallimento, eseguire `client.rewind_files(checkpoint_uuid)`.
- **Valore Utente:** L'utente vede i file cambiare e poi tornare indietro magicamente se il codice è rotto.

**B. Architettura Sub-agenti - ALTA**

Per alleggiare il carico del Main Agent (orchestratore) e creare un team minimale ma efficace, il sistema utilizza sub-agenti specializzati definiti programmaticamente tramite `AgentDefinition`.

- **Requisito Tecnico:** Definire sub-agenti tramite `AgentDefinition` seguendo le best practices Anthropic per l'orchestrazione di agenti autonomi.
- **Fonte di Definizione:** Le definizioni dettagliate dei sub-agenti (identità, principi core, activation sequence, skills, tools, workflows) sono documentate nei grafi DNA strutturati disponibili in `project_1/personas_per_agents/` (grafi per Developer, Scrum Master, Test Expert/Architect). Questi grafi forniscono il framework completo per definire un team perfetto, minimale al punto giusto e architettato ad hoc.
- **Sub-agenti MVP Minimi:**
  - **QA Specialist (Gatekeeper):** Il set di tool è **dinamico** e dipende dal tipo di progetto target. Il toolset base include sempre: tool di test CLI (`bash` per pytest/npm test/cargo test/go test) e tool di lettura (`read_file`). Il tool di browser automation (`mcp__puppeteer__...` o `mcp__playwright__...`) è aggiunto condizionalmente solo per progetti che richiedono test end-to-end e validazione UI. I tool `Edit` e `Write` devono essere **fisicamente rimossi** dalla configurazione, non solo vietati nel prompt. Questo garantisce tecnicamente che il QA agisca come "Gatekeeper" imparziale con capacità di validazione end-to-end quando appropriato, evitando falsi positivi e regressioni nascoste.
  - **Altri sub-agenti:** La definizione completa del team di sub-agenti (inclusa la valutazione di Docs Specialist o altri ruoli specializzati) sarà determinata durante la fase di design dettagliato, analizzando i DNA-grafi disponibili e seguendo le best practices Anthropic per ottimizzare l'efficienza e minimizzare il carico sul Main Agent. **Nota Isolamento:** L'isolamento rigoroso dei permessi (rimozione fisica dei tool `Edit` e `Write`) si applica **esclusivamente al QA Specialist**. Gli altri sub-agenti possono avere permessi differenziati basati sul loro ruolo specifico, come definito nei DNA-grafi e nelle best practices Anthropic.

- **Strategia Prompt Caching vs. Sub-Agenti - Trade-off Architetturale Aperto:** Questo è un **trade-off architetturale aperto** che richiede valutazione approfondita durante la fase di architettura. Durante la fase di design dettagliato verrà effettuata una consultazione approfondita dell'Agent SDK Anthropic per determinare *se* e *come* sia possibile combinare prompt caching e sub-agenti senza penalizzare le performance o i costi. La ricerca tecnica iniziale suggerisce che i sub-agenti definiti tramite `AgentDefinition` possono potenzialmente condividere lo stesso toolset preservando il prompt caching quando utilizzano tool comuni, ma questa ipotesi deve essere validata sperimentalmente durante l'implementazione. **Nota NFR:** Il context switching tra sub-agenti con set di tool diversi può invalidare la cache, riducendo l'efficacia del caching. Questo trade-off deve essere valutato attentamente bilanciando sicurezza strutturale (isolamento programmatico garantito) ed efficienza token (prompt caching preservato quando possibile). L'implementazione utilizzerà **sub-agenti distinti definiti tramite `AgentDefinition`** con toolset condivisi quando appropriato (per preservare prompt caching) e toolset differenziati quando necessario per isolamento (es. QA Specialist senza tool Edit/Write). Gli Hook SDK (`PreToolUse` con decisione `deny`) saranno utilizzati come meccanismo di enforcement aggiuntivo per garantire sicurezza strutturale, ma non come sostituto dell'isolamento tramite `AgentDefinition`. **Nota Implementazione:** Durante la fase di design dettagliato, verificare sperimentalmente il comportamento del prompt caching con sub-agenti che condividono toolset vs toolset differenziati per ottimizzare l'approccio finale.

**C. Integrazione Linear "Read-Only + Status Update" - ALTA**

Per evitare il "babysitting", l'agente deve sapere cosa fare da solo.

- **Requisito Tecnico:** Configurare `mcp_servers` per connettersi a Linear.

**C.1. Integrazione Tavily MCP per Ricerca Informazioni - MEDIA**

Per recuperare informazioni mancanti o non precise durante l'esecuzione, il sistema può utilizzare il server MCP Tavily.

- **Scopo del Tavily MCP Server:** Il Tavily MCP Server ha lo scopo di permettere ai client compatibili con il protocollo MCP di utilizzare direttamente le API di Tavily all'interno dei propri sistemi. Il suo utilizzo tipico è quello di potenziare gli agenti AI fornendo loro un accesso al web in tempo reale per eseguire ricerche, estrarre contenuti e svolgere indagini approfondite in modo autonomo.

- **Requisito Tecnico:** Il server MCP Tavily è già disponibile come Linear (configurato nel sistema MCP) e può essere utilizzato dall'agente orchestratore (Main Agent) o da sub-agenti per recuperare informazioni mancanti o non precise. **Non è richiesto codice aggiuntivo** - l'integrazione avviene esclusivamente tramite istruzioni nel prompt dell'agente o del sub-agente che specificano quando e come utilizzare i tool Tavily (`mcp__tavily__...`) per ricerche web e recupero informazioni.

- **Utilizzo:** L'agente orchestratore o i sub-agenti possono utilizzare Tavily quando necessitano di:
  - Documentazione tecnica aggiornata (es. SDK Anthropic, best practices)
  - Informazioni su librerie o framework specifici
  - Ricerche su pattern architetturali o soluzioni tecniche
  - Validazione di informazioni incerte o non precise
  - Esecuzione di ricerche web in tempo reale per indagini approfondite
  - Estrazione di contenuti da fonti web per analisi e validazione

- **Istruzioni Prompt:** I prompt dell'agente devono includere istruzioni esplicite su quando utilizzare Tavily per recuperare informazioni mancanti, specificando che il server MCP Tavily è disponibile e può essere invocato tramite i tool `mcp__tavily__...` quando necessario. L'uso di Tavily deve essere strategico e limitato a situazioni dove le informazioni non sono disponibili nel contesto locale o nel Memory Tool. Tavily fornisce accesso al web in tempo reale per potenziare le capacità di ricerca e investigazione autonoma dell'agente.
- **Logica Semplificata (CRITICA - Deterministica):**
  1. L'Harness (Python) esegue la query `mcp__linear__list_issues` **prima** di istanziare il `ClaudeSDKClient`, in codice Python "puro" fuori dal loop dell'agente.
  2. L'harness applica la logica di "Prioritizzazione Assoluta" hard-coded: seleziona ticket "In Progress" prima di "Todo", senza delegare questa decisione all'agente.
  3. **Context Recovery per Ticket "In Progress":** Se l'harness seleziona un ticket "In Progress" (interrotto da una sessione precedente), l'harness deve istruire l'agente (o passare nel prompt iniziale) di leggere esplicitamente la history dei commenti del ticket su Linear per ricostruire lo stato mentale della sessione interrotta. Questo è un requisito funzionale obbligatorio per garantire continuità tra sessioni. L'agente deve consultare i commenti precedenti per capire cosa è stato fatto, quali decisioni sono state prese, e da dove riprendere il lavoro.
  4. L'harness passa al *Main Agent* solo il ticket preselezionato (con contesto recuperato se "In Progress"), non la lista completa.
  5. A fine lavoro, l'agente usa il tool per marcare il ticket come "In Review" o commentare.
- **Profili di Sicurezza Differenziati:**
  - **Bootstrap Profile (Initializer Agent - Session 1):** Profilo di sicurezza con permessi elevati, usato solo durante il setup iniziale del progetto. Permessi completi di scrittura su Linear, inclusa creazione ticket (`mcp__linear__create_issue`) per popolare il backlog iniziale del progetto. Questo profilo è un'eccezione alla regola di sicurezza e deve essere utilizzato esclusivamente durante la fase di bootstrap.
  - **Runtime Profile (Coding Agent - Day-to-Day):** Profilo di sicurezza con permessi ristretti, usato per il loop operativo day-to-day. Read-Only + Status Update + Comment su Linear. **Esclusione MVP:** Non permettere al Coding Agent di *creare* nuovi ticket o modificare epiche complesse durante le operazioni day-to-day per evitare che un agente fuori controllo inondi il backlog di ticket spazzatura. La restrizione sulla creazione ticket si applica solo al Coding Agent durante le operazioni day-to-day, non all'Initializer Agent durante la fase di inizializzazione.

**D. Preflight Checks Rigorosi - CRITICA**

Per evitare crash a metà lavoro.

- **Requisito Tecnico:** Prima di lanciare `client.query()`, eseguire controlli su API Key, presenza di `git`, e connessione a Linear.
- **Valore:** Trasformare errori oscuri di runtime in messaggi chiari all'avvio (es. "Manca Claude Code CLI").

**E. Memory Tool Integration - ALTA**

Per garantire persistenza tecnica oltre Linear.

- **Requisito Tecnico:** Il Memory Tool è parte attiva del workflow standard, non solo meccanismo di crash recovery. L'agente deve: 1) **All'avvio ("Inizializzazione"):** consultare il Memory Tool per recuperare decisioni architetturali pregresse (ID progetto, scelte architetturali, piani di implementazione precedenti), 2) **Alla conclusione di Epic:** salvare decisioni chiave (piano di implementazione aggiornato ad alto livello, scelte architetturali significative) per garantire persistenza. **Uso Strategico e Cognitivo:** Dopo ricerca tecnica approfondita sulla documentazione Anthropic SDK e best practices di context management, è stato verificato che l'SDK non offre primitive native specifiche per "Context Compaction" o gestione memoria a lungo termine oltre il Memory Tool standard. L'implementazione del Memory Tool deve quindi essere **strategica e cognitiva**, non un dump indiscriminato di dati. Per bilanciare persistenza necessaria con efficienza token, il Memory Tool deve essere consultato all'inizializzazione e aggiornato **esclusivamente alla conclusione di una Epic**, non ad ogni iterazione o task singolo. L'obiettivo è la persistenza di alto livello, non il tracciamento granulare dello stato operativo (che rimane delegato a Linear e Git). Questo approccio garantisce continuità tra sessioni e previene amnesia su decisioni architetturali senza degradare le performance operative con overhead eccessivo di token.

- **Nota Implementazione Critica:** I requisiti FR28 e FR29 devono essere implementati con uso strategico e cognitivo del Memory Tool. I prompt dell'agente (`prompts.py`) devono riflettere questo approccio bilanciato, istruendo esplicitamente l'agente a utilizzare il Memory Tool solo ai checkpoint critici ben definiti, non in ogni ciclo. La definizione dei prompt deve bilanciare la necessità di persistenza con l'efficienza del consumo token, evitando overhead eccessivo che potrebbe degradare le performance operative. In fase di architettura, verificare che i prompt riflettano questo uso strategico senza aumentare eccessivamente la complessità del ciclo operativo. **Pattern di Utilizzo Documentato:** L'agente deve salvare nel Memory Tool solo informazioni "cognitivamente rilevanti" per la continuità del lavoro: ID progetto Linear, scelte architetturali chiave (non dettagli implementativi minori), piano di implementazione ad alto livello (non ogni singola modifica file), decisioni tecnologiche significative. Evitare di salvare log dettagliati di esecuzione o stato intermedio che può essere ricostruito da Linear o git.

#### Out of Scope for MVP

Per consegnare in 3 mesi, esclusi:

- **Gestione "Infinita" del Contesto:** Non implementare `Tool Result Clearing` o `Compaction` complessa. Se il task è troppo lungo e il contesto finisce, l'agente esegue la routine di salvataggio di emergenza (vedi Context Exhaustion) e chiede aiuto, piuttosto che cercare di gestire una memoria infinita.
- **Definizione Completa Sub-agenti:** La definizione finale del team di sub-agenti (inclusa la valutazione di ruoli come Docs Specialist, Coding Specialist, o altri ruoli specializzati) sarà determinata durante la fase di design dettagliato, analizzando i DNA-grafi disponibili in `project_1/personas_per_agents/` e seguendo le best practices Anthropic per creare un team minimale ma efficace che alleggerisca il carico sul Main Agent.
- **Apprendimento a Lungo Termine:** Il *Memory Tool* verrà usato solo per salvare `project_id` e decisioni vitali, non come knowledge base complessa.
- **Interfaccia Grafica:** L'output CLI è sufficiente.

#### MVP Success Criteria

Come sapremo di aver finito?

1. **Test del Disastro:** Iniettiamo volontariamente un bug nel codice generato. L'harness deve:
   - Rilevarlo tramite QA.
   - Eseguire il rollback automatico.
   - Lasciare il repo pulito come prima.

2. **Test dell'Autonomia:** Assegniamo un ticket Linear "Creare un file hello.py". L'utente lancia lo script e va a prendere il caffè. Al ritorno, il ticket su Linear deve essere "Done" e il file deve esistere.

**Sintesi per gli Stakeholder:**
L'MVP sarà un **"Developer Junior Cauto"**. Non sarà velocissimo e non gestirà progetti enormi, ma avrà una "rete di sicurezza" infallibile: o consegna codice funzionante e testato, o non tocca nulla. Questo risolve direttamente la paura principale dell'utente ("l'agente mi rompe il progetto").

### Growth Features (Post-MVP)

**Fase 2 (6-9 mesi):** Implementazione completa del pilastro "Gestione Contesto" con Tool Result Clearing e Context Compaction per sessioni multi-giorno.

**Fase 3 (9-12 mesi):** Docs Specialist autonomo e Memory Tool avanzato come knowledge base persistente.

**Fase 4 (12+ mesi):** Gestione di intere epic, apprendimento incrementale, e adozione da parte di utenti secondari (Product Owner, Engineering Manager).

### Vision (Future)

**Visione a lungo termine:**
L'harness evolve da "Developer Junior Cauto" a "Collaboratore Autonomo" capace di gestire progetti complessi multi-giorno senza degradazione delle performance, con memoria persistente avanzata e capacità di auto-apprendimento. Il sistema diventa un vero "collaboratore" che lavora su turni, gestisce intere epic autonomamente, e impara dai pattern di successo per migliorare continuamente la propria efficacia.

## User Journeys

### Journey 1: Alex - Dal "Babysitting" alla Fiducia

Alex è un Senior Software Engineer che lavora su progetti complessi che richiedono refactoring, nuove feature e manutenzione continua. Ha un background tecnico solido (Python/Node.js) e usa già strumenti come Linear per il project management e git per il versionamento.

Una sera, dopo aver passato 3 ore a riparare codice rotto da un agente AI che aveva esaurito il contesto a metà implementazione, lasciando il progetto in uno stato "half-implemented", Alex decide di provare il Linear Coding Agent Harness.

Il mattino seguente, Alex clona la repo dell'harness e configura `LINEAR_API_KEY` e le variabili per il file checkpointing. Lancia l'harness e riceve immediatamente un messaggio chiaro dal preflight check: "Preflight check completato: Python ✓, Node.js ✓, Git ✓, Linear connesso ✓, Claude Code CLI ✓". Nessun crash oscuro a metà esecuzione - se mancava qualcosa, avrebbe ricevuto un errore deterministico e azionabile.

Alex prepara il file `app_spec.txt` con i requisiti del progetto (generato da PRD + Architecture + Epics + Stories) e lo posiziona nella directory del progetto. Lancia l'harness e va a prendere un caffè, fiducioso che il sistema abbia una "Safety Net" attiva. L'**Initializer Agent** (Session 1 - bootstrap) rileva automaticamente la presenza di `app_spec.txt`, legge il file, e genera autonomamente l'infrastruttura Linear completa: progetto Linear con team ID, ticket strutturati ottimamente organizzati in epiche e milestone, tutto senza intervento manuale di Alex.

Al ritorno, Alex controlla Linear: il ticket è "In Progress" con un commento dell'agente che spiega il piano di implementazione salvato nel Memory Tool. Dopo un'ora, il ticket è "Done" con un link al commit git. Alex controlla il codice: è pulito, i test passano, e c'è un commento dettagliato su Linear che spiega le scelte implementative. Non deve passare ore a "riparare" ciò che l'agente ha rotto.

Il momento "aha!" arriva quella notte: Alex assegna un task complesso e va a dormire. Al mattino, trova un commento su Linear: "Test fallito alle 2:34 AM. Eseguito rollback automatico tramite `rewind_files()`. Tentativo con approccio alternativo in corso." Il progetto è intatto, nessun file corrotto. L'harness ha gestito l'errore autonomamente usando il checkpointing, senza che Alex debba intervenire manualmente via git per ripristinare lo stato.

Sei mesi dopo, Alex non è più un "Operatore di Chatbot" (scrivere prompt, correggere output, incollare file) ma un "Gestore di Flusso" (definire requisiti in Linear, revisionare PR, gestire eccezioni di alto livello). L'harness è diventato un "collaboratore autonomo" che lavora su turni, dotato di memoria a lungo termine e capacità di auto-correzione.

**Requisiti rivelati da questo journey:**
- Preflight checks rigorosi con messaggi chiari e azionabili
- Initializer Agent per bootstrap automatizzato progetto Linear da app_spec.txt
- File checkpointing e rollback automatico
- Memory Tool per persistenza del piano di implementazione
- Workflow autonomo che funziona senza "babysitting"
- Notifiche e aggiornamenti su Linear in tempo reale

### Journey 2: Sarah - Product Owner - Visibilità Senza Terminale

Sarah è Product Owner in un team distribuito che lavora su progetti complessi. Prima dell'harness, doveva chiedere ad Alex "a che punto è l'agente?" o richiedere accesso al terminale per controllare lo stato del lavoro, creando un collo di bottiglia nella comunicazione.

Con l'harness integrato con Linear, Sarah apre il suo workspace Linear e vede in tempo reale lo stato dei ticket: "In Progress" con commenti dettagliati dell'agente che spiegano il progresso, link ai commit git, e aggiornamenti automatici quando il lavoro viene completato o quando ci sono blocchi tecnici.

Il valore immediato per Sarah: può bloccare un task o cambiare priorità semplicemente spostando un ticket in Linear, senza dover interagire via terminale con l'agente o dipendere da Alex per aggiornamenti. La visibilità è trasparente e sempre aggiornata.

Quando un ticket viene completato, Sarah riceve una notifica e può vedere immediatamente il codice implementato, i test eseguiti, e i commenti dell'agente che spiegano le decisioni prese. Non deve più chiedere "è fatto?" - lo vede direttamente su Linear.

**Requisiti rivelati da questo journey:**
- Integrazione Linear con aggiornamenti in tempo reale
- Commenti automatici dell'agente su Linear con dettagli del progresso
- Link ai commit git nei commenti Linear
- Possibilità di modificare priorità e stato ticket senza interazione tecnica
- Notifiche quando ticket vengono completati o bloccati

### Journey 3: Marco - QA Engineer - PR Pulite e Fiducia

Marco è QA Engineer e riceve Pull Request per la revisione del codice generato dall'harness. Prima dell'harness, molte PR arrivavano con test non eseguiti o codice non testato, richiedendo tempo significativo per identificare problemi evidenti che avrebbero dovuto essere catturati prima.

Con l'harness, Marco riceve Pull Request dove il QA Specialist (sub-agente isolato) ha già eseguito e validato i test. Il codice ha superato una barriera di qualità automatizzata prima ancora che Marco lo guardi. Le PR sono più pulite, con test già passati e codice conforme alla "Definition of Done".

Il valore per Marco: meno tempo speso su bug evidenti che avrebbero dovuto essere catturati automaticamente, maggiore fiducia nella qualità del codice che arriva alla revisione umana, e capacità di concentrarsi su aspetti più sofisticati della qualità del codice piuttosto che su problemi banali.

Quando Marco trova un problema durante la review, può commentare su Linear e l'harness può riprendere il lavoro sul ticket, eseguendo automaticamente il rollback se necessario e riprovando con un approccio diverso.

**Requisiti rivelati da questo journey:**
- QA Specialist come sub-agente isolato con permessi ristretti
- Esecuzione automatica di test prima di marcare ticket come "Done"
- PR con codice già testato e validato
- Integrazione con workflow di review su Linear
- Capacità di riprendere lavoro basato su feedback della review

### Journey 4: Luca - DevOps Engineer - Configurazione e Monitoring

**Nota MVP vs Fase 2:** Per l'MVP, Luca configura l'harness in ambiente locale (PC/Mac/Linux) per test e validazione. Il deployment su server/container persistenti è differito alla Fase 2. Questo journey descrive la visione Fase 2 quando il sistema sarà deployato in produzione.

Luca è DevOps Engineer responsabile della configurazione e del monitoring dell'harness in produzione (Fase 2). Deve assicurarsi che l'harness funzioni correttamente su server/container persistenti e che sia monitorabile.

Luca configura l'harness su un server dedicato o container Docker, impostando le variabili d'ambiente necessarie (`LINEAR_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`, variabili per file checkpointing). Il preflight check gli fornisce feedback immediato su eventuali problemi di configurazione prima che l'harness inizi a lavorare.

Luca configura il monitoring per tracciare:
- Stato dei ticket Linear (quanti "In Progress", "Done", "Blocked")
- Frequenza di rollback automatici (per identificare pattern problematici)
- Consumo di token per finestra oraria (per evitare di raggiungere limiti dell'abbonamento)
- Tempo medio per completare un ticket

Quando l'harness raggiunge il limite di contesto e esegue la routine di salvataggio di emergenza, Luca riceve un alert che gli permette di intervenire se necessario o verificare che il salvataggio sia avvenuto correttamente.

**Requisiti rivelati da questo journey:**
- Configurazione ambiente con variabili d'ambiente chiare
- Preflight checks che validano configurazione prima dell'esecuzione
- Monitoring e observability del sistema
- Alerting per situazioni critiche (context exhaustion, errori ripetuti)
- Metriche di performance e consumo risorse


### Journey Requirements Summary

I journeys mappati rivelano i seguenti requisiti funzionali chiave:

**Onboarding e Configurazione:**
- Setup ambiente con variabili d'ambiente chiare
- Preflight checks rigorosi con messaggi azionabili
- Validazione configurazione prima dell'esecuzione

**Workflow Operativo:**
- Integrazione Linear per creazione, lettura e aggiornamento ticket
- Logica di prioritizzazione deterministica (hard-coded in Python)
- File checkpointing automatico prima di modifiche
- Rollback automatico su fallimento test (eseguito dall'Harness tramite `rewind_files`)
- Memory Tool per persistenza decisioni e piano implementazione (workflow standard: consultazione all'avvio, salvataggio esclusivamente alla conclusione di Epic; anche routine di salvataggio emergenza per context exhaustion)
- Git commit eseguito dall'Agente (dopo QA passa) con messaggi semantici e descrittivi
- Context Exhaustion halt condition con soglia conservativa (80-85% limite modello) e routine di salvataggio emergenza

**Sub-agenti e Isolamento:**
- QA Specialist isolato con permessi ristretti (tool Edit/Write fisicamente rimossi)
- Architettura sub-agenti definita tramite AgentDefinition seguendo best practices Anthropic
- Definizione team sub-agenti basata su DNA-grafi disponibili

**Monitoring e Observability:**
- Metriche di performance (tempo completamento ticket, frequenza rollback)
- Monitoring consumo token per finestra oraria
- Alerting per situazioni critiche
- Audit trail completo (git commits, Linear comments)

## Domain-Specific Requirements

### General Domain Compliance & Regulatory Overview

Il progetto **Linear Coding Agent Harness** opera nel dominio generale (`general`) con complessità bassa-media. Non richiede conformità a regolamenti specifici di settore (come FDA per healthcare, PCI DSS per fintech, o ISO 26262 per automotive), ma deve aderire a standard di sviluppo software generali e best practices per strumenti per sviluppatori.

Il dominio "general" richiede attenzione a requisiti standard che sono già integrati nel design del prodotto: sicurezza di base, performance, user experience, e pratiche di sviluppo software consolidate.

### Key Domain Concerns

Per il dominio generale, i principali concern sono già coperti nel PRD:

**Standard Requirements:**
- Conformità alle best practices di sviluppo software Python/Node.js
- Integrazione con strumenti standard del workflow sviluppatore (git, Linear, Claude SDK)
- Architettura modulare e manutenibile
- Documentazione tecnica chiara per sviluppatori

**Basic Security:**
- Gestione sicura delle API keys (Claude, Linear) tramite variabili d'ambiente
- Preflight checks per validare configurazione prima dell'esecuzione
- Isolamento dei sub-agenti con permessi controllati (isolamento rigoroso applicato esclusivamente al QA Specialist senza tool Edit/Write)
- File checkpointing e rollback automatico per prevenire modifiche distruttive
- Hook SDK (`PreToolUse` con decisione `deny`) per bloccare operazioni pericolose

**User Experience:**
- Messaggi di errore chiari e azionabili (preflight checks)
- Workflow autonomo senza "babysitting"
- Feedback trasparente su Linear con commenti dettagliati
- Integrazione seamless con workflow esistenti (git, Linear)

**Performance:**
- Efficienza token per evitare esaurimento contesto (monitoraggio consumo per finestra oraria)
- Gestione ottimale del contesto per sessioni long-running (Memory Tool, Context Exhaustion halt condition)
- Metriche di performance (tempo completamento ticket, frequenza rollback)

### Compliance Requirements

Non ci sono requisiti di compliance regolamentari specifici per il dominio generale. Il sistema deve aderire a:

- **Best Practices Software Development:** Codice pulito, testato, documentato
- **Security Best Practices:** Gestione sicura credenziali, validazione input, isolamento permessi
- **Version Control Standards:** Uso appropriato di git per tracciabilità e audit trail
- **API Integration Standards:** Conformità alle API di Linear MCP e Claude SDK

### Industry Standards & Best Practices

Il progetto aderisce a standard e best practices già documentati nel PRD:

**Claude SDK Best Practices:**
- Uso nativo di `enable_file_checkpointing` e `rewind_files()` per resilienza
- Architettura sub-agenti tramite `AgentDefinition` seguendo best practices Anthropic
- Gestione appropriata del contesto con Memory Tool per persistenza decisioni

**Development Tool Standards:**
- Preflight checks rigorosi per validare ambiente prima dell'esecuzione
- Isolamento rigoroso dei compiti tramite sub-agenti specializzati (isolamento permessi applicato esclusivamente al QA Specialist)
- Rollback automatico su fallimento test (QA Specialist come gatekeeper)
- Audit trail completo tramite git commits e Linear comments

**Python/Node.js Best Practices:**
- Gestione errori robusta con tipi di errore specifici SDK (`CLINotFoundError`, `CLIConnectionError`)
- Configurazione tramite variabili d'ambiente
- Architettura modulare e testabile

### Required Expertise & Validation

**Required Knowledge:**
- Comprensione approfondita delle capacità avanzate dell'SDK Claude Agent
- Best practices per l'orchestrazione di agenti autonomi
- Integrazione con Linear MCP e git
- Python/Node.js per sviluppo harness

**Validation Methodology:**
- Test del Disastro: Iniettare volontariamente bug e verificare rollback automatico
- Test dell'Autonomia: Assegnare ticket Linear e verificare completamento senza intervento umano
- Metriche di resilienza: Tasso di ripristino automatico > 90%, Integrità build 100%, Tasso regressione 0

### Implementation Considerations

I requisiti di dominio generale sono già integrati nel design MVP:

**Sicurezza:**
- Preflight checks bloccanti prima di spendere il primo token
- Isolamento sub-agenti con permessi fisicamente rimossi per QA Specialist (non solo vietati nel prompt)
- File checkpointing nativo SDK con rollback automatico

**Performance:**
- Memory Tool per persistenza decisioni chiave
- Context Exhaustion halt condition con routine di salvataggio emergenza
- Monitoring consumo token per finestra oraria

**User Experience:**
- Messaggi di errore chiari e azionabili
- Workflow autonomo senza "babysitting"
- Integrazione seamless con Linear e git

**Standard Requirements:**
- Architettura modulare e manutenibile
- Documentazione tecnica chiara
- Conformità alle best practices Python/Node.js

### Standard Requirements Summary

Il dominio "general" non introduce requisiti aggiuntivi oltre a quelli già documentati nel PRD. I requisiti standard (sicurezza, performance, UX, best practices) sono già integrati nel design MVP attraverso:

- Preflight checks rigorosi
- File checkpointing e rollback automatico
- Architettura sub-agenti isolata
- Memory Tool per persistenza
- Integrazione Linear e git
- Metriche di performance e monitoring

Non sono necessari requisiti regolamentari o compliance specifici di settore per questo dominio.

## Innovation & Novel Patterns

### Detected Innovation Areas

**Linear Coding Agent Harness** introduce un nuovo paradigma per l'orchestrazione di agenti AI autonomi, trasformando gli agenti Claude da esecutori lineari vulnerabili a sistemi resilienti capaci di auto-correzione. L'innovazione risiede nella combinazione unica di:

**1. Nuovo Paradigma di Resilienza Integrata:**
- **File Checkpointing Nativo SDK + Rollback Automatico:** Integrazione profonda delle primitive avanzate dell'SDK Claude (`rewind_files`, `enable_file_checkpointing`) con logica di business esterna (Linear, git) gestita programmaticamente dal codice Python dell'harness. Questo crea un sistema di "Safety Net" che previene errori distruttivi prima che diventino permanenti, trasformando l'esperienza da "l'agente mi rompe il progetto" a "l'agente si auto-corregge autonomamente".

**2. Orchestrazione Deterministica Pre-Agente:**
- **Prioritizzazione Assoluta Hard-Coded:** La logica di selezione ticket (Todo > In-Progress) è gestita **deterministicamente e hard-coded in Python** prima dell'istanziazione di `ClaudeSDKClient`. L'harness esegue le query MCP (`mcp__linear__list_issues`) **prima** di istanziare il client e passa all'agente solo il ticket preselezionato, evitando che l'agente possa "allucinare" la priorità o decidere autonomamente su cosa lavorare. Questo approccio elimina un punto di fallimento critico degli agenti AI autonomi: la decisione su cosa lavorare.

**3. Architettura Sub-Agenti Isolata e Programmabile:**
- **Definizione Programmatica con Isolamento Rigoroso:** I sub-agenti sono definiti via codice (`AgentDefinition`) per controllo dinamico e isolamento superiore, con separazione netta tra infrastruttura (codice harness) e intelligenza (prompting agente). L'innovazione chiave è la **rimozione fisica** dei tool di scrittura (`Edit`, `Write`) dalla configurazione del QA Specialist, non solo un'istruzione nel prompt. Questo garantisce tecnicamente che il QA agisca come "Gatekeeper" imparziale, evitando falsi positivi e regressioni nascoste.

**4. Persistenza Multi-Livello:**
- **Memory Tool + Linear MCP + Git:** Sistema di persistenza a tre livelli che garantisce recupero dello stato anche dopo riavvio sessione. Il Memory Tool salva decisioni architetturali chiave, Linear traccia lo stato operativo, e git fornisce audit trail consolidato. Questo risolve il problema critico della perdita di contesto nei progetti long-running.

**5. Timing Tecnologico Favorevole:**
- L'innovazione è resa possibile dal timing favorevole: l'SDK Claude ha reso disponibili nativamente le primitive necessarie (`rewind_files`, `enable_file_checkpointing`), e i modelli Claude 4.5 Sonnet/Opus con capacità di ragionamento estese abilitano sub-agenti specializzati affidabili come QA Specialist.

### Market Context & Competitive Landscape

**Posizionamento Competitivo:**

Il prodotto si posiziona in uno spazio emergente tra:
- **Agent Orchestration Tools** (es. LangChain, AutoGPT): Ma con resilienza integrata e auto-correzione che mancano negli strumenti esistenti
- **CI/CD Automation Tools** (es. GitHub Actions, GitLab CI): Ma con intelligenza AI che può adattarsi a task complessi invece di workflow rigidi
- **Code Generation Tools** (es. GitHub Copilot, Cursor): Ma con autonomia completa e gestione di progetti multi-giorno invece di assistenza incrementale

**Differenziazione Chiave:**

La difficoltà di replica deriva dall'orchestrazione precisa di feature beta avanzate (`mcp-client`) che devono lavorare in concerto, e dalla logica di "Prioritizzazione Assoluta" gestita deterministicamente in Python prima dell'istanziazione del client. Non è sufficiente avere accesso all'SDK Claude - serve una comprensione approfondita di come orchestrare queste primitive avanzate con logica di business esterna per creare un sistema resiliente.

**Competitive Advantage:**

Il vantaggio competitivo risiede nell'uso nativo e profondo delle funzionalità avanzate dell'SDK Claude integrate con logica di business esterna gestita programmaticamente. La combinazione di:
- File checkpointing nativo SDK
- Rollback automatico su fallimento test
- Sub-agenti isolati con permessi controllati
- Prioritizzazione deterministica hard-coded
- Persistenza multi-livello (Memory Tool + Linear + Git)

crea un sistema che non ha equivalenti diretti nel mercato attuale.

### Validation Approach

**Validazione dell'Innovazione:**

**1. Test del Disastro (Resilience Validation):**
- Iniettare volontariamente un bug nel codice generato
- Verificare che l'harness rilevi il problema tramite QA Specialist
- Verificare che esegua automaticamente il rollback tramite `rewind_files()`
- Verificare che lasci il repo pulito come prima
- **Success Criteria:** Rollback automatico funzionante nel 100% dei casi di test falliti

**2. Test dell'Autonomia (Autonomy Validation):**
- Assegnare un ticket Linear "Creare un file hello.py"
- Lanciare lo script e lasciare l'agente lavorare autonomamente
- Verificare che il ticket su Linear sia "Done" e il file esista
- Verificare che i test siano passati e il codice sia conforme alla "Definition of Done"
- **Success Criteria:** Completamento autonomo senza intervento umano per task di media complessità

**3. Test della Persistenza (Long-Running Validation):**
- Avviare un task complesso che richiede più sessioni
- Simulare un riavvio sessione (esaurimento contesto o crash)
- Verificare che l'harness recuperi lo stato dal Memory Tool e Linear
- Verificare che possa riprendere il lavoro dal punto di interruzione
- **Success Criteria:** Recupero completo dello stato dopo riavvio sessione

**4. Test della Prioritizzazione Deterministica:**
- Creare multiple ticket Linear con stati diversi (In Progress, Todo)
- Verificare che l'harness selezioni sempre il ticket "In Progress" prima di "Todo"
- Verificare che l'agente non possa "allucinare" la priorità o decidere autonomamente
- **Success Criteria:** Selezione ticket sempre deterministica e prevedibile

**5. Test dell'Isolamento Sub-Agenti:**
- Verificare che il QA Specialist non possa modificare codice di produzione
- Verificare che i tool `Edit` e `Write` siano fisicamente rimossi dalla configurazione
- Tentare di far modificare codice al QA Specialist e verificare che fallisca tecnicamente
- **Success Criteria:** Isolamento rigoroso verificato tecnicamente, non solo tramite prompt

### Risk Mitigation

**Rischi dell'Innovazione e Strategie di Mitigazione:**

**1. Rischio: Feature Beta SDK Instabili**
- **Mitigazione:** L'harness è progettato per essere resiliente a cambiamenti SDK. Il file checkpointing e rollback funzionano indipendentemente dalla stabilità delle altre feature. Se una feature beta diventa instabile, l'harness può continuare a funzionare con le primitive base.

**2. Rischio: Sub-Agenti Non Affidabili**
- **Mitigazione:** Il QA Specialist ha permessi rigorosamente limitati (tool Edit/Write fisicamente rimossi). Se il QA Specialist fallisce, il sistema esegue rollback automatico. Il fallback è sempre il rollback, non la modifica del codice.

**3. Rischio: Context Exhaustion in Sessioni Lunghe**
- **Mitigazione:** Context Exhaustion halt condition con routine di salvataggio emergenza. Il sistema salva lo stato attuale nel Memory Tool, aggiorna Linear con commento dettagliato, e si ferma con grazia. Il recupero è garantito dalla persistenza multi-livello.

**4. Rischio: Prioritizzazione Deterministica Non Scalabile**
- **Mitigazione:** La logica di prioritizzazione è hard-coded ma può essere estesa. Per l'MVP, la priorità è "Safety First" - meglio una logica semplice e deterministica che una complessa e imprevedibile. L'estensione è differita alla Fase 2.

**5. Rischio: Replica da Competitori**
- **Mitigazione:** La difficoltà di replica deriva dall'orchestrazione precisa di feature beta avanzate che devono lavorare in concerto. Non è sufficiente avere accesso all'SDK - serve comprensione approfondita e timing favorevole. Il vantaggio competitivo è temporaneo ma significativo durante la fase di adozione early.

**Fallback Strategy:**

Se l'innovazione non funziona come previsto:
- **Fallback 1:** Disabilitare sub-agenti e usare solo Main Agent con checkpointing base
- **Fallback 2:** Ridurre autonomia e richiedere conferma umana per operazioni critiche
- **Fallback 3:** Focus su resilienza invece di autonomia - mantenere Safety Net anche senza sub-agenti

L'MVP è progettato per essere "Safety First" - anche se l'autonomia fallisce, la resilienza deve funzionare.

## Developer Tool Specific Requirements

### Project-Type Overview

**Linear Coding Agent Harness** è uno strumento per sviluppatori (`developer_tool`) che opera come sistema di orchestrazione resiliente per agenti Claude autonomi. Il tool è progettato per essere integrato nel workflow di sviluppo esistente, lavorando con strumenti standard (git, Linear, Claude SDK) per trasformare agenti AI da esecutori lineari vulnerabili a sistemi autonomi capaci di auto-correzione.

Il tool si posiziona come infrastruttura Python che orchestra l'SDK Claude Agent, gestendo persistenza esterna dello stato, rollback automatico, e isolamento rigoroso dei compiti tramite sub-agenti specializzati. L'harness stesso è scritto in Python, ma può gestire progetti in qualsiasi linguaggio supportato dall'agente Claude (Python, JavaScript/TypeScript, Go, Rust, etc.).

### Technical Architecture Considerations

**Architettura Modulare:**
- **Harness Core (Python):** Gestisce orchestrazione, preflight checks, prioritizzazione deterministica, integrazione Linear MCP, e controllo del ciclo di vita degli agenti
- **Claude SDK Client:** Istanza di `ClaudeSDKClient` con:
  - `enable_file_checkpointing=True` e `permission_mode="acceptEdits"`
  - `extra_args={"replay-user-messages": None}` (obbligatorio per checkpointing)
  - Variabile d'ambiente `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1` (obbligatoria)
- **Sub-Agenti:** Definizioni programmatiche tramite `AgentDefinition` per controllo dinamico e isolamento
- **Memory Tool:** Persistenza decisioni chiave e stato tecnico
- **Linear MCP Integration:** Read-only + Status Update + Comment per tracking operativo (Runtime Profile), permessi completi per Bootstrap Profile
- **Tavily MCP Integration:** Server MCP Tavily già disponibile per ricerca informazioni mancanti o non precise, utilizzabile dall'agente orchestratore o da sub-agenti tramite istruzioni nel prompt
- **Git Integration:** Commit automatici quando QA passa, audit trail completo

**Separazione Infrastruttura/Intelligenza:**
- **Infrastruttura (Codice Harness):** Logica deterministica hard-coded in Python (prioritizzazione, preflight checks, orchestrazione)
- **Intelligenza (Prompting Agente):** Decisioni tecniche e implementazione gestite dall'agente Claude tramite prompt e sub-agenti

### Language Matrix

**Harness Implementation Language:**
- **Python 3.8+:** Linguaggio principale per l'implementazione dell'harness
- **Node.js:** Richiesto per MCP server Linear e integrazione con Claude Code CLI
- **Bash/Shell:** Utilizzato per preflight checks e integrazione con git

**Target Project Languages (Gestiti dall'Agente):**
L'harness può gestire progetti in qualsiasi linguaggio supportato dall'agente Claude, inclusi ma non limitati a:
- **Python:** Supporto completo per progetti Python con pytest, pip, virtualenv
- **JavaScript/TypeScript:** Supporto completo per progetti Node.js con npm, jest, yarn
- **Go:** Supporto per progetti Go con go test, go mod
- **Rust:** Supporto per progetti Rust con cargo, rustc
- **Altri linguaggi:** L'agente Claude può gestire progetti in qualsiasi linguaggio, purché il progetto abbia strumenti di test standard (test runner, package manager)

**Requisiti per Progetti Target:**
- Presenza di strumenti di test standard (pytest, npm test, go test, etc.)
- Presenza di git per versionamento e audit trail
- Struttura progetto standard (non richiesto, ma raccomandato)

### Installation Methods

**MVP Deployment Strategy:**
L'MVP è progettato per **esecuzione locale** su ambiente di sviluppo (PC/Mac/Linux). **Sviluppo Local-First:** L'infrastruttura, specialmente per quanto riguarda le dipendenze pesanti come il browser per i test visivi, sarà sviluppata inizialmente per funzionare in **locale (PC/Mac/Linux)**. Il deployment containerizzato (Docker) e le complessità serverless sono esplicitamente differiti alla **Fase 2**. L'MVP deve focalizzarsi sulla logica di orchestrazione e sicurezza in ambiente locale. **Priorità MVP:** Puppeteer (test end-to-end per progetti frontend/UI) è prioritario rispetto a Docker (deployment containerizzato). La grandezza dell'immagine Docker non è un problema per l'MVP poiché il deployment containerizzato è esplicitamente differito alla Fase 2 dopo validazione funzionale completa.

**Prerequisites (MVP - Ambiente Locale):**
- Python 3.8 o superiore
- Node.js (per MCP server Linear e Claude Code CLI)
- Git installato e configurato
- Claude Code CLI installato e configurato
- **Chrome/Chromium installato (per Puppeteer browser automation)** - Richiesto **solo per progetti che richiedono UI testing** (Web App frontend). Per progetti CLI/backend (Python, Rust, Go senza interfaccia grafica), Puppeteer è opzionale e non bloccante. Per l'MVP, Puppeteer è utilizzato esclusivamente in ambiente locale dove Chrome/Chromium è facilmente installabile. Il deployment containerizzato con Puppeteer sarà affrontato in Fase 2 dopo validazione funzionale completa.
- Dipendenze di sistema per browser headless (per Puppeteer) - Richieste solo se Puppeteer è necessario per il tipo di progetto. Installate automaticamente con Puppeteer o disponibili tramite package manager sistema operativo
- API Keys: `CLAUDE_CODE_OAUTH_TOKEN` (o API key Anthropic), `LINEAR_API_KEY`
- Variabile d'ambiente: `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1` (obbligatoria per checkpointing)

**Installation Steps:**

1. **Clone Repository:**
   ```bash
   git clone <repository-url>
   cd Linear-Coding-Agent-Harness
   ```

2. **Install Python Dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

3. **Configure Environment Variables:**
   ```bash
   export CLAUDE_CODE_OAUTH_TOKEN="your-token"
   export LINEAR_API_KEY="your-linear-api-key"
   export CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1  # Obbligatorio per checkpointing
   ```

4. **Verify Preflight Checks:**
   ```bash
   python agent.py --preflight-only
   ```

5. **Run Harness:**
   ```bash
   python agent.py
   ```

**Package Manager Support:**
- **pip:** Per dipendenze Python dell'harness
- **npm:** Per MCP server Linear (se necessario)
- **Non richiesto:** L'harness non richiede package manager specifici per progetti target (l'agente gestisce i package manager del progetto)


### API Surface

**Harness Core API:**

**Main Entry Point:**
- `agent.py`: Script principale che avvia l'harness e gestisce il ciclo di vita

**Key Classes/Modules:**
- `ClaudeSDKClient`: Wrapper per SDK Claude con checkpointing abilitato
- `LinearMCPClient`: Client per integrazione Linear MCP
- `AgentDefinition`: Definizione programmatica di sub-agenti
- `PreflightChecker`: Validazione ambiente prima dell'esecuzione
- `MemoryTool`: Gestione persistenza decisioni chiave

**Configuration:**
- `linear_config.py`: Configurazione Linear MCP
- `prompts.py`: Prompt per agenti e sub-agenti
- Environment variables: `CLAUDE_CODE_OAUTH_TOKEN`, `LINEAR_API_KEY`

**Command-Line Interface:**
- `python agent.py`: Avvia harness con prioritizzazione deterministica
- `python agent.py --preflight-only`: Esegue solo preflight checks
- `python agent.py --help`: Mostra opzioni disponibili

**MCP Integration:**
- `mcp__linear__list_issues`: Query Linear per lista ticket (eseguita da harness Python prima di istanziare client)
- `mcp__linear__update_issue`: Aggiornamento stato ticket (eseguito da agente)
- `mcp__linear__create_comment`: Creazione commenti su Linear (eseguito da agente)

**SDK Integration:**
- `client.enable_file_checkpointing(True)`: Abilita checkpointing file
- `client.rewind_files(uuid)`: Rollback automatico su fallimento
- `client.query()`: Query agente con sub-agenti e tool disponibili

### Code Examples

**Basic Usage:**

```python
# agent.py - Simplified example
from claude_sdk import ClaudeSDKClient
from linear_config import LinearMCPClient
from security import PreflightChecker

# Preflight checks (bloccanti)
checker = PreflightChecker()
checker.validate_all()  # Raises exception if validation fails

# Prioritizzazione deterministica (hard-coded in Python)
linear_client = LinearMCPClient()
issues = linear_client.list_issues()
selected_issue = prioritize_issues(issues)  # In-Progress > Todo

# Context Recovery per ticket "In Progress"
if selected_issue.status == "In Progress":
    # Istruire agente di leggere commenti Linear per ricostruire stato
    context_prompt = f"Read Linear issue comments to recover context: {selected_issue.id}"

# Istanziazione client con checkpointing (configurazione obbligatoria)
client = ClaudeSDKClient(
    enable_file_checkpointing=True,
    permission_mode="acceptEdits",
    extra_args={"replay-user-messages": None}  # Obbligatorio per checkpointing
)
# Nota: CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1 deve essere impostata come env var

# Definizione sub-agenti
qa_specialist = AgentDefinition(
    name="QA Specialist",
    tools=["bash", "read_file", "mcp__puppeteer__..."],  # Edit/Write fisicamente rimossi, Puppeteer per browser automation
    system_prompt=qa_prompt
)

# Query agente con ticket preselezionato
response = client.query(
    f"Work on Linear ticket: {selected_issue.id}",
    agents=[qa_specialist]
)

# Gestione rollback se QA fallisce
if qa_failed:
    client.rewind_files(checkpoint_uuid)
```

**Configuration Example:**

```python
# linear_config.py
LINEAR_API_KEY = os.getenv("LINEAR_API_KEY")
LINEAR_TEAM_ID = os.getenv("LINEAR_TEAM_ID", "default-team")

# Configurazione MCP
mcp_servers = {
    "linear": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-linear"],
        "env": {
            "LINEAR_API_KEY": LINEAR_API_KEY
        }
    }
}
```

**Sub-Agent Definition Example:**

```python
# Definizione QA Specialist con permessi ristretti
qa_specialist = AgentDefinition(
    name="QA Specialist",
    description="Gatekeeper imparziale per validazione codice",
    tools=["bash", "read_file"],  # Edit/Write NON inclusi
    system_prompt="""
    You are a QA Specialist. Your role is to validate code quality.
    CRITICAL: You CANNOT modify production code. You can only:
    - Run tests (bash pytest/npm test)
    - Validate UI/UX with browser automation (Puppeteer)
    - Read code files
    - Report issues
    """,
    max_turns=5
)
```

**Preflight Check Example:**

```python
# security.py
class PreflightChecker:
    def validate_all(self):
        self.check_python_version()
        self.check_nodejs_version()
        self.check_git_installed()
        self.check_claude_cli_installed()
        self.check_linear_connection()
        self.check_api_keys()
        self.check_checkpointing_env_var()  # Verifica CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1
        self.check_puppeteer_dependencies()  # Verifica Chrome/Chromium per Puppeteer
        
    def check_claude_cli_installed(self):
        try:
            result = subprocess.run(["claude", "--version"], 
                                  capture_output=True, check=True)
        except (subprocess.CalledProcessError, FileNotFoundError):
            raise CLINotFoundError("Claude Code CLI not found")
    
    def check_checkpointing_env_var(self):
        if not os.getenv("CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING"):
            raise ValueError("CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1 must be set")
    
    def check_puppeteer_dependencies(self):
        # Verifica presenza Chrome/Chromium per Puppeteer
        chrome_paths = ["/usr/bin/google-chrome", "/usr/bin/chromium", 
                       "/usr/bin/chromium-browser", "chromium"]
        found = False
        for path in chrome_paths:
            try:
                subprocess.run([path, "--version"], 
                             capture_output=True, check=True)
                found = True
                break
            except (subprocess.CalledProcessError, FileNotFoundError):
                continue
        if not found:
            raise RuntimeError("Chrome/Chromium not found. Required for Puppeteer browser automation.")
```

### Migration Guide

**Migrating from Manual Agent Usage:**

1. **Setup Environment:**
   - Installare dipendenze Python: `pip install -r requirements.txt`
   - Configurare variabili d'ambiente (CLAUDE_CODE_OAUTH_TOKEN, LINEAR_API_KEY)
   - Verificare preflight checks: `python agent.py --preflight-only`

2. **Configure Linear Integration:**
   - Creare team Linear se non esiste
   - Configurare LINEAR_API_KEY e LINEAR_TEAM_ID
   - Verificare connessione: `python agent.py --preflight-only`

3. **Migrate Existing Workflow:**
   - Creare ticket Linear per task esistenti
   - Tag ticket con `agent-task` per identificazione
   - Lanciare harness: `python agent.py`
   - Harness seleziona automaticamente ticket In-Progress

4. **Adapt Project Structure:**
   - Assicurarsi che progetto abbia strumenti di test standard
   - Configurare git se non già presente
   - Verificare che struttura progetto sia compatibile con agente Claude

**Migrating from Other Agent Orchestration Tools:**

1. **From LangChain/AutoGPT:**
   - Harness usa SDK Claude nativo invece di wrapper generici
   - File checkpointing integrato invece di gestione manuale
   - Sub-agenti definiti programmaticamente invece di configurazione file

2. **From Custom Scripts:**
   - Sostituire logica di orchestrazione con harness
   - Migrare configurazione a environment variables
   - Adattare prompt esistenti a formato AgentDefinition

**Backward Compatibility:**
- Harness è compatibile con progetti esistenti senza modifiche
- Non richiede refactoring codice progetto target
- Funziona con qualsiasi struttura progetto supportata da agente Claude

### Implementation Considerations

**Developer Experience:**
- **CLI-First:** Interfaccia principale è command-line, integrata con workflow sviluppatore esistente
- **Transparent Operations:** Tutte le operazioni sono tracciate su Linear con commenti dettagliati
- **Error Messages:** Messaggi di errore chiari e azionabili (preflight checks)
- **Documentation:** Documentazione tecnica completa per sviluppatori

**Integration Points:**
- **Git:** Commit automatici quando QA passa, audit trail completo
- **Linear:** Read-only + Status Update + Comment per tracking operativo
- **Claude SDK:** Integrazione nativa con file checkpointing e rollback
- **Terminal/Bash:** Preflight checks e integrazione con strumenti standard

**Extensibility:**
- **Sub-Agenti:** Facile aggiungere nuovi sub-agenti tramite AgentDefinition
- **Preflight Checks:** Estendibile con nuovi controlli validazione
- **Memory Tool:** Estendibile con nuove categorie di decisioni da persistere
- **Linear Integration:** Estendibile con nuove operazioni MCP (post-MVP)

**Performance Considerations:**
- **Token Efficiency:** Monitoring consumo token per finestra oraria
- **Context Management:** Context Exhaustion halt condition con routine salvataggio emergenza
- **Parallel Operations:** Sub-agenti possono operare in parallelo quando appropriato
- **Caching:** Memory Tool cache decisioni chiave per evitare ri-chiedere informazioni

**Security Considerations:**
- **API Keys:** Gestione sicura tramite environment variables
- **Preflight Checks:** Validazione ambiente prima di spendere token
- **Isolamento Sub-Agenti:** Permessi controllati fisicamente rimossi dalla configurazione
- **File Checkpointing:** Rollback automatico previene modifiche distruttive
- **Security Profiles Differenziati:**
  - **Bootstrap Profile (Initializer Agent):** Profilo con permessi elevati utilizzato esclusivamente durante la fase di setup iniziale (Session 1). Permette creazione ticket Linear (`mcp__linear__create_issue`) per popolare il backlog iniziale. Questo profilo è un'eccezione alla regola di sicurezza e deve essere utilizzato solo durante il bootstrap del progetto.
  - **Runtime Profile (Coding Agent):** Profilo con permessi ristretti utilizzato per tutte le operazioni day-to-day. Read-Only + Status Update + Comment su Linear. Non può creare ticket o modificare epiche complesse. Questo profilo previene che un agente fuori controllo inondi il backlog di ticket spazzatura durante le operazioni normali.

## Project Scoping & Phased Development

### MVP Strategy & Philosophy

**MVP Approach:** Problem-Solving MVP con focus su Safety First

Il Linear Coding Agent Harness adotta un approccio **Problem-Solving MVP** che risolve direttamente il problema critico identificato: la perdita di contesto e gli errori distruttivi nei progetti long-running con agenti AI autonomi. La filosofia MVP è **"Safety First, Autonomy Second"** - l'MVP mira alla robustezza, non a sessioni "infinite" o intelligenza suprema.

**Criterio Decisionale MVP:** *Se l'agente sbaglia, il progetto sopravvive?*

**Resource Requirements:**
- **Team Size:** 1-2 sviluppatori full-time (Python/Node.js)
- **Skills Required:** 
  - Comprensione approfondita SDK Claude Agent avanzato
  - Integrazione MCP (Linear)
  - Python per orchestrazione harness
  - Best practices orchestrazione agenti autonomi
- **Timeline:** 3 mesi per MVP
- **Budget Considerations:** Costi operativi per API Claude (token consumption), infrastruttura minima (CLI-based, no GUI)

**Strategic Rationale:**
L'MVP risolve il problema principale dell'utente primario (Alex - Senior Software Engineer): la paura che "l'agente mi rompa il progetto". Il sistema garantisce che o consegna codice funzionante e testato, o non tocca nulla. Questo crea fiducia iniziale necessaria per adozione, permettendo iterazione e miglioramento nelle fasi successive.

### MVP Feature Set (Phase 1)

**Core User Journeys Supported:**

**Journey 1: Alex - Dal "Babysitting" alla Fiducia (ESSENZIALE MVP)**
- Preflight checks rigorosi con messaggi chiari
- Integrazione Linear per creazione e tracking ticket
- File checkpointing e rollback automatico
- Memory Tool per persistenza piano implementazione (workflow standard)
- Workflow autonomo senza "babysitting"
- Git commit eseguito dall'Agente con messaggi semantici

**Journey 4: Luca - DevOps Engineer - Configurazione e Monitoring (ESSENZIALE MVP)**
- Configurazione ambiente con variabili d'ambiente chiare
- Preflight checks che validano configurazione
- Basic monitoring (metriche base, context exhaustion alerts)

**Journey 2: Sarah - Product Owner (POST-MVP)**
- Visibilità Linear in tempo reale (MVP supporta base, enhancement post-MVP)
- Notifiche avanzate (post-MVP)

**Journey 3: Marco - QA Engineer (MVP PARZIALE)**
- QA Specialist come sub-agente isolato (MVP)
- PR con codice testato (MVP base)
- Integrazione workflow review avanzata (post-MVP)

**Must-Have Capabilities:**

**A. Safety Net (Checkpointing & Rollback) - CRITICA**
- File checkpointing nativo SDK (`enable_file_checkpointing=True`)
- Rollback automatico su fallimento test (`rewind_files()`)
- Preflight checks rigorosi prima di spendere token
- Context Exhaustion halt condition con soglia conservativa (80-85%)

**B. Architettura Sub-Agenti - ALTA**
- QA Specialist isolato con permessi ristretti (tool Edit/Write fisicamente rimossi)
- Main Agent orchestratore
- Definizione programmatica tramite `AgentDefinition`

**C. Integrazione Linear "Read-Only + Status Update" - ALTA**
- Prioritizzazione deterministica hard-coded in Python (In-Progress > Todo)
- Query Linear eseguita da Harness prima di istanziare client
- Status update e commenti su Linear
- Esclusione MVP: creazione ticket complessi, modifica epiche

**D. Memory Tool Integration - ALTA**
- Consultazione all'avvio per recuperare decisioni pregresse
- Salvataggio piano implementazione all'inizio task
- Scrittura decisioni chiave alla fine ciclo
- Routine salvataggio emergenza per context exhaustion

**E. Git Integration - ALTA**
- Commit eseguito dall'Agente (dopo QA passa) con messaggi semantici
- Audit trail completo
- Rollback gestito da Harness (`rewind_files`)

**MVP Success Criteria:**
1. **Test del Disastro:** Bug iniettato volontariamente → QA rileva → Rollback automatico → Repo pulito
2. **Test dell'Autonomia:** Ticket Linear "Creare file hello.py" → Utente va via → Ticket "Done" → File esiste → Test passano

**MVP Deliverable:**
"Developer Junior Cauto" - Non velocissimo, non gestisce progetti enormi, ma ha "rete di sicurezza" infallibile: o consegna codice funzionante e testato, o non tocca nulla.

### Post-MVP Features

**Phase 2: Growth (6-9 mesi)**

**Focus:** Gestione Contesto Avanzata, Scalabilità e Deployment Containerizzato

**Key Features:**
- **Deployment Containerizzato:** Immagini Docker ottimizzate con Puppeteer e dipendenze browser headless, guide deployment per ambienti containerizzati, considerazioni per ambienti serverless
- **Tool Result Clearing:** Implementazione completa pulizia contesto server-side
- **Context Compaction:** Strategie avanzate compattazione per sessioni multi-giorno
- **Enhanced Sub-Agents:** Definizione completa team sub-agenti (Docs Specialist, Coding Specialist) basata su DNA-grafi
- **Advanced Linear Integration:** Creazione ticket complessi, gestione epiche
- **Enhanced Monitoring:** Dashboard avanzata, metriche dettagliate, alerting sofisticato

**User Journeys Enhanced:**
- Journey 2 (Sarah): Visibilità completa in tempo reale, notifiche avanzate
- Journey 3 (Marco): Integrazione workflow review completa
- Nuovi journeys per utenti secondari (Engineering Manager, Product Owner avanzato)

**Success Metrics:**
- Capacità di gestire sessioni multi-giorno senza degradazione performance
- Riduzione 50% tempo su bug fix triviali e boilerplate
- Adozione da parte di utenti secondari

**Phase 3: Expansion (9-12 mesi)**

**Focus:** Autonomia e Apprendimento

**Key Features:**
- **Docs Specialist Autonomo:** Generazione e aggiornamento documentazione automatica
- **Memory Tool Avanzato:** Knowledge base persistente complessa, apprendimento incrementale
- **Enhanced Autonomy:** Gestione task complessi multi-step senza intervento umano
- **Advanced Analytics:** Pattern recognition, ottimizzazione automatica workflow

**User Journeys Enhanced:**
- Gestione autonoma di task complessi multi-giorno
- Apprendimento da pattern di successo
- Ottimizzazione automatica basata su metriche

**Success Metrics:**
- Gestione autonoma di epic complesse senza intervento umano intermedio
- Apprendimento incrementale dimostrabile
- Adozione da parte di team distribuiti

**Phase 4: Platform (12+ mesi)**

**Focus:** Piattaforma e Ecosistema

**Key Features:**
- **Epic Management:** Gestione autonoma di intere epic end-to-end
- **Multi-Project Orchestration:** Gestione simultanea di progetti multipli
- **Ecosystem Integration:** Integrazione con altri tool (Jira, GitHub Actions, CI/CD)
- **Advanced Learning:** Apprendimento da pattern cross-project
- **Enterprise Features:** Multi-tenant, RBAC avanzato, compliance enterprise

**User Journeys Enhanced:**
- Gestione portfolio progetti
- Orchestrazione multi-team
- Enterprise adoption

**Success Metrics:**
- Gestione autonoma di portfolio progetti
- Adozione enterprise
- Ecosistema integrato con tool standard

### Risk Mitigation Strategy

**Technical Risks:**

**Rischio 1: Feature Beta SDK Instabili**
- **Mitigazione:** Design resiliente - file checkpointing e rollback funzionano indipendentemente da altre feature. Se feature beta instabile, harness continua con primitive base.
- **Contingency:** Fallback a primitive SDK stabili, disabilitazione feature beta problematiche

**Rischio 2: Sub-Agenti Non Affidabili**
- **Mitigazione:** QA Specialist con permessi rigorosamente limitati (tool Edit/Write fisicamente rimossi). Fallback sempre rollback, non modifica codice.
- **Contingency:** Disabilitare sub-agenti, usare solo Main Agent con checkpointing base

**Rischio 3: Context Exhaustion Frequente**
- **Mitigazione:** Soglia conservativa (80-85% limite modello), routine salvataggio emergenza, Memory Tool per persistenza
- **Contingency:** Ridurre complessità task MVP, richiedere conferma umana per task lunghi

**Rischio 4: Prioritizzazione Deterministica Non Scalabile**
- **Mitigazione:** Logica hard-coded semplice e deterministica per MVP. Estensione differita Fase 2.
- **Contingency:** Mantenere logica semplice anche in Fase 2, priorità "Safety First"

**Market Risks:**

**Rischio 1: Adozione Lenta**
- **Mitigazione:** MVP risolve problema critico immediato (paura "agente rompe progetto"). Focus su early adopters (Senior Engineers come Alex).
- **Validation:** Test con utenti beta prima di Fase 2, raccolta feedback continuo

**Rischio 2: Competizione da Tool Simili**
- **Mitigazione:** Vantaggio competitivo temporaneo da orchestrazione precisa feature beta avanzate. Difficoltà replica deriva da comprensione approfondita SDK.
- **Validation:** Monitorare competitività, iterare rapidamente su differenziazione

## Functional Requirements

### Environment Validation & Setup

- **FR1:** L'harness può validare l'ambiente prima di spendere il primo token, verificando API Keys (Claude, Linear), versioni Python/Node.js, presenza di Git, connessione a Linear, presenza di Claude Code CLI, variabile d'ambiente `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1`. Per progetti che richiedono UI testing (Web App frontend), l'harness può verificare anche dipendenze Puppeteer (Chrome/Chromium) come controllo condizionale non bloccante per progetti CLI/backend.

- **FR2:** L'harness può fornire messaggi di errore chiari e azionabili quando i preflight checks falliscono, identificando specificamente cosa manca o è configurato incorrettamente.

- **FR3:** L'harness può gestire tipi di errore specifici dell'SDK (`CLINotFoundError`, `CLIConnectionError`) e fallire gracefulmente con feedback azionabile invece di crash oscuri.

- **FR4:** L'utente può configurare l'harness tramite variabili d'ambiente senza modificare codice sorgente.

### Ticket Management & Prioritization

- **FR5:** L'harness può connettersi a Linear MCP e leggere la lista di ticket disponibili.

- **FR6:** L'harness può selezionare deterministicamente un ticket da lavorare applicando la logica hard-coded "In Progress" prima di "Todo", senza delegare questa decisione all'agente.

- **FR7:** L'harness può passare all'agente solo il ticket preselezionato, non la lista completa di ticket disponibili.

- **FR8:** L'agente può leggere ticket Linear esistenti per comprendere i requisiti e i criteri di accettazione.

- **FR9:** L'agente può aggiornare lo stato di un ticket Linear (es. "In Progress" → "Done", "In Review").

- **FR10:** L'agente può aggiungere commenti dettagliati su ticket Linear per comunicare progresso, decisioni prese, e blocchi tecnici.

- **FR11:** L'agente può recuperare il contesto di un ticket "In Progress" leggendo la history dei commenti Linear per ricostruire lo stato mentale della sessione interrotta.

- **FR12:** L'Initializer Agent può leggere `app_spec.txt` (già presente al trigger) per creare `feature_list.json` (progetto Linear in formato JSON) e generare infrastruttura Linear ottimale durante la fase di bootstrap (Session 1), inclusa creazione progetto Linear con team ID, generazione Linear issues ottimamente strutturate per ogni feature copiando da `feature_list.json` (deterministico), e organizzazione in epiche e milestone. **Requisito Essenziale:** Il file `feature_list.json` è un **requisito essenziale** per l'MVP, non opzionale. Questo file funge da **checkpoint fisico di recupero** tra la lettura di `app_spec.txt` e la creazione delle issue su Linear. **Scenario di Recovery:** Se l'Initializer Agent esaurisce la context window mentre crea le issue su Linear, il sistema deve poter terminare la sessione e avviarne una nuova che riprenda a generare le issue leggendo dal file `feature_list.json` già esistente, senza dover ri-ragionare sull'intera architettura. Questo garantisce resilienza e continuità operativa anche in caso di interruzioni durante la fase di bootstrap.

- **FR13:** Il Coding Agent non può creare nuovi ticket o modificare epiche complesse durante le operazioni day-to-day (restrizione MVP per prevenire spam di ticket).

### Code Execution & Safety

- **FR14:** L'harness può creare checkpoint automatici dei file prima di ogni modifica del codice, salvando `message.uuid` e `checkpoint_uuid` per ogni iterazione.

- **FR15:** L'harness può eseguire rollback automatico (`rewind_files`) quando i test falliscono, ripristinando lo stato dei file al checkpoint precedente.

- **FR16:** L'harness può bloccare operazioni pericolose (es. modifiche a `.env`, `rm -rf`) usando Hook SDK (`PreToolUse` con decisione `deny`) prima che vengano eseguite.

- **FR17:** L'agente può modificare file di codice nel progetto per implementare feature e fix.

- **FR18:** L'agente può leggere file di codice esistenti per comprendere la struttura del progetto e le convenzioni.

- **FR19:** L'harness può rilevare quando un'operazione distruttiva viene tentata e prevenire la sua esecuzione, eseguendo rollback se necessario.

### Quality Assurance & Testing

- **FR20:** Il QA Specialist può eseguire test CLI (pytest, npm test) per validare che il codice funzioni correttamente.

- **FR21:** Il QA Specialist può eseguire test end-to-end usando browser automation (Puppeteer) per validare l'interfaccia utente e il comportamento dell'applicazione.

- **FR22:** Il QA Specialist può leggere file di codice per analizzare la qualità e identificare potenziali problemi.

- **FR23:** Il QA Specialist non può modificare codice di produzione (tool `Edit` e `Write` fisicamente rimossi dalla configurazione).

- **FR24:** Il QA Specialist può riportare risultati dei test (passati/falliti) al Main Agent.

- **FR25:** L'harness può determinare se un ticket può essere marcato come "Done" basandosi sui risultati del QA Specialist.

- **FR26:** Il QA Specialist può validare che il codice sia conforme alla "Definition of Done" prima che il ticket venga marcato come completato.

### State Persistence & Recovery

- **FR27:** L'agente può consultare il Memory Tool all'avvio per recuperare decisioni architetturali pregresse (ID progetto, scelte architetturali, piani di implementazione precedenti).

- **FR28:** L'agente può salvare decisioni chiave nel Memory Tool **esclusivamente alla conclusione di una 'Epic'**, non ad ogni iterazione o task singolo. L'obiettivo è la persistenza di alto livello, non il tracciamento granulare dello stato operativo (che rimane delegato a Linear e Git). **Nota Implementazione:** Questo requisito funzionale deve essere esplicitamente riflesso nei prompt dell'agente (`prompts.py`) per essere effettivo. L'uso deve essere strategico e cognitivo, limitato a informazioni "cognitivamente rilevanti" per la continuità del lavoro (ID progetto, scelte architetturali chiave, piano implementazione ad alto livello) piuttosto che log dettagliati o stato intermedio ricostruibile. Il Memory Tool non è un meccanismo di tracciamento operativo dettagliato - Linear e Git gestiscono lo stato operativo granulare.

- **FR29:** L'agente può scrivere decisioni chiave nel Memory Tool **esclusivamente alla conclusione di una 'Epic'** (piano di implementazione aggiornato ad alto livello, ID progetto Linear, scelte architetturali significative). Il Memory Tool non è un meccanismo di tracciamento operativo dettagliato - Linear e Git gestiscono lo stato operativo granulare. **Nota Implementazione:** Questo requisito funzionale deve essere esplicitamente riflesso nei prompt dell'agente (`prompts.py`) per essere effettivo. La frequenza e il dettaglio delle scritture devono essere ottimizzati per la conclusione di Epic, salvando solo informazioni "cognitivamente rilevanti" per la continuità del lavoro. Evitare di salvare log dettagliati di esecuzione o stato intermedio che può essere ricostruito da Linear o git. Bilanciare persistenza necessaria con efficienza token evitando overhead eccessivo nel ciclo operativo.

- **FR30:** L'agente può eseguire una routine di salvataggio di emergenza quando il contesto si esaurisce, salvando lo stato attuale nel Memory Tool e aggiornando Linear con un commento dettagliato.

- **FR31:** L'harness può recuperare lo stato di un progetto interrotto combinando informazioni dal Memory Tool e dai commenti Linear.

### Context Management

- **FR32:** L'harness può monitorare l'utilizzo dei token tramite `usage` nei messaggi per determinare quando avvicinarsi al limite del contesto.

- **FR33:** L'harness può innescare una halt condition quando si raggiunge una soglia conservativa di token (80-85% del limite fisico), garantendo spazio sufficiente per salvataggio di emergenza. **Nota Comportamento:** La halt condition è un meccanismo tecnico di gestione memoria che permette all'agente di salvare stato e riprendere lavoro autonomamente, non una terminazione permanente del lavoro.

- **FR34:** L'agente può fermare l'esecuzione con grazia quando la halt condition viene raggiunta, eseguendo la routine di salvataggio di emergenza prima di terminare. **Meccanismo Tecnico:** L'harness Python non deve interrompere brutalmente l'agente (`client.interrupt()`), ma deve iniettare un messaggio di sistema (es. "ATTENZIONE: Token in esaurimento. Salva stato e chiudi.") nel flusso di conversazione quando si tocca la soglia. L'agente deve essere istruito nel System Prompt a reagire a questo specifico trigger con la routine di salvataggio (salvataggio Memory Tool, aggiornamento Linear, fermata con grazia). Questo garantisce che l'agente possa utilizzare i tool necessari per salvare lo stato prima di terminare, invece di essere interrotto dall'esterno senza possibilità di salvataggio. **Chiarimento:** Dopo il salvataggio, l'agente deve riprendere immediatamente il lavoro autonomamente, non terminare permanentemente. La halt condition è gestione memoria, non blocco operativo.

- **FR35:** L'harness può utilizzare clearing passivo del contesto (`clear_tool_uses_20250919`) come ottimizzazione server-side automatica per allungare la vita operativa dell'agente. **Nota Tecnica:** Questa funzionalità richiede l'header beta `context-management-2025-06-27` nell'SDK Claude. È un'ottimizzazione "passiva" essenziale per sessioni lunghe che riduce il consumo token senza aggiungere complessità di codice lato harness.

### Git Integration & Version Control

- **FR36:** L'agente può eseguire commit git con messaggi semantici e descrittivi dopo che il QA Specialist ha validato il codice.

- **FR37:** L'agente può includere link ai commit git nei commenti Linear per tracciabilità.

- **FR38:** L'harness può garantire che solo codice validato dal QA Specialist venga committato, evitando di sporcare la history con commit intermedi falliti.

- **FR39:** L'harness può utilizzare git come audit trail permanente, distinguendolo dal checkpointing effimero dell'SDK utilizzato per rollback tattico.

### Agent Orchestration & Sub-Agents

- **FR40:** L'harness può definire sub-agenti programmaticamente tramite `AgentDefinition` con permessi controllati e isolamento rigoroso.

- **FR41:** L'harness può configurare il QA Specialist come sub-agente isolato con toolset dinamico basato sul tipo di progetto. Il toolset base include sempre: tool di test CLI (`bash` per pytest/npm test/cargo test/go test) e tool di lettura (`read_file`). Il tool di browser automation (`mcp__puppeteer__...` o `mcp__playwright__...`) è aggiunto condizionalmente solo per progetti che richiedono test end-to-end e validazione UI. I tool di scrittura (`Edit`, `Write`) sono fisicamente rimossi dalla configurazione.

- **FR42:** Il Main Agent può orchestrare sub-agenti specializzati per distribuire il carico di lavoro e migliorare l'efficienza.

- **FR42.1:** L'agente orchestratore (Main Agent) o i sub-agenti possono utilizzare il server MCP Tavily (`mcp__tavily__...`) per recuperare informazioni mancanti o non precise durante l'esecuzione. Il Tavily MCP Server ha lo scopo di permettere ai client compatibili con il protocollo MCP di utilizzare direttamente le API di Tavily all'interno dei propri sistemi. Il suo utilizzo tipico è quello di potenziare gli agenti AI fornendo loro un accesso al web in tempo reale per eseguire ricerche, estrarre contenuti e svolgere indagini approfondite in modo autonomo. Tavily è già disponibile come Linear (configurato nel sistema MCP) e può essere utilizzato tramite istruzioni nel prompt senza richiedere codice aggiuntivo. L'uso di Tavily deve essere strategico e limitato a situazioni dove le informazioni non sono disponibili nel contesto locale o nel Memory Tool.

- **FR43:** L'harness può utilizzare profili di sicurezza differenziati: Bootstrap Profile (permessi elevati per Initializer Agent) e Runtime Profile (permessi ristretti per Coding Agent).

- **FR44:** L'harness può istanziare il `ClaudeSDKClient` con configurazione corretta (`enable_file_checkpointing=True`, `permission_mode="acceptEdits"`, `extra_args={"replay-user-messages": None}`).

### Workflow Orchestration

- **FR45:** L'harness può eseguire la sequenza operativa completa: checkpoint pre-modifica, coding loop, QA check, rollback se necessario, commit git se QA passa.

- **FR46:** L'harness può gestire il ciclo di vita completo di un ticket Linear dalla selezione al completamento, aggiornando lo stato e aggiungendo commenti appropriati.

- **FR47:** L'harness può gestire interruzioni di sessione, permettendo all'agente di riprendere il lavoro su un ticket "In Progress" recuperando il contesto dai commenti Linear e dal Memory Tool.

- **FR48:** L'harness può operare autonomamente senza richiedere intervento umano continuo ("babysitting") per task di media complessità.

### Error Handling & Resilience

- **FR49:** L'harness può gestire errori durante l'esecuzione (test falliti, eccezioni, operazioni bloccate) eseguendo rollback automatico quando appropriato.

- **FR50:** L'harness può lasciare il progetto in uno stato pulito e funzionante anche quando un task fallisce, prevenendo corruzione del codice.

- **FR51:** L'harness può comunicare chiaramente all'utente quando un task viene interrotto o fallisce, spiegando la situazione tramite commenti Linear.

### Monitoring & Observability

- **FR52:** L'harness può tracciare metriche di performance (tempo completamento ticket, frequenza rollback) per monitoring operativo.

- **FR53:** L'harness può monitorare il consumo di token per finestra oraria per evitare di raggiungere limiti dell'abbonamento.

- **FR54:** L'harness può generare alert quando si verificano situazioni critiche (context exhaustion, errori ripetuti, fallimenti preflight).

- **FR55:** L'harness può fornire audit trail completo tramite git commits e commenti Linear per tracciabilità delle operazioni.

### Implementation Considerations & Risk Mitigation

**Nota:** Questa sezione evidenzia aree critiche dove la definizione del PRD potrebbe creare frizione rispetto all'implementazione reale, richiedendo attenzione particolare durante la fase di architettura e sviluppo.

#### 1. Memory Tool Integration nel Workflow Standard

**Rischio Identificato:**
I requisiti FR28 e FR29 definiscono che il Memory Tool deve essere aggiornato **esclusivamente alla conclusione di una Epic**, non ad ogni iterazione o task singolo. Questo approccio bilancia persistenza necessaria con efficienza token, evitando overhead eccessivo che potrebbe degradare le performance operative.

**Considerazioni:**
- **Prompt Design Critico:** I requisiti FR28 e FR29 devono essere esplicitamente riflessi nei prompt dell'agente (`prompts.py`) per essere effettivi. Senza istruzioni esplicite nel prompt, l'agente potrebbe ignorare questi requisiti funzionali, aumentando il rischio di perdita di contesto tra sessioni.
- **Bilanciamento Token:** La frequenza ridotta (solo conclusione Epic) bilancia la necessità di persistenza con l'efficienza del consumo token, evitando overhead eccessivo. Il tracciamento granulare dello stato operativo rimane delegato a Linear e Git.
- **Verifica Coerenza:** In fase di architettura, verificare che i prompt riflettano questo approccio senza aumentare eccessivamente la complessità del ciclo operativo. Il Memory Tool è utilizzato per persistenza di alto livello (decisioni architetturali, piano implementazione), mentre Linear e Git gestiscono lo stato operativo granulare.

**Mitigazione:**
- Definire prompt espliciti che includano istruzioni obbligatorie per FR28 e FR29 con frequenza ridotta (solo conclusione Epic)
- Monitorare consumo token durante implementazione per validare efficacia dell'approccio
- Garantire che Linear e Git siano utilizzati per tracciamento operativo dettagliato

#### 2. Dipendenze Infrastrutturali Pesanti (Puppeteer)

**Rischio Identificato:**
Il PRD richiede Chrome/Chromium per Puppeteer per supportare test end-to-end e validazione UI **solo per progetti che richiedono UI testing**. Per progetti CLI/backend, Puppeteer è opzionale e non bloccante.

**Decisione Strategica MVP:**
Per l'MVP, Puppeteer è utilizzato esclusivamente in **ambiente locale** dove Chrome/Chromium è facilmente installabile, e solo per progetti che richiedono UI testing. Il deployment containerizzato (Docker) è **differito alla Fase 2**.

**Mitigazione:**
- **MVP Focus Locale:** L'MVP è progettato per esecuzione locale, evitando complessità deployment containerizzato iniziale.
- **Preflight Checks Condizionali:** Puppeteer è verificato solo per progetti che richiedono UI testing, non bloccante per progetti CLI/backend.

#### 3. Prompt Caching con Sub-Agenti - Risolto

**Rischio Identificato (Risolto):**
Inizialmente si temeva che i sub-agenti con set di tool diversi avrebbero invalidato il Prompt Caching, aumentando il consumo token.

**Risoluzione Tecnica:**
Dopo ricerca tecnica approfondita sulla documentazione Anthropic Agent SDK, è stato verificato che i sub-agenti definiti tramite `AgentDefinition` possono condividere lo stesso toolset preservando il prompt caching quando utilizzano tool comuni. L'SDK supporta pattern di isolamento tramite definizione programmatica dei sub-agenti senza necessariamente rompere il caching del prompt.

**Decisione Architetturale Finale:**
L'implementazione utilizzerà sub-agenti distinti definiti tramite `AgentDefinition` con toolset condivisi quando appropriato (per preservare prompt caching) e toolset differenziati quando necessario per isolamento (es. QA Specialist senza tool Edit/Write). Gli Hook SDK (`PreToolUse` con decisione `deny`) saranno utilizzati come meccanismo di enforcement aggiuntivo per garantire sicurezza strutturale, ma non come sostituto dell'isolamento tramite `AgentDefinition`.

**Mitigazione:**
- **Validazione Sperimentale:** Durante la fase di design dettagliato, verificare sperimentalmente il comportamento del prompt caching con sub-agenti che condividono toolset vs toolset differenziati per ottimizzare l'approccio finale.
- **Monitoraggio:** Monitorare consumo token durante implementazione per quantificare l'efficacia del prompt caching con l'approccio scelto.
- **Validazione Requisito:** Il requisito Puppeteer è coerente con l'intento di avere test "reali" (User Journey 3 - Marco QA Engineer) e funziona senza problemi in ambiente locale per l'MVP.

**Resource Risks:**

**Rischio 1: Team Più Piccolo del Previsto**
- **Mitigazione:** MVP progettato per 1-2 sviluppatori. Focus su core features critiche, deferimento feature nice-to-have.
- **Contingency:** Ridurre scope MVP a solo Safety Net + Linear base, deferire sub-agenti a Fase 2

**Rischio 2: Budget Token Limitato**
- **Mitigazione:** Monitoring consumo token, ottimizzazione prompt, soglia conservativa context exhaustion
- **Contingency:** Limitare complessità task MVP, richiedere approvazione per task lunghi

**Rischio 3: Timeline Slip**
- **Mitigazione:** MVP con scope chiaro e delimitato. Priorità assoluta su Safety Net, deferimento tutto il resto.
- **Contingency:** Ridurre scope MVP a core minimo (Safety Net + Linear base), deferire Memory Tool avanzato

**Scoping Decision Summary:**

**MVP In Scope (3 mesi):**
- Safety Net completo (checkpointing + rollback)
- QA Specialist isolato
- Linear Read-Only + Status Update
- Memory Tool workflow standard
- Preflight checks rigorosi
- Git commit da Agente
- Basic monitoring

**MVP Out of Scope:**
- Tool Result Clearing / Context Compaction avanzata
- Docs Specialist autonomo
- Creazione ticket complessi Linear
- Interfaccia grafica
- Apprendimento a lungo termine avanzato
- Gestione epic complesse

**Post-MVP Prioritization:**
1. **Fase 2 (6-9 mesi):** Context Management avanzato (highest priority per sessioni multi-giorno)
2. **Fase 3 (9-12 mesi):** Docs Specialist e Memory Tool avanzato (autonomia completa)
3. **Fase 4 (12+ mesi):** Epic management e piattaforma (scalabilità enterprise)

## Analisi Miglioramenti Sub-Agenti Basata su DNA-Grafi

### Sintesi Analisi DNA-Grafi

Dall'analisi dei grafi DNA strutturati in `project_1/personas_per_agents/` emergono tre personalità professionali con caratteristiche distintive che possono migliorare significativamente l'harness e i sub-agenti:

1. **Scrum Master (Bob)** - Generazione Infrastruttura Linear Ottimale
2. **Developer (Amelia)** - Story-Driven Development con Test-First Rigoroso
3. **Test Architect (Murat)** - Risk-Based Testing e Quality Gates Guidati da Dati

### Miglioramenti Identificati per l'Harness

#### 1. Linear Infrastructure Specialist (Basato su Scrum Master Bob) - Initializer Agent Enhancement

**Contesto Operativo:**
Al trigger dell'harness, `app_spec.txt` è già presente (generato da PRD + Architecture + Epics + Stories). Il flusso è: **app_spec.txt (già presente) → Initializer Agent crea feature_list.json → Initializer Agent crea Linear project + Linear issues**. Lo Scrum Master genera l'infrastruttura Linear ottimamente per predisporre il miglior terreno per il Developer. `feature_list.json` è il progetto Linear in formato JSON che viene creato dall'Initializer da `app_spec.txt` e serve come file di sistema per creare le issue Linear copiando da un file già definito, evitando che l'LLM generi issue per issue rischiando confusione. `feature_list.json` e Linear definiscono le stesse cose, ma `feature_list.json` viene curato in parallelo (si aggiorna solo lo stato 'passes') e serve all'Initializer per creare le issue Linear in modo deterministico.

**Caratteristiche Chiave dal DNA-Grafo:**
- Preparazione infrastruttura ottimale con zero ambiguità
- Confini rigorosi tra setup infrastruttura e implementazione
- Allineamento perfetto tra app_spec.txt, feature_list.json e struttura Linear
- Handoff precisi con infrastruttura Linear pronta per sviluppatori

**Miglioramenti Proposti per l'Harness:**

**A. Initializer Agent Enhancement (Scrum Master come Linear Infrastructure Specialist)**
- **Ruolo:** Generare infrastruttura Linear ottimale durante la fase di bootstrap (Session 1)
- **Input:** `app_spec.txt` (già presente al trigger dell'harness, generato da PRD + Architecture + Epics + Stories)
- **Capacità:**
  - Leggere `app_spec.txt` per estrarre feature e requisiti
  - Creare `feature_list.json` (progetto Linear in formato JSON) da `app_spec.txt`
  - Creare progetto Linear con team ID ottimamente configurato
  - Generare Linear issues strutturate ottimamente copiando da `feature_list.json` (evita che l'LLM generi issue per issue rischiando confusione) con:
    - Titoli chiari e descrittivi
    - Descrizioni dettagliate estratte da app_spec.txt
    - Criteri di accettazione testabili e non ambigui
    - Priorità e labels appropriate
    - Collegamenti tra issue (dipendenze, relazioni)
    - Epiche e milestone strutturate
  - Allineare struttura Linear con architettura e standard di progetto
  - Salvare configurazione Linear (project ID, team ID) nel Memory Tool
- **Workflow Bootstrap:**
  1. Initializer Agent legge `app_spec.txt` (già presente)
  2. Analizza struttura feature e requisiti da `app_spec.txt`
  3. Crea `feature_list.json` (progetto Linear in formato JSON) da `app_spec.txt`
  4. Crea progetto Linear con team ID
  5. Genera Linear issues ottimamente strutturate copiando da `feature_list.json` (deterministico, evita confusione LLM)
  6. Organizza issue in epiche e milestone
  7. Salva configurazione Linear (project ID, team ID) nel Memory Tool
  8. Predispone "terreno" Linear ottimale per il Developer
- **Valore:** Infrastruttura Linear ottimale riduce ambiguità per Developer, garantisce allineamento con PRD/Architecture, migliora qualità implementazione. L'uso di `feature_list.json` come file di sistema intermedio garantisce creazione deterministica delle issue Linear, evitando errori di generazione LLM.

**B. Integrazione con Flusso app_spec.txt → feature_list.json → Linear**
- L'Initializer Agent viene attivato quando `app_spec.txt` è disponibile (già presente al trigger)
- Crea `feature_list.json` (progetto Linear in formato JSON) da `app_spec.txt`
- Genera infrastruttura Linear completa copiando da `feature_list.json` prima che il Developer inizi a lavorare
- `feature_list.json` viene curato in parallelo a Linear (si aggiorna solo lo stato 'passes') e serve come file di sistema per creazione deterministica delle issue
- Valida che le Linear issues siano strutturate ottimamente (titoli chiari, criteri accettazione testabili, priorità appropriate)
- Garantisce che la struttura Linear rifletta fedelmente il contenuto di `app_spec.txt` tramite `feature_list.json` come intermediario deterministico
- **Requisito Essenziale:** Il file `feature_list.json` è un **requisito essenziale** per l'MVP, funzionando come checkpoint fisico di recupero. Se l'Initializer Agent esaurisce la context window mentre crea le issue su Linear, il sistema deve poter terminare la sessione e avviarne una nuova che riprenda a generare le issue leggendo dal file `feature_list.json` già esistente, senza dover ri-ragionare sull'intera architettura.

#### 2. Coding Specialist Enhancement (Basato su Developer Amelia)

**Caratteristiche Chiave dal DNA-Grafo:**
- Story-driven development rigoroso (file story come singola fonte di verità)
- Red-green-refactor cycle obbligatorio
- Test-first approach (scrivi test fallente prima, poi implementazione)
- Esecuzione sequenziale rigorosa dei task (nessun salto, nessun riordino)
- Quality gates dopo ogni task (suite test completa, non procedere con test fallenti)
- Code review avversariale (trova 3-10 problemi specifici, non accetta mai "sembra buono")

**Miglioramenti Proposti per l'Harness:**

**A. Metodologia Story-Driven Development**
- **Requisito:** Il Coding Agent deve trattare il ticket Linear (con specifiche preparate dal Story Preparation Specialist) come "singola fonte di verità"
- **Sequenza Operativa:**
  1. Leggere completamente il ticket Linear e le specifiche preparate
  2. Estrarre task/subtask sequenziali dalle specifiche
  3. Eseguire task/subtask NELL'ORDINE definito, senza salti o riordini
  4. Per ogni task: seguire ciclo red-green-refactor (test fallente → implementazione → refactor)
  5. Eseguire suite test completa dopo ogni task prima di procedere
  6. Non procedere mai con test fallenti
- **Valore:** Riduce allucinazioni, garantisce copertura test completa, migliora qualità codice

**B. Quality Gates Rigorosi**
- **Dopo Ogni Task:**
  - Eseguire suite test completa
  - Verificare che tutti i test passino al 100%
  - Se test falliscono, eseguire rollback automatico (`rewind_files`)
  - Non procedere al task successivo finché test non passano
- **Valore:** Previene accumulo di debito tecnico, garantisce qualità incrementale

**C. Code Review Specialist (Sub-Agente Opzionale MVP)**
- **Ruolo:** Eseguire code review avversariale dopo che il Coding Agent completa l'implementazione
- **Capacità:**
  - Analizzare codice per qualità, copertura test, conformità architettura, sicurezza, performance
  - Trovare minimo 3-10 problemi specifici in ogni implementazione
  - Non accettare mai "sembra buono" - deve trovare problemi
  - Può auto-correggere con approvazione utente
- **Tool:** `read_file`, `bash` (per analisi statica), ma NON `Edit`/`Write` (solo reporta problemi)
- **Valore:** Migliora qualità codice prima del QA Specialist, riduce problemi in produzione

#### 3. QA Specialist Enhancement (Basato su Test Architect Murat)

**Caratteristiche Chiave dal DNA-Grafo:**
- Risk-based testing (profondità scala con impatto)
- Quality gates guidati da dati (non soggettivi)
- Network-first safeguards (intercept-before-navigate, attese deterministiche)
- Risk scoring (Probability × Impact, scala 1-9)
- Test quality definition of done (deterministici, isolati, espliciti, focalizzati, veloci)
- NFR assessment (sicurezza, performance, affidabilità, manutenibilità)

**Miglioramenti Proposti per l'Harness:**

**A. Risk-Based Testing**
- **Requisito:** Il QA Specialist deve calcolare rischio vs valore per ogni decisione di testing
- **Metodologia:**
  - Analizzare ogni criterio di accettazione per probabilità di fallimento e impatto
  - Score Rischio = Probabilità × Impatto (scala 1-9)
  - Score ≥6 richiedono test approfonditi
  - Score = 9 richiedono test obbligatori prima di approvare ticket
- **Valore:** Ottimizza tempo testing, focus su aree ad alto rischio

**B. Quality Gates Guidati da Dati**
- **Requisito:** Decisioni gate basate su metriche oggettive, non soggettive
- **Criteri Gate:**
  - **PASS:** Tutti test verdi, copertura ≥80%, nessun rischio score ≥6 non mitigato
  - **CONCERNS:** Rischi score 6-8 con piani mitigazione, copertura parziale
  - **FAIL:** Test falliti, rischi score=9, copertura <80%, violazioni critiche
- **Valore:** Decisioni oggettive e riproducibili, riduce falsi positivi/negativi

**C. Network-First Safeguards**
- **Requisito:** Registrare intercettazioni network PRIMA di qualsiasi navigazione o azione utente
- **Pattern:**
  - Intercept-before-navigate (evita race conditions)
  - Attese deterministiche basate su risposte network (non hard waits)
  - Cattura HAR per debugging
- **Valore:** Riduce flakiness test, migliora stabilità test end-to-end

**D. Test Quality Definition of Done**
- **Criteri Qualità Test:**
  - Esecuzione < 3 minuti
  - < 300 righe
  - Deterministici (nessuna attesa hard, nessun condizionale)
  - Isolati (self-cleaning, parallel-safe)
  - Espliciti (asserzioni visibili nei corpi test)
- **Valore:** Test affidabili, veloci, manutenibili

**E. NFR Assessment**
- **Requisito:** Validare requisiti non funzionali tramite test automatizzati
- **Categorie:**
  - **Sicurezza:** Test auth/authz, validazione OWASP Top 10
  - **Performance:** Load testing (k6), p95 < 500ms, error rate < 1%
  - **Affidabilità:** Gestione errori graceful, retry implementati
  - **Manutenibilità:** Copertura ≥80%, duplicazione <5%
- **Valore:** Validazione completa qualità oltre funzionalità base

### Architettura Sub-Agenti Migliorata

#### MVP Enhancement: Team Minimale ma Efficace

**Sub-Agenti MVP Minimi (Aggiornati):**

1. **Initializer Agent Enhancement - Linear Infrastructure Specialist (Basato su Scrum Master Bob)**
   - **Ruolo:** Generare infrastruttura Linear ottimale durante bootstrap (Session 1) a partire da `app_spec.txt` (già presente al trigger)
   - **Tool:** `read_file`, `write_file`, `mcp__linear__create_project`, `mcp__linear__create_issue`, `mcp__linear__create_team`, Memory Tool
   - **Permessi:** Bootstrap Profile (permessi completi Linear per creazione progetto/team/issue)
   - **Workflow Bootstrap:** Legge app_spec.txt → Crea feature_list.json (progetto Linear in formato JSON) → Crea progetto Linear + team ID → Genera Linear issues ottimamente strutturate copiando da feature_list.json (deterministico) → Organizza epiche/milestone → Salva configurazione Memory Tool
   - **Valore:** Predispone terreno Linear ottimale per Developer, riduce ambiguità, garantisce allineamento PRD/Architecture. L'uso di feature_list.json come intermediario garantisce creazione deterministica delle issue Linear, evitando errori di generazione LLM.

2. **Coding Specialist (Enhancement del Main Agent)**
   - **Ruolo:** Implementare codice seguendo metodologia story-driven development rigorosa
   - **Tool:** `Edit`, `Write`, `bash`, `read_file`, Memory Tool
   - **Metodologia:** Red-green-refactor cycle, test-first, esecuzione sequenziale rigorosa
   - **Quality Gates:** Suite test completa dopo ogni task, non procedere con test fallenti

3. **QA Specialist (Enhancement)**
   - **Ruolo:** Gatekeeper con risk-based testing e quality gates guidati da dati
   - **Tool:** `bash`, `mcp__puppeteer__...`, `read_file` (Edit/Write fisicamente rimossi)
   - **Capacità:** Risk scoring, network-first safeguards, NFR assessment, test quality validation
   - **Decisioni Gate:** PASS/CONCERNS/FAIL basati su metriche oggettive

4. **Code Review Specialist (Opzionale MVP)**
   - **Ruolo:** Code review avversariale dopo implementazione Coding Specialist
   - **Tool:** `read_file`, `bash` (analisi statica), Memory Tool (Edit/Write fisicamente rimossi)
   - **Capacità:** Trova 3-10 problemi specifici, analisi qualità/architettura/sicurezza/performance
   - **Valore:** Migliora qualità prima del QA Specialist

#### Workflow Operativo Migliorato

**Sequenza Operativa Aggiornata:**

**Fase Bootstrap (Session 1 - Initializer Agent):**
1. **Input:** `app_spec.txt` (già presente al trigger, generato da PRD + Architecture + Epics + Stories)
2. **feature_list.json Creation:** Initializer Agent (Scrum Master) crea `feature_list.json` (progetto Linear in formato JSON) da `app_spec.txt`
3. **Linear Infrastructure Generation:** Initializer Agent genera infrastruttura Linear copiando da `feature_list.json`:
   - Progetto Linear con team ID ottimamente configurato
   - Linear issues strutturate ottimamente per ogni feature (copiate da `feature_list.json` per evitare confusione LLM)
   - Epiche e milestone organizzate
   - Configurazione salvata nel Memory Tool (project ID, team ID)
4. **Parallel Maintenance:** `feature_list.json` viene curato in parallelo a Linear (si aggiorna solo lo stato 'passes') come file di sistema

**Fase Operativa (Day-to-Day - Coding Agent):**
1. **Ticket Selection:** Harness seleziona ticket Linear deterministicamente (In-Progress > Todo)
2. **Context Recovery:** Se ticket "In Progress", recupera contesto da commenti Linear e Memory Tool
3. **Checkpoint Pre-Modifica:** Harness crea checkpoint automatico (`checkpoint_uuid`)
4. **Coding Loop:** Coding Specialist implementa seguendo ticket Linear (già strutturato ottimamente):
   - Esegue task/subtask in ordine sequenziale rigoroso
   - Per ogni task: red-green-refactor cycle
   - Suite test completa dopo ogni task
   - Non procede con test fallenti
6. **Code Review (Opzionale):** Code Review Specialist analizza codice per problemi
7. **QA Check:** QA Specialist esegue risk-based testing con quality gates guidati da dati
8. **Decision Gate:** QA Specialist determina PASS/CONCERNS/FAIL basato su metriche oggettive
9. **Rollback o Commit:**
   - Se QA fallisce: Harness esegue `rewind_files(checkpoint_uuid)`
   - Se QA passa: Coding Agent esegue `git commit` con messaggio semantico
10. **Update Linear:** Agente aggiorna ticket Linear con stato e commenti dettagliati

### Benefici dei Miglioramenti

**1. Riduzione Allucinazioni:**
- Initializer Agent (Scrum Master) genera Linear issues ottimamente strutturate da `app_spec.txt`, garantendo allineamento con PRD/Architecture
- Coding Specialist segue ticket Linear già strutturati ottimamente invece di interpretare requisiti vaghi
- Quality gates rigorosi prevengono implementazioni fuori scope

**2. Miglioramento Qualità:**
- Test-first approach garantisce copertura test completa
- Code Review Specialist trova problemi prima del QA
- Risk-based testing focus su aree ad alto rischio
- Quality gates guidati da dati eliminano decisioni soggettive

**3. Riduzione Debito Tecnico:**
- Red-green-refactor cycle garantisce codice pulito
- Test quality definition of done previene test flaky
- Network-first safeguards riducono flakiness
- NFR assessment valida qualità completa

**4. Miglioramento Efficienza:**
- Initializer Agent genera infrastruttura Linear ottimale una volta durante bootstrap, Coding Specialist lavora su ticket già strutturati
- Risk-based testing ottimizza tempo testing
- Quality gates guidati da dati riducono iterazioni
- Esecuzione sequenziale rigorosa previene rework

### Raccomandazioni Implementazione

**Per MVP (3 mesi):**
- **Priorità Alta:** Implementare Story Preparation Specialist per migliorare qualità specifiche
- **Priorità Alta:** Enhancement QA Specialist con risk-based testing e quality gates guidati da dati
- **Priorità Media:** Enhancement Coding Specialist con metodologia story-driven development rigorosa
- **Priorità Bassa:** Code Review Specialist (può essere differito a Fase 2)

**Per Fase 2 (6-9 mesi):**
- Code Review Specialist completo
- NFR assessment completo (sicurezza, performance, affidabilità, manutenibilità)
- Network-first safeguards avanzati
- Test quality definition of done completo

**Considerazioni Architetturali:**
- I sub-agenti devono essere definiti programmaticamente tramite `AgentDefinition`
- Tool `Edit` e `Write` devono essere fisicamente rimossi da QA Specialist e Code Review Specialist
- Story Preparation Specialist può avere permessi di scrittura su Memory Tool e commenti Linear
- Coding Specialist ha permessi completi di scrittura codice
- Workflow deve garantire sequenza operativa rigorosa con quality gates tra ogni fase

## Non-Functional Requirements

### Performance

**Efficienza Token e Gestione Contesto:**
- **Consumo Token per Finestra Oraria:** L'harness deve monitorare e controllare il consumo di token per finestra oraria per prevenire l'esaurimento della finestra di contesto prima dell'aggiornamento dello stato di sviluppo. L'obiettivo è preservare la pulizia e l'ordine dell'ambiente, garantendo che il sistema possa completare la routine di salvataggio di emergenza (salvataggio Memory Tool, aggiornamento Linear, fermata con grazia) quando necessario. Per utenti con abbonamento Claude (limite token su finestra temporale), il sistema deve prevenire l'esaurimento rapido del limite orario dovuto alla saturazione del contesto, garantendo spazio sufficiente per operazioni critiche di salvataggio stato.
- **Efficienza del Contesto:** L'harness deve utilizzare strategie di gestione del contesto per ottimizzare l'utilizzo della finestra di contesto disponibile. Il funzionamento del clearing passivo nativo Anthropic (`clear_tool_uses_20250919` con header beta `context-management-2025-06-27`) sarà approfondito in fase di architettura per determinare l'implementazione ottimale. L'obiettivo è allungare la vita operativa dell'agente senza aggiungere complessità di codice lato harness, mantenendo il consumo token controllato anche in sessioni long-running ("Long-running agents"). **Nota Prompt Caching:** Se si prevede di utilizzare Prompt Caching per ridurre costi, considerare che il context switching tra sub-agenti con set di tool diversi invaliderà spesso la cache, riducendo l'efficacia del caching. Valutare questo trade-off in fase di architettura.
- **Tempo di Completamento Ticket:** L'harness deve completare ticket Linear di media complessità senza richiedere intervento umano continuo ("babysitting"). Il sistema deve operare autonomamente per task di media complessità, con Intervention Ratio < 1 per ticket (idealmente solo la definizione iniziale e la review finale).

**Performance Operativa:**
- **Preflight Checks:** I controlli preflight devono completarsi in tempi ragionevoli prima di spendere il primo token, validando ambiente, API keys, e dipendenze. I tempi specifici di esecuzione saranno determinati in fase di architettura basandosi su best practices e requisiti operativi.
- **Rollback Automatico:** Il rollback tramite `rewind_files()` deve completarsi in tempi accettabili per ripristinare lo stato dei file al checkpoint precedente quando i test falliscono. I tempi specifici di esecuzione saranno determinati in fase di architettura basandosi su performance dell'SDK e requisiti operativi.

### Security

**Gestione Credenziali e Isolamento:**
- **Gestione API Keys:** Tutte le API keys (Claude, Linear) devono essere gestite esclusivamente tramite variabili d'ambiente, mai hardcoded nel codice sorgente. Il sistema deve validare la presenza e la validità delle API keys durante i preflight checks prima di spendere il primo token.
- **Isolamento Sub-Agenti:** I sub-agenti devono avere permessi controllati e isolamento rigoroso. **L'isolamento rigoroso dei permessi (rimozione fisica dei tool `Edit` e `Write`) si applica esclusivamente al QA Specialist**. Il QA Specialist deve avere i tool `Edit` e `Write` fisicamente rimossi dalla configurazione `AgentDefinition`, non solo vietati nel prompt. Questo garantisce tecnicamente che il QA agisca come "Gatekeeper" imparziale senza possibilità di modificare codice di produzione. Gli altri sub-agenti possono avere permessi differenziati basati sul loro ruolo specifico, come definito nei DNA-grafi e nelle best practices Anthropic.
- **Profili di Sicurezza Differenziati:** Il sistema deve utilizzare profili di sicurezza differenziati:
  - **Bootstrap Profile (Initializer Agent):** Permessi elevati utilizzati esclusivamente durante la fase di bootstrap (Session 1) per creare infrastruttura Linear iniziale.
  - **Runtime Profile (Coding Agent):** Permessi ristretti (Read-Only + Status Update + Comment) utilizzati per operazioni day-to-day, prevenendo che un agente fuori controllo inondi il backlog di ticket spazzatura.

**Protezione Operazioni Distruttive:**
- **Hook SDK Pre-Tool Use:** Il sistema deve utilizzare Hook SDK (`PreToolUse` con decisione `deny`) per bloccare operazioni pericolose (es. modifiche a `.env`, `rm -rf`) prima che vengano eseguite, prevenendo modifiche distruttive al progetto.
- **File Checkpointing:** Il sistema deve creare checkpoint automatici dei file prima di ogni modifica del codice, salvando `message.uuid` e `checkpoint_uuid` per ogni iterazione, garantendo capacità di rollback immediato in caso di errori.

### Reliability

**Auto-Correzione e Resilienza:**
- **Tasso di Ripristino Automatico (Self-Correction Rate):** Il sistema deve gestire autonomamente > 90% degli errori tecnici durante il coding loop tramite `rewind_files()` senza richiedere intervento umano. Gli errori gestiti includono: test falliti, eccezioni durante l'esecuzione, operazioni bloccate.
- **Integrità della Build (Build Stability):** Il sistema deve garantire che il 100% dei ticket marcati come "Done" dall'agente passino effettivamente la CI/CD pipeline principale al primo colpo. Il QA Specialist deve agire da gatekeeper rigoroso; se la build è rossa, il ticket non deve mai passare a "Done".
- **Tasso di Regressione:** Il sistema deve garantire un tasso di regressione pari a 0. Nessuna funzionalità esistente deve essere rotta dall'implementazione di una nuova feature, grazie all'isolamento del contesto e ai test di regressione del QA Specialist.

**Gestione Errori e Recovery:**
- **Rollback Automatico:** Quando i test falliscono, il sistema deve eseguire automaticamente `rewind_files(checkpoint_uuid)` per ripristinare lo stato dei file al checkpoint precedente, lasciando il progetto in uno stato pulito e funzionante anche quando un task fallisce.
- **Context Exhaustion Recovery:** Quando il contesto si esaurisce (soglia conservativa 80-85% del limite fisico), il sistema deve eseguire una routine di salvataggio di emergenza: salvare lo stato attuale nel Memory Tool (piano di implementazione, progresso parziale, decisioni prese), aggiornare il ticket Linear con un commento dettagliato che spiega la situazione e il punto di ripartenza, e fermare l'esecuzione con grazia. Questo previene crash per limiti di contesto e permette il recupero dello stato in una sessione successiva.
- **Recupero Stato Interrotto:** Il sistema deve poter recuperare lo stato di un progetto interrotto combinando informazioni dal Memory Tool e dai commenti Linear, permettendo all'agente di riprendere il lavoro su un ticket "In Progress" recuperando il contesto dalla history dei commenti Linear.

**Gestione Errori SDK:**
- **Error Handling SDK:** Il sistema deve gestire tipi di errore specifici dell'SDK (`CLINotFoundError`, `CLIConnectionError`) e fallire gracefulmente con feedback azionabile invece di crash oscuri. I preflight checks devono trasformare errori oscuri di runtime in messaggi chiari all'avvio (es. "Manca Claude Code CLI").

### Scalability

**Gestione Sessioni Long-Running:**
- **Sessioni Multi-Giorno:** Il sistema deve supportare sessioni di lavoro continue su progetti complessi che richiedono giorni di sviluppo senza degradazione delle performance. Il Memory Tool deve garantire persistenza decisioni chiave tra sessioni, permettendo continuità operativa anche dopo riavvio sessione.
- **Context Management:** Il sistema deve monitorare l'utilizzo dei token tramite `usage` nei messaggi per determinare quando avvicinarsi al limite del contesto. Quando si raggiunge una soglia conservativa (80-85% del limite fisico), il sistema deve innescare una halt condition garantendo spazio sufficiente per: 1) generare il file di memoria completo nel Memory Tool, 2) creare un commento dettagliato su Linear, 3) eseguire la routine di salvataggio di emergenza senza errori.

**Scalabilità Operativa:**
- **Efficienza della Memoria (Context Efficiency):** L'efficacia del Memory Tool e del Context Clearing nel preservare informazioni critiche deve garantire che l'agente non chieda mai due volte la stessa specifica architetturale presente nel Memory Tool. Misurabile tramite il numero di volte che l'agente deve "ri-chiedere" informazioni già fornite o presenti nel progetto.
- **Gestione Multi-Ticket:** Il sistema deve gestire il ciclo di vita completo di ticket Linear dalla selezione al completamento, con capacità di gestire interruzioni di sessione e riprendere il lavoro su ticket "In Progress" recuperando il contesto dai commenti Linear e dal Memory Tool.

### Maintainability

**Architettura e Documentazione:**
- **Architettura Modulare:** Il sistema deve essere architettato seguendo le best practices Anthropic e delle dipendenze scelte per questo progetto. La documentazione ufficiale Anthropic SDK e delle dipendenze sarà consultata in fase di architettura per garantire conformità alle best practices e ottimizzazione dell'implementazione. La separazione netta tra infrastruttura (codice harness) e intelligenza (prompting agente) deve essere mantenuta, con moduli separati (Harness Core, Claude SDK Client, Sub-Agenti, Memory Tool, Linear MCP Integration) per facilitare manutenzione e estensibilità.
- **Documentazione Tecnica:** Il sistema deve fornire documentazione tecnica completa per sviluppatori, inclusa configurazione ambiente, API surface, code examples, e migration guide. La documentazione deve essere chiara e azionabile, permettendo a sviluppatori di configurare e utilizzare l'harness senza modificare codice sorgente.
- **Configurazione Tramite Environment Variables:** L'utente deve poter configurare l'harness tramite variabili d'ambiente senza modificare codice sorgente. Tutte le configurazioni critiche (API keys, checkpointing, Linear team ID) devono essere gestite tramite environment variables.

**Estensibilità:**
- **Sub-Agenti Estendibili:** Il sistema deve permettere facile aggiunta di nuovi sub-agenti tramite `AgentDefinition`, seguendo le best practices Anthropic per l'orchestrazione di agenti autonomi.
- **Preflight Checks Estendibili:** Il sistema deve permettere estensione con nuovi controlli di validazione ambiente senza modificare codice core.
- **Memory Tool Estendibile:** Il sistema deve permettere estensione con nuove categorie di decisioni da persistere nel Memory Tool.

**Monitoring e Observability:**
- **Metriche di Performance:** Il sistema deve tracciare metriche di performance (tempo completamento ticket, frequenza rollback) per monitoring operativo.
- **Alerting:** Il sistema deve generare alert quando si verificano situazioni critiche (context exhaustion, errori ripetuti, fallimenti preflight).
- **Audit Trail:** Il sistema deve fornire audit trail completo tramite git commits e commenti Linear per tracciabilità delle operazioni.

---

## Appendice: Post-MVP Deployment (Fase 2)

**Nota Importante:** Questa sezione descrive considerazioni per Fase 2, non per MVP. L'MVP è esplicitamente **Local-First** (PC/Mac/Linux) e non include deployment containerizzato o complessità infrastrutturali avanzate.

### Deployment Containerizzato

Il deployment Docker/containerizzato è differito alla Fase 2 per permettere:
- Focus MVP su funzionalità core e test locale
- Validazione completa del sistema prima di affrontare complessità infrastrutturali
- Ottimizzazione deployment basata su apprendimenti da test locale

**La Fase 2 includerà:**
- Immagini Docker ottimizzate con Puppeteer e dipendenze browser headless
- Guide deployment per ambienti containerizzati
- Considerazioni per ambienti serverless (se applicabili)

### Considerazioni Infrastrutturali per Fase 2

**Dipendenze Infrastrutturali Pesanti (Puppeteer):**

**Footprint Container:**
Immagini Docker devono includere Chrome/Chromium (~200-300MB aggiuntivi) e dipendenze di sistema per browser headless (libnss3, libatk-bridge2.0, libdrm2, etc.), aumentando significativamente la dimensione dell'immagine. Questo sarà affrontato in Fase 2 dopo validazione funzionale completa.

**Compatibilità Serverless:**
Ambienti serverless spesso non supportano esecuzione browser headless o hanno limitazioni significative. Questa considerazione sarà valutata in Fase 2 quando necessario.

**Ambiente Locale MVP:**
Puppeteer funziona senza problemi in ambiente locale (PC/Mac/Linux) dove Chrome/Chromium è facilmente installabile tramite package manager standard. Per l'MVP, questo è sufficiente.

**Fase 2 Deployment:**
Il deployment Docker/containerizzato sarà implementato in Fase 2 con immagini ottimizzate e guide specifiche basate su apprendimenti da test locale.


