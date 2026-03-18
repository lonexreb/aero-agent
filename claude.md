# AeroAgent — Multi-Domain F1 Car Development Platform

## What This Project Is

AeroAgent is a multi-domain F1 car development platform with **6 specialized domain agents** (Aero, Materials, Chemistry, Structural, Thermal, ERS/Strategy). Each agent runs an autonomous research-to-simulation pipeline — synthesizing scientific literature, correlating findings with simulation results and on-track telemetry, and producing ranked design modification proposals.

F1 teams operate under FIA Aerodynamic Testing Restrictions (ATR) — every CFD run has a capped computational cost. Beyond aerodynamics, F1 car development spans 6 tightly coupled domains where a change in one (e.g., lighter floor material) cascades into others (aero flex behavior, structural compliance, thermal limits). AeroAgent makes every simulation count across all domains.

**Target users:** F1 aero departments, materials engineers, powertrain engineers, race strategists, Formula E teams, motorsport engineering consultancies, university FSAE teams.

**Why F1 specifically:** The ATR constraint means teams can't brute-force their way to solutions. An AI that helps prioritize which simulations to run — and resolves cross-domain trade-offs — has direct competitive value.

> **Full architecture specification:** See [ARCHITECTURE.md](ARCHITECTURE.md) for the complete 4-layer architecture, cross-domain dependency graph, per-domain specifications, and implementation roadmap.

---

## Tech Stack

### Core Infrastructure
- **Language:** Python 3.11+
- **Package Management:** uv >= 0.5
- **Linting & Formatting:** Ruff >= 0.9
- **Type Checking:** basedpyright
- **Data Validation:** Pydantic v2 >= 2.10
- **Testing:** pytest >= 8.0 (with pytest-asyncio, pytest-cov, pytest-mock, pytest-xdist)
- **API:** FastAPI >= 0.115

### AI / RAG Pipeline
- **RAG Framework:** LlamaIndex >= 0.11
- **Orchestration:** LangGraph >= 0.2
- **LLM Gateway:** LiteLLM >= 1.55
- **LLM:** GPT-4o (literature analysis), Claude Sonnet (CFD result interpretation)
- **Vector Database:** Qdrant >= 1.12
- **Relational DB:** PostgreSQL >= 16
- **Embeddings:** Cohere embed-v4 / Jina v3
- **PDF Parsing:** Docling >= 2.0 (primary), Marker >= 1.0 (batch fallback)

### Research Pipeline
- **Pipeline Framework:** AutoResearchClaw (8-phase research-to-report)
- **Agent Runtime:** NemoClaw (Phase 5 — sandboxed always-on operation)
- **Literature APIs:** OpenAlex, Semantic Scholar, arXiv, SAE MOBILUS, AIAA
- **MCP Servers:** openfoam-mcp-server, mcp.science, Scite MCP

### Domain-Specific Tools

| Domain | Tools |
|:--|:--|
| **Aerodynamics** | OpenFOAM / SU2 / Star-CCM+, foamlib >= 1.2, PyVista >= 0.47, FastF1 >= 3.8 |
| **Materials** | Materials Project API, AFLOW, LAMMPS |
| **Chemistry** | Cantera, RDKit, RMG, GROMACS |
| **Structural** | CalculiX, Code_Aster, preCICE |
| **Thermal** | OpenFOAM CHT, CoolProp, Elmer |
| **ERS/Strategy** | CasADi, OpenMDAO, Gymnasium, FastF1 |

### Surrogate Models
- **Prototype:** DeepXDE >= 1.15 (PINNs)
- **Production:** NVIDIA PhysicsNeMo (FNO-based surrogates)

### Dashboard
- **MVP:** Streamlit
- **Production:** Plotly Dash

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

## Directory Structure

