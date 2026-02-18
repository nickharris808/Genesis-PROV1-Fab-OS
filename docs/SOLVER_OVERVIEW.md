# Solver Architecture Overview (Non-Confidential)

## Genesis PROV 1: Fab OS -- Computational Framework

This document describes the mathematical foundations and validation approach for the Fab OS computational framework. It covers the partial differential equations solved, the control theory applied, and the validation methodology used. It does not disclose solver implementation details, source code, or proprietary algorithms.

---

## 1. Biharmonic FEM Solver

### Governing Equation

The core solver implements the **Kirchhoff-Love thin plate equation**, a fourth-order biharmonic partial differential equation that governs out-of-plane deflection of thin elastic plates:

```
D * nabla^4 w(x,y) = q(x,y)
```

where:
- `D = E * h^3 / [12(1 - nu^2)]` is the flexural rigidity (N*m)
- `w(x,y)` is the out-of-plane deflection field (m)
- `q(x,y)` is the distributed load from thermal curvature (N/m^2)
- `E` is Young's modulus (Pa)
- `h` is the plate thickness (m)
- `nu` is Poisson's ratio (dimensionless)

### Thermal Loading

For thermal loading, the equivalent distributed load derives from the thermal curvature:

```
kappa_T = alpha * delta_T / h
q_thermal = D * nabla^2(kappa_T)
```

For uniform temperature differential, the analytical maximum deflection of a simply-supported circular plate is:

```
w_max = (alpha * delta_T / h) * R^2 / 2
```

This is the Timoshenko reference solution used for validation.

### Azimuthal Stiffness Law

The solver handles arbitrary chuck stiffness distributions through the azimuthal stiffness modulation law:

```
K(r, theta) = K_0 * [1 + k_edge * (r/R)^alpha] * [1 + k_azi * cos(n * theta)]
```

where:
- `K_0` is the baseline spring stiffness (N/m)
- `k_edge` is the edge stiffness parameter (dimensionless)
- `k_azi` is the azimuthal stiffness parameter (dimensionless, 0 to <1)
- `R` is the wafer radius (m)
- `n` is the azimuthal harmonic order (integer)
- `alpha` is the radial exponent (dimensionless)

The **physics cliff** occurs when `k_azi` exceeds a material-dependent critical threshold (~0.9 for silicon), causing warpage amplitude to explode nonlinearly.

### Discretization

- **Domain:** Circular plate (300mm diameter wafer)
- **Elements:** Triangular finite elements with cotangent-Laplacian stiffness
- **Mesh:** Structured polar grid with center fan and annular rings
- **Boundary conditions:** Simply-supported (w=0 on edge) or clamped
- **Solve method:** Sparse direct factorization (SciPy)

### Validation

| Benchmark | Error | Method |
|:----------|:------|:-------|
| Timoshenko analytical (clamped plate) | 0.18% | Direct comparison at 6 temperature levels |
| Mesh convergence (10-level h-refinement) | O(h^1.8) | Richardson extrapolation to 4 significant figures |
| CalculiX cross-validation | 60 cases | .inp input deck generation for C3D8 elements |
| ABAQUS cross-validation | 45 cases | S4R shell element comparison at 3 mesh densities |

---

## 2. Iterative Learning Control (ILC)

### Control Law

The ILC controller operates in a Zernike-decomposed modal space:

```
u_{n+1} = u_n + gamma_n * L^{-1} * (w_target - w_measured)
```

where:
- `u_n` is the actuation field at iteration n
- `gamma_n = gamma_0 * decay^n` is the decreasing gain schedule
- `L^{-1}` is the inverse plant model (from the FEM solver)
- `w_target` is the desired deflection (typically zero)
- `w_measured` is the measured deflection from the current iteration

### Zernike Decomposition

The deflection field is projected onto Zernike polynomial basis functions:

```
w(rho, theta) = sum_{n,m} a_n^m * Z_n^m(rho, theta)
```

Zernike polynomials are orthogonal on the unit disk and correspond to physically meaningful deformation modes:
- `Z_0^0`: Piston (constant offset)
- `Z_2^0`: Defocus (parabolic)
- `Z_2^2`, `Z_2^{-2}`: Astigmatism
- `Z_3^1`, `Z_3^{-1}`: Coma
- Higher orders: Trefoil, spherical aberration, etc.

The controller applies mode-specific gain multipliers, with stronger correction for low-order modes (defocus, astigmatism) and lighter correction for high-order modes to prevent noise amplification.

### Convergence Properties

The decreasing gain schedule ensures monotonic convergence:
- **Default configuration (A):** gamma_0 = 0.5, EMA = 0.5, achieves 90.8% reduction
- **Benchmark configuration (B):** gamma_0 = 0.6, EMA = 0.6, achieves 96.5% reduction

Convergence is patience-based: the controller terminates after N iterations without improvement.

### Validation

| Scenario | Reduction | Iterations |
|:---------|:----------|:-----------|
| Silicon, default config | 90.8% | 23 |
| Silicon, benchmark config | 96.5% | 15 |
| InP, benchmark config | 97.6% | 25 |
| 315-case batch (5 materials, 7 k_azi, 3 loads, 3 dT) | 90.3% mean | 15-60 |
| +/-20% plant mismatch (100 MC trials) | 83.4% mean | varies |

---

## 3. Reduced-Order Model (ROM)

### Architecture

The ROM is a gradient-boosting ensemble trained on FEM solver outputs:
- **Input features:** k_azi, delta_T, material (encoded), load_pattern (encoded)
- **Output:** Peak-to-valley warpage (nm)
- **Training data:** 540 balanced FEM solutions spanning the parameter space
- **Validation:** 5-fold cross-validation, R^2 = 0.9937 +/- 0.0054

### Feature Importance

| Feature | Importance |
|:--------|:----------|
| delta_T | 46.0% |
| load_pattern | 31.4% |
| material | 15.4% |
| k_azi | 7.2% |

Note: k_azi has low feature importance for *amplitude prediction* because delta_T dominates the amplitude scale. However, k_azi has a 61% effect on warpage *variation*, which is the cliff phenomenon. The ROM captures amplitude; the cliff is a separate structural finding.

---

## 4. SECS/GEM Protocol Interface

### Standards Implemented

The ASML scanner interface implements:
- **SEMI E5 (SECS-II):** Binary message encoding and decoding
- **SEMI E37 (HSMS):** TCP/IP transport layer with header framing
- **GEM messages:** S1F13/S1F14 (establish communication), S2F41/S2F42 (host command), S6F11/S6F12 (event report)

### Status

Protocol wire-format encoding has been validated (52/52 tests pass). No physical scanner connection has been tested. Integration with standard SECS/GEM libraries is estimated at 2 weeks of engineering effort.

---

## 5. Cross-Validation Infrastructure

### CalculiX Validation Suite

60 independent validation cases with complete `.inp` input decks:
- 5 materials x 4 temperatures x 3 k_azi levels
- C3D8 8-node hexahedral elements
- Full material cards, boundary conditions, and thermal loads
- Analytical Timoshenko comparison at k_azi = 0

### ABAQUS Comparison Framework

45 comparison cases:
- 5 materials x 3 temperatures x 3 mesh densities
- S4R quadrilateral shell elements
- Coarse, medium, and fine mesh levels

Both suites generate standard FEA input decks that any buyer with commercial FEA licenses can run independently.

---

*This document describes mathematical foundations only. Solver source code, implementation details, and proprietary algorithms are available in the full data room upon request.*
