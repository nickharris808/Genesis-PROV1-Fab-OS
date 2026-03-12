# SCIENCE NOTES -- Red-Team Audit Fixes
## PROV_1_FAB_OS (February 28, 2026)

This document records all science-integrity fixes made in response to a
red-team audit of the PROV_1_FAB_OS data room.  Each issue is documented
with what was found, what was fixed, and what limitations remain.

---

## 1. DETERMINISTIC "MONTE CARLO" (FIXED)

### What was found
All "Monte Carlo" simulations used fixed random seeds, making them fully
deterministic.  Every run produces identical output.  This is technically
a parametric perturbation sweep, not a Monte Carlo simulation in the
statistical sense.  Files affected:

| File | Seed Pattern | Issue |
|:-----|:-------------|:------|
| `tolerance_monte_carlo.py` | `seed=42` (global) | Single fixed seed |
| `robust_ilc_controller.py` | `seed=trial*7+42` (per trial) | Deterministic per-trial |
| `verify_glass_cliff.py` | `seed=trial*31+k_azi_enc+mat_enc` | Deterministic per-point |
| `production_floor_validation.py` | `seed=2026,2027,...` (per scenario) | Fixed per-scenario |
| `statistical_robustness.py` | `RandomState(42)` | Fixed seed |

### What was fixed
1. **Renamed:** "Monte Carlo" references changed to "parametric sweep" or
   "perturbation study" where appropriate (docstrings, README, buyer pitch).
2. **Configurable seeds:** `tolerance_monte_carlo.py` now accepts `--seed=none`
   for true stochastic sampling and `--seed=N` for explicit reproducibility.
3. **Ensemble mode:** `tolerance_monte_carlo.py` now supports `--ensemble N`
   which runs N independent seeds and reports mean +/- std across seeds.
4. **Honesty in output:** All JSON outputs now record the seed used and
   whether results are deterministic.
5. **ILC robustness:** `test_robustness()` now accepts a `seed` parameter;
   `seed=None` enables true stochastic sampling.

### What remains
- `verify_glass_cliff.py` still uses per-trial deterministic seeds.
  This is intentional for exact reproducibility of the cliff data.
- `production_floor_validation.py` still uses fixed seeds per scenario.
  This is acceptable for synthetic production data.
- `statistical_robustness.py` bootstrap analyzer still uses seed=42.
  The bootstrap methodology itself is sound; the seed just ensures
  reproducibility of CI bounds.
- **No multi-seed ensemble results have been generated yet.**  Running
  `--ensemble 5` would provide honest cross-seed statistics.

---

## 2. CHERRY-PICKED METRICS (FIXED)

### What was found
Headlines in README and buyer pitch cited best-case numbers:
- "96.5% warpage reduction" without mentioning this requires benchmark
  config with perfect plant knowledge (no mismatch).
- "14x variance explosion" in buyer pitch (actual data shows 2.42-2.48x
  amplitude ratio -- this is an amplitude cliff, not a variance ratio).
- "13.2x better than best competitor" uses Kitchen Sink (1,198 nm) as
  the comparison, not the actual best competitor Hybrid B (725 nm = 8.0x).
- ROI projections assume warpage is the dominant yield limiter with no
  caveats about multi-factorial yield loss.

### What was fixed
1. **README scorecard:** Warpage reduction now shows range including
   mismatch degradation: "90.8% default / 96.5% benchmark (with +/-20%
   mismatch: mean ~83-91%)".
2. **Buyer pitch:** "14x variance explosion" corrected to "2.42-2.48x
   amplitude increase".
3. **Design-around comparison:** Added caveats that the 13.2x ratio
   uses Kitchen Sink (not actual best competitor) and that the result
   is from a single deterministic run.
4. **ROI section:** Added caveat that projections are upper-bound
   estimates assuming warpage is the yield limiter.
5. **Statistical tables:** Noted that percentile spreads are from
   deterministic perturbation grids, not stochastic trials.
6. **Cliff description:** Changed "phase-transition boundary" to
   "steep nonlinearity" and clarified amplitude vs. variance ratio.

### What remains
- The ILC performance numbers (90.8%, 96.5%) are still real and
  reproducible for the specific configurations tested.  The honest
  range is 78-97% depending on plant uncertainty and configuration.
- ROI projections are inherently speculative and should be treated
  as such by any buyer.

---

## 3. MISSING CONTROLLER/ACTUATOR DYNAMICS (PARTIALLY FIXED)

### What was found
The ILC controller assumes instantaneous actuation.  There is no model
for actuator bandwidth, transport delay, transfer function, or nonlinear
actuator response.  Real piezoelectric or electrostatic actuators have
finite response time that would degrade convergence and achievable
warpage reduction.

