
### 1. La Filosofia dell'MVP: "Safety First, Autonomy Second"

Per risolvere il problema critico (errori distruttivi e perdita di contesto), l'MVP non deve mirare a sessioni "infinite" o intelligenza suprema, ma alla **robustezza**.
Il criterio decisionale è: *Se l'agente sbaglia, il progetto sopravvive?*

### 2. Selezione dei Pilastri per l'MVP

Ecco come filtrare i quattro pilastri del design per i primi 3 mesi:

| Pilastro | Priorità MVP | Decisione |
| :--- | :--- | :--- |
| **Sicurezza** (Checkpointing) | 🔴 **CRITICA** | **Implementare subito.** È il core differentiator. Senza `rewind_files`, l'harness è pericoloso,. |
| **Isolamento** (Sub-agenti) | 🟡 **ALTA** | **Implementare parzialmente.** Servono solo 2 ruoli: *Coding* (scrive) e *QA* (testa). Il *Docs Specialist* è differito,. |
| **Persistenza** (Linear MCP) | 🟡 **ALTA** | **Essenziale per l'autonomia.** Senza Linear, l'umano deve fare da "balia". Serve un'integrazione base (Read/Update status),. |
| **Gestione Contesto** | 🟢 **BASSA** | **Differire.** Tool Result Clearing e Compaction complessa non sono essenziali per task atomici iniziali. Basta riavviare la sessione tra un ticket e l'altro,. |

---

### 3. Funzionalità Core dell'MVP (Scope)

Per l'obiettivo "Safe Harness", l'MVP deve includere esclusivamente queste 4 componenti:

#### A. Il "Safety Net" (Checkpointing & Rollback)
Questa è la funzionalità "Aha!". L'harness deve poter annullare le modifiche se i test falliscono.
*   **Requisito Tecnico:** Abilitare `enable_file_checkpointing=True` e `permission_mode="acceptEdits"` (per evitare blocchi su ogni file).
*   **Logica:** Prima di ogni iterazione del *Coding Agent*, salvare il `message.uuid`. Se il *QA Agent* riporta fallimento, eseguire `client.rewind_files(uuid)`.
*   **Valore Utente:** L'utente vede i file cambiare e poi tornare indietro magicamente se il codice è rotto.

#### B. Il "Gatekeeper" (QA Sub-agent)
Non basta scrivere codice; serve qualcuno che dica "NO".
*   **Requisito Tecnico:** Definire programmaticamente un sub-agente `QA Specialist` tramite `AgentDefinition`,.
*   **Configurazione:** Questo agente deve avere accesso ai tool di test (`bash` per pytest/npm test) ma, crucialmente, istruzioni rigide di **non modificare** il codice di produzione, solo i file di test se necessario.

#### C. Integrazione Linear "Read-Only + Status Update"
Per evitare il "babysitting", l'agente deve sapere cosa fare da solo.
*   **Requisito Tecnico:** Configurare `mcp_servers` per connettersi a Linear,.
*   **Logica Semplificata:**
    1. L'Harness (Python) interroga Linear per ticket con tag "Ready".
    2. Passa il contesto del ticket al *Main Agent*.
    3. A fine lavoro, l'agente usa il tool per marcare il ticket come "In Review" o commentare.
*   **Esclusione MVP:** Non permettere all'agente di *creare* nuovi ticket o modificare epiche complesse.

#### D. Preflight Checks Rigorosi
Per evitare crash a metà lavoro.
*   **Requisito Tecnico:** Prima di lanciare `client.query()`, eseguire controlli su API Key, presenza di `git`, e connessione a Linear,.
*   **Valore:** Trasformare errori oscuri di runtime in messaggi chiari all'avvio (es. "Manca Claude Code CLI").

---

### 4. Cosa NON costruire (Ambizione vs Velocità)

Per consegnare in 3 mesi, dobbiamo tagliare spietatamente:
*   ❌ **Gestione "Infinita" del Contesto:** Non implementare `Tool Result Clearing` o `Compaction` complessa. Se il task è troppo lungo e il contesto finisce, l'agente fallisce "gracefully" e chiede aiuto, piuttosto che cercare di gestire una memoria infinita.
*   ❌ **Docs Specialist:** La documentazione può essere aggiornata dall'umano o dal Coding Agent per ora.
*   ❌ **Apprendimento a Lungo Termine:** Il *Memory Tool* verrà usato solo per salvare `project_id` e decisioni vitali, non come knowledge base complessa.
*   ❌ **Interfaccia Grafica:** L'output CLI è sufficiente.

### 5. Criteri di Successo dell'MVP

Come sapremo di aver finito?
1.  **Test del Disastro:** Iniettiamo volontariamente un bug nel codice generato. L'harness deve:
    *   Rilevarlo tramite QA.
    *   Eseguire il rollback automatico.
    *   Lasciare il repo pulito come prima.
2.  **Test dell'Autonomia:** Assegniamo un ticket Linear "Creare un file hello.py". L'utente lancia lo script e va a prendere il caffè. Al ritorno, il ticket su Linear deve essere "Done" e il file deve esistere.

### Sintesi per gli Stakeholder
L'MVP sarà un **"Developer Junior Cauto"**. Non sarà velocissimo e non gestirà progetti enormi, ma avrà una "rete di sicurezza" infallibile: o consegna codice funzionante e testato, o non tocca nulla. Questo risolve direttamente la paura principale dell'utente ("l'agente mi rompe il progetto").



Ecco le fonti utilizzate per la definizione dell'MVP, basate sul documento di design fornito e sulla documentazione tecnica dell'SDK:

*   **Design Document (Fonte Primaria):**
    *   `design-harness.md` (File locale del progetto)

*   **Documentazione Claude Agent SDK & Tools:**
    *   https://platform.claude.com/docs/en/agent-sdk/file-checkpointing.md (Implementazione tecnica di *Rewind* e *Checkpointing* per il "Safety Net"),
    *   https://platform.claude.com/docs/en/agent-sdk/subagents.md (Definizione programmatica dei *Sub-agenti* per il ruolo QA),
    *   https://platform.claude.com/docs/en/agent-sdk/mcp.md (Integrazione *Linear MCP* nell'SDK),
    *   https://platform.claude.com/docs/en/agent-sdk/python.md (Riferimento API Python per `ClaudeSDKClient`, gestione errori `CLINotFoundError` e opzioni preflight),,
    *   https://platform.claude.com/docs/en/agent-sdk/permissions.md (Configurazione dei permessi come `acceptEdits` per l'automazione MVP),