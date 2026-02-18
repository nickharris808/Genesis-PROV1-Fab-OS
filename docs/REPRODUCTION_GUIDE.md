# Reproduction Guide

## Genesis PROV 1: Fab OS -- How to Independently Verify Claims

This guide explains how to verify the headline claims in the Fab OS white paper using the materials provided in this repository and, optionally, the full data room.

---

## 1. Quick Verification (This Repository Only)

The self-contained verification script re-derives key metrics from published equations and compares them against canonical reference values.

### Prerequisites

- Python 3.9 or later
- NumPy >= 1.21
- SciPy >= 1.7

### Running the Verification

```bash
cd verification/
pip install -r requirements.txt

# Human-readable output
python verify_claims.py

# Machine-readable JSON output
python verify_claims.py --json
```

### What It Checks

| Check | Method | Expected Result |
|:------|:-------|:----------------|
| Biharmonic plate deflection | Timoshenko analytical formula vs. published FEM result | < 1% error |
| Physics cliff ratio | Stiffness modulation formula at k_azi = 0.99 | Ratio > 2.0x |
| ILC convergence | Simulated gain decay schedule | Monotonic error reduction |
| Material universality | Flexural rigidity computation for 5 materials | All D values physically consistent |
| Mesh convergence | Convergence rate estimation from published data | O(h^2) or better |

Each check prints PASS or FAIL with the computed value, expected value, and tolerance.

---

## 2. Analytical Verification (Paper and Pencil)

The core claims can be verified without running any code.

### Claim: FEM accuracy of 0.18%

The Timoshenko analytical solution for a simply-supported circular plate under uniform thermal curvature is:

```
w_max = (alpha * dT / h) * R^2 / 2
```

For silicon (alpha = 2.6e-6 /K, dT = 0.01 K, h = 775e-6 m, R = 0.15 m):

```
kappa_T = 2.6e-6 * 0.01 / 775e-6 = 3.3548e-5 /m
w_max   = 3.3548e-5 * 0.15^2 / 2  = 3.7742e-7 m = 377.42 nm
```

The FEM solver reports 376.73 nm, giving an error of (377.42 - 376.73) / 377.42 = 0.18%.

### Claim: Physics cliff at k_azi = 0.99

The azimuthal stiffness law is:

```
K(r, theta) = K_0 * [1 + k_azi * cos(n * theta)]
```

At k_azi = 0.99, the stiffness varies from K_0 * 0.01 (near zero) to K_0 * 1.99. The near-zero stiffness in certain azimuthal directions removes structural support, causing the plate to deflect much more in those directions. The 2.44x amplitude ratio is the integrated peak-to-valley increase due to this asymmetric support loss.

### Claim: Material independence

The flexural rigidity for each material is:

```
D = E * h^3 / [12 * (1 - nu^2)]
```

| Material | E (GPa) | nu   | D (N*m) |
|:---------|--------:|:----:|--------:|
| Silicon  | 130.0   | 0.28 | 5.472   |
| Glass    | 73.6    | 0.23 | 2.941   |
| InP      | 61.0    | 0.36 | 2.255   |
| GaAs     | 85.5    | 0.31 | 3.371   |
| SiC      | 410.0   | 0.14 | 16.701  |

Despite a 7x range in flexural rigidity, all materials exhibit the cliff because it is driven by the stiffness distribution geometry, not the material stiffness itself.

---

## 3. Full Data Room Verification

With access to the complete data room, the following comprehensive verifications are available.

### One-Command Full Verification

```bash
python3 04_Source_Code/due_diligence_runner.py
```

This runs all verification checks and produces a complete audit report.

### Individual Component Verification

```bash
# FEM solver accuracy (Timoshenko comparison)
python3 04_Source_Code/fem_wafer_solver.py

# ILC controller benchmark (90.8% / 96.5%)
python3 04_Source_Code/robust_ilc_controller.py

# Physics cliff (5 materials, 15 k_azi values, 100 MC trials each)
python3 04_Source_Code/verify_glass_cliff.py

# 315-case batch validation
python3 04_Source_Code/batch_validation_suite.py

# ASML SECS/GEM protocol tests (52/52)
python3 04_Source_Code/asml_scanner.py

# CalculiX validation (60 cases)
python3 04_Source_Code/run_calculix_validation.py

# ABAQUS comparison (45 cases)
python3 04_Source_Code/abaqus_comparison.py

# Production floor simulation (1,250 wafers)
python3 04_Source_Code/production_floor_validation.py
```

### CalculiX Independent Verification

If you have CalculiX installed, you can run the generated input decks directly:

```bash
# Generate 60 validation cases
python3 04_Source_Code/run_calculix_validation.py

# Run a specific case with CalculiX
cd 03_Simulation_Data/calculix_validation/
ccx case_silicon_dT0.01_k0.0
```

### ABAQUS Independent Verification

If you have ABAQUS installed, you can run the generated input decks:

```bash
# Generate 45 comparison cases
python3 04_Source_Code/abaqus_comparison.py

# Run a specific case with ABAQUS
cd 03_Simulation_Data/abaqus_comparison/
abaqus job=case_silicon_dT0.01_k0.0_fine
```

---

## 4. Key Data Files for Verification

| File | Contents |
|:-----|:---------|
| `03_Simulation_Data/convergence_proof.json` | 10-level mesh convergence with Richardson extrapolation |
| `03_Simulation_Data/dense_cliff_data.json` | Physics cliff data: 5 materials, 15 k_azi values, 100 MC trials |
| `03_Simulation_Data/batch_results_v2.json` | Full 315-case batch results |
| `03_Simulation_Data/statistical_robustness_report.json` | ILC confidence intervals (p10/p50/p90) |
| `05_Models/rom_v3_metrics.json` | ROM cross-validation metrics |
| `05_Models/MODEL_TRAINING_MANIFEST.json` | Full model lineage (39 models with SHA-256 hashes) |

---

## 5. Expected Verification Outcomes

A successful verification should confirm:

1. **FEM accuracy:** w_max within 0.2% of Timoshenko analytical solution for all tested temperatures
2. **Physics cliff:** Warpage ratio > 2.0x at k_azi = 0.99 for all 5 materials
3. **ILC convergence:** Monotonically decreasing error with > 85% reduction in default config
4. **Batch robustness:** All 315 cases achieve > 50% reduction with 0 failures
5. **ROM accuracy:** Cross-validated R^2 > 0.99
6. **Mesh convergence:** Error decreasing monotonically with mesh refinement at approximately O(h^2) rate

If any check fails, please report the specific failure to enable investigation. Contact: [nmk.ai/contact](https://nmk.ai/contact)

---

*This guide enables independent verification at three levels: analytical (paper), computational (this repo's verify_claims.py), and comprehensive (full data room). Each level provides increasing confidence in the headline claims.*
