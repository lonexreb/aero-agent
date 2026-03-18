<p align="center">
  <img src="assets/banner.svg" alt="AeroAgent Banner" width="100%"/>
</p>

<p align="center">
  <strong>CFD-Aware Research Assistant for Formula 1 Aerodynamicists</strong><br/>
  <em>Synthesize literature. Correlate CFD. Optimize every simulation hour.</em>
</p>

<p align="center">
  <a href="#quickstart"><img src="https://img.shields.io/badge/python-3.11+-3776AB?style=flat-square&logo=python&logoColor=white" alt="Python 3.11+"/></a>
  <a href="#tech-stack"><img src="https://img.shields.io/badge/LLM-GPT--4o_%7C_Claude-8A2BE2?style=flat-square" alt="LLM"/></a>
  <a href="#tech-stack"><img src="https://img.shields.io/badge/RAG-LlamaIndex_%2B_Qdrant-00d4ff?style=flat-square" alt="RAG"/></a>
  <a href="#tech-stack"><img src="https://img.shields.io/badge/CFD-OpenFOAM_%7C_SU2-FF6B35?style=flat-square" alt="CFD"/></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-green?style=flat-square" alt="License"/></a>
  <a href="#contributing"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square" alt="PRs Welcome"/></a>
</p>

---

## The Problem

F1 teams operate under **FIA Aerodynamic Testing Restrictions (ATR)** — every CFD simulation hour is capped based on constructor standings. Teams can't brute-force their way to aerodynamic solutions. A wrong simulation costs real competitive advantage.

**AeroAgent makes every CFD run count.**

It ingests aerodynamics literature, correlates findings with your CFD results and on-track telemetry, then produces ranked design modification proposals — each with literature backing, expected aero delta, CFD budget cost, and confidence level.

```
┌─────────────────────────────────────────────────────────────────┐
│  "The literature suggests vortex generators at 60% chord        │
│   improve diffuser performance by 3-5%. Your CFD shows          │
│   separation at 55% chord. Telemetry confirms rear instability  │
│   at high speed. Estimated gain: +8 counts Cl_rear.             │
│   CFD cost: 2 ATR-hours. ROI score: 4.0"                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architecture

```
                    ┌──────────────────────────────────┐
                    │        KNOWLEDGE LAYER           │
                    │                                  │
                    │  arXiv ─┐                        │
                    │  SAE   ─┼─▶ Docling ─▶ Chunks   │
                    │  AIAA  ─┘     │          │       │
                    │  OpenAlex     ▼          ▼       │
                    │  Semantic  Paper      Qdrant     │
                    │  Scholar   Analyzer   (pgvector) │
                    │               │          │       │
                    │  FIA Regs ────┘          │       │
                    └──────────────┬───────────┘
                                  │
                    ┌─────────────▼────────────────────┐
                    │        ANALYSIS LAYER            │
                    │                                  │
                    │  CFD Results ──┐                 │
                    │  (OpenFOAM,    ├─▶ Correlation   │
                    │   SU2,         │    Engine        │
                    │   Star-CCM+)   │      │          │
                    │                │      ▼          │
                    │  FastF1 ───────┘  Insight        │
                    │  Telemetry        Ranker         │
                    │                      │           │
                    │  ATR Budget ─────────┘           │
                    │  Tracker                         │
                    └──────────────┬───────────────────┘
                                  │
                    ┌─────────────▼────────────────────┐
                    │       SUGGESTION LAYER           │
                    │                                  │
                    │  1. VG on diffuser    (ROI: 4.0) │
                    │  2. Floor edge fence  (ROI: 3.2) │
                    │  3. Endplate cutout   (ROI: 2.8) │
                    │                                  │
                    │  Each with: literature citations, │
                    │  expected ΔCl/ΔCd, CFD cost,     │
                    │  confidence, risk assessment      │
                    └──────────────────────────────────┘