```
aero-agent/
├── CLAUDE.md
├── ARCHITECTURE.md
├── README.md
├── pyproject.toml
├── assets/
│   └── banner.svg
├── src/
│   └── aero_agent/
│       ├── __init__.py
│       ├── main.py                     # FastAPI application entrypoint
│       ├── config.py                   # Pydantic Settings configuration
│       │
│       ├── domains/                    # 6 domain agents
│       │   ├── __init__.py
│       │   ├── base.py                 # Base domain agent (AutoResearchClaw pipeline)
│       │   │
│       │   ├── aero/
│       │   │   ├── __init__.py
│       │   │   ├── agent.py            # Aero domain agent
│       │   │   ├── scanner.py          # Multi-source paper discovery (OpenAlex, S2, arXiv)
│       │   │   ├── paper_analyzer.py   # LLM-powered aero insight extraction
│       │   │   ├── cfd_parser.py       # Parse OpenFOAM (foamlib) / SU2 / Star-CCM+
│       │   │   ├── force_analyzer.py   # Cd, Cl, COP, moment coefficient analysis
│       │   │   ├── flow_detector.py    # Detect separation, vortex structures
│       │   │   ├── design_modifier.py  # Propose geometry modifications
│       │   │   ├── budget_tracker.py   # FIA ATR CFD budget tracking
│       │   │   └── schemas.py          # AeroInsight, CFDResult, AeroProposal
│       │   │
│       │   ├── materials/
│       │   │   ├── __init__.py
│       │   │   ├── agent.py            # Materials domain agent
│       │   │   ├── matproject_client.py # Materials Project API wrapper
│       │   │   ├── aflow_client.py     # AFLOW REST API wrapper
│       │   │   ├── lammps_runner.py    # LAMMPS molecular dynamics integration
│       │   │   └── schemas.py          # MaterialInsight, MaterialProposal
│       │   │
│       │   ├── chemistry/
│       │   │   ├── __init__.py
│       │   │   ├── agent.py            # Chemistry domain agent
│       │   │   ├── cantera_runner.py   # Cantera combustion simulation
│       │   │   ├── rdkit_analyzer.py   # Molecular property analysis
│       │   │   ├── rmg_client.py       # Reaction mechanism generation
│       │   │   └── schemas.py          # FuelInsight, ChemistryProposal
│       │   │
│       │   ├── structural/
│       │   │   ├── __init__.py
│       │   │   ├── agent.py            # Structural domain agent
│       │   │   ├── calculix_runner.py  # CalculiX FEA integration
│       │   │   ├── precice_coupler.py  # preCICE fluid-structure interaction
│       │   │   └── schemas.py          # StructuralInsight, StructuralProposal
│       │   │
│       │   ├── thermal/
│       │   │   ├── __init__.py
│       │   │   ├── agent.py            # Thermal domain agent
│       │   │   ├── cht_runner.py       # OpenFOAM CHT integration
│       │   │   ├── coolprop_client.py  # CoolProp thermodynamic properties
│       │   │   └── schemas.py          # ThermalInsight, ThermalProposal
│       │   │
│       │   └── ers/
│       │       ├── __init__.py
│       │       ├── agent.py            # ERS/Strategy domain agent
│       │       ├── casadi_optimizer.py # CasADi optimal control
│       │       ├── openmda_runner.py   # OpenMDAO multidisciplinary optimization
│       │       ├── strategy_env.py     # Gymnasium RL environment for race strategy
│       │       └── schemas.py          # ERSInsight, StrategyProposal
│       │
│       ├── research/                   # AutoResearchClaw pipeline
│       │   ├── __init__.py
│       │   ├── pipeline.py             # 8-phase research pipeline (A-H)
│       │   ├── scoping.py              # Phase A: Define development targets
│       │   ├── discovery.py            # Phase B: Literature search
│       │   ├── synthesis.py            # Phase C: Cluster & hypothesize
│       │   ├── design.py               # Phase D: Generate simulation setup
│       │   ├── execution.py            # Phase E: Run simulation
│       │   ├── analysis.py             # Phase F: Evaluate results
│       │   ├── writing.py              # Phase G: Engineering report
│       │   └── finalization.py         # Phase H: Archive & learn
│       │
│       ├── orchestration/              # LangGraph state machine
│       │   ├── __init__.py
│       │   ├── graph.py                # LangGraph workflow definition
│       │   ├── dependency_resolver.py  # Cross-domain dependency management
│       │   ├── conflict_resolver.py    # Multi-domain conflict resolution
│       │   ├── scheduler.py            # Simulation scheduling under budget
│       │   └── pareto_ranker.py        # Multi-objective Pareto ranking
│       │
│       ├── mcp/                        # MCP server clients
│       │   ├── __init__.py
│       │   ├── openfoam_mcp.py         # openfoam-mcp-server client
│       │   ├── science_mcp.py          # mcp.science client
│       │   └── scite_mcp.py            # Scite MCP client
│       │
│       ├── runtime/                    # NemoClaw integration (Phase 5)
│       │   ├── __init__.py
│       │   ├── sandbox.py              # NemoClaw sandbox management
│       │   ├── inference_router.py     # LiteLLM model routing
│       │   └── monitor.py              # Resource & budget monitoring
│       │
│       ├── literature/                 # Shared literature utilities
│       │   ├── __init__.py
│       │   ├── knowledge_base.py       # RAG index over knowledge (Qdrant)
│       │   ├── regulation_parser.py    # FIA technical regulation parsing
│       │   └── citation_verifier.py    # Citation accuracy verification
│       │
│       ├── correlation/                # Cross-domain correlation
│       │   ├── __init__.py
│       │   ├── lit_to_sim.py           # Correlate literature with simulation data
│       │   ├── telemetry_validator.py  # Validate predictions vs on-track telemetry
│       │   └── insight_ranker.py       # Rank suggestions by expected gain/cost
│       │
│       ├── telemetry/
│       │   ├── __init__.py
│       │   ├── fastf1_client.py        # FastF1 API wrapper with caching
│       │   ├── speed_trace.py          # Straight/corner speed analysis
│       │   └── aero_balance.py         # Front/rear aero balance estimation
│       │
│       └── api/
│           ├── __init__.py
│           ├── routes.py               # FastAPI route definitions
│           └── schemas.py              # API request/response models
│
├── tests/
│   ├── conftest.py
│   ├── test_domains/
│   │   ├── test_aero/
│   │   ├── test_materials/
│   │   ├── test_chemistry/
│   │   ├── test_structural/
│   │   ├── test_thermal/
│   │   └── test_ers/
│   ├── test_research/
│   ├── test_orchestration/
│   ├── test_correlation/
│   └── test_telemetry/
│
├── data/
│   ├── regulations/                    # FIA technical regulation PDFs
│   ├── aero_knowledge_base/            # Seeded aero papers and reference material
│   ├── reference_geometries/           # Open-source simplified F1 geometry
│   └── chemistry/
│       └── mechanisms/                 # Cantera reaction mechanism files
│
├── scripts/
│   ├── scan_papers.py                  # CLI: discover and ingest new papers
│   ├── ingest_cfd_results.py           # CLI: parse and index CFD results
│   └── correlate_telemetry.py          # CLI: run telemetry-CFD correlation
│
└── docker/
    ├── Dockerfile
    └── docker-compose.yml              # Qdrant + PostgreSQL + AeroAgent
```

