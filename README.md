# Distributed Quantum Circuit Partitioning
### An implementation of the circuit partitioning framework from Ferrari et al. (arXiv:2305.02969)

---

## What is this project?

Quantum computers today are small. Current NISQ devices have maybe a few hundred qubits at best, and most practical quantum algorithms need far more than that. One solution the research community is actively exploring is **Distributed Quantum Computing (DQC)** — instead of one big quantum processor, you have multiple smaller QPUs connected over a quantum network, cooperating to run a circuit that none of them could handle alone.

But here is the catch: every time a two-qubit gate needs to act on qubits sitting on different QPUs, you need to consume an **EPR pair** — a shared entangled state that has to be generated and distributed across a quantum channel. That is expensive. Really expensive. So the key question becomes: **how do you split the circuit across QPUs in a way that minimizes the number of these cross-QPU gates?**

That is the circuit partitioning problem, and that is what this project implements.

The implementation is based on the research paper:

> **Ferrari et al., "A Modular Quantum Compilation Framework for Distributed Quantum Computing"**
> arXiv:2305.02969 — [Read the paper](https://arxiv.org/abs/2305.02969)

---

## The Full Workflow

The notebook walks through the following pipeline, step by step.

### Step 1 — Turning the circuit into a graph

The first step takes a quantum circuit (loaded from a QASM file) and converts it into a **weighted interaction graph** using NetworkX. The qubits are the nodes, and the edges represent two-qubit gates between them. The edge weight is the total number of two-qubit gates between a pair of qubits. Heavier edge = more interaction = more expensive to split those two qubits across different QPUs.

Qiskit makes it easy to iterate through the circuit's instructions and extract qubit indices. The graph visualization also allows you to visually sanity-check the partitioning for smaller circuits before scaling to larger ones.

### Step 2 — Partitioning with METIS

Once the interaction graph is built, it is fed into **METIS** (via PyMETIS) — a multilevel k-way graph partitioning algorithm. METIS is designed to partition a graph into k roughly equal parts while minimising the total weight of edges crossing partition boundaries. That maps perfectly onto this problem: minimise inter-QPU gate cost.

You give METIS your adjacency structure, edge weights, and the number of partitions (QPUs), and it gives back a partition assignment for every node.

The initial METIS partition is a good starting point — but it is not optimal. It does not know anything about the actual hardware constraints of each QPU, like how many communication qubits each one has. This is where the next steps come in.

### Step 3 — Enforcing hardware topology constraints

Each QPU in a real DQC architecture has a fixed number of **data qubits** and **communication qubits**. The constraint enforced here is:

```
communication_qubits(QPU_i) >= data_qubits_needing_remote_gates(QPU_i)
sum(data_qubits across all QPUs) >= total circuit qubits
```

If you violate this, the partition is physically infeasible — you literally do not have enough quantum channels to execute the required remote operations.

The `tpwgts` parameter in METIS is used to weight target partition sizes proportionally based on each QPU's data qubit capacity. Beyond that, a balancing step shifts qubits from overloaded QPUs to underutilised ones. Two approaches were considered for this balancing:

- **Greedy balancing**: iterate over every overloaded QPU and all target QPUs, execute the best available move
- **Min Cost Flow formulation**: model overloaded QPUs as sources and underloaded ones as sinks — a cleaner formulation for the general case

### Step 4 — Gain-based local search refinement

The METIS partition is the starting point. The real algorithm is what happens next: a **gain-based local search loop** inspired by the Kernighan-Lin heuristic.

The idea is to look at every qubit that sits on a partition boundary — meaning it has at least one gate with a qubit in a different QPU — and ask: if I moved this qubit to a different QPU, would the total communication cost go down?

To answer that efficiently, the implementation builds a **`link_qubit_register`** — a dictionary for each qubit that tracks how much gate weight it shares with each QPU. This is the key data structure. Instead of recomputing everything from scratch each time, when a qubit moves, the register is updated for the qubit itself and all its neighbors.

The gain function computes, for a qubit `q` moving to a target partition:

```
gain = (weight of edges to target partition) - (weight of edges to current partition)
```

If the gain is positive, the move reduces communication cost and is executed.

One important design decision: the original gain function included a **balance penalty term** (penalising moves that make partition sizes unequal). Since hardware capacity constraints are already enforced explicitly, that soft penalty was interfering — causing the search to reject perfectly good moves. Setting `alpha=0` in the gain function, or removing the penalty term altogether, resolves this.

The loop runs until no qubit can be moved to reduce communication cost further, or until a safety cap of **10,000 iterations** is hit to avoid infinite loops in edge cases.

### Step 5 — Output and visualisation

After refinement, the notebook outputs:

- The **before and after communication cost** (number of weighted cross-QPU edges)
- A **visualisation of each QPU's subgraph** showing which qubits ended up where
- The final **partition assignment** mapping each qubit to a QPU

---

## What I Learned Building This (The Honest Version)

This was not a straight line. A few things that went wrong before arriving at the current version:

**The first lookahead was too greedy.** The initial implementation was a single-pass greedy scan — go through each boundary qubit once, move it if beneficial, stop. That was faster but completely missed cases where moving qubit A first would unlock a better move for qubit B. The iterative version fixed that.

**The `link_qubit_register` was hardcoded for 3 QPUs.** Embarrassingly obvious in hindsight. It had to be generalised to work for an arbitrary number of QPUs, which is what makes it actually useful for general DQC topologies.

**The `tpwgts` bug.** At one point this parameter was accidentally commented out, causing METIS to produce completely unbalanced partitions with no obvious error message. Subtle and annoying to track down.

**The balance penalty in `compute_gain`.** The alpha term felt right at first — you want balanced partitions. But since capacity is enforced as a hard constraint elsewhere, the soft penalty was double-counting and blocking valid moves. Removing it improved results noticeably.

---

## Repository Structure

```
├── DQC.ipynb                  # Main Jupyter notebook (full pipeline)
├── requirements.txt           # Python dependencies
├── sample_circuit.qasm        # Sample input circuit to test with
└── README.md                  # This file
```

---

## Getting Started

### Prerequisites

- Python 3.8 or higher
- Jupyter Notebook or JupyterLab

### Installation

Clone the repository:

```bash
git clone https://github.com/yourusername/yourreponame.git
cd yourreponame
```

Install all dependencies:

```bash
pip install -r requirements.txt
```

The `requirements.txt` includes:

```
qiskit
qiskit[visualization]
qiskit-qasm3-import
networkx
pymetis
matplotlib
pylatexenc
```

### Running the notebook

```bash
jupyter notebook DQC.ipynb
```

The notebook will prompt you for:
1. A QASM circuit file (a sample one is included in the repo)
2. The number of QPUs to partition across
3. The communication qubit capacity per QPU

A sample circuit is included (`sample_circuit.qasm`) so you can run the full pipeline immediately without needing to generate your own input.

---

## Inputs and Outputs

**Inputs:**
- A quantum circuit in OpenQASM 2.0 format (`.qasm` file)
- Number of QPUs `k`
- Communication qubit capacity per QPU

**Outputs:**
- Weighted interaction graph visualisation
- Initial METIS partition assignment and communication cost
- Refined partition assignment after local search
- Per-QPU subgraph visualisations
- Before/after communication cost comparison

---

## Example Output

After running the pipeline on a sample circuit with 3 QPUs, you should see something like:

```
Edge cut cost (METIS initial):     47
Communication cost after refinement: 31
```

Along with visualisations of each QPU's qubit subgraph showing the local connectivity within each partition.

---

## What's Next

The current implementation covers the circuit partitioning phase of the framework. The next planned steps are:

- Completing the `shift` function for all edge cases
- Implementing a **remote gate scheduling pass** that takes the partition and generates the actual TeleGate / TeleData operations
- Benchmarking against alternative partitioning methods (Fiduccia-Mattheyses, Spectral Partitioning, ILP for small circuits)
- Extending to circuits with arbitrary gate sets beyond CZ

---

## How to Contribute

Contributions are very welcome — whether that is fixing a bug, improving the algorithm, adding support for new circuit types, or just trying it on an interesting circuit and reporting what happened.

**To contribute:**

1. Fork the repository (click the Fork button on the top right of the repo page)
2. Clone your fork locally:
   ```bash
   git clone https://github.com/your-username/yourreponame.git
   ```
3. Create a new branch for your changes:
   ```bash
   git checkout -b your-feature-name
   ```
4. Make your changes in the notebook or any other file
5. Commit and push to your fork:
   ```bash
   git add .
   git commit -m "describe what you changed"
   git push origin your-feature-name
   ```
6. Open a **Pull Request** from your fork back to this repo

If you find a bug or have a suggestion but do not want to write code, please open an **Issue** — that is just as valuable.

**Some areas where contributions would be especially interesting:**
- Implementing the Fiduccia-Mattheyses algorithm as an alternative refinement step
- Adding support for hypergraph partitioning (for circuits with multi-qubit gates)
- Benchmarking on standard circuit families (QFT, VQE, Graph State circuits)
- Implementing the Min Cost Flow balancing approach
- Generalising to heterogeneous QPU topologies

---

## Reference

```
@article{ferrari2023modular,
  title={A Modular Quantum Compilation Framework for Distributed Quantum Computing},
  author={Ferrari, Davide and Carretta, Stefano and Amoretti, Michele},
  journal={arXiv preprint arXiv:2305.02969},
  year={2023}
}
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Qiskit | Circuit loading, gate extraction |
| NetworkX | Interaction graph construction and visualisation |
| PyMETIS | k-way graph partitioning |
| Matplotlib | Visualisation |
| Python 3 | Everything |

---

## License

MIT License — feel free to use, modify, and build on this.

---

*This project is part of an ongoing academic implementation effort. A LaTeX paper documenting the full implementation journey — including the wrong turns — is being written alongside the code.*