```

---

## Tech Stack

AeroAgent uses a modern, production-grade Python stack. Every choice is deliberate.

### Core Infrastructure

| Component | Tool | Why |
|:--|:--|:--|
| **Package Management** | [uv](https://github.com/astral-sh/uv) `>= 0.5` | 10x faster than Poetry. Industry standard for Python in 2025+. |
| **Linting & Formatting** | [Ruff](https://github.com/astral-sh/ruff) `>= 0.9` | Replaces Black, isort, Flake8, and 12 other tools. Millisecond speed. |
| **Type Checking** | [basedpyright](https://github.com/DetachHead/basedpyright) | Stricter, faster alternative to mypy. |
| **Data Validation** | [Pydantic v2](https://docs.pydantic.dev/) `>= 2.10` | Rust core. 5-50x faster than v1. |
| **Testing** | [pytest](https://docs.pytest.org/) `>= 8.0` | With `pytest-asyncio`, `pytest-cov`, `pytest-mock`, `pytest-xdist`. |
| **API** | [FastAPI](https://fastapi.tiangolo.com/) `>= 0.115` | Async, Pydantic-native, OpenAPI docs. |

### AI / RAG Pipeline

| Component | Tool | Why |
|:--|:--|:--|
| **RAG Framework** | [LlamaIndex](https://docs.llamaindex.ai/) `>= 0.11` | Best ingestion/retrieval for scientific docs. 150+ connectors. |
| **Orchestration** | [LangGraph](https://langchain-ai.github.io/langgraph/) `>= 0.2` | Stateful graph workflows for multi-step correlation analysis. |
| **LLM Gateway** | [LiteLLM](https://github.com/BerriAI/litellm) `>= 1.55` | Unified API across GPT-4o + Claude. Cost tracking, fallback routing. |
| **Vector Database** | [Qdrant](https://qdrant.tech/) `>= 1.12` | Rich payload filtering for aero metadata (Cd, Cl, flow regime, Re). |
| **Relational DB** | [PostgreSQL](https://www.postgresql.org/) `>= 16` | ATR budget tracking, CFD run history, telemetry sessions. |
| **Embeddings** | Cohere embed-v4 / [Jina v3](https://jina.ai/embeddings/) | Top MTEB scores. Jina v3's late chunking preserves cross-section context in papers. |
| **PDF Parsing** | [Docling](https://github.com/docling-project/docling) `>= 2.0` | IBM Granite VLM. Best structural extraction for scientific PDFs (tables, equations, figures). |
| **PDF Fallback** | [Marker](https://github.com/VikParuchuri/marker) `>= 1.0` | Fast GPU/MPS batch processing for bulk paper ingestion. |

### Domain-Specific

| Component | Tool | Why |
|:--|:--|:--|
| **F1 Telemetry** | [FastF1](https://docs.fastf1.dev/) `>= 3.8` | De facto standard for F1 timing/telemetry data. |
| **OpenFOAM Parsing** | [foamlib](https://github.com/gerlero/foamlib) `>= 1.2` | Modern, typed, async. Published in JOSS 2025. |
| **SU2 Parsing** | pysu2 (native) + pandas | SU2's native Python wrapper + CSV force history parsing. |
| **CFD Visualization** | [PyVista](https://docs.pyvista.org/) `>= 0.47` | Pythonic VTK wrapper. Reads VTK, STL, OpenFOAM, CGNS. Used at NASA. |
| **Surrogate Models** | [DeepXDE](https://deepxde.readthedocs.io/) `>= 1.15` (prototype) / [NVIDIA PhysicsNeMo](https://github.com/NVIDIA/physicsnemo) (production) | PINNs for prototype. FNO-based surrogates via PhysicsNeMo for production. |
| **Paper Discovery** | [OpenAlex](https://openalex.org/) + [Semantic Scholar](https://www.semanticscholar.org/) + [arXiv](https://arxiv.org/) | OpenAlex for broad discovery, S2 for citation graphs + SPECTER2 embeddings, arXiv for preprints. |
| **Dashboard** | [Streamlit](https://streamlit.io/) (MVP) / [Plotly Dash](https://dash.plotly.com/) (production) | Streamlit for rapid prototyping; Dash for complex interactive engineering dashboards. |

### LLM Routing Strategy

| Task | Model | Rationale |
|:--|:--|:--|
| Paper analysis & structured extraction | GPT-4o | Strong structured output, reliable JSON |
| CFD result interpretation | Claude Sonnet | Nuanced reasoning over numerical data |
| Regulation parsing | Claude Sonnet | Long-context precision |
| Quick metadata queries | GPT-4o-mini | Cost-efficient for simple lookups |
| Surrogate model code generation | Claude Sonnet | Superior code generation |

Routing is handled by LiteLLM proxy with tag-based model groups — the application code calls a single endpoint.

---

## Project Structure

```
aero-agent/
├── pyproject.toml                  # uv-managed, PEP 735 dependency groups
├── README.md
├── LICENSE
├── assets/
│   └── banner.svg
├── src/
│   └── aero_agent/
│       ├── __init__.py
│       ├── main.py                 # FastAPI application entrypoint
│       ├── config.py               # Pydantic Settings configuration
│       ├── literature/
│       │   ├── scanner.py          # Multi-source paper discovery (OpenAlex, S2, arXiv)
│       │   ├── paper_analyzer.py   # LLM-powered aero insight extraction (Cd, Cl, flow)
│       │   ├── knowledge_base.py   # RAG index over aero knowledge (Qdrant)
│       │   ├── regulation_parser.py # FIA technical regulation parsing (Docling)
│       │   └── citation_verifier.py # Verify citation accuracy and provenance
│       ├── cfd/
│       │   ├── result_parser.py    # Parse OpenFOAM (foamlib) / SU2 (pysu2) / Star-CCM+
│       │   ├── force_analyzer.py   # Cd, Cl, COP, moment coefficient analysis
│       │   ├── flow_detector.py    # Detect separation, vortex structures, recirculation
│       │   ├── design_modifier.py  # Propose geometry modifications from insights
│       │   └── budget_tracker.py   # FIA ATR CFD budget tracking & ROI thresholds
│       ├── correlation/
│       │   ├── lit_to_cfd.py       # Correlate literature insights with team CFD data
│       │   ├── telemetry_validator.py  # Validate CFD predictions vs on-track telemetry
│       │   └── insight_ranker.py   # Rank suggestions by expected ΔCl/ΔCd per ATR-hour
│       ├── suggestions/
│       │   ├── design_generator.py # Generate ranked design modification proposals
│       │   ├── roi_estimator.py    # Estimate aerodynamic gain per CFD-hour cost
│       │   └── optimizer.py        # Feedback loop for suggestion quality improvement
│       ├── telemetry/
│       │   ├── fastf1_client.py    # FastF1 API wrapper with caching
│       │   ├── speed_trace.py      # Straight/corner speed extraction and analysis
│       │   └── aero_balance.py     # Estimate front/rear aero balance from telemetry
│       └── api/
│           ├── routes.py           # FastAPI route definitions
│           └── schemas.py          # API request/response Pydantic models
├── tests/
│   ├── conftest.py
│   ├── test_literature/
│   ├── test_cfd/
│   ├── test_correlation/
│   ├── test_suggestions/
│   └── test_telemetry/
├── data/
│   ├── regulations/                # FIA technical regulation PDFs
│   ├── aero_knowledge_base/        # Seeded aero papers and reference material
│   └── reference_geometries/       # Open-source simplified F1 geometry
├── scripts/
│   ├── scan_papers.py              # CLI: discover and ingest new papers
│   ├── ingest_cfd_results.py       # CLI: parse and index CFD results
│   └── correlate_telemetry.py      # CLI: run telemetry-CFD correlation
└── docker/
    ├── Dockerfile
    └── docker-compose.yml          # Qdrant + PostgreSQL + AeroAgent
