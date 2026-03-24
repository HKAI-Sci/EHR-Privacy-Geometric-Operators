<div align="center">

# 🏥 Privacy-Preserving EHR Transformation with Mathematical Guarantees

**A Human–AI Co-Designed Solution for Making Clinical Data Available and Visible**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)
[![CPU Only](https://img.shields.io/badge/compute-CPU--only-green.svg)](#)
[![EHR](https://img.shields.io/badge/domain-Clinical%20EHR-red.svg)](#)

*Maolin Wang, Beining Bao, Gan Yuan, Hongyu Chen, Bingkun Zhao, Baoshuo Kan, Jiming Xu, Qi Shi, Yinggong Zhao, Yao Wang, Wei-Ying Ma, Jun Yan*

HKAI-Sci · City University of Hong Kong · YIDU TECH

</div>

---

## Problem

Electronic health records (EHRs) are essential for clinical AI, but **models can move — data cannot**. Regulatory, ethical, and governance constraints keep high-quality clinical data locked inside hospital intranets, creating persistent data silos.

Existing privacy-preserving methods (MPC, HE, DP, federated learning) make data **usable but invisible**: protected data can be computed on, but clinicians and researchers cannot directly inspect cohort-level distributions, perform exploratory data analysis (EDA), or visually validate temporal patterns — activities that are central to real-world clinical research workflows.

**This framework bridges the gap**: it produces transformed numeric views that preserve medical semantics and major statistical properties, while provably breaking linkage to protected patient-level attributes under a clearly specified threat model.

---

## Approach

### Human–AI Co-Design with SciencePal

Rather than letting an AI agent use high-compute tools, we ask it to **design low-compute tools**. The AI role is played by **SciencePal**, which explored the space of possible operators under human-specified constraints. Human researchers define the problem, validate every claim, and conduct all attacks.

The co-design protocol proceeds in five iterative steps:

1. **Humans specify constraints** C1–C4 and threat model C5
2. **SciencePal searches literature** — finds no existing operator satisfying all constraints simultaneously
3. **SciencePal proposes candidates** T1, T2, T3
4. **Humans prove properties and attack** — keep T1/T2, reject T3 (negative case)
5. **Co-design Q-mix extension** — observing residual reconstruction risk, humans and SciencePal design per-stay orthogonal mixing

### Geometric Foundation: The Mean–Variance Manifold

All operators operate in **z-score space** on the manifold $\mathcal{M}(0,1) = \{z \in \mathbb{R}^n : \mu(z) = 0, \sigma^2(z) = 1\}$ — the intersection of a zero-mean hyperplane and a unit-norm sphere. Any transform that starts and ends on this manifold exactly preserves cohort-level means and variances after de-standardization.

A single **unified privacy knob $\alpha$** controls the $\ell_\infty$ perturbation bound in z-score space across all variables, automatically adapting to each variable's intrinsic variability.

---

## Design Constraints

All operators satisfy four hard constraints:

| # | Constraint | What it means |
|---|---|---|
| **C1** | Mean & variance preserved | Downstream statistics unaffected, to machine precision |
| **C2** | Unified $\ell_\infty$ bound $\alpha$ | One privacy knob for all variables — no per-variable tuning |
| **C3** | Full variability | No "silent" time points — virtually all values are moved |
| **C4** | $O(n)$ complexity, CPU-only | Runs on existing hospital ETL infrastructure overnight |

### Threat Model (C5)

- **No-key, structure-aware adversary**: knows operator definitions, pseudocode, and public hyperparameters ($\alpha$, window lengths, Q-mix block sizes)
- No long-term shared cryptographic key; randomness is hospital-internal and ephemeral
- Three leakage levels: **L0** (output only), **L1** (0.01% paired samples), **L2** (20% paired samples)

---

## Core Operators

### T1: Triplet Rotation
Random orthogonal rotation over local 3-point windows in the mean-zero subspace. Preserves short-range autocorrelation while injecting controlled noise.

| Property | Value |
|---|---|
| Reconstruction R² (HR, α=1.0, L2) | ~0.82 |
| Utility (LOS AUROC drop) | < 0.01 |

### T2: Noise + Projection
Gaussian noise added in standardized space, then re-projected onto $\mathcal{M}(0,1)$ by re-centering and re-scaling. Strong preservation of marginals and multivariate correlation.

| Property | Value |
|---|---|
| Reconstruction R² (HR, α=1.0, L2) | ~0.97 |
| Utility (LOS AUROC drop) | < 0.01 |

### T3: Householder Reflection *(Negative Case)*
Global reflection in the mean-zero subspace. Nearly preserves all statistics but is **highly invertible** (R² ≈ 0.9999) — serves as a cautionary example that satisfying C1–C4 does not guarantee privacy.

### Q-mix + T1/T2: The Privacy Cliff
Per-stay orthogonal mixing applied **before** T1/T2. The adversary knows the algorithm but not the per-stay matrix realization.

| Metric | Without Q-mix | With Q-mix |
|---|---|---|
| Reconstruction R² (HR, α=1.0) | 0.83–0.97 | **≈ 0.0** 🔒 |
| Attribute inference R² (max HR) | 0.92–0.98 | 0.08–0.11 |
| LOS AUROC change | — | ±0.01 |
| Shape utility (1 − KS) | — | 0.93+ |

> **Q-mix creates a qualitative privacy step at fixed $\alpha$**: reconstruction collapses to baseline while downstream tasks remain stable.

---

## System: EHR-Privacy-Agent

The framework is implemented as an **in-hospital nightly pipeline** with a three-layer architecture:

```
┌─────────────────────────────────────────────────────┐
│  Skill Layer (YAML)                                 │
│  ┌──────────────────┐  ┌──────────────────────────┐ │
│  │ In-hospital QA   │  │ Export (strong privacy)  │ │
│  │ α=0.5, no Q-mix  │  │ α=1.0, Q-mix on HR/Gluc │ │
│  └──────────────────┘  └──────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│  Module Layer                                        │
│  ColumnTransform · CrossColumn · TemporalSlicing     │
├─────────────────────────────────────────────────────┤
│  Operator Layer                                      │
│  T1 (triplet rotation) · T2 (noise + projection)    │
│  T3 (Householder) · Q-mix (per-stay orthogonal)     │
├─────────────────────────────────────────────────────┤
│  Raw EHR → (never leaves production system)          │
└─────────────────────────────────────────────────────┘
```

- **Skills** are version-controlled YAML configs mapping deployment scenarios to operator configurations
- The **LLM-based agent** interacts only with skill-defined privacy views, never with raw EHR
- **Automated evaluation** runs the full attack protocol on every skill update, detecting privacy/utility regressions

---

## Evaluation Results

### End-to-End Downstream Tasks (MIMIC-IV ICU)

| Task | Raw AUROC | T1+T2 @α=0.5 | CTGAN Synthetic | Gaussian Noise |
|---|---|---|---|---|
| Mortality | — | −0.01 | −0.05–0.08 | −0.03 |
| LOS (3-class) | ~0.70 | −0.01 | −0.05–0.07 | −0.03–0.05 |
| Readmission (30d) | ~0.70 | −0.01 | larger drop | −0.03 |

### Privacy Attack Suite

| Attack | Goal | Metric | Q-mix @α=1.0 |
|---|---|---|---|
| **A** — Reconstruction | Recover original time series | R² | ≈ 0.0 ✅ |
| **B** — Record Linkage | Re-link perturbed → original patients | Re-id@1 | ~random baseline ✅ |
| **C** — Membership Inference | Was this patient in the dataset? | AUC | ~0.5 ✅ |
| **D** — Attribute Inference | Infer sensitive attributes (max HR, p90 flag) | R² | 0.005–0.11 ✅ |

### Compute Efficiency

The geometric pipeline is **1–2 orders of magnitude faster** than CTGAN training + sampling, runs CPU-only, and is trivially amortized across nightly ETL runs.

---

## Scenario-Based Recommendations

| Scenario | Recommended Config | Privacy | Utility |
|---|---|---|---|
| **In-hospital** (research, QA, teaching) | T1+T2 @α=0.5, no Q-mix | Moderate | Near-lossless |
| **Export** (multi-center, external) | T1+T2 @α=1.0, Q-mix on high-risk vars | Strong | < 0.03 AUROC drop |

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
├── paper/                              # 📑 Technical report PDF
├── repo_discovery.py                   #    Runtime path resolution
├── requirements.txt
└── LICENSE
```

---

## Getting Started

### Installation

```bash
git clone https://github.com/HKAI-Sci/EHR-Privacy-Geometric-Operators.git
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

### Quick Start

```bash
# 🎮 Try demos (no real data needed)
cd agent_demo
python demo_numeric_pipeline.py          # operator pipeline on synthetic data
python demo_privacy_attacks_synthetic.py # attack simulation on synthetic data

# 🔬 Run experiments
cd experiments/A2_operator_grid/code
python exp_operators_mimic.py --variables HR Glucose SBP

cd experiments/A3_theory_validation/code
python exp_a3_sanity_check.py

cd experiments/A7_privacy_attacks/code
python exp_a7_privacy.py --variables HR Glucose --K 5

cd experiments/P7_qmix_pilot/code
python run_qmix_pilot.py --variables HR Glucose --alphas 1.0 --seed 42

cd experiments/B4_privacy_utility_tradeoff/code
PYTHONPATH=. python exp_b4_timeline_icu_tasks.py --max-stays 500

# 🔍 Full privacy evaluation protocol
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
<<<<<<< HEAD
### Attack Families

| Attack | Goal | Random Baseline |
|---|---|---|
| **A** — Reconstruction | Recover original time series from perturbed output | R² = 0 |
| **B** — Record Linkage | Re-link perturbed records to original patients | Re-id@1 = 1/m |
| **C** — Membership Inference | Determine if a patient was in the dataset | AUC = 0.5 |
| **D** — Attribute Inference | Infer sensitive attributes (e.g., max HR, above-p90 flag) | R² = 0 |
=======



## License

MIT © 2025 HKAI-Sci, City University of Hong Kong
