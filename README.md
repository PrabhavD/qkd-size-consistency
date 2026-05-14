# QKD-Size-Consistency

Project to determine if the Quantum Krylov Diagonalization algorithm (KQD) and its sample-based counterpart (SKQD) are size-consistent, and hence serve as helpful tools for simulating model chemistry on quantum computers.

***

## 🔬 Latest Work: H2 Exact-Evolution KQD & Size Consistency Analysis

> **This is the primary research contribution of this repository.** The notebooks below establish a clean, Trotter-error-free baseline for KQD on the hydrogen molecule and rigorously probe size consistency — a necessary condition for any method to be chemically meaningful.

### What is Size Consistency?

A method is **size-consistent** if the energy of two non-interacting fragments A and B computed jointly equals the sum of their energies computed separately:

$$E(A \cdots B) = E(A) + E(B)$$

This is a non-trivial requirement for variational quantum algorithms. KQD passes this test exactly for STO-3G and to within sub-chemical accuracy for 6-31G (see results below).

***

### `krylov_h2_exact_sc.ipynb` — KQD on H₂ with Exact Evolution

This notebook benchmarks KQD on the hydrogen molecule using **exact matrix exponentiation** (`scipy.linalg.expm`), completely bypassing Trotterization. This isolates the KQD algorithm's intrinsic convergence from any Trotter-product error.

#### Hamiltonian Construction

Hamiltonians are built via **Qiskit Nature + PySCF**, with a `ParityMapper` 2-qubit reduction exploiting Z₂ particle-number and spin-parity symmetries:

| Property | STO-3G | 6-31G |
|---|---|---|
| Qubits (after 2Q reduction) | 2 | 6 |
| Hilbert space dimension | 4 | 64 |
| Pauli terms | 5 | 159 |
| Exact GS (total, Ha) | −1.13730604 | −1.15161432 |

The **Hartree-Fock statevector** is used as the Krylov reference state — a product state requiring only X-gate preparation, consistent with the efficient Hadamard-test circuit model.

#### Algorithm

$$|k\rangle = e^{-iHk \cdot dt}|\text{ref}\rangle, \quad S_{ij} = \langle i | j \rangle, \quad \tilde{H}_{ij} = \langle i | H | j \rangle$$

The generalised eigenvalue problem $\tilde{H}\mathbf{c} = E\,\tilde{S}\mathbf{c}$ is solved with a regularised SVD-based solver (threshold $10^{-10}$).

#### Performance Optimisations

Three algorithmic bugs in the naive implementation were fixed, yielding a significant speedup for the 6-31G sweeps:

| Bug | Problem | Fix |
|---|---|---|
| Redundant `expm` calls | O(d) full matrix exponentials per `run_kqd` call | Compute $U = e^{-iH \cdot dt}$ **once**, apply iteratively: $|k\rangle = U^k|\text{ref}\rangle$ |
| Redundant state rebuilding | States recomputed from scratch for each Krylov dim in sweep | Build all states at `max_dim` once; extract by slicing |
| Redundant matrix construction | S and H rebuilt per dim | Build full matrices once; extract submatrices by index |

**Net result:** `len(dt_scales) × 1` `expm` calls instead of `sum(dims) × len(dt_scales)`.

#### Convergence Results

| | STO-3G | 6-31G |
|---|---|---|
| Exact GS (Ha) | −1.1373060358 | −1.1516143199 |
| KQD GS (Ha) | −1.1373060358 | −1.1516143199 |
| Absolute error (Ha) | 4.44 × 10⁻¹⁶ | 6.22 × 10⁻¹⁵ |
| Krylov dim to converge | 2 | 7–10 (dt-dependent) |
| Chemical accuracy (1.6 mHa)? | ✅ | ✅ |
| Machine precision? | ✅ | ✅ |

STO-3G saturates the full Hilbert space at Krylov dimension 2 (the space is only 4-dimensional). 6-31G requires up to dim 10 at suboptimal `dt`, but reliably hits machine precision at `dt_opt`.

***

### Size Consistency Analysis

The **non-interacting dimer** Hamiltonian is constructed as the tensor-product:

$$H_{AB} = H_A \otimes I_B + I_A \otimes H_B$$

with reference state $|\text{ref}_{AB}\rangle = |\text{ref}_A\rangle \otimes |\text{ref}_B\rangle$. This is mathematically equivalent to running PySCF at $R = 1000$ Å (where all inter-molecular integrals vanish), but avoids re-mapping a 4-electron system through a different symmetry sector.

The **size consistency error** is defined as:

$$\Delta E_{\text{SC}} = |E_{\text{KQD}}(A \cdots B) - 2\, E_{\text{KQD}}(A)|$$

#### Converged Size Consistency (Krylov dim = 10, `dt = dt_opt`)

| | STO-3G | 6-31G |
|---|---|---|
| 2 × E_KQD(H₂) (Ha) | −2.27461207 | −2.30322864 |
| E_KQD(H₂⋯H₂) (Ha) | −2.27461207 | −2.30322844 |
| **SC error (Ha)** | **8.88 × 10⁻¹⁶** | **1.98 × 10⁻⁷** |

STO-3G achieves **exact** size consistency (machine precision). 6-31G exhibits a residual error of ~0.2 µHa — four orders of magnitude below chemical accuracy — attributable to the larger Hilbert space and tighter regularisation demands.

#### Size Consistency vs. `dt` Sweep

A sweep over `dt` scales from $10^{-2}$ to $3 \times dt_{\text{opt}}$ characterises the robustness of size consistency:

- **`dt` too small** → consecutive Krylov vectors become nearly identical → S rank-deficient → SC breaks
- **`dt` too large** → Krylov vectors wrap/randomise → S ill-conditioned or singular → SC breaks
- **Sweet spot** → S well-conditioned → KQD converges and remains size-consistent

The minimum eigenvalue of S is tracked as a conditioning proxy alongside the SC error and monomer KQD error, for both monomer and dimer, in both basis sets.

#### Output Figures

| File | Description |
|---|---|
| `kqd_h2_exact_convergence.png` | KQD error vs Krylov dim, 4 dt scales, both bases |
| `kqd_sc_vs_dt.png` | SC error, monomer error & S conditioning vs dt (4×2 panel) |
| `kqd_h2_sc_bar.png` | Converged energy bar chart: 2×E_mono vs E_dimer vs 2×E_exact |
| `kqd_sc_overlay.png` | STO-3G vs 6-31G overlay (3 panels) |
| `kqd_h2_size_consistency.png` | Full Krylov-dim × dt SC sweep (6 panels) |

***

## Krylov Quantum Diagonalization (KQD) — Ground State Estimation via AerSimulator

A quantum simulation of the KQD algorithm estimating the ground state energy of a Hamiltonian using a Hadamard-test-based circuit, implemented with [Qiskit](https://qiskit.org/) and executed locally on `AerSimulator`.

### Overview

This notebook implements KQD using time evolution to build a unitary Krylov subspace:

$$K^U_r = \text{span}\{|\psi_0\rangle,\, U|\psi_0\rangle,\, U^2|\psi_0\rangle,\, \ldots,\, U^{r-1}|\psi_0\rangle\}$$

where $U = e^{-iH\,dt}$ is realised via a first-order Lie–Trotter product formula. The overlap matrix **S** and effective Hamiltonian matrix **H** are assembled from expectation values measured via ancilla-qubit Hadamard tests. The generalised eigenvalue problem $H\mathbf{c} = E\,S\mathbf{c}$ is then solved classically to extract the ground state energy estimate.

### What Changed from the IBM Learning Course Notebook

The original notebook ([IBM Quantum Learning — KQD](https://quantum.cloud.ibm.com/learning/en/courses/quantum-diagonalization-algorithms/krylov)) targets real IBM Quantum hardware via `QiskitRuntimeService`. This version replaces all QPU-specific infrastructure with a fully local `AerSimulator` execution path, preserving the KQD algorithm logic exactly.

| Component | Original (QPU) | This Version (AerSimulator) |
|---|---|---|
| Backend | `service.least_busy()` / `ibm_fez` | `AerSimulator.from_backend(FakeAuckland())` |
| Execution primitive | `Batch` + `EstimatorV2` | `EstimatorV2(mode=backend)` |
| Error mitigation | ZNE / PEA / TREX | None (4096 shots) |
| Credentials required | Yes | No |
| Estimated runtime | ~17 minutes QPU time | Seconds (local) |

### Algorithm Steps (Qiskit Patterns)

1. **Map** — Define the target Hamiltonian as a `SparsePauliOp` and construct the parameterised Hadamard-test circuit with `PauliEvolutionGate` + `LieTrotter`
2. **Optimize** — Transpile to the `FakeAuckland` basis gate set using `generate_preset_pass_manager` at `optimization_level=3`
3. **Execute** — Run all parameter sweeps in a single PUB using `EstimatorV2` on `AerSimulator`
4. **Post-process** — Assemble **S** and **H** matrices from `results.data.evs`, solve the regularised generalised eigenvalue problem, and plot convergence

### Key Parameters

| Parameter | Value |
|---|---|
| Krylov dimension | 4 |
| Time step `dt` | configurable |
| Shots | 4096 |
| Transpile optimisation level | 3 |
| Noise model | `FakeAuckland` device noise |

### Dependencies

```bash
pip install qiskit qiskit-aer qiskit-ibm-runtime
```

***

## Sample-based Krylov Quantum Diagonalization (SKQD)

A quantum simulation of the SKQD algorithm for estimating the ground state energy of a spin-chain Hamiltonian, implemented using [Qiskit](https://qiskit.org/).

### Overview

This project implements SKQD on a 22-site antiferromagnetic XXZ spin-1/2 chain with periodic boundary conditions:

$$H = \sum_{i,j} J_{xy}(X_i X_j + Y_i Y_j) + Z_i Z_j$$

The algorithm builds a quantum Krylov subspace by repeatedly applying a Trotterized time-evolution operator to a Néel reference state, samples bitstrings from each Krylov vector, and classically diagonalises the projected Hamiltonian to estimate the ground state energy.

### Key Parameters

| Parameter | Value |
|---|---|
| System size | 22 spins |
| Krylov dimension | 5 |
| Time step `dt` | 0.15 |
| Trotter steps | 6 |
| Shots | 100,000 |

### Dependencies

```bash
pip install qiskit qiskit-aer qiskit-ibm-runtime qiskit-addon-sqd qiskit-addon-utils
```

### Results

The estimated ground state energy converges toward the exact value (−23.934) with increasing Krylov dimension, demonstrating the provable convergence guarantees of SKQD.

***

## References

- Yu et al., *Quantum-Centric Algorithm for Sample-Based Krylov Diagonalization* (2025). [arXiv:2501.09702](https://arxiv.org/abs/2501.09702)
- Epperly, Lin & Nakatsukasa, *A theory of quantum subspace diagonalization*, SIAM (2022)
- Hatano & Suzuki, *Finding Exponential Product Formulas of Higher Orders* (2005)
- IBM Quantum Learning, *Krylov Quantum Diagonalization* course. [quantum.cloud.ibm.com](https://quantum.cloud.ibm.com/learning/en/courses/quantum-diagonalization-algorithms/krylov)