```

---

## Quickstart

### Prerequisites

- Python 3.11+
- [uv](https://docs.astral.sh/uv/getting-started/installation/) (package manager)
- Docker (for Qdrant + PostgreSQL)

### Installation

```bash
# Clone the repository
git clone https://github.com/your-username/aero-agent.git
cd aero-agent

# Install dependencies
uv sync

# Start infrastructure (Qdrant + PostgreSQL)
docker compose -f docker/docker-compose.yml up -d

# Copy and configure environment
cp .env.example .env
# Edit .env with your API keys (OpenAI, Anthropic, Cohere, etc.)

# Run the application
uv run python -m aero_agent.main
```

### First Steps

```bash
# 1. Scan and ingest aerodynamics papers
uv run python scripts/scan_papers.py --query "ground effect diffuser F1" --limit 50

# 2. Ask a question
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What does recent research suggest about diffuser optimization for ground effect cars?"}'

# 3. Ingest CFD results (OpenFOAM example)
uv run python scripts/ingest_cfd_results.py --case-dir ./data/cfd_runs/baseline/

# 4. Get ranked design suggestions
curl -X POST http://localhost:8000/api/suggestions \
  -H "Content-Type: application/json" \
  -d '{"component": "diffuser", "budget_hours": 10}'
```

---

## Roadmap

### Phase 1 — Literature MVP (8 weeks)

- [ ] Multi-source paper scanner (OpenAlex, Semantic Scholar, arXiv)
- [ ] PDF parsing pipeline (Docling + Marker fallback)
- [ ] LLM-powered aero insight extraction (Cd, Cl, flow structures, Re)
- [ ] RAG knowledge base with Qdrant (hybrid vector + metadata search)
- [ ] FIA regulation parser
- [ ] FastF1 telemetry integration
- [ ] Natural language query API
- [ ] Streamlit dashboard (MVP)

### Phase 2 — CFD Integration (12 weeks)

- [ ] CFD result parser (OpenFOAM via foamlib, SU2 via pysu2)
- [ ] Literature-CFD correlation engine
- [ ] Design suggestion generator with ROI ranking
- [ ] ATR budget optimization with dynamic ROI thresholds
- [ ] Telemetry-CFD validation pipeline
- [ ] PyVista-powered flow visualization

### Phase 3 — Surrogate Models & Production (8 weeks)

- [ ] DeepXDE-based PINN surrogates for rapid Cl/Cd prediction
- [ ] NVIDIA PhysicsNeMo FNO integration for production surrogates
- [ ] Plotly Dash production dashboard
- [ ] Docker deployment with Qdrant + PostgreSQL
- [ ] Feedback loop for suggestion quality (optimizer)

---

## Key Concepts

### ATR Budget Optimization

The FIA allocates CFD hours based on constructor championship position — the lower you finish, the more hours you get. As budget depletes, AeroAgent exponentially increases the ROI threshold for proposed simulations:

```python
def roi_threshold(utilization: float) -> float:
    """Require higher ROI as budget depletes."""
    return 1.0 + 2.0 * (utilization ** 2)
    # At 50% used: threshold = 1.5
    # At 80% used: threshold = 2.28
    # At 95% used: threshold = 2.80
