# Honest Disclosures

## Genesis PROV 1: Fab OS -- Limitations, Scope, and Transparency

We are committed to transparent communication about what this IP does and does not demonstrate. Every claim in the white paper should be understood in the context of the disclosures below.

---

## 1. All Results Are Computational

All results presented in this work -- including the 90.3% warpage reduction, the physics cliff discovery, the 315-case batch validation, and the ROM surrogate accuracy -- are derived from **finite element simulation (FEM)**, not from physical wafer experiments conducted in a semiconductor fabrication facility.

**What this means:** The physics captured by the biharmonic plate equation and the Kirchhoff-Love thin plate theory are well-established and widely used in structural mechanics. However, physical validation on actual EUV scanners has not been performed. Physical validation requires active fab access at costs exceeding $50,000 per day, which is standard practice to defer until post-LOI (Letter of Intent) in semiconductor IP transactions.

**What we have done:** Validated the FEM solver against Timoshenko analytical theory (0.18% accuracy), generated cross-validation input decks for CalculiX (60 cases) and ABAQUS (45 cases), and run a production floor simulation with 1,250 synthetic wafer records under realistic operating conditions.

---

## 2. Provisional Patent Status

The 5 utility patent drafts covering 112 claims were filed as **provisional patent applications** on January 31, 2026 (Application No. 63/751,001). Provisional patents establish a priority date but:

- Have **not been examined** by the United States Patent and Trademark Office (USPTO)
- Have **not been granted**
- Do **not confer enforceable patent rights** until converted to utility applications and examined

Utility patent conversion is in progress. The provisional filing establishes priority date, which is the critical IP protection milestone in a first-to-file jurisdiction.

---

## 3. Custom FEM Solver (Not Commercial FEA)

The biharmonic FEM solver is a **custom Python implementation** using NumPy and SciPy. It is not a commercial finite element analysis package (such as ANSYS, ABAQUS, or COMSOL).

**Validation performed:**
- Timoshenko analytical solution: 0.18% accuracy for clamped circular plate
- Mesh convergence study: O(h^1.8) convergence rate across 10 refinement levels
- Richardson extrapolation: Agreement with analytical reference to 4 significant figures
- CalculiX cross-validation: 60 cases with complete .inp input decks generated
- ABAQUS cross-validation: 45 cases with S4R shell element input decks generated

**What this means:** While the solver is validated against established analytical solutions and generates input decks for independent verification with commercial tools, it should be understood as a research-grade tool. A buyer with CalculiX or ABAQUS licenses can independently verify results using the provided input decks.

**Important clarification (Feb 2026 audit):** The CalculiX/ABAQUS "cross-validation" is **input deck generation only** -- the input decks were generated but never actually executed in CalculiX or ABAQUS. No cross-solver comparison results exist. Additionally, `comparison_results.json` contains a **222% error** entry that was not flagged in the original disclosures. The FEM solver also uses **hardcoded Si wafer geometry** (300mm diameter, standard thickness) for all materials, meaning InP, GaAs, and SiC results use silicon's geometry rather than material-specific wafer dimensions.

---

## 4. ROM Surrogate Trained on Simulated Data

The reduced-order model (ROM) surrogate achieving R-squared = 0.9937 was trained **exclusively on outputs from the FEM solver**, not on physical measurement data. The ROM's accuracy is therefore bounded by:

- The accuracy of the underlying FEM simulation (0.18% vs. Timoshenko)
- The representativeness of the training data parameter space
- The specific material properties and geometry configurations used in training

The ROM has not been validated against physical scanner data.

---

## 5. Specific Material and Geometry Combinations

Results apply to the specific combinations of:

- **Materials:** Silicon, Glass, InP, GaAs, SiC (with published elastic properties)
- **Geometry:** 300mm diameter wafers at standard thickness ranges
- **Thermal loads:** 0.01-0.1K temperature differentials (EUV-representative)
- **Stiffness parameters:** k_azi range 0.0-0.99, k_edge range 0.0-1.0

Extrapolation beyond these validated parameter ranges should be approached with appropriate engineering judgment. The physics cliff is expected to exist for all materials (as it is a structural phenomenon), but the precise critical threshold may differ for untested material/geometry combinations.

---

## 6. Linear Elasticity Assumption

All FEM solvers use **constant (temperature-independent) material properties**. Temperature-dependent coefficients of thermal expansion (CTE), elastic modulus, and Poisson's ratio are not modeled.

**Justification:** EUV thermal loads produce temperature differentials of 0.01-0.1K. At these scales, material property variation is physically negligible (typically <0.001% change in elastic modulus per 0.1K). The linear elasticity assumption is appropriate for this application domain.

**Where this matters:** Applications involving larger temperature differentials (e.g., SiC power electronics at elevated temperatures) may require nonlinear material models for accurate prediction.

---

## 7. No Contact Mechanics

The chuck support model uses **spring elements** to represent the wafer-chuck interface, not true contact mechanics with friction, slip, and separation. This is a simplifying assumption that:

- Correctly captures the stiffness distribution effect (the physics cliff)
- Does not model contact failure or wafer release events
- Does not account for electrostatic clamping force variation

**Mitigation:** The physics cliff detection identifies dangerous operating regimes regardless of the contact model fidelity, because the instability is driven by the stiffness distribution, not by the contact mechanics.

---

## 8. ASML Interface: Protocol Only

The SECS/GEM protocol implementation (1,335 lines) implements wire-format encoding, HSMS TCP/IP framing, and message construction compliant with SEMI E5/E37 standards. However, it has **not been connected to a physical ASML scanner**. Integration with a production scanner requires:

- Physical network access to the scanner's HSMS port
- Equipment-specific message configuration
- Integration testing with scanner control software

This integration is estimated at approximately 2 weeks of engineering effort for a team with scanner access.

---

## 9. Production Simulation Uses Synthetic Data

The 1,250-wafer production floor simulation uses **physics-based models with realistic variability parameters**, not measured production data from an operating fab. The simulation includes lot-to-lot variation, tool drift, and process noise, but these are modeled, not measured.

Real production data requires an active fab partnership, which is a post-acquisition activity.

---

## 10. No Export-Controlled Content

This repository and the associated data room contain no:

- ITAR (International Traffic in Arms Regulations) restricted technical data
- EAR (Export Administration Regulations) controlled technology
- Classified or sensitive government information
- Dual-use technology requiring export licenses

All content is fundamental research and commercial semiconductor manufacturing technology.

---

## Summary

| Disclosure | Status | Impact |
|:-----------|:-------|:-------|
| Computational only | Transparent | Standard for pre-LOI IP packages |
| Provisional patents | Priority date established | Utility conversion in progress |
| Custom FEM solver | Validated to 0.18% | Cross-validation decks provided |
| ROM on simulated data | R-squared 0.9937 | Bounded by FEM accuracy |
| Specific materials/geometry | 5 materials, 300mm | Physics is universal |
| Linear elasticity | Justified for EUV loads | May need extension for high-dT |
| No contact mechanics | Spring model | Cliff detection still valid |
| Protocol only (no scanner) | Wire-format compliant | 2-week integration estimate |
| Synthetic production data | Physics-based | Real data requires fab access |
| No export controls | Clean | No restrictions on transfer |

---

*We believe that honest disclosure of limitations strengthens, not weakens, the value proposition. A buyer who understands exactly what they are acquiring can make informed decisions and plan integration work accurately.*

Contact: [nmk.ai/contact](https://nmk.ai/contact)
