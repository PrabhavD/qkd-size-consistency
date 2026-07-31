# QKD-Size-Consistency

Project to determine if the Quantum Krylov Diagonalization algorithm (KQD) and its sample-based counterpart (SKQD) are size-consistent, and hence serve as helpful tools for simulating model chemistry on quantum computers.

***

## Size Consistency — Correct Finite-N Definition

A method is **size-consistent** if the energy of two non-interacting fragments A and B computed jointly equals the sum of their energies computed separately:

```text
E(A⋯B) = E(A) + E(B)
```

The **correct finite-N test** compares the joint (or protocol) energy at large separation against **same-N QKD monomers**, not exact monomer energies:

```text
ΔE_SC(N) = |E_method(A⋯B; N) − E_A^QKD(N) − E_B^QKD(N)|
           at R = 1000 Å
```

Comparing a converged dimer energy to **exact** monomers can look "size-consistent" by coincidence — that is a reference coincidence, not a finite-N SC proof. Key consequences established in this repo:

- **Joint sequential Krylov** at fixed Krylov order N is generally **not** size-consistent.
- **Factorized / Kronecker product bases** restore size consistency by construction in the R → ∞ limit.
- Apparent SC "lift-off" at large N often tracks **SVD truncation of a singular overlap S**, not physical SC loss.

***

## Latest Work: Interacting Krylov Protocols

> **Primary research contribution.** Fragment-structured Krylov bases for interacting H₂⋯H₂ (and H₂⋯H₄), testing size consistency against interacting accuracy. Chemical accuracy threshold: 1.6 mHa.

### Hamiltonian partition

```text
H_tot = H_a + H_b + H_ab
H_A   = H_a + (1/2) H_ab
H_B   = H_b + (1/2) H_ab
Δt    = π / ‖H_tot‖₂
```

### Protocol taxonomy

| Name | Basis states | Raw dim | Notes |
|---|---|---|---|
| **P1 / joint** | `U_tot^k \|HF⟩` | N | Supermolecular KQD |
| **P2** | Same joint energy | N | Δ vs exact monomers (not finite-N SC) |
| **P3 / novel** | `U_A^i U_B^j \|HF⟩` | N² | Dressed product; SC-friendly as H_ab → 0 |
| **both_order** | AB ∪ BA grids | ≤ 2N² | Uses [H_A, H_B] ≠ 0 at close contact |
| **multidt** | Union of AB grids at several Δt | ≤ 3N² | Spectral sampling; same product grammar |
| **hybrid** | novel ∪ short joint chain | ≤ N² + N | Dressed core + entanglement patch |
| **uab** | `exp(-i(k H_A + l H_B) Δt) \|HF⟩` | N² | Dressed sum; independent (k, l) |
| **cross_product** | Same form with H = H_A ⊗ I + I ⊗ H_B | N² | Ideal non-interacting / Kronecker limit |

### Notebooks

| Notebook | Role |
|---|---|
| [`krylov_interacting_protocols_sto3g_631g.ipynb`](krylov_interacting_protocols_sto3g_631g.ipynb) | Central: P1–P3, both_order, hybrid, multidt; STO-3G + 6-31G |
| [`krylov_interacting_protocols_sto3g.ipynb`](krylov_interacting_protocols_sto3g.ipynb) | STO-3G-only precursor |
| [`krylov_dressed_sum_uab_protocols.ipynb`](krylov_dressed_sum_uab_protocols.ipynb) | Dressed-sum U_AB |
| [`krylov_cross_product_protocols.ipynb`](krylov_cross_product_protocols.ipynb) | Kronecker R → ∞ SC |

### Headline results

| Setting | Method | Result |
|---|---|---|
| STO-3G, R = 1.1 Å, N = 6 | novel P3 | GS / interaction error ~ 9×10⁻⁷ Ha |
| STO-3G, R = 1000 Å | novel P3 | SC ~ 10⁻¹⁴ Ha |
| 6-31G, R = 1.1 Å, N = 8 | novel P3 | Plateaus at **~5.6 mHa** (eff_dim ≪ N²) |
| 6-31G, R = 1.1 Å, N = 6 | both_order / hybrid | **~1.30 / ~1.22 mHa** (chemical accuracy) |
| 6-31G, R = 1.1 Å, N = 6 | multidt | ~5.45 mHa — does not fix the grammar bottleneck |
| 6-31G H₂+H₂, N = 6 | uab dressed-sum | **0.85 mHa**; strongly SC-friendly |
| Kronecker R → ∞ | cross_product | Machine-precision SC when monomers are converged |
| Kronecker R → ∞ | joint | Clear finite-N SC bumps (e.g. STO-3G ~5×10⁻⁴ Ha at N = 2) |

**Paper-level conclusion.** STO-3G novel succeeds by finite-space overcompleteness. On 6-31G, fixed-N dressed product is an O(N), commutator-biased subspace missing inter-fragment entanglement. Hybrid / both_order / uab are the pragmatic fixes for interacting accuracy while preserving SC-friendly structure; Kronecker cross-product proves SC by construction in the non-interacting limit.

Figures and sweep CSVs live under `output/krylov_interacting_protocols_multibasis/`, `output/krylov_dressed_sum_uab/`, and `output/krylov_cross_product/`.

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

```text
|k⟩ = exp(−i H k · dt) |ref⟩
S_ij = ⟨i|j⟩
H̃_ij = ⟨i|H|j⟩
```

