# AeroAgent Architecture

> Multi-Domain F1 Car Development Platform — Technical Architecture Specification

---

## 1. System Overview

AeroAgent is a multi-domain F1 car development platform built around **6 specialized domain agents**, each running an autonomous research-to-simulation pipeline. The system synthesizes scientific literature, correlates findings with simulation results and on-track telemetry, and produces ranked design modification proposals — all under FIA resource constraints.

### Domain Agents

| Agent | Domain | Core Question |
|:--|:--|:--|
| **Aero** | Aerodynamics | How do we maximize downforce while minimizing drag under ATR budget? |
| **Materials** | Materials Science | What material/layup gives the best stiffness-to-weight for this component? |
| **Chemistry** | Combustion & Fluids | How do fuel formulation and lubricant properties affect power and reliability? |
| **Structural** | Structural Engineering | Does this design survive crash loads, fatigue cycles, and suspension forces? |
| **Thermal** | Thermal Management | Can we cool this component without excessive cooling drag penalty? |
| **ERS/Strategy** | Energy & Race Strategy | How do we deploy energy optimally across a stint/race? |

### Key Design Principles

1. **Each agent is autonomous** — runs the full AutoResearchClaw 8-phase pipeline (literature → hypothesis → simulation → evaluation) parameterized by domain
2. **Orchestrated by LangGraph** — state machine manages cross-domain dependencies, simulation scheduling, and conflict resolution
3. **Connected via MCP** — simulation tools exposed as Model Context Protocol servers for standardized tool access
4. **Always-on capability** — NemoClaw sandbox runtime (Phase 5) enables continuous autonomous operation with network isolation
5. **ATR-aware globally** — every simulation proposal is evaluated against remaining FIA budget with exponentially increasing ROI thresholds

---

## 2. The Karpathy Loop Applied to F1

The core feedback loop follows the autoresearch pattern, adapted for multi-domain F1 engineering:

