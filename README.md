# QKD-size-consistency
Project to determine if the Quantum Krylov Diagonalization algorithm (QKD) and it's sample-based counterpart (SQKD) is size-consistent, and hence serve helpful for simulating a model chemistry on quantum computers

## Sample-based Krylov Quantum Diagonalization (SKQD)

A quantum simulation of the **Sample-based Krylov Quantum Diagonalization** algorithm for estimating the ground state energy of a spin-chain Hamiltonian, implemented using the [Qiskit](https://qiskit.org/) framework.

### Overview

This project implements SKQD on a **22-site antiferromagnetic XXZ spin-1/2 chain** with periodic boundary conditions:

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

> **Note:** For results closer to real hardware convergence, initialise the simulator with a depolarizing noise model to improve bitstring diversity in the sampled Krylov subspace.

### Results

The estimated ground state energy converges toward the exact value (−23.934) with increasing Krylov dimension, demonstrating the provable convergence guarantees of SKQD.

### References

- Yu et al., *Quantum-Centric Algorithm for Sample-Based Krylov Diagonalization* (2025). [arXiv:2501.09702](https://arxiv.org/abs/2501.09702)
- Epperly, Lin & Nakatsukasa, *A theory of quantum subspace diagonalization*, SIAM (2022)
- Hatano & Suzuki, *Finding Exponential Product Formulas of Higher Orders* (2005)

### License

© IBM Corp., 2017–2026
