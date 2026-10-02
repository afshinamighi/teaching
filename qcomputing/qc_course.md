# Introduction to Quantum Computing for Software Engineers

## Course Introduction

Quantum computing introduces a different model of computation in which information is represented and manipulated using quantum states. For software engineers, understanding this model requires learning to think beyond classical bits and Boolean operations and becoming familiar with concepts such as qubits, superposition, quantum gates, measurement, phase, and interference.

This 2-ECTS elective provides a practical and accessible introduction to these ideas. The course deliberately keeps the mathematical and physical prerequisites to a minimum and approaches quantum computing primarily from a programmer’s perspective. Students gradually move from classical bits to single- and two-qubit systems, experiment with the $X$, $H$, and CNOT gates, construct quantum circuits, and execute them using Python and Qiskit.

Deutsch’s algorithm is used as the central example of quantum algorithm design. Students first learn to construct and execute the algorithm and then investigate the mechanisms behind it, particularly quantum parallelism, phase kickback, and interference. In this way, the course connects the basic building blocks of quantum computing to an example in which a quantum algorithm solves a specific problem with fewer oracle queries than a deterministic classical solution.

The course is designed with a practice-first approach, allowing students to experience the practical aspects of quantum computing as early as possible. Mathematical and theoretical concepts are introduced alongside hands-on activities, such as constructing quantum circuits, executing them in Qiskit, and inspecting the resulting quantum states. This allows students to connect new theoretical concepts directly to their implementation and observable effects.

### Outcome

By the end of the course, students should be able to connect the main concepts introduced throughout the course: classical bits, qubits, superposition, quantum gates, two-qubit systems, quantum oracles, Deutsch’s algorithm, quantum parallelism, and phase kickback. The intended outcome is not simply that students can implement a quantum algorithm, but that they can explain why it works and recognize how quantum computational reasoning differs from classical computation.

To keep the mathematical requirements accessible, this introductory course works with real-valued amplitudes only. Although quantum computing generally requires complex numbers, the selected examples and algorithms can be understood without developing the full complex-number formalism. Similarly, only the linear algebra needed to understand the presented quantum states and operations is introduced.

After completing the course, students should have a conceptual and practical foundation for continuing with more advanced quantum computing topics. To progress further, however, they will need to strengthen their knowledge of linear algebra and complex-number calculations, as these become increasingly important when studying more general quantum states, gates, and algorithms.

## Target Group

This elective is designed for students with a software engineering or computer science background who want a first practical introduction to quantum computing and quantum programming.

No previous knowledge of quantum mechanics or quantum computing is expected. The emphasis is on computational thinking and experimentation rather than on the underlying physics.

Required Background

Students are expected to be comfortable with basic programming in Python and with fundamental concepts from classical computing, including bits, Boolean operations, and functions.

Only basic mathematics is required. Students should be familiar with elementary algebra and simple two-dimensional vectors and matrices. The necessary mathematical concepts are reviewed when they are needed. Previous study of quantum physics or advanced linear algebra is not required.

## Learning Objectives

After successfully completing this course, the student can:

1. Represent and manipulate simple one- and two-qubit quantum states using computational basis notation and the $X$, $H$, and CNOT gates, and construct and execute the corresponding quantum circuits using Qiskit.
2. Explain and demonstrate how Deutsch’s algorithm uses quantum computational principles—particularly superposition, quantum parallelism, and phase kickback to distinguish constant from balanced functions using fewer oracle queries than a deterministic classical approach.

### Six-Week Programme

#### Week 1 — From Classical Bits to Qubits

We begin with familiar concepts from classical computing and use them as a bridge toward quantum information.

**Topics:** classical bits and Boolean operations; vectors as representations of states; computational basis states $\ket{0}$ and $\ket{1}$; the qubit as a quantum state; amplitudes and measurement; and an intuitive introduction to superposition.

**Practice:** representing simple qubit states mathematically and experimenting with single-qubit states using Python and Qiskit.

⸻

#### Week 2 — Manipulating a Qubit

Students learn how quantum gates transform quantum states.

**Topics:** quantum gates as transformations; the $X$ gate; the Hadamard $H$ gate; normalization; the states $\ket{+}$ and $\ket{-}$; measurement probabilities; and an introductory view of unitary transformations.

**Practice:** constructing, running, and inspecting simple single-qubit circuits with $X$ and $H$.

⸻

#### Week 3 — Working with Two Qubits

The computational model is extended from one qubit to two qubits.

**Topics:** two-qubit computational basis states $\ket{00}$, $\ket{01}$, $\ket{10}$, and $\ket{11}$; tensor products at an intuitive level; two-qubit superposition; the CNOT gate; control and target qubits; and statevector representation of two-qubit systems.

**Practice:** constructing two-qubit circuits, applying combinations of $X$, $H$, and CNOT gates, and interpreting the resulting states.

⸻

#### Week 4 — From Classical Functions to Quantum Oracles

Students connect familiar programming concepts such as functions and XOR with their quantum counterparts.

**Topics:** classical functions $f(x)$; reversible quantum operations; representing a function as a quantum oracle $U_f$; the transformation $\ket{x}\ket{y}\rightarrow\ket{x}\ket{y\oplus f(x)}$; constructing simple oracles; and the relationship between CNOT and the function $f(x)=x$.

**Practice:** implementing and testing different one-bit function oracles in Qiskit and inspecting their effects on quantum states.

⸻

#### Week 5 — From Classical Limits to Quantum Power: Deutsch’s Algorithm

Students now combine the concepts from the previous weeks into their first complete quantum algorithm.

**Topics:** the Deutsch problem; constant and balanced functions; the deterministic classical approach; construction of the quantum solution; preparation of the input and auxiliary qubits; application of $U_f$; the final Hadamard gate; measurement; and comparison with the classical approach.

**Practice:** reconstructing Deutsch’s algorithm, implementing constant and balanced oracles, executing the circuits in Qiskit, and verifying the final measurement results.

⸻

#### Week 6 — Behind Quantum Power: Parallelism and Phase Kickback

The final week revisits Deutsch’s algorithm to understand more deeply why it works.

**Topics:** quantum parallelism; eigenstates at an intuitive level; global and relative phase; phase kickback; interference; and how these mechanisms appear inside Deutsch’s algorithm.

**Practice:** inspecting intermediate quantum states, experimenting with phase kickback using $\ket{-}$, tracing the effect of different oracles, and explaining how phase information is eventually converted into a measurable result.

## Study Material

- [QC: A Quick Introduction for Programmers](readme.md)