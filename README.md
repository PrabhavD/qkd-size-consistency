# QKD-Size-Consistency

Project to determine if the Quantum Krylov Diagonalization algorithm (QKD) and its sample-based counterpart (QKD) are size-consistent, and hence serve as helpful tools for simulating model chemistry on quantum computers.

***

## Size Consistency — Correct Finite-\(N\) Definition

A method is **size-consistent** if the energy of two non-interacting fragments \(A\) and \(B\) computed jointly equals the sum of their energies computed separately:

$$E(A \cdots B) = E(A) + E(B)$$

The **correct finite-\(N\) test** compares the joint (or protocol) energy at large separation against **same-\(N\) QKD monomers**, not exact monomer energies:

$$\Delta E_{\mathrm{SC}}(N)=\bigl|E_{\mathrm{method}}(A{\cdots}B;N)-E_A^{\mathrm{QKD}}(N)-E_B^{\mathrm{QKD}}(N)\bigr|
\quad\text{at }R=1000\,\text{Å}.$$

Comparing a converged dimer energy to **exact** monomers can look “size-consistent” by coincidence — that is a reference coincidence, not a finite-\(N\) SC proof. Key consequences established in this repo:

- **Joint sequential Krylov** at fixed Krylov order \(N\) is generally **not** size-consistent.
- **Factorized / Kronecker product bases** restore size consistency by construction in the \(R\to\infty\) limit.
- Apparent SC “lift-off” at large \(N\) often tracks **SVD truncation of a singular overlap \(S\)**, not physical SC loss.

***

## Latest Work: Interacting Krylov Protocols

> **Primary research contribution.** Fragment-structured Krylov bases for interacting \(\mathrm{H}_2{\cdots}\mathrm{H}_2\) (and \(\mathrm{H}_2{\cdots}\mathrm{H}_4\)), testing size consistency against interacting accuracy. Chemical accuracy threshold: \(1.6\,\mathrm{mHa}\).

### Hamiltonian partition

$$H_{\mathrm{tot}}=H_a+H_b+H_{ab},\qquad
H_A=H_a+\tfrac12 H_{ab},\qquad
H_B=H_b+\tfrac12 H_{ab},\qquad
\Delta t=\pi/\|H_{\mathrm{tot}}\|_2.$$

### Protocol taxonomy

| Name | Basis states | Raw dim | Notes |
|---|---|---|---|
| **P1 / joint** | \(U_{\mathrm{tot}}^k\|\mathrm{HF}\rangle\) | \(N\) | Supermolecular KQD |
| **P2** | Same joint energy | \(N\) | \(\Delta\) vs exact monomers (not finite-\(N\) SC) |
| **P3 / novel** | \(U_A^i U_B^j\|\mathrm{HF}\rangle\) | \(N^2\) | Dressed product; SC-friendly as \(H_{ab}\to0\) |
| **both_order** | AB ∪ BA grids | \(\le 2N^2\) | Uses \([H_A,H_B]\neq0\) at close contact |
| **multidt** | Union of AB grids at several \(\Delta t\) | \(\le 3N^2\) | Spectral sampling; same product grammar |
| **hybrid** | novel ∪ short joint chain | \(\le N^2+N\) | Dressed core + entanglement patch |
| **uab** | \(e^{-i(kH_A+lH_B)\Delta t}\|\mathrm{HF}\rangle\) | \(N^2\) | Dressed sum; independent \((k,l)\) |
| **cross_product** | Same form with \(H=H_A\otimes I+I\otimes H_B\) | \(N^2\) | Ideal non-interacting / Kronecker limit |

### Notebooks

| Notebook | Role |
|---|---|
| [`krylov_interacting_protocols_sto3g_631g.ipynb`](krylov_interacting_protocols_sto3g_631g.ipynb) | Central: P1–P3, both_order, hybrid, multidt; STO-3G + 6-31G |
| [`krylov_interacting_protocols_sto3g.ipynb`](krylov_interacting_protocols_sto3g.ipynb) | STO-3G-only precursor |
| [`krylov_dressed_sum_uab_protocols.ipynb`](krylov_dressed_sum_uab_protocols.ipynb) | Dressed-sum \(U_{AB}\) |
| [`krylov_cross_product_protocols.ipynb`](krylov_cross_product_protocols.ipynb) | Kronecker \(R\to\infty\) SC |
| [`uab_sc_check.ipynb`](uab_sc_check.ipynb) | Minimal OpenFermion sanity check |