```
┌─────────────────────────────────────────────────────────────┐
│                    KARPATHY LOOP (per domain)                │
│                                                              │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│   │ RESEARCH  │───▶│ SIMULATE │───▶│ EVALUATE │             │
│   │           │    │          │    │          │             │
│   │ Literature│    │ CFD/FEA/ │    │ ΔCl, ΔCd,│             │
│   │ + propose │    │ thermal/ │    │ mass,    │             │
│   │ design    │    │ chemistry│    │ stress,  │             │
│   │ change    │    │ sim      │    │ temp,    │             │
│   └──────────┘    └──────────┘    │ lap time │             │
│        ▲                           └────┬─────┘             │
│        │                                │                    │
│        │         ┌──────────┐           │                    │
│        └─────────│ KEEP /   │◀──────────┘                    │
│                  │ REVERT   │                                │
│                  │          │                                │
│                  │ ROI >    │                                │
│                  │ threshold│                                │
│                  └──────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

| Karpathy Original | F1 Adaptation |
|:--|:--|
| `modify train.py` | Research literature + propose design modification |
| `train` | Run CFD / FEA / thermal / combustion simulation |
| `evaluate val_bpb` | Evaluate ΔCl, ΔCd, mass, stress, temperature, lap time |
| `keep/revert` | Keep if ROI > threshold, revert otherwise |

**Key difference from vanilla autoresearch:** Multi-objective Pareto optimality across 6 coupled domains. A change that improves downforce may increase mass (materials), require more cooling (thermal), or violate crash test requirements (structural). The orchestration layer resolves these cross-domain trade-offs.

---

## 3. Four-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 4 — RUNTIME (NemoClaw Sandbox)                               │
│  Network isolation · Filesystem restriction · Inference routing     │
│  Continuous operation · Resource monitoring                          │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 3 — ORCHESTRATION (LangGraph State Machine)                  │
│                                                                      │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Cross-  │  │ Simul-  │  │ Conflict │  │ Multi-   │             │
│  │ Domain  │  │ ation   │  │ Resolut- │  │ Objective│             │
│  │ Deps    │  │ Sched-  │  │ ion      │  │ Ranking  │             │
│  │ Graph   │  │ uler    │  │          │  │ (Pareto) │             │
│  └─────────┘  └─────────┘  └──────────┘  └──────────┘             │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 2 — DOMAIN AGENTS                                            │
│                                                                      │
│  ┌────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────┐ ┌───┐│
│  │ AERO   │ │MATERIALS │ │CHEMISTRY │ │STRUCTURAL│ │THERMAL│ │ERS││
│  │        │ │          │ │          │ │          │ │       │ │   ││
│  │OpenFOAM│ │Materials │ │Cantera   │ │CalculiX  │ │OF-CHT │ │F1 ││
│  │SU2     │ │Project   │ │RDKit     │ │Code_Aster│ │CoolPrp│ │CAS││
│  │foamlib │ │AFLOW     │ │RMG       │ │preCICE   │ │Elmer  │ │ADi││
│  │PyVista │ │LAMMPS    │ │GROMACS   │ │          │ │       │ │   ││
│  └────────┘ └──────────┘ └──────────┘ └──────────┘ └───────┘ └───┘│
│                                                                      │
│  Each agent: own Qdrant collection · domain schemas · eval metrics  │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 1 — MCP TOOL LAYER                                           │
│                                                                      │
│  ┌────────────────┐ ┌────────────┐ ┌──────────┐ ┌───────────────┐  │
│  │openfoam-mcp-   │ │mcp.science │ │Scite MCP │ │Domain-specific│  │
│  │server           │ │(MatProj,   │ │(citation │ │MCP servers    │  │
│  │                 │ │ GPAW,      │ │ verify)  │ │(Cantera,      │  │
│  │                 │ │ Jupyter)   │ │          │ │ CalculiX,     │  │
│  │                 │ │            │ │          │ │ CoolProp)     │  │
│  └────────────────┘ └────────────┘ └──────────┘ └───────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer 1 — MCP Tool Layer

Simulation tools are exposed as [Model Context Protocol](https://modelcontextprotocol.io/) servers, providing standardized tool access for LLM agents.

| MCP Server | Purpose | Domain(s) |
|:--|:--|:--|
| `openfoam-mcp-server` | Run OpenFOAM cases, read results | Aero, Thermal |
| `mcp.science` | Materials Project queries, GPAW DFT, Jupyter execution | Materials, Chemistry |
| `Scite MCP` | Citation verification and smart citation search | All |
| Custom Cantera MCP | Combustion/reaction simulations | Chemistry |
| Custom CalculiX MCP | FEA stress/fatigue analysis | Structural |
| Custom CoolProp MCP | Thermodynamic property lookups | Thermal |
| FastF1 MCP | Telemetry data access | Aero, ERS/Strategy |
| CasADi MCP | Optimization problem setup and solve | ERS/Strategy |

### Layer 2 — Domain Agent Layer

Each agent is an instance of the AutoResearchClaw 8-phase pipeline, parameterized by:
- **Research sources** — which APIs and databases to query
- **Simulation tools** — which MCP servers to invoke
- **Qdrant collection** — domain-specific vector index with typed metadata
- **Evaluation metrics** — domain-specific success criteria
- **Pydantic schemas** — structured data models for insights, results, proposals

### Layer 3 — Orchestration Layer

LangGraph state machine that:
1. **Manages cross-domain dependencies** — ensures a thermal analysis runs before an aero agent finalizes sidepod geometry
2. **Schedules simulations** — respects ATR budget and computational resource limits
3. **Resolves conflicts** — when two domains disagree (e.g., aero wants thin wing, structural needs thickness for stiffness)
4. **Ranks proposals** — multi-objective Pareto ranking across all 6 domains, weighted by current development priorities

### Layer 4 — Runtime Layer (Phase 5)

NemoClaw-based sandbox providing:
- **Network isolation** — agents can only reach whitelisted paper APIs and local simulation tools
- **Filesystem restriction** — agents cannot access team data outside designated paths
- **Inference routing** — LiteLLM proxy routes to optimal model per task type
- **Continuous operation** — always-on loop for overnight simulation campaigns
- **Resource monitoring** — track CPU/GPU usage, memory, and ATR budget in real-time

---

## 4. Cross-Domain Dependency Graph

Domain agents don't operate in isolation. The following bidirectional dependencies define the constraints that the orchestration layer must manage:

```
                    ┌──────────┐
            ┌───────│   AERO   │───────┐
            │       └────┬─────┘       │
            │            │             │
     cooling drag   aeroelastic    aero loads
     trade-off      flutter/flex   on structure
            │            │             │
            ▼            ▼             ▼
     ┌──────────┐  ┌──────────┐  ┌──────────┐
     │ THERMAL  │  │STRUCTURAL│  │MATERIALS │
     └────┬─────┘  └────┬─────┘  └────┬─────┘
          │              │             │
     thermal        thermal       CFRP layup
     fatigue/       stress/       crash
     creep          fatigue       absorption
          │              │             │
          └──────┬───────┘             │
                 │                     │
          ┌──────▼─────┐              │
          │ CHEMISTRY  │──────────────┘
          │            │  fuel energy density
          └──────┬─────┘  lubricant properties
                 │
          battery chemistry
          fuel formulation
                 │
          ┌──────▼─────┐
          │    ERS /   │
          │  STRATEGY  │
          └────────────┘
