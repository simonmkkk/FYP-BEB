# Full Duplex MAC Protocol Reproduction – Documentation Outline

## 1. Executive Summary

- **Project objective:** Recreate the analytical and simulation results from *"Performance Analysis of a Full Duplex MAC Protocol with Binary Exponential Backoff"* using a modern Python toolchain.
- **Scope of work:** Implement the fixed-point analytical model, develop a discrete-event simulator capturing the full-duplex MAC with BEB, and reproduce key figures/tables for performance metrics (request failure probability, mean delay, delay variance, parameter sweeps).
- **Current status:** Documentation scaffold established; FYP paper reviewed; design phase in progress prior to coding. No Python modules implemented yet—this document guides subsequent development milestones.

## 2. Literature Context & Key Assumptions

- **Paper highlights:** Proposes a TDMA-based full-duplex MAC protocol leveraging binary exponential backoff, derives closed-form expressions for request failure probability and packet delay statistics, and validates the model with C++ discrete-event simulations.
- **Key assumptions:**
  - Saturated uplink traffic for all user equipment (UE); each UE always has packets to send.
  - Frames partitioned into request, full-duplex data, control (information), and downlink subframes with fixed slot counts (m, d, b) and durations ($t$, $T$).
  - BEB parameters: minimum contention window $W$, exponential factor $\lambda=2$, retry thresholds $r$ (window growth cap) and $R$ (packet drop limit).
  - Symmetric duplex traffic assumed for analytical tractability; downlink subframe included for asymmetry studies.
- **Assumptions to revisit:**
  - Hardware self-interference cancellation model not simulated; Python effort will focus on MAC-layer effects.
  - Traffic symmetry can be relaxed in simulation if needed; analytical derivations remain under saturated traffic.
  - Time slot quantization (mini-slot definition) to follow paper defaults unless system-level requirements dictate adjustments.

## 3. System Overview

- **Planned repository layout:**
  - `src/analytics/` – analytical model implementation (fixed-point solver, delay metrics, plotting helpers).
  - `src/simulation/` – SimPy-based discrete-event simulator, parameter loaders, metrics exporters.
  - `notebooks/` – exploratory analysis, reproduction of figures, comparison studies.
  - `tests/` – unit and regression tests for analytics and simulation components.
  - `configs/` – YAML/JSON parameter sets mirroring paper scenarios.
- **Detailed structure:**

```text
FYP-BEB/
├── README.md                      # High-level project introduction (to be created)
├── docs/
│   ├── README.md                  # Project manual (this file)
│   ├── config-reference.md        # Schema & field descriptions for configuration files
│   └── methodology-notes.md       # Supplemental derivations, research notes
├── configs/
│   ├── baseline.yaml              # Primary scenario matching paper defaults (m=d=12, b=12)
│   ├── sensitivity/
│   │   ├── n_sweep.yaml           # UE count sweep definitions
│   │   ├── d_variants.yaml        # Data-slot availability experiments
│   │   └── downlink_sweep.yaml    # Downlink subframe variations
│   └── templates/
│       └── config.schema.json     # JSON Schema for validation tooling
├── src/
│   ├── __init__.py
│   ├── analytics/
│   │   ├── __init__.py
│   │   ├── parameters.py          # Dataclasses for frame/BEB parameters, validation helpers
│   │   ├── fixed_point.py         # Core equations for p, p_c, p_d; iterative solvers
│   │   ├── delay.py               # Delay statistics (E[X], Var[X]) calculations
│   │   ├── distributions.py       # Utility functions for Q(j), q(j) computations
│   │   └── plots.py               # Matplotlib helpers for analytical figures
│   ├── simulation/
│   │   ├── __init__.py
│   │   ├── entities.py            # UE/BS classes with state machines
│   │   ├── events.py              # Frame transitions, request handling, slot assignment
│   │   ├── metrics.py             # Observers collecting delays, collisions, utilization
│   │   ├── runner.py              # SimPy environment orchestration
│   │   └── seeds.py               # RNG utilities ensuring reproducibility
│   ├── experiments/
│   │   ├── __init__.py
│   │   ├── run.py                 # CLI entry point orchestrating batch experiments
│   │   ├── reporting.py           # Aggregation of results, MAPE calculations
│   │   └── exporters.py           # Persist artefacts (CSV, JSON, LaTeX tables)
│   └── cli.py                     # Typer-based unified command surface
├── notebooks/
│   ├── analysis.ipynb             # Analytical vs simulation comparison, figure reproduction
│   ├── sensitivity.ipynb          # Parameter sweep exploration
│   └── sandbox.ipynb              # Free-form experimentation
├── tests/
│   ├── __init__.py
│   ├── test_analytics.py          # Unit tests for analytical computations
│   ├── test_fixed_point.py        # Edge cases & convergence scenarios
│   ├── test_simulation_entities.py# UE/BS state machine tests
│   ├── test_simulation_runner.py  # Integration tests of simulation pipeline
│   └── test_experiments.py        # Regression tests for batch execution outputs
├── reports/
│   ├── figures/                   # Generated plots (PNG, PDF) with metadata
│   │   └── README.md              # Documentation of figure naming conventions
│   ├── simulation/                # Raw simulation outputs, CSV/Parquet
│   └── analytics/                 # Analytical output snapshots
├── scripts/
│   ├── bootstrap_env.ps1          # Optional Windows bootstrap for virtual environment
│   ├── bootstrap_env.sh           # POSIX bootstrap script
│   └── regenerate_figures.py      # Automation helper to rebuild reported figures
├── pyproject.toml                 # Project metadata, dependency specifications
├── requirements.txt               # Frozen dependency list (if using pip)
├── uv.lock / Pipfile.lock         # Environment lock file (tool-dependent)
├── .pre-commit-config.yaml        # Linting/formatting hooks configuration
├── .github/workflows/ci.yml       # CI pipeline running tests and linting
└── .gitignore                     # Ignored files and directories
```