### Headline results

| Setting | Method | Result |
|---|---|---|
| STO-3G, \(R=1.1\) Å, \(N=6\) | novel P3 | GS / interaction error \(\sim 9\times10^{-7}\) Ha |
| STO-3G, \(R=1000\) Å | novel P3 | SC \(\sim10^{-14}\) Ha |
| 6-31G, \(R=1.1\) Å, \(N=8\) | novel P3 | Plateaus at **\(\sim5.6\,\mathrm{mHa}\)** (eff_dim \(\ll N^2\)) |
| 6-31G, \(R=1.1\) Å, \(N=6\) | both_order / hybrid | **\(\sim1.30\) / \(\sim1.22\,\mathrm{mHa}\)** (chemical accuracy) |
| 6-31G, \(R=1.1\) Å, \(N=6\) | multidt | \(\sim5.45\,\mathrm{mHa}\) — does not fix the grammar bottleneck |
| 6-31G H₂+H₂, \(N=6\) | uab dressed-sum | **\(0.85\,\mathrm{mHa}\)**; strongly SC-friendly |
| Kronecker \(R\to\infty\) | cross_product | Machine-precision SC when monomers are converged |
| Kronecker \(R\to\infty\) | joint | Clear finite-\(N\) SC bumps (e.g. STO-3G \(\sim5\times10^{-4}\) Ha at \(N=2\)) |

**Paper-level conclusion.** STO-3G novel succeeds by finite-space overcompleteness. On 6-31G, fixed-\(N\) dressed product is an \(O(N)\), commutator-biased subspace missing inter-fragment entanglement. Hybrid / both_order / uab are the pragmatic fixes for interacting accuracy while preserving SC-friendly structure; Kronecker cross-product proves SC by construction in the non-interacting limit.

Figures and sweep CSVs live under `output/krylov_interacting_protocols_multibasis/`, `output/krylov_dressed_sum_uab/`, and related folders.

***

## Non-Interacting Baseline: H₂ Exact-Evolution KQD

[`krylov_h2_exact_sc.ipynb`](krylov_h2_exact_sc.ipynb) benchmarks KQD on H₂ with **exact matrix exponentiation** (`scipy.linalg.expm`), isolating algorithmic convergence from Trotter error. Extended analysis is in [`krylov_h2_exact_sc_copy_4.ipynb`](krylov_h2_exact_sc_copy_4.ipynb) (Steps 14–18).

### Hamiltonian construction

Hamiltonians via **Qiskit Nature + PySCF**, `ParityMapper` 2-qubit reduction:

| Property | STO-3G | 6-31G |
|---|---|---|
| Qubits (after 2Q reduction) | 2 | 6 |
| Hilbert space dimension | 4 | 64 |
| Pauli terms | 5 | 159 |
| Exact GS (total, Ha) | −1.13730604 | −1.15161432 |

Hartree–Fock statevector as Krylov reference. Algorithm:

$$|k\rangle = e^{-iHk \cdot dt}|\mathrm{ref}\rangle, \quad S_{ij} = \langle i | j \rangle, \quad \tilde{H}_{ij} = \langle i | H | j \rangle$$

GEVP \(\tilde{H}\mathbf{c} = E\,\tilde{S}\mathbf{c}\) with regularised SVD (threshold \(10^{-10}\)).

### Performance optimisations

| Bug | Problem | Fix |
|---|---|---|
| Redundant `expm` calls | O(d) full matrix exponentials per `run_kqd` | Compute \(U = e^{-iH \cdot dt}\) once; \(|k\rangle = U^k|\mathrm{ref}\rangle\) |
| Redundant state rebuilding | States recomputed per Krylov dim | Build at `max_dim` once; slice |
| Redundant matrix construction | S and H rebuilt per dim | Build full matrices once; extract submatrices |

### Convergence (non-interacting monomer)

| | STO-3G | 6-31G |
|---|---|---|
| Absolute error (Ha) | \(4.44\times10^{-16}\) | \(6.22\times10^{-15}\) |
| Krylov dim to converge | 2 | 7–10 (`dt`-dependent) |
| Chemical accuracy? | Yes | Yes |

### Non-interacting dimer SC (converged)

Tensor-product dimer \(H_{AB}=H_A\otimes I_B+I_A\otimes H_B\). At Krylov dim 10, `dt = dt_opt`:

| | STO-3G | 6-31G |
|---|---|---|
| SC error (Ha) | \(8.88\times10^{-16}\) | \(1.98\times10^{-7}\) |

This is the **converged** non-interacting baseline. It does **not** imply joint KQD is size-consistent at arbitrary finite \(N\) (see protocols section and Steps 16–18 below).

### Extensions (`krylov_h2_exact_sc_copy_4.ipynb`)

| Step | Finding |
|---|---|
| **14** | Exact GS recovery requires \(K\ge M\) (# eigenstates with nonzero ref overlap). No \(t\) rescues \(K<M\). |
| **15** | Two-level “bird” \(t\)-sweeps; dimer 2-level product activates 4 states ⇒ \(K=2\) floor \(\sim0.23\) Ha. |
| **16** | SC vs Krylov dim; mismatched monomer/dimer dims can inflate SC error. |
| **17** | Heterogeneous H₂+H₄ additivity via product Krylov. |
| **18** | Shared sequential Krylov on \(N\) replicated monomers: \(\delta=E(AA)-2E(A)\) grows \(\sim N^{6}\) at fixed \(K\). |

***

## Spectral Error Theory

Synthetic and continuum models that explain *why* KQD errors appear and how spectra are resolved.

| Notebook | Focus |
|---|---|
| [`kqd_error_characterisation_fourier_box.ipynb`](kqd_error_characterisation_fourier_box.ipynb) | Linear DOS; reference-state and \(dt\) drivers |
| [`kqd_error_energy_discretisation.ipynb`](kqd_error_energy_discretisation.ipynb) | Continuum binning; Fourier ↔ energy dictionary |
| [`kqd_krylov_energy_discretisation_elimination.ipynb`](kqd_krylov_energy_discretisation_elimination.ipynb) | High→low residual elimination front at \(dt_{\mathrm{opt}}\) |
| [`kqd_fourier_box_with_effdim.ipynb`](kqd_fourier_box_with_effdim.ipynb) | Effective-dimension threshold |
| [`kqd_free_particle_1d_v7.ipynb`](kqd_free_particle_1d_v7.ipynb) | Free particle; grid vs Krylov ceilings; corrected \(d_{\mathrm{eff}}\) |

**Main points:**

- **Nyquist** \(dt_{\mathrm{opt}}=\pi/\|H\|_2\) (or \(\pi/E_{\max}\)) places the top phase node at \(z=-1\).
- At that \(dt\), Krylov acts as a time-domain Prony / Vandermonde method: modes resolve **high energy → low energy**.
- Error landscape is **asymmetric**: small-\(dt\) cliff (near-parallel vectors / rank-deficient \(S\)) steeper than large-\(dt\) wrap-around.
- **Effective dimension** \(d_{\mathrm{eff}}=\exp(-\sum |c_k|^2\ln|c_k|^2)\) sets the Krylov budget; error floors until \(d\gtrsim d_{\mathrm{eff}}\) when the target is in the support of \(|\psi_0\rangle\).
- **Free-particle caveat:** Krylov span = support of \(|\psi_0\rangle\); a packet away from \(k_{\min}\) cannot reach \(E_{\min}\).

***

## Ising Size Extensivity

Critical TFIM (\(J=h=1\), PBC, Néel reference): is KQD **size-extensive** (\(E_0/N\to\mathrm{const}\))?

| Notebook | Focus |
|---|---|
| [`kqd_ising_size_extensivity_extended.ipynb`](kqd_ising_size_extensivity_extended.ipynb) | \(N=2\ldots12\); CFT \(1/N^2\) fits vs ED |
| [`kqd_ising_size_extensivity_multi_dt.ipynb`](kqd_ising_size_extensivity_multi_dt.ipynb) | \(dt\) scales at fixed Krylov dim |
| [`kqd_ising_size_extensivity_multi_kd.ipynb`](kqd_ising_size_extensivity_multi_kd.ipynb) | Krylov-dim schedules vs \(N\) |

Exact ED is extensive (\(E_0/N\to -4/\pi\approx -1.273\)). KQD at **fixed** Krylov dim is accurate for small \(N\), then absolute and fractional errors grow — **method extensivity fails** unless the Krylov budget scales with system size. Near \(dt_{\mathrm{opt}}\) is best; too-small \(dt\) worsens rank deficiency.

***

## Analytic Drafts and Tooling

| Path | Contents |
|---|---|
| [`novel-qkd-analytics/qkd_interacting_analytic.tex`](novel-qkd-analytics/qkd_interacting_analytic.tex) | Subspace fidelity \(F_N\), commutator, residual diagnostics for the 6-31G plateau |
| [`paper/qkd_interacting_analytic_proof.tex`](paper/qkd_interacting_analytic_proof.tex) | Pure analytic proof: novel spans \(\psi_0\) on STO-3G but not at fixed \(N\) on 6-31G |
| [`scripts/step3f_analytic_diagnostics.py`](scripts/step3f_analytic_diagnostics.py) | Reproducible PySCF / Qiskit Nature diagnostics pipeline |

Generated figures and CSVs are under `output/`.

***

## Krylov Quantum Diagonalization (KQD) — AerSimulator

A quantum simulation of KQD estimating the ground-state energy via a Hadamard-test circuit, implemented with [Qiskit](https://qiskit.org/) and executed locally on `AerSimulator` ([`krylov_aer.ipynb`](krylov_aer.ipynb)).

### Overview

Builds a unitary Krylov subspace

$$K^U_r = \mathrm{span}\{|\psi_0\rangle,\, U|\psi_0\rangle,\, U^2|\psi_0\rangle,\, \ldots,\, U^{r-1}|\psi_0\rangle\}$$

where \(U = e^{-iH\,dt}\) is realised via a first-order Lie–Trotter product formula. Overlap **S** and effective Hamiltonian **H** are assembled from Hadamard-test expectation values; the GEVP \(H\mathbf{c} = E\,S\mathbf{c}\) is solved classically.

### What changed from the IBM Learning Course notebook

The original ([IBM Quantum Learning — KQD](https://quantum.cloud.ibm.com/learning/en/courses/quantum-diagonalization-algorithms/krylov)) targets real IBM hardware. This version replaces QPU infrastructure with local `AerSimulator`, preserving the algorithm logic.

| Component | Original (QPU) | This version (AerSimulator) |
|---|---|---|
| Backend | `service.least_busy()` / `ibm_fez` | `AerSimulator.from_backend(FakeAuckland())` |
| Execution primitive | `Batch` + `EstimatorV2` | `EstimatorV2(mode=backend)` |
| Error mitigation | ZNE / PEA / TREX | None (4096 shots) |
| Credentials required | Yes | No |
| Estimated runtime | ~17 minutes QPU time | Seconds (local) |

### Key parameters

| Parameter | Value |
|---|---|
| Krylov dimension | 4 |
| Time step `dt` | configurable |
| Shots | 4096 |
| Transpile optimisation level | 3 |
| Noise model | `FakeAuckland` device noise |

```bash
pip install qiskit qiskit-aer qiskit-ibm-runtime
```

***

## Sample-based Krylov Quantum Diagonalization (SKQD)

SKQD on a 22-site antiferromagnetic XXZ spin-1/2 chain with periodic boundary conditions ([`skqd.ipynb`](skqd.ipynb)):

$$H = \sum_{i,j} J_{xy}(X_i X_j + Y_i Y_j) + Z_i Z_j$$

The algorithm builds a Krylov subspace by Trotterized evolution of a Néel reference, samples bitstrings from each Krylov vector, and classically diagonalises the projected Hamiltonian.

| Parameter | Value |
|---|---|
| System size | 22 spins |
| Krylov dimension | 5 |
| Time step `dt` | 0.15 |
| Trotter steps | 6 |
| Shots | 100,000 |

```bash
pip install qiskit qiskit-aer qiskit-ibm-runtime qiskit-addon-sqd qiskit-addon-utils
```

The estimated ground-state energy converges toward the exact value (−23.934) with increasing Krylov dimension.

***

## References

- Yu et al., *Quantum-Centric Algorithm for Sample-Based Krylov Diagonalization* (2025). [arXiv:2501.09702](https://arxiv.org/abs/2501.09702)
- Epperly, Lin & Nakatsukasa, *A theory of quantum subspace diagonalization*, SIAM (2022)
- Hatano & Suzuki, *Finding Exponential Product Formulas of Higher Orders* (2005)
- IBM Quantum Learning, *Krylov Quantum Diagonalization* course. [quantum.cloud.ibm.com](https://quantum.cloud.ibm.com/learning/en/courses/quantum-diagonalization-algorithms/krylov)