```

### Dependency Annotations

| Dependency | Direction | Constraint |
|:--|:--|:--|
| Aero ↔ Thermal | Bidirectional | Cooling drag trade-off — sidepod inlet sizing, brake duct flow vs. wheel wake |
| Aero ↔ Structural | Bidirectional | Aeroelastic flutter — wing flex under load changes effective AoA |
| Aero ↔ Materials | Aero → Materials | Aero loads define material requirements (front wing, floor, diffuser) |
| Materials ↔ Structural | Bidirectional | CFRP layup schedule determines both stiffness and crash absorption |
| Chemistry ↔ ERS | Bidirectional | Fuel energy density affects range/weight; battery chemistry affects power/thermal |
| Thermal ↔ ERS | Bidirectional | Battery thermal limits constrain deployment; MGU-K temperature affects efficiency |
| Structural ↔ Thermal | Bidirectional | Thermal fatigue and creep in exhaust/brake components |
| Chemistry ↔ Materials | Chemistry → Materials | Tire compound behavior depends on material science; fuel compatibility with seals |

---

## 5. Per-Domain Specification

### 5.1 Aerodynamics Agent

**Research Sources:** arXiv (physics.flu-dyn), SAE MOBILUS, AIAA, OpenAlex, Semantic Scholar

**Simulation Tools:** OpenFOAM (via foamlib + openfoam-mcp-server), SU2 (via pysu2), Star-CCM+ (file I/O)

**Optimization Targets:** Maximize Cl (downforce), minimize Cd (drag), optimize COP (center of pressure balance)

**Key Schemas:**
```python
class AeroInsight(BaseModel):
    component: str              # "front_wing", "floor", "diffuser", "rear_wing"
    flow_feature: str           # "separation", "vortex", "wake", "ground_effect"
    reynolds_number: float | None
    cl_delta: float | None      # Expected change in lift coefficient
    cd_delta: float | None      # Expected change in drag coefficient
    confidence: float           # 0.0-1.0
    source_papers: list[str]    # DOIs

class CFDResult(BaseModel):
    solver: Literal["openfoam", "su2", "starccm"]
    cl: float
    cd: float
    cop: float                  # Center of pressure (% from front axle)
    atr_hours_used: float
    separation_regions: list[SeparationRegion]
    vortex_structures: list[VortexStructure]

