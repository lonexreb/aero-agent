# AeroAgent — CFD-Aware Research Assistant for F1 Aerodynamicists

## What This Project Is

AeroAgent is a research assistant that synthesizes aerodynamics literature, correlates findings with CFD simulation results, and suggests design modifications ranked by expected downforce/drag improvement per CFD-hour budget. F1 teams operate under FIA Aerodynamic Testing Restrictions (ATR) — every CFD run has a capped computational cost. AeroAgent makes each run count.

**Target users:** F1 aero departments, Formula E teams, motorsport engineering consultancies, university FSAE teams.

**Why F1 specifically:** The ATR constraint means teams can't brute-force their way to solutions. An AI that helps prioritize which simulations to run has direct competitive value.

---

## Tech Stack

- **Language:** Python 3.11+
- **LLM:** GPT-4o (literature analysis), Claude Sonnet (CFD result interpretation)
- **Literature APIs:** Semantic Scholar, arXiv, SAE MOBILUS, AIAA
- **CFD:** OpenFOAM / SU2 (open-source), Star-CCM+ integration via file I/O
- **Telemetry:** FastF1 (open F1 telemetry API)
- **Surrogate models:** PyTorch, DeepXDE (for physics-informed surrogates)
- **RAG:** LlamaIndex or LangChain with pgvector
- **Database:** PostgreSQL + pgvector
- **Dashboard:** Streamlit / Plotly Dash
- **API:** FastAPI

---

## Directory Structure

```
aero-agent/
├── claude.md
├── pyproject.toml
├── src/
│   ├── main.py
│   ├── config.py
│   ├── literature/
│   │   ├── aero_scanner.py        # Monitor aero-specific publications
│   │   ├── paper_analyzer.py      # Extract aero insights (Cd, Cl, flow structures)
│   │   ├── knowledge_base.py      # RAG index of aero knowledge
│   │   ├── regulation_parser.py   # Parse FIA technical regulations
│   │   └── citation_verifier.py
│   ├── cfd/
│   │   ├── result_parser.py       # Parse OpenFOAM/SU2/Star-CCM+ results
│   │   ├── force_analyzer.py      # Cd, Cl, COP analysis
│   │   ├── flow_feature_detector.py # Detect separation, vortex structures
│   │   ├── design_modifier.py     # Propose geometry modifications
│   │   └── budget_tracker.py      # Track ATR CFD budget usage
│   ├── correlation/
│   │   ├── lit_to_cfd.py          # Correlate literature insights with team's CFD data
│   │   ├── telemetry_validator.py # Validate CFD predictions against on-track telemetry
│   │   └── insight_ranker.py      # Rank design suggestions by expected gain/cost
│   ├── suggestions/
│   │   ├── design_generator.py    # Generate ranked design modification proposals
│   │   ├── roi_estimator.py       # Estimate ΔCl/ΔCd per CFD-hour
│   │   └── optimizer.py           # Karpathy Loop for suggestion quality
│   ├── telemetry/
│   │   ├── fastf1_client.py       # FastF1 API wrapper
│   │   ├── speed_trace_analyzer.py
│   │   └── aero_balance_estimator.py # Estimate front/rear aero balance from telemetry
│   └── api/
│       └── routes.py
├── tests/
├── data/
│   ├── regulations/               # FIA technical regulation PDFs
│   ├── aero_knowledge_base/       # Seeded aero papers and textbooks
│   └── reference_geometries/      # Open-source F1 geometry (simplified)
└── scripts/
    ├── scan_papers.py
    ├── ingest_cfd_results.py
    └── correlate_telemetry.py
```

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  KNOWLEDGE LAYER                     │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐│
│  │ Aero Papers  │  │ FIA Regs     │  │ Textbooks  ││
│  │ (arXiv, SAE, │  │ (parsed      │  │ (Katz &    ││
│  │  AIAA)       │  │  annually)   │  │  Plotkin,  ││
│  └──────┬───────┘  └──────┬───────┘  │  Anderson) ││
│         │                  │          └─────┬──────┘│
│         ▼                  ▼                ▼       │
│  ┌──────────────────────────────────────────────┐   │
│  │           RAG Knowledge Base (pgvector)       │   │
│  │  Indexed by: concept, component, flow regime  │   │
│  └──────────────────────┬───────────────────────┘   │
└─────────────────────────┼───────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                 ANALYSIS LAYER                        │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐│
│  │ Team's CFD   │  │ FastF1       │  │ ATR Budget ││
│  │ Results      │  │ Telemetry    │  │ Tracker    ││
│  │ (forces,     │  │ (speed,      │  │ (remaining ││
│  │  pressures,  │  │  throttle,   │  │  CFD hours)││
│  │  flow vis)   │  │  lateral g)  │  │            ││
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘│
│         │                  │                │       │
│         ▼                  ▼                ▼       │
│  ┌──────────────────────────────────────────────┐   │
│  │        Correlation & Insight Engine            │   │
│  │                                                │   │
│  │  "The literature suggests vortex generators    │   │
│  │   at 60% chord improve diffuser performance    │   │
│  │   by 3-5%. Your CFD shows separation at 55%    │   │
│  │   chord on the current design. Telemetry       │   │
│  │   confirms rear instability at high speed.     │   │
│  │   Estimated gain: +8 counts Cl_rear.           │   │
│  │   CFD cost: 2 ATR-hours."                      │   │
│  └──────────────────────┬───────────────────────┘   │
└─────────────────────────┼───────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│              SUGGESTION LAYER                        │
│                                                      │
│  Ranked design proposals:                            │
│  1. VG placement on diffuser (ΔCl/cost = 4.0)      │
│  2. Floor edge fence geometry (ΔCl/cost = 3.2)      │
│  3. Rear wing endplate cutout (ΔCl/cost = 2.8)      │
│                                                      │
│  Each includes:                                      │
│  - Literature backing (cited papers)                 │
│  - Expected aerodynamic delta                        │
│  - CFD budget required                               │
│  - Confidence level                                  │
│  - Risk assessment                                   │
└─────────────────────────────────────────────────────┘
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

## MVP Scope

### Literature-only MVP (8 weeks)
- [ ] Aero paper scanner (arXiv, SAE, AIAA)
- [ ] Paper analyzer for aerodynamic insights
- [ ] RAG knowledge base with pgvector
- [ ] FIA regulation parser
- [ ] FastF1 telemetry integration
- [ ] Natural language query: "What does recent research suggest about diffuser optimization for ground effect cars?"
- [ ] Streamlit dashboard

### Full CFD integration (additional 12 weeks)
- [ ] CFD result parser (OpenFOAM/SU2)
- [ ] Literature-CFD correlation engine
- [ ] Design suggestion generator with ROI ranking
- [ ] ATR budget optimization
- [ ] Karpathy Loop for suggestion quality

---

## Success Metrics

- **Literature coverage:** Index >500 relevant aero papers with structured insights
- **Query quality:** Aero engineers rate >75% of RAG answers as "relevant and accurate"
- **Suggestion quality:** >50% of design suggestions are deemed "worth investigating" by aero engineers
- **Telemetry correlation:** CFD-to-track speed predictions within 2% for drag, 5% for downforce
