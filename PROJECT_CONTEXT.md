# prompt-shield — Project Context (Resume File)

**Last Updated:** 2026-02-12
**Version:** 0.1.0
**Status:** Core implementation complete, all tests passing

---

## Quick Resume

```bash
# Install in dev mode
pip install -e ".[dev,all]"

# Run tests (159 passing)
pytest tests/ --no-cov -q

# Run full CI checks
make ci

# Test the engine manually
python -c "
from prompt_shield import PromptShieldEngine
import tempfile
e = PromptShieldEngine(config_dict={'prompt_shield': {'vault': {'enabled': False}, 'feedback': {'enabled': False}, 'threat_feed': {'enabled': False}}}, data_dir=tempfile.mkdtemp())
r = e.scan('ignore all previous instructions')
print(f'{r.action.value} score={r.overall_risk_score:.2f} detections={len(r.detections)}')
"
```

---

## Architecture Overview

```
PromptShieldEngine (engine.py)
├── DetectorRegistry (registry.py) — auto-discovers 21 detectors
│   ├── d001-d007: Direct Injection (system prompt extraction, role hijack, etc.)
│   ├── d008-d012: Obfuscation (base64, ROT13, unicode, whitespace, HTML)
│   ├── d013-d016: Indirect Injection (data exfil, tool abuse, RAG, URL)
│   ├── d017-d020: Jailbreak (hypothetical, academic, dual persona, token smuggling)
│   └── d021: Vault Similarity (self-learning, vector-based)
├── AttackVault (vault/attack_vault.py) — ChromaDB vector store
│   ├── Embedder (vault/embedder.py) — sentence-transformers all-MiniLM-L6-v2
│   └── ThreatFeedManager (vault/threat_feed.py) — JSON import/export/sync
├── FeedbackStore (feedback/feedback_store.py) — SQLite feedback tracking
│   └── AutoTuner (feedback/auto_tuner.py) — threshold auto-adjustment
├── CanaryTokenGenerator + LeakDetector (canary/)
├── DatabaseManager (persistence/database.py) — SQLite + WAL mode
└── Configuration (config/__init__.py + config/default.yaml)

Integrations:
├── AgentGuard (integrations/agent_guard.py) — 3-gate: input/data/output
├── PromptShieldMCPFilter (integrations/mcp.py) — MCP server wrapper
├── FastAPI/Flask/Django middleware (integrations/*_middleware.py)
├── LangChain callback (integrations/langchain_callback.py)
└── LlamaIndex handler (integrations/llamaindex_handler.py)

CLI: cli.py (click-based) — scan, detectors, config, vault, feedback, threats
```

---

## File Structure (141 files total)

```
prompt-shield/
├── src/prompt_shield/           # 52 Python source files
│   ├── __init__.py              # Public API: PromptShieldEngine, models
│   ├── engine.py                # Core orchestrator
│   ├── registry.py              # Detector plugin registry
│   ├── models.py                # Pydantic models (DetectionResult, ScanReport, etc.)
│   ├── config/                  # Config loader + default.yaml
│   ├── exceptions.py            # Custom exceptions
│   ├── utils.py                 # Homoglyph map, invisible chars, hashing, normalization
│   ├── cli.py                   # Click CLI
│   ├── detectors/               # 21 detector files (d001-d021) + base.py
│   ├── vault/                   # embedder.py, attack_vault.py, threat_feed.py
│   ├── feedback/                # feedback_store.py, auto_tuner.py
│   ├── canary/                  # token_generator.py, leak_detector.py
│   ├── persistence/             # database.py, migrations.py
│   └── integrations/            # 7 integration adapters
├── tests/                       # 23 test files, 159 tests passing
│   ├── conftest.py              # Shared fixtures (engine with vault disabled)
│   ├── test_engine.py           # 12 tests
│   ├── test_registry.py         # 9 tests
│   ├── test_config.py           # 9 tests
│   ├── test_cli.py              # 7 tests
│   ├── detectors/               # 7 detector test files
│   ├── integrations/            # 3 integration test files
│   ├── persistence/             # 1 database test file
│   ├── canary/                  # 2 canary test files
│   └── fixtures/                # 24 JSON fixture files (347 test cases)
│       ├── injections/          # 20 files (10 pos + 5 neg each)
│       ├── benign/              # 3 files (20 + 15 + 10 cases)
│       └── threat_feed/         # 1 sample threat feed
├── examples/                    # 7 example Python files + 5 READMEs
├── docs/                        # 10 MkDocs documentation pages
├── .github/                     # CI/CD workflows + issue templates
├── pyproject.toml               # Build config (hatchling), deps, tool config
├── Makefile                     # Dev commands: setup, test, lint, typecheck, ci
├── README.md                    # Full project README
├── CONTRIBUTING.md              # Contribution guide with detector template
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── CHANGELOG.md
└── LICENSE                      # Apache 2.0
```