class AeroProposal(BaseModel):
    modification: str
    component: str
    expected_cl_delta: float
    expected_cd_delta: float
    cfd_hours_required: float
    roi_score: float            # ΔCl / cfd_hours_required
    literature_backing: list[AeroInsight]
    confidence: float
    risk: Literal["low", "medium", "high"]
```

### 5.2 Materials Agent

**Research Sources:** Materials Project API, AFLOW REST API, OpenAlex, Semantic Scholar

**Simulation Tools:** LAMMPS (molecular dynamics), CalculiX (FEA), custom materials DB

**Optimization Targets:** Maximize specific stiffness (E/ρ), maximize fatigue life, minimize mass

**Key Schemas:**
```python
class MaterialInsight(BaseModel):
    material_class: str         # "CFRP", "titanium_alloy", "ceramic_composite"
    property_type: str          # "tensile_strength", "fatigue_life", "thermal_conductivity"
    value: float
    units: str
    conditions: dict            # Temperature, strain rate, layup orientation
    source_papers: list[str]

class MaterialProposal(BaseModel):
    component: str
    current_material: str
    proposed_material: str
    mass_delta_kg: float
    stiffness_delta_pct: float
    fatigue_life_delta_pct: float
    cost_delta: float
    simulation_hours: float
    confidence: float
```

### 5.3 Chemistry Agent

**Research Sources:** PubChem, ChemRxiv, SAE (fuel/lubricant papers), OpenAlex

**Simulation Tools:** Cantera (combustion kinetics), RDKit (molecular properties), RMG (reaction mechanism generation), GROMACS (molecular dynamics)

**Optimization Targets:** Fuel lower heating value (LHV), lubricant viscosity profile, tire compound grip/degradation

**Key Schemas:**
```python
class FuelInsight(BaseModel):
    formulation: str
    lhv_mj_per_kg: float
    ron: float                  # Research Octane Number
    sustainable_fraction: float # FIA requires increasing % sustainable fuel
    source_papers: list[str]

class ChemistryProposal(BaseModel):
    domain: Literal["fuel", "lubricant", "tire_compound", "hydraulic_fluid"]
    modification: str
    performance_delta: dict     # Domain-specific metrics
    simulation_hours: float
    confidence: float
```

### 5.4 Structural Agent

**Research Sources:** AIAA, SAE, ASME, OpenAlex

**Simulation Tools:** CalculiX (FEA), Code_Aster (advanced FEA), preCICE (fluid-structure coupling)

**Optimization Targets:** FIA crash test compliance, fatigue life > N cycles, suspension load paths, minimum mass

**Key Schemas:**
```python
class StructuralInsight(BaseModel):
    component: str              # "monocoque", "front_wing", "suspension_arm"
    load_case: str              # "crash_front", "fatigue_cycling", "static_aero"
    failure_mode: str           # "buckling", "delamination", "fatigue_crack"
    safety_factor: float
    source_papers: list[str]

class StructuralProposal(BaseModel):
    component: str
    modification: str           # "layup_reorientation", "rib_addition", "topology_opt"
    mass_delta_kg: float
    safety_factor_delta: float
    fia_compliance: bool
    simulation_hours: float
    confidence: float
```

### 5.5 Thermal Agent

**Research Sources:** ASME, SAE thermal management papers, OpenAlex

**Simulation Tools:** OpenFOAM CHT (conjugate heat transfer), CoolProp (thermodynamic properties), Elmer (multiphysics)

**Optimization Targets:** Keep components within temperature limits, minimize cooling drag penalty

**Key Schemas:**
```python
class ThermalInsight(BaseModel):
    component: str              # "brakes", "PU_radiator", "battery", "MGU-K"
    max_temp_c: float
    cooling_method: str         # "forced_convection", "heat_pipe", "phase_change"
    drag_penalty_counts: float  # Drag counts from cooling apertures
    source_papers: list[str]