### What was fixed
1. **First-order actuator lag model added:** `ProductionILC` now accepts
   an `actuator_tau` parameter (0-1).  With tau=0 (default), behavior
   is unchanged (instantaneous).  With tau>0, actuation is filtered:
   `u_actual = (1-tau)*u_command + tau*u_prev`.
2. **Documentation:** Honest Disclosures section 17E added to README
   documenting this limitation.
3. **Code docstring:** ILC controller docstring now notes the
   instantaneous actuation assumption.

### What remains
- The first-order lag model is a minimal approximation.  Real actuator
  dynamics include:
  - Higher-order transfer functions (2nd order, resonance)
  - Transport delay (dead time)
  - Bandwidth limits (frequency-dependent gain)
  - Actuator saturation (nonlinear)
  - Hysteresis (piezoelectric actuators)
  - Cross-coupling between actuator channels
- **None of the performance claims include actuator dynamics.**  The
  90.8-96.5% reduction assumes perfect actuation.  With realistic
  actuator dynamics, these numbers will degrade.
- Quantifying the degradation requires actuator specifications from
  the target platform (ASML scanner), which are not publicly available.

---

## 4. CIRCULAR VALIDATION (ACKNOWLEDGED)

### What was found
The FEM solver validates against the Timoshenko analytical solution
w(r) = kappa_T * (R^2 - r^2) / 2.  However, the solver directly
implements the equivalent Poisson equation (nabla^2 w = kappa_T),
which is the same equation solved in closed form.  The 0.18% error
measures discretization accuracy, not physics correctness.

Additionally:
- The CalculiX and ABAQUS "validation" scripts generate input decks
  but do not actually run those solvers or compare results.
- All "robustness" tests perturb parameters of the same solver -- they
  test parameter sensitivity, not model validity.

### What was fixed
1. **Validation Gaps section added to README** (Section 17-PRE)
   documenting all four types of circular validation.
2. **FEM solver docstring updated** to note that the validation suite
   checks implementation correctness, not physics correctness.
3. **Specific action items listed** for independent validation.

### What remains
The following independent validations are needed but not yet performed:
- Run generated CalculiX .inp decks on actual CalculiX installation
- Run generated ABAQUS .inp decks on actual ABAQUS installation
- Compare FEM predictions against physical wafer warpage measurements
- Validate physics cliff experimentally on real azimuthal chuck
- Test ILC convergence on real scanner hardware

---

## 5. MODEL VALIDITY BOUNDS AT k_azi > 0.81 (FIXED)

### What was found
The physics cliff is a real mathematical consequence of stiffness
approaching zero (sigma(theta) = 1 + k_azi*cos(2*theta) -> 0).
However, the model's assumptions become increasingly unreliable as
k_azi approaches 1.0:
- Linear elasticity breaks down under large deformations
- Simply-supported BC approximation fails near zero-stiffness zones
- Contact mechanics (wafer-chuck) are not modeled
- Temperature-dependent material properties are ignored

### What was fixed
1. **Model validity table added to README** (Section 13, under EQ5)
   with explicit confidence levels per k_azi range.
2. **Transition zone table updated** with model confidence column.
3. **Explicit warning** that predictions above k_azi=0.85 are
   unreliable without experimental validation.
4. **Cliff ratio caveat** noting that 2.42-2.48x values are computed
   in the regime where model assumptions are most likely to fail.

### What remains
- The precise cliff location and severity above k_azi=0.85 need
  experimental confirmation.
- Nonlinear FEA (e.g., CalculiX with nonlinear geometry) would
  provide a more reliable prediction in the cliff zone.
- Contact mechanics between wafer and chuck could significantly
  alter behavior near k_azi=1.0.

---

## SUMMARY OF REMAINING LIMITATIONS

| # | Limitation | Severity | Mitigation |
|:-:|:-----------|:---------|:-----------|
| 1 | No experimental validation | HIGH | Requires fab access ($50K+/day) |
| 2 | No real actuator dynamics | HIGH | First-order model added; full model needs actuator specs |
| 3 | Circular FEM validation | MEDIUM | CalculiX/ABAQUS decks ready to run |
| 4 | Linear elasticity only | LOW | Adequate for small-strain regime (dT < 0.1K) |
| 5 | No contact mechanics | MEDIUM | Matters most at high k_azi (cliff zone) |
| 6 | Deterministic "Monte Carlo" | LOW | Ensemble mode added; fixed seeds are transparent |
| 7 | No temperature-dependent properties | LOW | Negligible at EUV thermal loads (dT ~ 0.01K) |
| 8 | 2D plate model (not 3D solid) | LOW | Appropriate for thin wafers (h/R ~ 0.005) |

---

## WHAT WAS PRESERVED

