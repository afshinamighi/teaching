## From a Classical Function to a Quantum Oracle

The goal of this part is to make a clear connection between three ideas that can easily be mixed up when learning the Deutsch algorithm for the first time: the classical function $f$, the quantum oracle $U_f$, and the quantum circuit that implements $U_f$. We first look at what the function $f$ means as an ordinary classical function. We then see how this function is embedded into a reversible quantum operation $U_f$, and finally determine which quantum gates can implement that operation. Understanding these three levels separately makes the oracle used in the Deutsch algorithm much easier to understand.

### The Function $f$

In the Deutsch problem, we are given a function [ with this test text ] $f: \{ 0,1 \} \rightarrow \{ 0,1 \} $

The function accepts one bit as input and produces one bit as output. Since the input can only be $0$ or $1$, there are exactly four possible functions:

| Function | $f(0)$ | $f(1)$ | Type     |
|----------|--------|--------|----------|
| $f_0$    | $0$    | $0$    | Constant |
| $f_1$    | $1$    | $1$    | Constant |
| $f_2$    | $0$    | $1$    | Balanced |
| $f_3$    | $1$    | $0$    | Balanced |

A function is constant when it produces the same output for both possible inputs. It is balanced when half of its inputs produce $0$ and the other half produce $1$. Since there are only two possible inputs here, a balanced function produces a different output for $0$ and $1$.

For example, $f_2$ can simply be written as $f_2(x)=x$, because $f_2(0)=0$ and $f_2(1)=1$.

#### $f$ Is Not the Same Thing as $U_f$

An important distinction is that the function $f$ itself is not the quantum oracle.
If we think classically, we might imagine implementing $f$ as

$$
x\longrightarrow f(x).
$$

For example, if $f(x)=0$, both possible inputs would be mapped to $0$:

$$
0\longrightarrow0
$$

and

$$
1\longrightarrow0.
$$

This is perfectly acceptable in ordinary classical computation. However, this transformation loses information: after seeing the output $0$, we cannot determine whether the original input was $0$ or $1$.

Quantum operations must be reversible. Therefore, we cannot generally implement an arbitrary classical function by simply replacing $x$ with $f(x)$.

Instead, we introduce a second qubit, $y$, and construct a reversible operation called $U_f$:

$$
U_f\ket{x,y}=\ket{x,y\oplus f(x)},
$$

where $\oplus$ represents XOR.

Notice the important difference:

$$
f:x\longrightarrow f(x)
$$

describes the classical function, while

$$
U_f:\ket{x,y}\longrightarrow\ket{x,y\oplus f(x)}
$$

describes the reversible quantum operation that contains the behaviour of $f$.

The input $x$ is preserved, while the value $f(x)$ is XORed with the second qubit.

#### Why Can This Be Confusing?

This distinction can feel unusual at first because we are used to ignoring similar distinctions in classical programming.

Consider a simple classical function:
```python
def double(x):
    return 2 * x
```
We naturally say things such as “call `double(3)`” or “apply the `double` function.” But several different concepts are involved. `double` is the definition of a function, `double(3)` is an invocation of that function with a particular input, and the CPU instructions executed by the computer are the physical implementation of that function which at the end there are a sequence of logical gates that are actually producing the results.

Another simple example is the classical NOT operation. We may define it mathematically as

$$
f(0)=1,\qquad f(1)=0.
$$

We can describe this as a truth table, write it as a Boolean expression, implement it using a NOT gate, or implement it using a programming-language expression such as `not x`. These are different representations of the same intended behaviour.

Experienced programmers usually move between these levels without thinking much about them. For a beginner, however, the distinction between a function definition, calling the function, and the mechanism implementing the function can initially be confusing.

Something similar happens in quantum computing, but the distinction becomes more visible because an ordinary classical function $f$ cannot necessarily be used directly as a quantum operation. **We have to construct a reversible operation $U_f$ and then find a quantum circuit that implements $U_f$**.