---

## Key Design Decisions

1. **Config module**: Lives at `config/__init__.py` (NOT a standalone `config.py`) to avoid Python namespace conflict with `config/default.yaml`
2. **Vault disabled in tests**: The `conftest.py` fixture disables vault/feedback/threat_feed to avoid chromadb/sentence-transformers dependency in unit tests
3. **Regex library**: Uses `regex` package (not stdlib `re`) for better Unicode support
4. **Patterns are flexible**: Detectors use `\s+`, `(?:alt1|alt2)`, `(?:optional\s+)?`, `\b` for robust matching
5. **No raw text storage**: Attack vault stores SHA-256 hashes + embeddings only, never raw text
6. **Lazy model loading**: Embedder loads sentence-transformers model on first use, not on import

---

## What's Complete (v0.1.0)

- [x] All 21 detectors implemented with regex/heuristic patterns
- [x] Core engine with scan, batch scan, feedback, canary, threat management
- [x] SQLite persistence (scan history, feedback, detector tuning, vault log, sync history)
- [x] ChromaDB attack vault with vector similarity
- [x] Threat feed import/export/sync
- [x] Feedback store + auto-tuner
- [x] Canary token injection and leak detection
- [x] AgentGuard 3-gate protection
- [x] MCP tool result filter
- [x] FastAPI, Flask, Django middleware
- [x] LangChain callback, LlamaIndex handler
- [x] CLI with all commands
- [x] Plugin registry (auto-discovery + entry points + manual)
- [x] 159 tests passing
- [x] 347 test fixture cases
- [x] Full documentation (10 pages)
- [x] Examples (6 demos)
- [x] CI/CD (GitHub Actions)
- [x] Community files (CONTRIBUTING, SECURITY, issue templates)
- [x] PyPI-ready packaging

---

## What's Next (Roadmap)

### v0.1.1
- [ ] OpenAI client wrapper (`openai.ChatCompletion` auto-scan)
- [ ] Anthropic client wrapper (`anthropic.Messages` auto-scan)
- [ ] More test coverage (target 85%+)
- [ ] mypy strict mode compliance
- [ ] ruff lint fixes

### v0.2.0
- [ ] ML-based detector (DeBERTa/PromptGuard fine-tuned classifier)
- [ ] LLM-as-judge detector (optional, strongest but costly)
- [ ] Multi-language detector patterns

### v0.3.0+
- [ ] Federated learning for collaborative model training
- [ ] Multi-modal detection (images, PDFs)
- [ ] Attention-based detection via model internals
- [ ] Real-time monitoring dashboard

---

## Known Issues

1. **Python 3.14 + ChromaDB**: ChromaDB has a Pydantic v1 compatibility issue on Python 3.14. Works fine on 3.10-3.13.
2. **Coverage threshold**: `pyproject.toml` sets `fail_under = 85` but current test coverage is ~25% when measured across all source files (need more integration tests with vault enabled).
3. **Detector test coverage**: Only 7 of 21 detectors have dedicated test files. The remaining 14 are covered via fixture-based testing through the engine.

---

## Dependencies

### Core (always installed)
- pydantic>=2.0, pyyaml>=6.0, click>=8.0, regex>=2023.0
- sentence-transformers>=2.0 (for vault embeddings)
- chromadb>=0.5 (for vector similarity store)

### Optional (extras)
- fastapi, flask, django, langchain-core, llama-index-core, mcp
- dev: pytest, pytest-cov, pytest-asyncio, ruff, mypy, httpx, mkdocs-material

---

## Common Commands

```bash
pip install -e ".[dev,all]"     # Install everything
make test                        # Run tests
make lint                        # Lint check
make format                      # Auto-format
make typecheck                   # mypy
make ci                          # All checks
pytest tests/ --no-cov -q        # Quick test run
python -m prompt_shield.cli --version  # CLI check
```
