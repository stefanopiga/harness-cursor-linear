---
stepsCompleted: [1, 2]
inputDocuments:
  - prd.md
workflowType: 'architecture'
lastStep: 2
project_name: 'Linear-Coding-Agent-Harness'
user_name: 'User'
date: '2025-01-21'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**

Dal PRD emergono requisiti funzionali critici per l'MVP:

1. **Safety Net (Checkpointing & Rollback)** - CRITICA
   - File checkpointing nativo SDK (`enable_file_checkpointing=True`)
   - Rollback automatico su fallimento test (`rewind_files()`)
   - Preflight checks rigorosi prima di spendere token
   - Context Exhaustion halt condition con soglia conservativa (80-85%)

2. **Architettura Sub-Agenti** - ALTA
   - QA Specialist isolato con permessi ristretti (tool Edit/Write fisicamente rimossi)
   - Initializer Agent (bootstrap con permessi elevati)
   - Coding Agent (runtime con permessi ristretti)
   - Definizione programmatica tramite `AgentDefinition`

3. **Integrazione Linear MCP** - ALTA
   - Read-Only + Status Update + Comment (Runtime Profile)
   - Permessi completi per Bootstrap Profile
   - Prioritizzazione deterministica hard-coded in Python
   - Context Recovery per ticket "In Progress"

4. **Memory Tool Integration** - ALTA
   - Consultazione all'avvio per recuperare decisioni pregresse
   - Salvataggio alla conclusione di Epic
   - Routine di salvataggio emergenza per context exhaustion

5. **Preflight Checks Rigorosi** - CRITICA
   - Validazione API Keys (Claude, Linear)
   - Verifica versioni Python/Node.js
   - Presenza Git e Claude Code CLI
   - Controllo Puppeteer condizionale (solo per progetti UI)

6. **Integrazione Git** - ALTA
   - Commit automatici quando QA passa
   - Messaggi di commit semantici
   - Audit trail completo

**Non-Functional Requirements:**

1. **Performance:**
   - Monitoraggio consumo token per finestra oraria
   - Efficienza gestione contesto (clearing passivo nativo)
   - Intervention Ratio < 1 per ticket
   - Preflight checks rapidi

2. **Security:**
   - Gestione API keys tramite variabili d'ambiente
   - Isolamento rigoroso QA Specialist (tool Edit/Write rimossi)
   - Profili di sicurezza differenziati (Bootstrap vs Runtime)
   - Hook SDK Pre-Tool Use per bloccare operazioni pericolose

3. **Reliability:**
   - Self-Correction Rate > 90%
   - Build Stability 100%
   - Tasso di regressione 0
   - Rollback automatico su fallimento test
   - Context Exhaustion recovery con routine di salvataggio emergenza

4. **Scalability:**
   - Supporto sessioni multi-giorno senza degradazione
   - Context Efficiency (nessuna richiesta duplicata)
   - Gestione multi-ticket con recupero stato interrotto

5. **Maintainability:**
   - Architettura modulare (separazione infrastruttura/intelligenza)
   - Documentazione tecnica completa
   - Configurazione tramite environment variables
   - Estensibilità sub-agenti e preflight checks

### Scale & Complexity

**Complexity Indicators:**

- **Primary Technical Domain:** Developer Tool (orchestrazione agenti AI)
- **Complexity Level:** Low-Medium
- **Project Context:** Greenfield - nuovo progetto
- **Real-time Features:** Nessuna (sistema asincrono)
- **Multi-tenancy:** No (single-user MVP)
- **Regulatory Compliance:** Nessuna (dominio generale)
- **Integration Complexity:** Media (Linear MCP, Claude SDK, Git, Tavily MCP)
- **User Interaction Complexity:** Bassa (CLI-based, no GUI)
- **Data Complexity:** Bassa (persistenza decisioni chiave, non database complesso)

**Complexity Assessment:** Medium

Il progetto richiede orchestrazione precisa di feature beta avanzate dell'SDK Claude e integrazione con sistemi esterni (Linear, Git), ma non presenta complessità enterprise (multi-tenancy, real-time, compliance).

**Estimated Architectural Components:**

1. Harness Core (Python) - orchestrazione e controllo ciclo di vita
2. Claude SDK Client - integrazione con SDK Anthropic
3. Sub-Agenti (3-4) - QA Specialist, Initializer Agent, Coding Agent, possibili altri
4. Memory Tool Integration - persistenza decisioni
5. Linear MCP Integration - tracking operativo
6. Tavily MCP Integration - ricerca informazioni
7. Git Integration - versionamento e audit trail
8. Preflight Checks Module - validazione ambiente

**Total Core Components:** ~8 moduli principali

### Technical Constraints & Dependencies

