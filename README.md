# Quantum Computing — Noisy Circuit Simulation with Qiskit

A Jupyter notebook project that builds a small entangling quantum circuit, runs it on Qiskit's ideal and noisy Aer simulators, and submits the same circuit to IBM Quantum Runtime backends. The notebook compares ideal vs. noisy measurement outcomes and visualizes the underlying quantum state with histograms, Q-spheres, density matrices, city plots, and Bloch spheres.

---

## Table of Contents

1. [Overview](#overview)
2. [Project Structure](#project-structure)
3. [Requirements](#requirements)
4. [Installation](#installation)
5. [IBM Quantum Account Setup](#ibm-quantum-account-setup)
6. [Running the Notebook](#running-the-notebook)
7. [What the Notebook Does](#what-the-notebook-does)
8. [Circuit Description](#circuit-description)
9. [Noise Model](#noise-model)
10. [Visualizations Produced](#visualizations-produced)
11. [Security Notice](#security-notice)
12. [Troubleshooting](#troubleshooting)
13. [References](#references)

---

## Overview

This project demonstrates a basic end-to-end workflow for working with quantum circuits on real and simulated quantum hardware using IBM's Qiskit SDK:

- Construct a 4-qubit entangling circuit using Hadamard, CNOT, and CZ gates.
- Simulate the circuit on a noiseless `AerSimulator` to obtain the ideal probability distribution of measurement outcomes.
- Build a custom **bit-flip noise model** and run the circuit on a noisy simulator to see how errors degrade the result.
- Submit the same circuit to IBM Quantum Runtime backends (`simulator_statevector`, `ibmq_qasm_simulator`) using the `SamplerV2` primitive.
- Inspect the underlying quantum state through state-vector representations, density matrices, and several visualizations (histogram, Q-sphere, city plot, Bloch sphere).

It is intended as a learning resource for students and newcomers to quantum computing who want to understand circuit construction, noise modeling, and the difference between simulation and execution on cloud backends.

---

## Project Structure

```
Quantum-Computing/
├── quantum.ipynb       # Main Jupyter notebook with the full workflow
├── QC_Project.docx     # Project write-up / report document
├── README.md           # This file
└── README.txt          # Original short install notes
```

---

## Requirements

- **Python** 3.10 or newer (the notebook was authored with Python via Miniconda)
- **Jupyter Notebook** or **JupyterLab**
- An **IBM Quantum account** (free tier is sufficient) if you want to run cells that submit jobs to IBM hardware/cloud simulators

### Python packages

The notebook depends on the following packages (versions shown are those captured at the end of the notebook's `pip list`):

| Package               | Version  |
| --------------------- | -------- |
| `qiskit`              | 1.0.2    |
| `qiskit-aer`          | 0.14.1   |
| `qiskit-ibm-runtime`  | 0.23.0   |
| `numpy`               | 1.26.4   |
| `matplotlib`          | 3.8.4    |
| `seaborn`             | 0.13.2   |
| `pylatexenc`          | 2.10     |
| `jupyter` / `ipykernel` | latest |

---

## Installation

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd Quantum-Computing
```

### 2. (Recommended) Create an isolated environment

Using **conda**:

```bash
conda create -n quantum python=3.11
conda activate quantum
```

Or using **venv**:

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate
```

### 3. Install the dependencies

```bash
pip install qiskit qiskit-aer qiskit-ibm-runtime numpy matplotlib seaborn pylatexenc jupyter
```

> `pylatexenc` is required by Qiskit's `mpl` circuit drawer to render gate labels.

#### Original install notes (from `README.txt`)

```
========================libraries the must be installed===================
pip install qiskit
pip install matplotlib
pip install jupyter
pip install numpy
pip install qiskit_ibm_runtime
pip install qiskit_aer
pip install seaborn
--refer to the pip list at the end of the notebook
==========assuming you have python already installed :)=======================
```

---

## IBM Quantum Account Setup

The notebook uses `QiskitRuntimeService` to talk to IBM Quantum cloud backends. You will need a personal API token.

1. Sign up / log in at <https://quantum.ibm.com/>.
2. Copy your API token from your account dashboard.
3. Save it locally **once** with:

   ```python
   from qiskit_ibm_runtime import QiskitRuntimeService

   QiskitRuntimeService.save_account(
       channel="ibm_quantum",
       token="<YOUR_PERSONAL_API_TOKEN>",
       set_as_default=True,
       overwrite=True,
   )
   ```

Once saved, future sessions can simply do `service = QiskitRuntimeService()` without re-entering the token.

> **Important:** Do **not** commit your real token to the repository. See the [Security Notice](#security-notice) below.

---

## Running the Notebook

From the project directory, launch Jupyter:

```bash
jupyter notebook quantum.ipynb
```

or, with JupyterLab:

```bash
jupyter lab quantum.ipynb
```

Run the cells top-to-bottom. Cells that submit jobs to IBM Runtime require an active internet connection and a configured account.

---

## What the Notebook Does

The notebook is organized as a sequence of small experiments:

1. **Account configuration** — Initializes `QiskitRuntimeService` and (optionally) saves your IBM Quantum credentials.
2. **Noise model construction** — Defines a single-qubit bit-flip error with probability `p = 0.20` and tensors it for two-qubit gates. The model is attached to `cx`, `cz`, and `h` gates.
3. **Circuit construction** — Builds a 4-qubit, 2-classical-bit circuit using a register-based API.
4. **Ideal simulation** — Runs the circuit on `AerSimulator()` with no noise and plots the resulting histogram.
5. **Noisy simulation** — Transpiles the circuit for the noisy backend (`optimization_level=3`) and plots the degraded distribution.
6. **Cloud execution** — Submits the circuit to IBM Runtime simulators using `SamplerV2`, retrieves results by job ID, and plots their counts.
7. **State analysis** — Re-creates the unmeasured circuit, extracts its `Statevector`, evolves it, computes the resulting `DensityMatrix`, and visualizes it.

---

## Circuit Description

The circuit acts on a 4-qubit quantum register `q` and a 2-bit classical register `c`:

```
q_0: ──H──■───────────────────────X──
          │                       │
q_1: ──H──X──■───────────■────────┼───M(c_0)
             │           │        │
q_2: ──H─────X──■────────┼────────■───M(c_1)
                │        │
q_3: ──H────────Z────────X────────────
```

Operations applied (in order):

1. **Hadamard** on all four qubits — creates a uniform superposition.
2. **CNOT** `q_0 → q_1`
3. **CNOT** `q_1 → q_2`
4. **CZ**   `q_2 ↔ q_3`
5. **CNOT** `q_1 → q_3`
6. **CNOT** `q_2 → q_0`
7. **Measure** `q_1 → c_0`, `q_2 → c_1`

The result is a highly entangled 4-qubit state of which only two qubits are measured.

---

## Noise Model

A **bit-flip channel** is constructed with Pauli error probabilities:

- Single-qubit channel: applies `X` with probability `p = 0.20`, otherwise identity `I`.
- Two-qubit channel: tensor product of two independent single-qubit bit-flip channels.

The channel is attached to the basis gates as follows:

| Gate         | Error channel              |
| ------------ | -------------------------- |
| `h`          | Single-qubit bit-flip      |
| `cx`, `cz`   | Two-qubit bit-flip (tensor)|

This produces a `NoiseModel` with basis gates `['cx', 'cz', 'h', 'id', 'rz', 'sx']`.

---

## Visualizations Produced

The notebook generates the following visualizations:

- **Circuit diagram** (`qc.draw('mpl')`) — pretty matplotlib rendering of the quantum circuit.
- **Histogram** (`plot_histogram`) — measurement-outcome distributions for both the ideal and noisy runs and for IBM cloud results.
- **Q-sphere** (`Statevector.draw('qsphere')`) — geometric view of the full multi-qubit state.
- **Latex matrix** (`Statevector.draw('latex')`) — symbolic rendering of the state vector.
- **City plot** (`DensityMatrix.draw('city')`) — 3D bar plot of the real and imaginary parts of the density matrix.
- **Bloch sphere** (`DensityMatrix.draw('bloch')`) — Bloch-sphere representation per qubit.

---

## Security Notice

> ⚠️ **The current `quantum.ipynb` contains a hard-coded IBM Quantum API token.**
>
> If this notebook has ever been committed to a public repository, **treat the token as compromised and revoke it immediately** at <https://quantum.ibm.com/account>.
>
> Recommended hygiene going forward:
>
> - Never paste real tokens into notebooks. Use `QiskitRuntimeService.save_account(...)` once locally, then call `QiskitRuntimeService()` with no arguments.
> - Add `*.ipynb_checkpoints/` and any local credential files to `.gitignore`.
> - Consider clearing notebook outputs (`Cell → All Output → Clear`) before committing.

---

## Troubleshooting

**`ModuleNotFoundError: No module named 'pylatexenc'`**
Install it with `pip install pylatexenc`. It is required by Qiskit's `mpl` circuit drawer.

**`name 'noise_bit_flip' is not defined`**
The noisy-simulator cell references `noise_bit_flip`, but the noise model in the previous cell is named `noise_model`. Either rename one to match the other or change the line to:

```python
sim_noise = AerSimulator(noise_model=noise_model)
```

**Cloud simulators deprecated warning**
`simulator_statevector` and `ibmq_qasm_simulator` were deprecated in May 2024. Use Qiskit Runtime's local testing mode (`qiskit-ibm-runtime ≥ 0.22`) or submit to a real backend such as `ibm_brisbane` / `ibm_kyoto`.

**`QiskitRuntimeService` cannot find an account**
Run `QiskitRuntimeService.save_account(...)` once with your token, or pass `channel` and `token` directly to the constructor.

---

## References

- [Qiskit documentation](https://docs.quantum.ibm.com/)
- [Qiskit Aer noise module](https://qiskit.github.io/qiskit-aer/apidocs/aer_noise.html)
- [IBM Quantum Runtime](https://docs.quantum.ibm.com/run)
- [Qiskit visualization tools](https://docs.quantum.ibm.com/build/visualizing-quantum-circuits)