The generalised eigenvalue problem H̃ c = E S̃ c is solved with a regularised SVD-based solver (threshold 10⁻¹⁰).

### Performance optimisations

| Bug | Problem | Fix |
|---|---|---|
| Redundant `expm` calls | O(d) full matrix exponentials per `run_kqd` call | Compute U = exp(−i H · dt) **once**, apply iteratively |
| Redundant state rebuilding | States recomputed from scratch for each Krylov dim in sweep | Build all states at `max_dim` once; extract by slicing |
| Redundant matrix construction | S and H rebuilt per dim | Build full matrices once; extract submatrices by index |

**Net result:** `len(dt_scales) × 1` `expm` calls instead of `sum(dims) × len(dt_scales)`.

### Convergence (non-interacting monomer)

| | STO-3G | 6-31G |
|---|---|---|
| Exact GS (Ha) | −1.1373060358 | −1.1516143199 |
| Absolute error (Ha) | 4.44×10⁻¹⁶ | 6.22×10⁻¹⁵ |
| Krylov dim to converge | 2 | 7–10 (dt-dependent) |
| Chemical accuracy (1.6 mHa)? | Yes | Yes |

### Non-interacting dimer SC (converged)

The non-interacting dimer Hamiltonian is the tensor product:

```text
H_AB = H_A ⊗ I_B + I_A ⊗ H_B
```

with reference |ref_AB⟩ = |ref_A⟩ ⊗ |ref_B⟩. At Krylov dim 10, `dt = dt_opt`:

| | STO-3G | 6-31G |
|---|---|---|
| SC error (Ha) | 8.88×10⁻¹⁶ | 1.98×10⁻⁷ |

This is the **converged** non-interacting baseline. It does **not** imply joint KQD is size-consistent at arbitrary finite N (see protocols section and Steps 16–18 below).

### Extensions (`krylov_h2_exact_sc_copy_4.ipynb`)

| Step | Finding |
|---|---|
| **14** | Exact GS recovery requires K ≥ M (# eigenstates with nonzero ref overlap). No t rescues K < M. |
| **15** | Two-level "bird" t-sweeps; dimer 2-level product activates 4 states ⇒ K = 2 floor ~0.23 Ha. |
| **16** | SC vs Krylov dim; mismatched monomer/dimer dims can inflate SC error. |
| **17** | Heterogeneous H₂+H₄ additivity via product Krylov. |
| **18** | Shared sequential Krylov on N replicated monomers: δ = E(AA) − 2E(A) grows ~ N⁶ at fixed K. |

Output figures under `output/krylov_h2_exact_sc_figures/`.

***

## Spectral Error Theory

Synthetic models that explain *why* KQD errors appear and how spectra are resolved.

| Notebook | Focus |
|---|---|
| [`kqd_error_characterisation_fourier_box.ipynb`](kqd_error_characterisation_fourier_box.ipynb) | Linear DOS; reference-state and dt drivers |
| [`kqd_krylov_energy_discretisation_elimination.ipynb`](kqd_krylov_energy_discretisation_elimination.ipynb) | High→low residual elimination front at dt_opt |

**Main points:**

- **Nyquist** dt_opt = π / ‖H‖₂ (or π / E_max) places the top phase node at z = −1.
- At that dt, Krylov acts as a time-domain Prony / Vandermonde method: modes resolve **high energy → low energy**.
- Error landscape is **asymmetric**: small-dt cliff (near-parallel vectors / rank-deficient S) steeper than large-dt wrap-around.
- **Effective dimension** of the reference state's spectral support sets the Krylov budget; error floors until the Krylov dimension covers that support.

***

## Ising Size Extensivity

Critical TFIM (J = h = 1, PBC, Néel reference): is KQD **size-extensive** (E₀/N → const)?

| Notebook | Focus |
|---|---|
| [`kqd_ising_size_extensivity_extended.ipynb`](kqd_ising_size_extensivity_extended.ipynb) | N = 2…12; CFT 1/N² fits vs ED |
| [`kqd_ising_size_extensivity_multi_kd.ipynb`](kqd_ising_size_extensivity_multi_kd.ipynb) | Krylov-dim schedules vs N |
| [`qkd_ising_sweep.ipynb`](qkd_ising_sweep.ipynb) | Early Ising sweep / convergence plots |

Exact ED is extensive (E₀/N → −4/π ≈ −1.273). KQD at **fixed** Krylov dim is accurate for small N, then absolute and fractional errors grow — **method extensivity fails** unless the Krylov budget scales with system size. Near dt_opt is best; too-small dt worsens rank deficiency.

***

## Krylov Quantum Diagonalization (KQD) — AerSimulator

A quantum simulation of KQD estimating the ground-state energy via a Hadamard-test circuit, implemented with [Qiskit](https://qiskit.org/) and executed locally on `AerSimulator` ([`krylov_aer.ipynb`](krylov_aer.ipynb)).

### Overview

Builds a unitary Krylov subspace:

```text
Kᵁ_r = span{ |ψ₀⟩, U|ψ₀⟩, U²|ψ₀⟩, …, U^(r−1)|ψ₀⟩ }
```

where U = exp(−i H dt) is realised via a first-order Lie–Trotter product formula. Overlap **S** and effective Hamiltonian **H** are assembled from Hadamard-test expectation values; the GEVP H c = E S c is solved classically.

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

```text
H = Σ_{i,j} J_xy (X_i X_j + Y_i Y_j) + Z_i Z_j
```

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