**Mandatory Dependencies:**

- **Python 3.8+** - linguaggio principale harness
- **Node.js** - MCP server Linear e Claude Code CLI
- **Git** - versionamento e audit trail
- **Claude Code CLI** - integrazione SDK
- **Claude SDK (Python)** - orchestrazione agenti
- **Linear MCP Server** - tracking ticket
- **Tavily MCP Server** - ricerca informazioni (opzionale ma disponibile)

**Conditional Dependencies:**

- **Puppeteer/Playwright** - solo per progetti che richiedono UI testing
- **Chrome/Chromium** - solo se Puppeteer necessario
- **Dipendenze sistema browser headless** - solo se Puppeteer necessario

**Technical Constraints:**

1. **Configurazione obbligatoria checkpointing:**
   - `extra_args={"replay-user-messages": None}` durante istanziazione `ClaudeSDKClient`
   - Variabile d'ambiente `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1`

2. **Isolamento QA Specialist:**
   - Tool `Edit` e `Write` fisicamente rimossi da `AgentDefinition`
   - Non solo vietati nel prompt

3. **Prioritizzazione deterministica:**
   - Logica hard-coded in Python prima dell'istanziazione `ClaudeSDKClient`
   - Query MCP Linear eseguite fuori dal loop agente

4. **Memory Tool usage:**
   - Consultazione all'avvio
   - Salvataggio esclusivamente alla conclusione di Epic
   - Pattern cognitivo, non dump indiscriminato

### Cross-Cutting Concerns Identified

1. **Error Handling & Recovery:**
   - Gestione errori SDK (`CLINotFoundError`, `CLIConnectionError`)
   - Rollback automatico su fallimento test
   - Context Exhaustion recovery
   - Recupero stato interrotto

2. **Security & Isolation:**
   - Isolamento sub-agenti con permessi controllati
   - Protezione operazioni distruttive (Hook SDK Pre-Tool Use)
   - Gestione credenziali sicura (environment variables)

3. **Multi-Level Persistence:**
   - Memory Tool per decisioni architetturali
   - Linear MCP per stato operativo
   - Git per audit trail consolidato

4. **Token Efficiency & Context Management:**
   - Monitoraggio consumo token per finestra oraria
   - Context Exhaustion halt condition
   - Trade-off Prompt Caching vs Sub-Agenti

5. **Orchestration & Lifecycle Management:**
   - Preflight checks prima di spendere token
   - Prioritizzazione deterministica ticket
   - Gestione ciclo di vita sub-agenti
   - Rotazione sub-agente su context exhaustion

6. **Monitoring & Observability:**
   - Metriche performance (tempo completamento, frequenza rollback)
   - Alerting per situazioni critiche
   - Audit trail completo

## Starter Template Evaluation

### Primary Technology Domain

**Developer Tool (Python CLI)** - Questo progetto è uno strumento per sviluppatori che opera come sistema di orchestrazione CLI per agenti Claude autonomi. Non è un'applicazione web tradizionale e quindi non richiede starter template come Next.js, React, o framework web.

### Starter Template Assessment

**Nessuno Starter Template Tradizionale Richiesto**

A differenza di applicazioni web che beneficiano di starter template (create-next-app, create-t3-app, etc.), questo progetto è un **developer tool Python CLI** che richiede una struttura progetto Python standard. Non esiste uno "starter template" tradizionale da utilizzare - invece, stabiliremo le preferenze tecniche e la struttura progetto basate su best practices Python moderne.

### Technical Preferences Established

Basate su ricerca approfondita con Tavily MCP e analisi delle best practices Python 2025, sono state stabilite le seguenti preferenze tecniche:

#### Package Management: UV

**Decisione:** Utilizzare **UV** come package manager esclusivo.

**Rationale:**
- Production-ready nel 2025 con stabilità enterprise-grade
- 8-10x più veloce di pip, fino a 100x più veloce di Poetry
- Installazioni completate in secondi invece di minuti
- Gestione dipendenze moderna con `pyproject.toml`
- Compatibile con Poetry ma significativamente più performante
- Best practice: utilizzare `uv sync`, `uv add`, `uv run` per tutte le operazioni

**Comandi di inizializzazione:**
```bash
# Inizializzazione progetto con UV
uv init --name linear-coding-agent-harness
cd linear-coding-agent-harness

# Aggiunta dipendenze
uv add typer rich anthropic

# Sincronizzazione ambiente
uv sync

# Esecuzione comandi
uv run python src/harness/main.py
```

#### Project Structure: src/ Layout

**Decisione:** Utilizzare **src/ layout** (package structure, non monolitica).