class ThermalProposal(BaseModel):
    component: str
    modification: str
    temp_reduction_c: float
    cooling_drag_delta: float   # Negative = less drag = good
    simulation_hours: float
    confidence: float
```

### 5.6 ERS / Strategy Agent

**Research Sources:** FastF1 (telemetry), SAE (hybrid powertrain), IEEE (battery/motor), OpenAlex

**Simulation Tools:** CasADi (optimal control), OpenMDAO (multidisciplinary optimization), Gymnasium (RL for strategy)

**Optimization Targets:** Optimal energy deployment per lap/stint, active aero strategy (DRS+), race strategy

**Key Schemas:**
```python
class ERSInsight(BaseModel):
    topic: str                  # "deployment_map", "battery_chemistry", "mgu_efficiency"
    lap_time_delta_s: float
    energy_delta_kj: float
    source_papers: list[str]

class StrategyProposal(BaseModel):
    strategy_type: Literal["energy_deployment", "tire_strategy", "active_aero"]
    modification: str
    lap_time_delta_s: float
    energy_cost_kj: float
    simulation_hours: float
    confidence: float
```

---

## 6. AutoResearchClaw Integration Pattern

Each domain agent runs the [AutoResearchClaw](https://github.com/TechxGenus/AutoResearchClaw) 8-phase pipeline, adapted for F1 engineering:

| Phase | AutoResearchClaw | F1 Adaptation |
|:--|:--|:--|
| **A — Scoping** | Define research question | Team development targets (e.g., "+15 points Cl_rear by Silverstone") |
| **B — Discovery** | Literature search | Domain-specific queries to OpenAlex, Semantic Scholar, arXiv, SAE |
| **C — Synthesis** | Cluster and synthesize | Cluster insights by component/flow-feature, generate design hypotheses |
| **D — Design** | Design experiment | Generate simulation setup files (OpenFOAM case, CalculiX input, Cantera mechanism) |
| **E — Execution** | Run experiment | Run simulation in sandbox via MCP server |
| **F — Analysis** | Analyze results | Evaluate against targets → PROCEED / REFINE / PIVOT |
| **G — Writing** | Write paper | Generate structured engineering report with citations and CFD figures |
| **H — Finalization** | Archive + MetaClaw | Archive to Qdrant for cross-run learning; update surrogate models |

### Phase Flow

```
  A (Scoping)
     │
     ▼
  B (Discovery) ──────────────────────┐
     │                                 │
     ▼                                 │
  C (Synthesis)                        │
     │                                 │
     ▼                                 │
  D (Design) ◄───── REFINE ────┐      │
     │                          │      │
     ▼                          │      │
  E (Execution)                 │      │
     │                          │      │
     ▼                          │      │
  F (Analysis) ─── PROCEED ─┐  │      │
     │                       │  │      │
     ├── REFINE ─────────────┘  │      │
     │                          │      │
     └── PIVOT ─────────────────┘──────┘
                                       │
  G (Writing) ◄────── PROCEED ─────────┘
     │
     ▼
  H (Finalization)
     │
     ▼
  [Next iteration or new target]