---

## Architecture (Summary)

AeroAgent uses a 4-layer architecture. See [ARCHITECTURE.md](ARCHITECTURE.md) for the full specification.

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4 — RUNTIME: NemoClaw sandbox (Phase 5)                  │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 3 — ORCHESTRATION: LangGraph state machine               │
│  Cross-domain deps · Simulation scheduling · Pareto ranking     │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 2 — DOMAIN AGENTS                                        │
│  ┌──────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌───────┐ ┌───┐│
│  │ Aero │ │Materials│ │Chemistry│ │Structural│ │Thermal│ │ERS││
│  └──────┘ └─────────┘ └─────────┘ └──────────┘ └───────┘ └───┘│
│  Each: AutoResearchClaw 8-phase pipeline + Qdrant collection    │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 1 — MCP TOOLS                                            │
│  openfoam-mcp · mcp.science · Scite MCP · domain MCP servers   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Components

### ATR Budget Tracker (`budget_tracker.py`)

The FIA limits CFD usage based on constructor standings. This is a real constraint.

```python
@dataclass
class ATRBudget:
    team_position: int                    # Constructor championship position
    total_atr_hours_per_period: float     # Allocated hours
    used_hours: float
    remaining_hours: float
    period_end_date: date

    def can_afford(self, proposed_run_hours: float) -> bool:
        return self.remaining_hours >= proposed_run_hours

    def roi_threshold(self) -> float:
        """As budget depletes, require higher expected ROI per run."""
        utilization = self.used_hours / self.total_atr_hours_per_period
        # Exponentially increase ROI threshold as budget depletes
        return 1.0 + 2.0 * (utilization ** 2)
```

