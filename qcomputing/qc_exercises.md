qc_exercises

20 Ket-Algebra Exercises

Single-qubit — fundamentals

1. Write $\ket{0}$ and $\ket{1}$ as column vectors.
2. Expand the following state in the computational basis:
    $$
    \ket{+}=\frac{1}{\sqrt{2}}(\ket{0}+\ket{1}).
    $$
    Then write it as a column vector.
3. Starting from the definition of $X$, calculate:
    $$
    X\ket{0},\qquad X\ket{1}.
    $$
4. Calculate and simplify:
    $$
    X\left(\frac{1}{\sqrt{2}}(\ket{0}+\ket{1})\right).
    $$
5. Calculate and simplify:
    $$
    X\left(\frac{1}{\sqrt{2}}(\ket{0}-\ket{1})\right).
    $$
    Express your answer using $\ket{+}$ or $\ket{-}$ if possible.
6. Using
    $$
    H\ket{0}=\frac{\ket{0}+\ket{1}}{\sqrt{2}},
    \qquad
    H\ket{1}=\frac{\ket{0}-\ket{1}}{\sqrt{2}},
    $$
    calculate:
    $$
    H\ket{+}.
    $$
7. Calculate and simplify:
    $$
    H\ket{-}.
    $$
8. Work from right to left and calculate:
    $$
    HX\ket{0}.
    $$
    Then calculate
    $$
    XH\ket{0}.
    $$
    Are the results the same?

Two-qubit — tensor products and gates

9. Expand the following state completely in the two-qubit computational basis:
    $$
    \ket{+}\ket{0}.
    $$
10. Expand and simplify:
    $$
    \ket{+}\ket{-}.
    $$
    Your final answer should contain only $\ket{00}$, $\ket{01}$, $\ket{10}$, and $\ket{11}$.
11. Starting from $\ket{00}$, calculate:
    $$
    (H\otimes I)\ket{00}.
    $$
12. Calculate:
    $$
    (I\otimes X)\frac{\ket{00}+\ket{10}}{\sqrt{2}}.
    $$
13. Let the first qubit be the control and the second qubit the target. Calculate:
    $$
    \operatorname{CNOT}
    \left(
    \frac{\ket{00}+\ket{10}}{\sqrt{2}}
    \right).
    $$
14. Calculate the complete transformation:
    $$
    \ket{00}
    \xrightarrow{H\otimes I}
    ?
    \xrightarrow{\operatorname{CNOT}}
    ?
    $$
    Expand and simplify the state after each operation.
15. Consider
    $$
    \ket{\psi}=\ket{+}\ket{-}.
    $$
    First expand $\ket{\psi}$ in the computational basis. Then apply a CNOT with the first qubit as control and the second as target. Simplify the result and, if possible, factor it back into two single-qubit states.
16. Starting with $\ket{00}$, calculate and simplify:
    $$
    (H\otimes H)(X\otimes I)\ket{00}.
    $$
    Do not perform matrix multiplication. Instead, use the known actions of $X$ and $H$ on $\ket{0}$ and $\ket{1}$.

Three-qubit — combining the techniques

For the following exercises, use the ordering

$$
\ket{q_0q_1q_2}.
$$

17. Expand completely:
    $$
    \ket{+}\ket{0}\ket{1}.
    $$
18. Expand completely and simplify:
    $$
    \ket{+}\ket{-}\ket{0}.
    $$
    Your final expression should contain only three-qubit computational basis states such as $\ket{000}$ and $\ket{010}$.
19. Start with
    $$
    \ket{000}.
    $$
    Apply $H$ to $q_0$, followed by a CNOT with $q_0$ as control and $q_1$ as target:
    $$
    \ket{000}
    \xrightarrow{H_0}
    ?
    \xrightarrow{\operatorname{CNOT}_{0,1}}
    ?
    $$
    Expand and simplify the state after each step. What happens to $q_2$?
20. Challenge. Start with
    $$
    \ket{000}.
    $$
    Apply the following operations in order:

$$
X_2,\qquad
H_0,\qquad
H_2,\qquad
\operatorname{CNOT}_{0,2},\qquad
H_0.
$$