- **Structure notes:**
  - `pyproject.toml` preferred for dependency management; retain `requirements.txt` for compatibility.
  - Use lock file matching chosen tool (`uv`, `pipenv`, or `poetry`).
  - `reports/` contents should be reproducible; document provenance in accompanying README files.
  - Automation scripts under `scripts/` should be cross-platform where feasible.
- **Core dependencies:** Python 3.11+, `numpy`, `scipy`, `simpy`, `pandas`, `matplotlib`, `seaborn`, `typer` (CLI scaffolding), `pytest`.
- **Environment setup:** Recommend `uv` or `pipenv` for deterministic environments; alternatively, provide `requirements.txt`. Initial steps: clone repository, create virtual environment, install dependencies (`pip install -r requirements.txt`), run smoke tests once added.
- **Documentation tooling:** Continue expanding this `docs/README.md`; consider MkDocs/Sphinx if narrative grows beyond single file.

## 4. Analytical Model

- **Key formulations:**
  - Fixed-point system involving request collision probability $p_c$, data-slot shortage probability $p_d$, and overall failure probability $p$ per equations (1), (3), (5) of the paper.
  - Average backoff duration $B_{avg}$ and attempt probability $\tau$ leading to truncated binomial distribution $Q(j)$ for successful requests.
  - Delay random variables $U$, $V$, and $Y^{(i)}$ composing packet delay moments $E[X]$, $Var[X]$.
