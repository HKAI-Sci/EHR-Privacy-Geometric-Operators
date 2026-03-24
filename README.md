<div align="center">

# 🏥 EHR Privacy Geometric Operators

**Privacy-preserving transformation of EHR time series via geometric operators on the mean–variance manifold.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)
[![CPU Only](https://img.shields.io/badge/compute-CPU--only-green.svg)](#)
[![EHR](https://img.shields.io/badge/domain-Clinical%20EHR-red.svg)](#)

</div>

---

## What is this?

EHR data rarely leaves the hospital — yet clinical AI teams need to share, explore, and model it. This framework provides **geometric operators** that transform EHR time series to be:

- ✅ **Visible** — still looks like real clinical data; histograms, correlations, and temporal patterns are preserved
- ✅ **Usable** — downstream models (mortality, LOS, readmission) lose < 1–2% AUROC
- ✅ **Hard to reconstruct** — pointwise inversion is substantially degraded, especially with Q-mixing

All operations run **CPU-only**, with no long-term shared key, making nightly hospital ETL deployment practical.

---

## Core Operators

<table>
<tr>
<th>Operator</th>
<th>Mechanism</th>
<th>Reconstruction R²<br><small>(HR, α=1.0, L2 attacker)</small></th>
<th>Utility</th>
</tr>
<tr>
<td><b>T1</b> — Triplet Rotation</td>
<td>Random orthogonal rotation over local 3-point windows</td>
<td align="center">~0.83</td>
<td align="center">⭐⭐⭐⭐⭐</td>
</tr>
<tr>
<td><b>T2</b> — Noise + Projection</td>
<td>Gaussian noise re-projected onto mean–variance manifold</td>
<td align="center">~0.97</td>
<td align="center">⭐⭐⭐⭐⭐</td>
</tr>
<tr>
<td><b>T3</b> — Householder <em>(negative case)</em></td>
<td>Global reflection in mean-zero subspace</td>
<td align="center">~0.9999 ❌</td>
<td align="center">⭐⭐⭐⭐⭐</td>
</tr>
<tr>
<td><b>Q-mix</b> + T1/T2</td>
<td>Per-stay orthogonal mixing before T1/T2</td>
<td align="center"><b>~0.000</b> 🔒</td>
<td align="center">⭐⭐⭐⭐</td>
</tr>
</table>

> **Q-mix creates a privacy cliff**: at fixed α=1.0, reconstruction R² collapses from ~0.8–0.97 down to ~0 — while downstream LOS prediction AUROC remains stable.

---

## Design Principles

All operators operate in z-score space and satisfy four hard constraints:

| # | Constraint | What it means |
|---|---|---|
| C1 | Mean & variance preserved | Downstream statistics unaffected, to machine precision |
| C2 | Unified ℓ∞ bound α | One privacy knob for all variables — no per-variable tuning |
| C3 | Full variability | No "silent" time points — virtually all values are moved |
| C4 | O(n) complexity, CPU-only | Runs on existing hospital ETL infrastructure overnight |

---

## Repository Structure

```
EHR-Privacy-Geometric-Operators/
│
├── ehr_privacy/                        # 📦 Core framework
│   ├── numeric_operators.py            #    T1, T2, T3, Q-mix
│   ├── non_numeric_operators.py        #    Text, categorical, ID operators
│   └── agent.py                        #    PrivacyAgent + SkillRegistry
│
├── agent_demo/                         # 🎮 Runnable demos (no real data needed)
│
├── privacy_evaluation_protocol/        # 🔍 Attack suite (A/B/C/D)
│   └── code/run_privacy_protocol.py
│
├── experiments/                        # 🧪 All experiments
│   ├── A2_operator_grid/               #    Operator application & parameter grid
│   ├── A3_theory_validation/           #    C1–C4 constraint verification
│   ├── A4_single_column_distribution/  #    KS / marginal distribution analysis
│   ├── A5_multivariate_correlation/    #    Correlation structure preservation
│   ├── A6_temporal_structure/          #    ACF / spectral analysis
│   ├── A7_privacy_attacks/             #    Reconstruction & privacy attacks
│   ├── B2_privacy_utility/             #    Privacy grid search & pipeline planning
│   ├── B3_theory_e2e/                  #    End-to-end theory constraint checks
│   ├── B4_privacy_utility_tradeoff/    #    Downstream tasks (LOS / mortality / readmission)
│   ├── B5_complexity_compute/          #    Runtime & scaling benchmarks
│   ├── B6_agent_ablation/              #    Operator & agent ablation
│   ├── P3_weak_interpolation/          #    Weak interpolation pipeline
│   ├── P6_alpha_hierarchy/             #    α-hierarchy Pareto plots
│   └── P7_qmix_pilot/                  #    Q-mix privacy cliff experiments
│
├── docs/                               # 📄 Documentation
├── repo_discovery.py                   #    Runtime path resolution
├── requirements.txt
└── LICENSE
```

---

## Getting Started

### Installation

```bash
git clone https://github.com/MorinClaw/EHR-Privacy-Geometric-Operators.git
cd EHR-Privacy-Geometric-Operators
pip install -r requirements.txt
```

```bash
# Optional: CTGAN baseline and gradient boosting
pip install ctgan xgboost
```

### Data Setup

Place MIMIC-IV-style preprocessed data under `experiment_extracted/`, anywhere on disk:

```
data_preparation/
└── experiment_extracted/
    ├── ts_48h/
    │   ├── ts_single_column_HR_zscore.csv
    │   ├── ts_single_column_Glucose.csv
    │   └── ...
    ├── patient_profile.csv
    ├── timeline_events.csv
    └── cohort_icu_stays.csv
```

`repo_discovery.py` resolves all paths at runtime — no hardcoded paths anywhere.

---

## Quick Start

### 🎮 Try demos (no real data needed)

```bash
cd agent_demo
python demo_numeric_pipeline.py          # operator pipeline on synthetic data
python demo_privacy_attacks_synthetic.py # attack simulation on synthetic data
```

### 🔬 Run experiments

```bash
# Operator grid (A2)
cd experiments/A2_operator_grid/code
python exp_operators_mimic.py --variables HR Glucose SBP

# Theory validation: verify C1–C4 on real data (A3)
cd experiments/A3_theory_validation/code
python exp_a3_sanity_check.py

# Privacy attacks: reconstruction R², multi-run variability (A7)
cd experiments/A7_privacy_attacks/code
python exp_a7_privacy.py --variables HR Glucose --K 5

# Q-mix privacy cliff (P7)
cd experiments/P7_qmix_pilot/code
python run_qmix_pilot.py --variables HR Glucose --alphas 1.0 --seed 42

# Downstream tasks: LOS / mortality / readmission (B4)
PYTHONPATH=. python experiments/B4_privacy_utility_tradeoff/code/exp_b4_timeline_icu_tasks.py \
  --max-stays 500
```

### 🔍 Full privacy evaluation protocol

```bash
cd privacy_evaluation_protocol/code
python run_privacy_protocol.py \
  --data-dir ../../data_preparation/experiment_extracted/ts_48h \
  --perturbed-dir ../../experiments/A2_operator_grid/results/perturbed \
  --attack A B C D \
  --variables HR Glucose \
  --alpha 1.0 \
  --seed 42
```

---
### Attack Families

| Attack | Goal | Random Baseline |
|---|---|---|
| **A** — Reconstruction | Recover original time series from perturbed output | R² = 0 |
| **B** — Record Linkage | Re-link perturbed records to original patients | Re-id@1 = 1/m |
| **C** — Membership Inference | Determine if a patient was in the dataset | AUC = 0.5 |
| **D** — Attribute Inference | Infer sensitive attributes (e.g., max HR, above-p90 flag) | R² = 0 |

---

## License

MIT © 2025 HKAI-Sci, City University of Hong Kong