### Telemetry-CFD Correlation (`telemetry_validator.py`)

```python
def correlate_cfd_with_track(cfd_result: CFDResult, session: FastF1Session) -> Correlation:
    """
    Compare CFD predictions with on-track behavior.

    Key correlations:
    - Predicted vs. actual top speed (drag accuracy)
    - Predicted vs. actual cornering speed (downforce accuracy)
    - Predicted vs. actual aero balance (COP accuracy)
    """
    # Extract speed traces for high-speed straights (drag indicator)
    straight_speeds = extract_straight_speeds(session)
    predicted_top_speed = estimate_top_speed(cfd_result.cd, car_params)

    # Extract cornering speeds for high-downforce corners
    corner_speeds = extract_corner_speeds(session, corner_type="high_speed")
    predicted_corner_speed = estimate_corner_speed(cfd_result.cl, car_params)

    return Correlation(
        drag_error=abs(predicted_top_speed - max(straight_speeds)) / max(straight_speeds),
        downforce_error=...,
        balance_error=...,
    )
```

---

## Implementation Roadmap

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed phase breakdowns.

| Phase | Focus | Duration |
|:--|:--|:--|
| **Phase 1** | Literature MVP — multi-domain scanner + RAG + Streamlit | 8 weeks |
| **Phase 2** | Aero Agent — CFD integration, ATR budget, telemetry correlation | 10 weeks |
| **Phase 3** | Materials + Chemistry Agents — Materials Project, Cantera, LAMMPS | 8 weeks |
| **Phase 4** | Structural + Thermal + ERS Agents — FEA, CHT, energy strategy | 8 weeks |
| **Phase 5** | NemoClaw Integration — always-on autonomous loop, MCP orchestration | 6 weeks |

---

## Success Metrics

### Aerodynamics (Phase 2+)
- **Literature coverage:** Index >500 relevant aero papers with structured insights
- **Query quality:** Aero engineers rate >75% of RAG answers as "relevant and accurate"
- **Suggestion quality:** >50% of design suggestions deemed "worth investigating" by aero engineers
- **Telemetry correlation:** CFD-to-track speed predictions within 2% for drag, 5% for downforce

### Materials & Chemistry (Phase 3+)
- **Materials coverage:** Index >200 relevant CFRP/composite papers
- **Property accuracy:** Material property predictions within 10% of experimental values
- **Chemistry accuracy:** Combustion simulation results within 5% of dyno measurements

### Structural & Thermal (Phase 4+)
- **FEA correlation:** Stress predictions within 8% of physical test results
- **Thermal accuracy:** Component temperature predictions within 5°C of track measurements
- **Cross-domain proposals:** >30% of proposals span multiple domains with resolved trade-offs

### Autonomous Operation (Phase 5)
- **Uptime:** >95% continuous operation over 24-hour campaigns
- **Proposal throughput:** >10 ranked proposals per overnight campaign
- **Budget efficiency:** <5% ATR budget waste on low-ROI simulations