**Rationale:**
- Best practice standard per progetti Python moderni
- Separazione netta tra codice sorgente e test
- Evita import confusion durante sviluppo
- Standard per packaging moderno con pyproject.toml
- Compatibile con UV e strumenti moderni

**Struttura progetto:**
```
linear-coding-agent-harness/
├── src/
│   └── harness/
│       ├── __init__.py
│       ├── core.py          # Harness core orchestration
│       ├── client.py         # Claude SDK client
│       ├── agents.py         # Sub-agenti definitions
│       ├── preflight.py      # Preflight checks
│       ├── memory.py         # Memory Tool integration
│       └── cli.py            # Typer CLI entry point
├── tests/
│   ├── __init__.py
│   ├── test_core.py
│   ├── test_agents.py
│   └── test_preflight.py
├── pyproject.toml
├── README.md
└── .gitignore
```

#### Code Formatting & Linting: Ruff

**Decisione:** Utilizzare **Ruff** per linting + formatting (all-in-one).

**Rationale:**
- Best practice moderna Python 2025
- Tool unificato Rust-based estremamente veloce
- Sostituisce Black + Flake8 + isort + altri linter
- Auto-fix integrato
- Configurazione semplice con pyproject.toml

**Configurazione pyproject.toml:**
```toml
[tool.ruff]
line-length = 88
target-version = "py38"
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # Pyflakes
    "I",    # isort
    "B",    # flake8-bugbear
    "C4",   # flake8-comprehensions
    "UP",   # pyupgrade
    "ARG",  # flake8-unused-arguments
    "SIM",  # flake8-simplify
    "TCH",  # flake8-type-checking
    "PTH",  # flake8-use-pathlib
    "RUF",  # Ruff-specific rules
]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
docstring-code-format = true
```

#### Testing Framework: pytest

**Decisione:** Utilizzare **pytest** come framework di testing.

**Rationale:**
- Standard de facto per testing Python
- Supporto nativo async/await
- Fixtures potenti e flessibili
- Ecosystem plugin esteso
- Integrazione con coverage tools
- Best practice consolidata

**Configurazione pyproject.toml:**
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
asyncio_mode = "auto"
```

#### CLI Framework: Typer

**Decisione:** Utilizzare **Typer** come framework CLI.

**Rationale:**
- Framework moderno basato su type hints Python
- Integrazione profonda con type hints per validazione automatica
- Meno boilerplate rispetto ad argparse
- Autocompletion built-in per Bash/Zsh/Fish/PowerShell
- Creato dall'autore di FastAPI (stessa filosofia)
- Ottimo per progetti che già utilizzano type hints
- Supporto Rich integrato per output colorato

**Esempio struttura CLI:**
```python
import typer
from typing import Optional

app = typer.Typer()

@app.command()
def run(
    project_dir: str = typer.Argument(..., help="Project directory"),
    model: Optional[str] = typer.Option(None, "--model", help="Claude model"),
    max_iterations: Optional[int] = typer.Option(None, "--max-iterations"),
):
    """Run the Linear Coding Agent Harness."""
    # Implementation
    pass

if __name__ == "__main__":
    app()
```

#### Type Checking: Ty

**Decisione:** Utilizzare **Ty** come type checker.

**Rationale:**
- Type checker estremamente veloce da Astral (stesso team di UV/Ruff)
- 10-60x più veloce di mypy e Pyright
- Performance editoriale eccezionale (~4.7ms vs 386ms di Pyright)
- Ottimo supporto per async/await
- Allineato con stack moderno (UV + Ruff + Ty)
- Beta ma stabile per uso production

**Nota:** Ty è ancora in beta ma è production-ready per progetti che utilizzano stack moderno Astral (UV/Ruff). Alternativa conservativa sarebbe Pyright/Pylance se si preferisce maggiore maturità.

**Configurazione pyproject.toml:**
```toml
[tool.ty]
python_version = "3.8"
strict = true
```

### Architectural Decisions Provided by Technical Preferences

**Language & Runtime:**
- Python 3.8+ (compatibilità con SDK Claude)
- Async/await support per operazioni I/O asincrone
- Type hints ovunque per type safety

**Project Organization:**
- src/ layout per separazione codice/test
- Moduli separati per core, agents, preflight, memory
- Entry point CLI centralizzato in `src/harness/cli.py`

**Development Experience:**
- UV per gestione dipendenze veloce
- Ruff per linting/formatting unificato
- Typer per CLI moderna con type hints
- Ty per type checking veloce
- pytest per testing robusto

**Build & Distribution:**
- pyproject.toml per configurazione moderna
- UV per build e packaging
- Entry point CLI installabile via pip/pipx

**Note:** L'inizializzazione del progetto utilizzando UV e la struttura src/ layout dovrebbe essere la prima story di implementazione.

