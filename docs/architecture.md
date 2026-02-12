# Architecture

This document describes the internal architecture of prompt-shield.

## System Overview

```
                    +-------------------+
                    |  PromptShieldEngine |
                    +-------------------+
                    |  scan()           |
                    |  feedback()       |
                    |  export_threats() |
                    +--------+----------+
                             |
          +------------------+------------------+
          |                  |                  |
    +-----v-----+    +------v------+    +------v------+
    |  Registry  |    |   Vault     |    |  Feedback   |
    +-----+-----+    +------+------+    +------+------+
          |                  |                  |
    +-----v-----+    +------v------+    +------v------+
    | Detectors  |    |  ChromaDB   |    | Auto-Tuner  |
    | (d001-d021)|    |  Embedder   |    | FeedbackStore|
    +------------+    +------+------+    +-------------+
                             |
                      +------v------+
                      | Threat Feed |
                      | (import/    |
                      |  export/    |
                      |  sync)      |
                      +-------------+
```

## Core Components

### PromptShieldEngine (`engine.py`)

The central orchestrator. On construction it:

1. Loads configuration (YAML + env vars + dict overrides)
2. Initializes the data directory and SQLite database
3. Creates the attack vault (ChromaDB) if enabled
4. Creates the feedback store and auto-tuner if enabled
5. Creates the canary token generator and leak detector if enabled
6. Creates the threat feed manager if the vault is available
7. Auto-discovers and registers all detectors
8. Compiles allowlist/blocklist regex patterns

On `scan()`:

1. Checks the allowlist (immediate pass) and blocklist (immediate block)
2. Runs all enabled detectors, applying per-detector thresholds (including auto-tuned values)
3. Aggregates results: risk score = max confidence across detections
4. Determines action based on severity-to-action mapping
5. Logs the scan to SQLite history
6. Auto-stores detected attacks in the vault
7. Periodically triggers the auto-tuner

### DetectorRegistry (`registry.py`)

Manages detector lifecycle with three registration methods:

- **Auto-discovery**: Scans the `prompt_shield.detectors` package for `BaseDetector` subclasses
- **Entry points**: Discovers third-party detectors via the `prompt_shield.detectors` entry point group
- **Manual**: `engine.register_detector()` at runtime

### Detectors (`detectors/`)

Each detector is a Python class extending `BaseDetector`. All 21 built-in detectors follow a consistent pattern:

1. Define class attributes (`detector_id`, `name`, `severity`, etc.)
2. Define detection patterns
3. Implement `detect()` returning a `DetectionResult`

Detectors are stateless and independent. They receive the input text and optional context dict, and return a structured result.

### Attack Vault (`vault/attack_vault.py`)

ChromaDB-backed vector store with these responsibilities:

- **Store**: Embeds detected attack text using sentence-transformers and stores the embedding + metadata. Raw text is never stored; only the SHA-256 hash.
- **Query**: Finds the N most similar stored embeddings to an input using cosine distance
- **Import/Export**: Bulk operations for threat feed integration
- **Deduplication**: Skips entries with duplicate `pattern_hash` values

The embedder (`vault/embedder.py`) wraps sentence-transformers to produce 384-dimensional embeddings.

### Feedback System (`feedback/`)

Two components:

**FeedbackStore** (`feedback_store.py`): SQLite-based storage for user feedback. Records `(scan_id, detector_id, is_correct, timestamp, notes)` tuples. Provides per-detector accuracy statistics.

**AutoTuner** (`auto_tuner.py`): Reads accumulated feedback stats and adjusts per-detector confidence thresholds. Stores tuned values in the `detector_tuning` SQLite table. The engine queries this table on every scan to get the effective threshold.

### Canary System (`canary/`)

**CanaryTokenGenerator** (`token_generator.py`): Generates unique tokens and injects them into system prompts using a configurable format.

**LeakDetector** (`leak_detector.py`): Checks LLM responses for the presence of canary tokens. A leaked token indicates the model was tricked into revealing its system prompt.

### Threat Feed (`vault/threat_feed.py`)

Manages import, export, and remote sync of threat intelligence:

- **Export**: Serializes locally-detected vault entries as a `ThreatFeed` JSON file
- **Import**: Validates a feed file and bulk-imports entries into the vault with deduplication
- **Sync**: Downloads a remote feed URL, saves locally, and imports

The feed format includes embedding vectors, pattern hashes, severity, and metadata but never raw attack text.

### Persistence (`persistence/`)

**DatabaseManager** (`database.py`): Manages the SQLite database with WAL mode, migrations, and connection pooling.

**Migrations** (`migrations.py`): Schema creation and migration scripts for `scan_history`, `feedback`, `detector_tuning`, and `sync_history` tables.

### Configuration (`config.py`, `config/default.yaml`)

Layered configuration with four sources merged in priority order:

1. Environment variables (`PROMPT_SHIELD_*`)
2. `config_dict` parameter
3. YAML file
4. Built-in defaults (`config/default.yaml`)

### Integrations (`integrations/`)

Framework adapters that wrap the engine:

| Module | Class | Purpose |
|---|---|---|
| `fastapi_middleware.py` | `PromptShieldMiddleware` | Starlette middleware for HTTP body scanning |
| `flask_middleware.py` | `PromptShieldMiddleware` | WSGI middleware for Flask |
| `django_middleware.py` | `PromptShieldMiddleware` | Django middleware |
| `langchain_callback.py` | `PromptShieldCallback` | LangChain lifecycle callback handler |
| `llamaindex_handler.py` | `PromptShieldHandler` | LlamaIndex query/retrieval scanner |
| `agent_guard.py` | `AgentGuard` | 3-gate protection for agent loops |
| `mcp.py` | `PromptShieldMCPFilter` | Transparent MCP server proxy |

### CLI (`cli.py`)

Click-based CLI with subcommands: `scan`, `detectors list/info`, `config init/validate`, `vault stats/search/clear`, `feedback`, `threats export/import/sync/stats`, `test`, `benchmark`.

## Data Flow

```
User Input
    |
    v
[Allowlist check] --> Pass (if matched)
    |
[Blocklist check] --> Block (if matched)
    |
    v
[Run detectors d001-d021]
    |
    v
[Aggregate: max confidence, highest severity]
    |
    v
[Determine action: block/flag/log/pass]
    |
    +--> [Log to scan_history]
    |
    +--> [Store in vault if confidence >= threshold]
    |
    +--> [Auto-tune if scan_count % interval == 0]
    |
    v
ScanReport
```

## Models (`models.py`)

All data structures are Pydantic v2 models:

- `Severity` -- Enum: LOW, MEDIUM, HIGH, CRITICAL
- `Action` -- Enum: BLOCK, FLAG, LOG, PASS
- `MatchDetail` -- Pattern, matched text, position, description
- `DetectionResult` -- Single detector result
- `ScanReport` -- Aggregated scan result
- `GateResult` -- AgentGuard gate result
- `ThreatEntry` -- Single threat feed entry
- `ThreatFeed` -- Complete threat feed document
