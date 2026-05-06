# QKD-size-consistency

Project to determine if the Quantum Krylov Diagonalization algorithm (QKD) and its sample-based counterpart (SQKD) is size-consistent, and hence serve as helpful tools for simulating model chemistry on quantum computers.

***

## Krylov Quantum Diagonalization (KQD) — Ground State Estimation via AerSimulator

A quantum simulation of the Krylov Quantum Diagonalization algorithm estimating the ground state energy of a Hamiltonian using a Hadamard-test-based circuit, implemented with [Qiskit](https://qiskit.org/) and executed locally on `AerSimulator`.

### Overview

This notebook implements KQD using time evolution to build a unitary Krylov subspace:

$$K^U_r = \text{span}\{|\psi_0\rangle,\, U|\psi_0\rangle,\, U^2|\psi_0\rangle,\, \ldots,\, U^{r-1}|\psi_0\rangle\}$$

where $$U = e^{-iH\,dt}$$ is realised via a first-order Lie–Trotter product formula. The overlap matrix **S** and effective Hamiltonian matrix **H** are assembled from expectation values measured via ancilla-qubit Hadamard tests. The generalised eigenvalue problem $$H\mathbf{c} = E\,S\mathbf{c}$$ is then solved classically to extract the ground state energy estimate.

The plot below illustrates convergence of the KQD estimate toward the exact ground state energy as the Krylov space dimension grows from 1 to 4:



### What Changed from the IBM Learning Course Notebook

The original notebook ([IBM Quantum Learning — KQD](https://quantum.cloud.ibm.com/learning/en/courses/quantum-diagonalization-algorithms/krylov)) targets real IBM Quantum hardware via `QiskitRuntimeService`. This version replaces all QPU-specific infrastructure with a fully local `AerSimulator` execution path, preserving the KQD algorithm logic exactly.

| Component | Original (QPU) | This Version (AerSimulator) |
|---|---|---|
| Backend | `service.least_busy()` / `ibm_fez` | `AerSimulator.from_backend(FakeAuckland())` |
| Execution primitive | `Batch` + `EstimatorV2` | `EstimatorV2(mode=backend)` |
| Error mitigation | ZNE / PEA / TREX (`layer_noise_learning`) | None (4096 shots) |
| Credentials required | Yes (IBM Quantum account) | No |
| Estimated runtime | ~17 minutes QPU time | Seconds (local) |

### Algorithm Steps (Qiskit Patterns)

1. **Map** — Define the target Hamiltonian as a `SparsePauliOp` and construct the parameterised Hadamard-test circuit with `PauliEvolutionGate` + `LieTrotter`
2. **Optimize** — Transpile to the `FakeAuckland` basis gate set using `generate_preset_pass_manager` at `optimization_level=3`
3. **Execute** — Run all parameter sweeps in a single PUB (Primitive Unified Bloc) using `EstimatorV2` on `AerSimulator`
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

### Usage

Open and run `krylov_aer.ipynb` sequentially. No IBM Quantum credentials are needed.

```python
# Backend setup (replaces QiskitRuntimeService)
from qiskit_aer import AerSimulator
from qiskit_ibm_runtime.fake_provider import FakeAuckland

backend = AerSimulator.from_backend(FakeAuckland())

# Execution (replaces Batch context manager)
from qiskit_ibm_runtime import EstimatorV2 as Estimator

estimator = Estimator(mode=backend)
estimator.options.default_shots = 4096
job = estimator.run([pub])
results = job.result()[0]
```

### Results

The KQD estimate converges toward the exact ground state energy with increasing Krylov space dimension. The `FakeAuckland` noise model introduces realistic gate-level errors, so results exhibit small deviations from the noiseless exact value — consistent with expected behaviour on near-term hardware.

***

## Sample-based Krylov Quantum Diagonalization (SKQD)

A quantum simulation of the Sample-based Krylov Quantum Diagonalization algorithm for estimating the ground state energy of a spin-chain Hamiltonian, implemented using the [Qiskit](https://qiskit.org/) framework.

### Overview

This project implements SKQD on a 22-site antiferromagnetic XXZ spin-1/2 chain with periodic boundary conditions:

$$H = \sum_{i,j} J_{xy}(X_i X_j + Y_i Y_j) + Z_i Z_j$$

The algorithm builds a quantum Krylov subspace by repeatedly applying a Trotterized time-evolution operator to a Néel reference state, samples bitstrings from each Krylov vector, and classically diagonalises the projected Hamiltonian to estimate the ground state energy.

### Algorithm Steps (Qiskit Patterns)

1. **Map** — Generate the XXZ Hamiltonian and construct Krylov quantum circuits using `LieTrotter` synthesis
2. **Optimize** — Transpile circuits to the target backend using a preset pass manager
3. **Execute** — Sample circuits using `SamplerV2` on a local `AerSimulator` (no IBM Quantum account required)
4. **Post-process** — Aggregate counts, apply spin post-selection, project and diagonalise with `qiskit-addon-sqd`

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

### Usage

Open and run `skqd.ipynb` sequentially. No IBM Quantum credentials are needed — execution runs entirely on the local `AerSimulator`.

> **Note:** For results closer to real hardware convergence, initialise the simulator with a depolarising noise model to improve bitstring diversity in the sampled Krylov subspace.

### Results

The estimated ground state energy converges toward the exact value (−23.934) with increasing Krylov dimension, demonstrating the provable convergence guarantees of SKQD.

***

## References

- Yu et al., *Quantum-Centric Algorithm for Sample-Based Krylov Diagonalization* (2025). [arXiv:2501.09702](https://arxiv.org/abs/2501.09702)
- Epperly, Lin & Nakatsukasa, *A theory of quantum subspace diagonalization*, SIAM (2022)
- Hatano & Suzuki, *Finding Exponential Product Formulas of Higher Orders* (2005)
- IBM Quantum Learning, *Krylov Quantum Diagonalization* course. [quantum.cloud.ibm.com](https://quantum.cloud.ibm.com/learning/en/courses/quantum-diagonalization-algorithms/krylov)

***