```

---

## 7. Five-Phase Implementation Roadmap

### Phase 1 — Literature MVP (8 weeks)

**Goal:** Generic multi-domain literature scanner + RAG + query interface

- Multi-source paper scanner (OpenAlex, Semantic Scholar, arXiv)
- PDF parsing pipeline (Docling primary + Marker fallback)
- LLM-powered insight extraction parameterized by domain
- RAG knowledge base with Qdrant (hybrid vector + metadata search)
- FIA regulation parser
- FastF1 telemetry integration
- Natural language query API (FastAPI)
- Streamlit dashboard (MVP)

**Deliverable:** Ask "What does recent research suggest about diffuser optimization for ground effect cars?" and get cited, structured answers.

### Phase 2 — Aero Agent (10 weeks)

**Goal:** Full aero domain agent with CFD integration and ATR budget tracking

- CFD result parser (OpenFOAM via foamlib, SU2 via pysu2)
- Flow feature detection (separation, vortex structures)
- Literature-CFD correlation engine
- Telemetry-CFD validation (FastF1)
- Design suggestion generator with ROI ranking
- ATR budget optimization with dynamic ROI thresholds
- PyVista flow visualization
- AutoResearchClaw 8-phase pipeline for aero domain

**Deliverable:** Ingest CFD results → get ranked design proposals with literature backing, expected ΔCl/ΔCd, and CFD budget cost.

### Phase 3 — Materials + Chemistry Agents (8 weeks)

**Goal:** Two additional domain agents with simulation integration

- Materials Agent: Materials Project API, AFLOW, LAMMPS integration
- Chemistry Agent: Cantera combustion sims, RDKit molecular analysis, RMG mechanisms
- Domain-specific Qdrant collections and schemas
- Cross-domain dependency handling (Materials ↔ Structural, Chemistry ↔ ERS)
- MCP servers for mcp.science, Cantera, CalculiX

**Deliverable:** Multi-domain proposals — e.g., "lighter CFRP layup for floor saves 0.8kg, requires CFD re-evaluation of flex behavior."

### Phase 4 — Structural + Thermal + ERS Agents (8 weeks)

**Goal:** Complete the 6-agent constellation

- Structural Agent: CalculiX FEA, Code_Aster, preCICE FSI coupling
- Thermal Agent: OpenFOAM CHT, CoolProp thermodynamics, Elmer multiphysics
- ERS/Strategy Agent: CasADi optimal control, OpenMDAO, Gymnasium RL
- Full cross-domain dependency graph active
- Multi-objective Pareto ranking across all 6 domains
- LangGraph orchestration for conflict resolution

**Deliverable:** Unified design proposals ranked across all 6 domains — e.g., "sidepod redesign improves aero by +12 Cl counts, requires thermal re-evaluation, structural impact minimal."

### Phase 5 — NemoClaw Integration (6 weeks)

**Goal:** Always-on autonomous operation in sandboxed runtime

- NemoClaw sandbox deployment (Docker-based)
- Network isolation (whitelisted paper APIs only)
- Filesystem restriction (designated data paths only)
- Inference routing via LiteLLM proxy
- Continuous overnight simulation campaigns
- MetaClaw-style cross-run learning
- Resource and ATR budget monitoring dashboard
- Surrogate model training (DeepXDE prototype → PhysicsNeMo production)

**Deliverable:** Leave it running overnight → wake up to ranked proposals from autonomous simulation campaigns, with full audit trail.

---

## 8. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA FLOW                                     │
│                                                                      │
│  Paper APIs                  Team Data                               │
│  ┌─────────┐                 ┌───────────┐                          │
│  │OpenAlex │                 │ CFD Runs   │                          │
│  │Sem.Schol│                 │ (OpenFOAM, │                          │
│  │arXiv    │──┐              │  SU2)      │──┐                      │
│  │SAE      │  │              └───────────┘  │                      │
│  │AIAA     │  │              ┌───────────┐  │                      │
│  └─────────┘  │              │ Telemetry  │  │                      │
│               │              │ (FastF1)   │──┤                      │
│               ▼              └───────────┘  │                      │
│         ┌──────────┐         ┌───────────┐  │                      │
│         │ Docling / │         │ ATR Budget│──┤                      │
│         │ Marker    │         │ Tracker   │  │                      │
│         └────┬─────┘         └───────────┘  │                      │
│              │                              │                      │
│              ▼                              ▼                      │
│    ┌──────────────────────────────────────────────┐                │
│    │              Qdrant Vector DB                  │                │
│    │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │                │
│    │  │ aero   │ │material│ │ chem   │ │ struct │ │                │
│    │  │collectn│ │collectn│ │collectn│ │collectn│ │                │
│    │  └────────┘ └────────┘ └────────┘ └────────┘ │                │
│    │  ┌────────┐ ┌────────┐                        │                │
│    │  │thermal │ │  ers   │                        │                │
│    │  │collectn│ │collectn│                        │                │
│    │  └────────┘ └────────┘                        │                │
│    └──────────────────┬───────────────────────────┘                │
│                       │                                             │
│                       ▼                                             │
│    ┌──────────────────────────────────────────────┐                │
│    │          6 Domain Agents (Layer 2)            │                │
│    │    (AutoResearchClaw 8-phase pipeline)        │                │
│    └──────────────────┬───────────────────────────┘                │
│                       │                                             │
│                       ▼                                             │
│    ┌──────────────────────────────────────────────┐                │
│    │       Simulation MCP Servers (Layer 1)        │                │
│    │  OpenFOAM · CalculiX · Cantera · CoolProp    │                │
│    └──────────────────┬───────────────────────────┘                │
│                       │                                             │
│                       ▼                                             │
│    ┌──────────────────────────────────────────────┐                │
│    │       LangGraph Orchestrator (Layer 3)        │                │
│    │  Cross-domain deps · Conflict resolution      │                │
│    │  Multi-objective Pareto ranking                │                │
│    └──────────────────┬───────────────────────────┘                │
│                       │                                             │
│                       ▼                                             │
│    ┌──────────────────────────────────────────────┐                │
│    │          RANKED PROPOSALS                      │                │
│    │  1. VG on diffuser (Aero: +8 Cl, 2 ATR-hrs)  │                │
│    │  2. CFRP layup opt (Mat: -0.8kg, 4 FEA-hrs)  │                │
│    │  3. Sidepod inlet  (Therm+Aero: -3 Cd, 6hrs) │                │
│    └──────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Security Model

### Trust Boundaries

| Boundary | Policy |
|:--|:--|
| **Paper APIs** | Whitelisted: OpenAlex, Semantic Scholar, arXiv, SAE, AIAA, PubChem, Materials Project, AFLOW |
| **Simulation execution** | Local only — all CFD/FEA/chemistry sims run on team infrastructure, never cloud |
| **LLM inference** | LiteLLM proxy (dev) → NemoClaw inference router (production) with model-per-task routing |
| **Team CFD data** | Never leaves local filesystem — Qdrant and PostgreSQL run in local Docker containers |
| **Databases** | Local Docker only — Qdrant (vectors), PostgreSQL (relational), no external database connections |
| **MCP servers** | Local-only transport (stdio/SSE on localhost) — no remote MCP connections |

### NemoClaw Sandbox (Phase 5)

```
┌─────────────────────────────────────────┐
│         NemoClaw Sandbox                 │
│                                          │
│  ┌──────────┐    ┌──────────────────┐   │
│  │ Agent    │    │ Allowed network   │   │
│  │ Process  │───▶│ (paper APIs only) │   │
│  └──────────┘    └──────────────────┘   │
│       │                                  │
│       │          ┌──────────────────┐   │
│       └─────────▶│ Allowed filesystem│   │
│                  │ /data/            │   │
│                  │ /simulations/     │   │
│                  │ /results/         │   │
│                  └──────────────────┘   │
│                                          │
│  ✗ No access to: /home, /etc, .env      │
│  ✗ No outbound to: arbitrary URLs       │
│  ✗ No execution of: non-whitelisted bins │
└─────────────────────────────────────────┘
```

### Data Classification

| Data Type | Classification | Storage |
|:--|:--|:--|
| Published papers | Public | Qdrant (indexed) |
| FIA regulations | Public | Qdrant (indexed) |
| Team CFD results | Confidential | Local PostgreSQL + filesystem |
| Telemetry (FastF1) | Public | Cached locally |
| Design proposals | Confidential | Local PostgreSQL |
| ATR budget data | Confidential | Local PostgreSQL |
| API keys | Secret | `.env` (never committed) |