- **Notation table:** Document symbols ($m$, $d$, $b$, $t$, $T$, $W$, $\lambda$, $r$, $R$, $N$, $p$, $p_c$, $p_d$, $\tau$) with descriptions and units to ensure Python modules remain traceable to the paper.
- **Implementation plan:**
  - `analytics/parameters.py`: Dataclasses encapsulating frame configuration and BEB parameters; validation helpers.
  - `analytics/fixed_point.py`: Functions to compute $B_{avg}$, $\tau$, $p_c$, $p_d$, and solve for $p$ via iterative methods (e.g., Newton-Raphson fallback to bisection). Include convergence tolerance, iteration cap, diagnostic logging.
  - `analytics/delay.py`: Derive $E[X]$ and $Var[X]` using computed probabilities and slot distributions; provide vectorized evaluation across parameter grids.
  - `analytics/plots.py`: Utilities to plot probability curves and delay metrics mirroring Figures 3–5.
- **Validation methodology:**
  - Unit tests comparing against closed-form sanity checks (e.g., low-load approximation where $p \approx 0$).
  - Reproduce reference scenarios (m = d = 12, N sweep) to ensure curves align with paper within acceptable tolerance.
  - Provide regression fixtures storing reference outputs for automated verification.
- **Artefact expectations:** Tables/plots for $p$ vs. $N$, mean/Std delay vs. $N$, impact of varying $b$; LaTeX-ready exports.

## 5. Discrete-Event Simulation

- **Simulation design:**
  - Use SimPy environment with processes for each UE and the base station (BS). Frame clock governs request, duplex data, information, and downlink subframes.
  - UE process: manages BEB state machine (backoff counter, retry count), submits requests in chosen slots, handles success/failure notifications, schedules data transmissions, records packet delays.
  - BS process: collects requests per frame, resolves collisions, assigns data slots, publishes FD/DL maps, and serves downlink transmissions.
- **Data structures:**
  - `simulation/entities.py`: UE and BS classes encapsulating state (contention window, queue, assigned slots).
  - `simulation/events.py`: Event helper functions for frame transitions, request handling, data slot allocation.
  - `simulation/metrics.py`: Observers capturing per-packet delays, request outcomes, slot utilization; exports to Pandas DataFrame.
- **Configuration & randomization:** Seeded RNG for reproducibility; parameter loading from `configs/*.yaml` to align with analytical scenarios.
- **Performance considerations:** Batch metrics aggregation to minimize per-event overhead; optional progress logging for long simulations.
- **Cross-validation:**
  - Compare empirical $p_c$, $p_d$, mean delay against analytical outputs for identical parameter sets.
  - Conduct sensitivity analysis to quantify discrepancies; document causes (finite run time, variance).
  - Establish acceptance thresholds (e.g., relative error < 5% over benchmark runs).

## 6. Experiments & Parameter Studies

- **Experiment matrix:**
  - Baseline: $m = d = 12$, $b = 12$, $N \in [5, 40]$ (step 5) to replicate Figures 3–4.
  - Data-slot availability study: fix $N=20$, vary $d \in \{8, 10, 12\}$ to observe delay sensitivity.
  - Downlink subframe sweep: $b \in \{2, 4, 8, 12\}$ with near-symmetric traffic scenarios to mirror Figure 5.
  - Contention window impact: vary $W \in \{4, 8, 16\}$, $r \in \{2, 3, 4\}$ to test stability boundaries.
- **Methodology:**
  - For each parameter set, compute analytical metrics and run multiple simulation seeds (e.g., 5 runs × 5,000 s) to gather confidence intervals.
  - Use consistent random seeds across sweeps for comparative fairness.
  - Automate batch execution via CLI (`python -m experiments.run --config ...`).
- **Results presentation:**
  - Overlay analytical and simulation curves with error bars.
  - Summaries in tables capturing mean absolute percentage error (MAPE) between models.
  - Store figures under `reports/figures/` with metadata JSON for reproducibility.
- **Insights to capture:**
  - Breakpoints where BEB saturation causes sharp delay increases.
  - Effectiveness of full duplex on reducing delay versus hypothetical half-duplex baseline (optional extension).
  - Identify parameter regimes where analytical model diverges, document hypotheses.

## 7. Validation & Testing

- **Testing layers:**
  - Unit tests for analytical computations (e.g., verifying $B_{avg}$, $\tau$, fixed-point solver convergence, delay moments) with deterministic fixtures.
  - Unit tests for simulation entities (UE/BS state transitions) using short deterministic scenarios.
  - Integration tests comparing analytical vs. simulation metrics for small networks (e.g., $N=5$, short run) to ensure coherence.
- **Automation:**
  - Configure `pytest` with coverage thresholds (e.g., 85%).
  - Optional `pre-commit` hooks to run linting (`ruff`, `black`) and unit tests on staged files.
  - CI pipeline (GitHub Actions or similar) executing full analytical calculations and short simulations; cache heavy artefacts where possible.
- **Reproducibility controls:** Fixed RNG seeds exposed via CLI/Config; document expected variance bounds.
- **Acceptance criteria:** Analytical vs. simulation MAE < 5% for benchmark scenarios; unit tests pass; documentation updated alongside code changes.

## 8. Usage Guide

- **Command-line workflows:**
  - `python -m analytics.compute --config configs/baseline.yaml` to output request probabilities and delay stats (CSV/JSON).
  - `python -m simulation.run --config configs/baseline.yaml --seed 42` to execute simulations and store metrics under `reports/simulation/`.
- **Notebook workflows:**
  - `notebooks/analysis.ipynb` for interactive comparisons and figure generation.
  - `notebooks/sensitivity.ipynb` for parameter sweeps and plotting utilities.
- **Plot generation:** Scripts/notebooks populate `reports/figures/`; specify DPI, labels, and save raw data for LaTeX integration.
- **Configuration examples:** Provide starter YAML with frame parameters and toggles (e.g., enable downlink subframe). Document required fields in `docs/config-reference.md` (future item).

## 9. Future Work & Open Questions

- **Planned enhancements:**
  - Implement half-duplex baseline for comparative analysis.
  - Extend simulator to heterogeneous traffic loads (non-saturated UEs, arrival processes).
  - Introduce adaptive slot allocation policies and study impact.
- **Open research questions:**
  - Analytical handling of asymmetric traffic with dynamic downlink subframe sizing.
  - Incorporating physical-layer self-interference models into delay analysis.
  - Scalability limits for large-cell deployments (N > 100) with distributed contention.
- **Tooling gaps:** Need robust logging pipeline, potential migration to structured configs (Pydantic) for validation, optional GPU acceleration for large Monte Carlo runs.

## 10. Appendix

- **Symbol glossary:** Table summarizing notation with domain units, referencing equation numbers.
- **Derivations:** Step-by-step calculation bridging equations (1)–(15), including intermediate expectations and variances.
- **Supplementary material:** Links to raw simulation datasets, LaTeX figure templates, and any auxiliary scripts.