It is therefore useful to think in three levels:

1. What does $f$ calculate?
2. What transformation must $U_f$ perform?
3. Which quantum gates implement that transformation?

Let us apply this reasoning to two of the four possible functions.

#### Example 1: The Balanced Function $f_2$

Consider

$$
f_2(0)=0,\qquad f_2(1)=1.
$$

This can be written simply as

$$
f_2(x)=x.
$$

The function itself is balanced because its two possible inputs produce different outputs.

Now substitute $f_2(x)=x$ into the definition of the oracle: $U_{f_2}\ket{x,y} = \ket{x,y\oplus f_2(x)}$.

Therefore, $U_{f_2}\ket{x,y} = \ket{x,y\oplus x}$.

Let us examine all four possible computational basis states.

When $x=0$,

$$
\ket{00}\longrightarrow\ket{00}
$$

and

$$
\ket{01}\longrightarrow\ket{01}.
$$

The second qubit does not change because $y\oplus0=y$

When $x=1$,

$$
\ket{10}\longrightarrow\ket{11}
$$

and

$$
\ket{11}\longrightarrow\ket{10}.
$$

The second qubit is flipped because $y\oplus1=\neg y$

We can therefore describe the behaviour of the oracle as: If $x=0$, leave $y$ unchanged, but if $x=1$, flip $y$.

This is exactly the behaviour of a CNOT gate with $x$ as the control qubit and $y$ as the target qubit.

We have therefore moved through our three levels:

We started with $f_2(x)=x$ and then we formulated $U_{f_2}\ket{x,y}=\ket{x,y\oplus x}$ and then we chose the CNOT gate with $x$ controlling $y$ to implement $U_f$.

This is an important point: **the CNOT gate is not the function $f_2$. The CNOT gate is a circuit implementation of the reversible oracle $U_{f_2}$ that encodes the behaviour of $f_2$**.

#### Example 2: The Constant Function $f_0$

Now consider: $f_0(0)=0,\qquad f_0(1)=0$.

This can be written as $f_0(x)=0$.

This function is constant because it always returns $0$, independently of its input.

Again, substitute the function into the oracle definition (we need a reversible transformation): $U_{f_0}\ket{x,y} = \ket{x,y\oplus f_0(x)}$

Since $f_0(x)=0$, therefore $U_{f_0}\ket{x,y}=\ket{x,y\oplus0}$.

XOR with $0$ leaves a bit unchanged: $y\oplus0=y$. Therefore, $U_{f_0}\ket{x,y} = \ket{x,y}$.

Checking all four computational basis states gives

$$
\ket{00}\longrightarrow\ket{00},
$$

$$
\ket{01}\longrightarrow\ket{01},
$$

$$
\ket{10}\longrightarrow\ket{10},
$$

and

$$
\ket{11}\longrightarrow\ket{11}.
$$

Nothing changes.

Therefore, the oracle for $f_0$ is simply the identity operation: $U_{f_0}=I$.

No gate is required to change either qubit.

Again, we can see the three levels clearly:
from $f_0(x)=0$, we concluded $U_{f_0}\ket{x,y}=\ket{x,y\oplus0}=\ket{x,y}$ and then we chose the Identity operation (does nothing).

Exercises: Constructing the Remaining Oracles

We have now constructed the oracle for one balanced function, $f_2$, and one constant function, $f_0$. Try to construct the remaining two yourself.

Exercise 1: Follow the same steps and define $U_{f_1}$ and $U_{f_3}$. Then, implement the oracles with proper quantum gates.

## Summary 

Keeping these three levels separate is especially useful when studying the rest of the Deutsch algorithm. Once the oracle is placed between Hadamard gates and its input can be in superposition, exactly the same $U_f$ operates on a quantum state containing multiple computational-basis components. That is where the quantum part of the algorithm becomes particularly interesting.