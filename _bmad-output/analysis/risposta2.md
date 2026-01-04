
### 1. Persona Primaria: "L'Architect/Lead Developer"

**Nome e Contesto:**
Chiamiamolo **Alex**. È un Senior Software Engineer o Tech Lead in un team agile. Ha un background tecnico solido (Python/Node.js) e lavora su progetti complessi che richiedono refactoring, nuove feature e manutenzione continua. Usa già strumenti come Linear per il project management e git per il versionamento.

**Esperienza Attuale del Problema (Il "Dolore"):**
Oggi Alex usa strumenti AI, ma è frustrato dalla loro fragilità. Vive il problema della "memoria volatile": se assegna all'agente un compito lungo (es. "rifai l'autenticazione"), l'agente spesso esaurisce il contesto a metà, lasciando il codice in uno stato rotto ("half-implemented") o allucinando di aver finito,.
I suoi workaround attuali sono costosi: deve fare "babysitting" all'agente, spezzettare i task in micro-prompt manuali e intervenire fisicamente per revertire i file quando l'agente commette errori distruttivi,.

**Visione del Successo:**
Per Alex, il successo non è un agente che scrive codice più velocemente, ma un agente che **non rompe la build**.
*   **Il momento "Aha!":** Avviene quando Alex vede che un test è fallito durante la notte, ma invece di trovare il progetto rotto al mattino, scopre che l'harness ha eseguito automaticamente un `rewind_files()`, ha ripristinato lo stato sicuro e ha provato una strategia alternativa o ha chiesto aiuto senza distruggere il lavoro fatto,.
*   **Valore:** La capacità di delegare un ticket Linear complesso sapendo che il sistema ha una "Safety Net" (rete di sicurezza) attiva.

### 2. Utenti Secondari

**Product Owner / Engineering Manager:**
*   **Beneficio:** Visibilità trasparente. Non devono chiedere "a che punto è l'agente?"; vedono lo stato dei ticket aggiornarsi in tempo reale su Linear (In Progress -> Done) con commenti e link ai commit,.
*   **Ruolo:** Oversight. Possono bloccare un task o cambiare priorità semplicemente spostando un ticket in Linear, senza dover interagire via terminale con l'agente.

**QA Engineer / Reviewer:**
*   **Beneficio:** Ricevono Pull Request più pulite. Grazie al "QA Specialist" (sub-agente isolato) che esegue i test prima di dichiarare il lavoro finito, il codice che arriva alla revisione umana ha già passato una barriera di qualità automatizzata,.

### 3. User Journey: Dalla Scoperta alla Routine

Ecco come Alex interagisce con il *Linear Coding Agent Harness*:

#### Fase 1: Onboarding e Configurazione (Setup)
Alex clona la repo dell'harness.
1.  **Configurazione Ambientale:** Imposta `LINEAR_API_KEY` e le variabili per `file_checkpointing`.
2.  **Preflight Check:** Lancia l'harness. Il sistema esegue immediatamente una sequenza di controlli ("Preflight Checklist"): verifica la presenza di Python, Node.js, git e la connessione a Linear.
    *   *Valore immediato:* Se manca qualcosa, riceve un errore deterministico e azionabile (es. "Manca Claude Code CLI"), invece di un crash oscuro a metà esecuzione.

#### Fase 2: Attivazione del Workflow (Day-to-Day)
Invece di scrivere un lungo prompt in chat, Alex va su Linear.
1.  **Input:** Crea un ticket Linear: "Implementare endpoint API per login utenti" con i Criteri di Accettazione. Lo lascia in stato "Todo".
2.  **Esecuzione:** Lancia l'harness (o questo gira su un server/container persistente).
3.  **Prioritizzazione Automatica:** L'harness interroga Linear. Nota che non ci sono issue "In Progress" interrotte (che avrebbero precedenza assoluta) e seleziona il ticket "Todo" appena creato.

#### Fase 3: Monitoraggio e Intervento (Il Loop Autonomo)
Mentre Alex fa altro, l'harness lavora ciclicamente:
1.  **Inizializzazione:** L'Agente Principale legge il ticket e pianifica il lavoro, salvando il piano nel Memory Tool per persistenza.
2.  **Coding & Safety Net:** Il "Coding Specialist" scrive il codice. L'SDK crea checkpoint automatici dei file prima delle modifiche.
3.  **Gatekeeping:** Il "QA Specialist" esegue i test.
    *   *Scenario Errore:* I test falliscono. L'harness intercetta l'errore, esegue il rollback dei file al checkpoint precedente e l'Agente Principale riprova con un approccio diverso,.
4.  **Chiusura:** Se i test passano, il "Docs Specialist" aggiorna la documentazione e l'harness marca il ticket Linear come "Done",.

#### Fase 4: Percezione del Valore
Alex riceve una notifica da Linear: "Ticket completato". Controlla il codice:
*   I file sono puliti (grazie al rollback degli errori intermedi).
*   C'è un file `feature_list.json` o una memoria persistente aggiornata che traccia il progresso.
*   Non deve passare ore a "riparare" ciò che l'agente ha rotto.

### Sintesi del Cambiamento
Il passaggio dall'harness attuale a quello proposto trasforma il lavoro di Alex da **"Operatore di Chatbot"** (scrivere prompt, correggere output, incollare file) a **"Gestore di Flusso"** (definire requisiti in Linear, revisionare PR, gestire eccezioni di alto livello). L'harness diventa un "impiegato virtuale" che lavora su turni, dotato di memoria a lungo termine e capacità di auto-correzione,.