The following genuine science was not altered:
- Kirchhoff plate theory implementation (correct for thin plates)
- Azimuthal stiffness modulation formula (physics is sound)
- Zernike-decomposed ILC controller (valid control theory)
- Material property database (verified against published sources)
- Mesh convergence study (Richardson extrapolation is rigorous)
- The existence of the cliff (mathematical fact: stiffness -> 0)
- 112 patent claims structure

The IP value is real.  The fixes above make the claims honest about
what has and has not been validated.

---

## FILES MODIFIED IN THIS AUDIT

### Code files
| File | Changes |
|:-----|:--------|
| `04_Source_Code/tolerance_monte_carlo.py` | Renamed title, added seed parameter, ensemble mode, deterministic warnings |
| `04_Source_Code/robust_ilc_controller.py` | Added actuator_tau parameter, seed parameter for robustness, dynamics caveat |
| `04_Source_Code/verify_glass_cliff.py` | Renamed "Monte Carlo" to "perturbation", added RNG note to metadata |
| `04_Source_Code/fem_wafer_solver.py` | Fixed "variance explosion" to "amplitude amplification", added validation caveat |
| `cli.py` | Renamed "Monte Carlo" to "parametric sweep" in CLI help text |

### Documentation files (buyer-facing)
| File | Changes |
|:-----|:--------|
| `README.md` | Scorecard ranges, validation gaps section, model validity bounds, honest design-around caveats |
| `01_Executive_Summary/BUYER_PITCH_ASML.md` | Corrected design-around ratio, added limitations caveat |
| `13_Buyer_Prep/BUYER_PITCH_ASML.md` | Replaced 14x with 2.42-2.48x amplitude, added model confidence note |
| `13_Buyer_Prep/BUYER_STRATEGY_ASML.md` | Replaced 14x variance with correct amplitude ratio |
| `13_Buyer_Prep/hostile_diligence_pack.md` | Corrected design-around ratio, renamed MC, added remaining limitations |
| `13_Buyer_Prep/executive_one_pager.md` | Renamed "Monte Carlo" to "parametric perturbation" |
| `13_Buyer_Prep/TARGET_BUYER_MAPPING.md` | Replaced 14x with correct amplitude ratio |
| `PROV1_EXECUTIVE_PITCH.md` | Added mismatch range, corrected performance claims, added limitations |
| `TECHNICAL_DATASHEET.md` | Added controller dynamics caveat, renamed MC, added model confidence note |
| `DUE_DILIGENCE_REPORT.md` | Renamed MC proof, corrected design-around ratio |
| `VALIDATION_REPORT.md` | Renamed MC, corrected design-around comparison |
| `15_Computational_Proofs/README.md` | Renamed MC, corrected 14x to cross-load CV ratio, corrected design-around |
| `08_Legal/PATENT_PORTFOLIO_MASTER_INDEX.md` | Replaced 14x and 19x glass claims with correct values |

### Files intentionally NOT modified
- **Patent filings** (`02_Patent_Draft/`, `10_Utility_Patents/`): Legal documents
  already filed; modifications would require patent counsel review.
- **Legacy archives** (`04_Source_Code/00_LEGACY_ARCHIVE/`): Historical code;
  modifications would destroy audit trail.
- **Simulation data files** (`03_Simulation_Data/`): Generated outputs; would
  need to be regenerated by re-running scripts.
- **Deep dive audit** (`DEEP_DIVE_AUDIT_PROV1.md`): Historical audit document
  that correctly identifies the 14x issue; modifying would destroy the record.
- **Red team report** (`RED_TEAM_AUDIT_REPORT.md`): Historical finding document.

### Remaining "14x variance" references in legacy/legal files
The following files still contain "14x variance explosion" references but were
intentionally not modified because they are either (a) filed legal documents,
(b) historical audit records, or (c) legacy code archives:
- `10_Utility_Patents/utility_patent_A_azimuthal_stiffness.md` (patent filing)
- `02_Patent_Draft/PROVISIONAL_PATENT_1_FAB_OS_COMPLETE.md` (patent filing)
- `PROVISIONAL_PATENT_1_FAB_OS_COMPLETE.md` (patent filing)
- `DEEP_DIVE_AUDIT_PROV1.md` (historical audit)
- `RED_TEAM_AUDIT_REPORT.md` (historical audit finding)
- `04_Source_Code/00_LEGACY_ARCHIVE/` (archived code)
- `04_Source_Code/optical_bridge.py` (data analysis script -- outputs 14x from
  crossload data; the number is real for that specific comparison but the
  framing as "variance explosion" is misleading)
- `04_Source_Code/statistical_robustness.py` (validates the 14x ratio -- this
  is a valid cross-load CV ratio, just not an amplitude ratio)

---

*Generated: February 28, 2026*
*Updated: February 28, 2026 (second pass -- additional doc files fixed)*
*Purpose: Science integrity audit response*