```

### Literature-CFD Correlation

AeroAgent doesn't just find relevant papers — it correlates literature insights with your actual CFD data and on-track performance:

```
Literature says:  "VGs at 60% chord → +3-5% diffuser Cl"
Your CFD shows:   Separation onset at 55% chord
Telemetry shows:  Rear instability above 280 km/h
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Suggestion:       Add VG array at 58% chord on diffuser
Expected gain:    +8 counts Cl_rear
CFD cost:         2 ATR-hours
Confidence:       HIGH (3 corroborating sources)
```

### Hybrid RAG Search

Papers are indexed with both vector embeddings and structured metadata, enabling queries like:

> "Find papers about vortex shedding on front wing endplates at Re > 5×10⁶ published after 2022"

This combines semantic similarity (vector) with exact metadata filtering (Re, year, component) — powered by Qdrant's payload filtering during HNSW search.

---

## Target Users

| User | Use Case |
|:--|:--|
| **F1 Aero Departments** | Maximize aerodynamic gains per ATR-hour across development cycles |
| **Formula E Teams** | Literature-driven design optimization under tighter budgets |
| **Motorsport Consultancies** | Rapid literature review and CFD prioritization for clients |
| **University FSAE Teams** | Access professional-grade aero research tooling on student budgets |
| **Aero Researchers** | Structured knowledge base of aerodynamics literature with CFD context |

---

## Success Metrics

| Metric | Target |
|:--|:--|
| Literature coverage | Index >500 relevant aero papers with structured insights |
| Query quality | >75% of RAG answers rated "relevant and accurate" by aero engineers |
| Suggestion quality | >50% of design proposals deemed "worth investigating" |
| Telemetry correlation | CFD-to-track: within 2% for drag, 5% for downforce |

---

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

```bash
# Development setup
uv sync --group dev --group test
uv run ruff check .
uv run pytest
```

---

## License

MIT License. See [LICENSE](LICENSE) for details.

---

<p align="center">
  <sub>Built for the aero engineers who make 0.001s count.</sub>
</p>
