# Quantum Computing and Cryptographic Applications

![Python](https://img.shields.io/badge/python-3.9-blue.svg)
![Cirq](https://img.shields.io/badge/cirq-1.3+-orange.svg)
![Qiskit](https://img.shields.io/badge/qiskit-0.45+-purple.svg)
![Sympy](https://img.shields.io/badge/sympy-1.12-green.svg)

## Overview

This project explores the foundations of quantum computing with a focus on cryptographic applications, particularly Shor's algorithm for integer factorization and its implications for RSA encryption security. Quantum computing represents a shift from classical computation through properties like superposition and entanglement. Unlike classical bits that exist in states 0 or 1, qubits exist in superposition states described by $|\psi\rangle = \alpha|0\rangle + \beta|1\rangle$, where $\alpha$ and $\beta$ are complex amplitudes satisfying $|\alpha|^2 + |\beta|^2 = 1$. For an $n$-qubit system, this allows simultaneous access to $2^n$ basis states, which becomes relevant for problems like integer factorization.

## Theoretical Background

### Quantum Gates and Circuits

We implemented quantum circuits using fundamental gates. The Hadamard gate creates superpositions:

$$H = \frac{1}{\sqrt{2}}\begin{pmatrix} 1 & 1 \\ 1 & -1 \end{pmatrix}$$

The CNOT gate creates entanglement between qubits. For a two-qubit system, it's represented as:

$$\text{CNOT} = \begin{pmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 0 & 1 \\ 0 & 0 & 1 & 0 \end{pmatrix}$$

The Quantum Fourier Transform (QFT) extracts periodicity from quantum states. For an $n$-qubit system:

$$\text{QFT}|j\rangle = \frac{1}{\sqrt{2^n}} \sum_{k=0}^{2^n-1} e^{2\pi ijk/2^n}|k\rangle$$

This transformation is central to Shor's algorithm, enabling period-finding in polynomial time.

### RSA and Factorization

RSA security depends on the difficulty of factoring the product $n = p \times q$ of two large primes. Key generation involves computing $\phi(n) = (p-1)(q-1)$, selecting a public exponent $e$ coprime to $\phi(n)$, and calculating the private exponent $d \equiv e^{-1} \pmod{\phi(n)}$. Encryption transforms plaintext $m$ into ciphertext $c \equiv m^e \pmod{n}$, while decryption recovers $m \equiv c^d \pmod{n}$.

The best classical factorization algorithms have subexponential complexity $O(\exp((64/9)^{1/3} (\ln n)^{1/3} (\ln \ln n)^{2/3}))$. Shor's algorithm reduces this to polynomial time $O((\log n)^3)$ by finding the period of the modular exponentiation function $f(x) = a^x \bmod n$.

## Implementation

### RSA Encryption

We implemented two versions of RSA. The simple version uses SymPy for educational purposes:
```python
def generate_keypair(bits):
    p = sympy.randprime(2**(bits//2), 2**(bits//2 + 1))
    q = sympy.randprime(2**(bits//2), 2**(bits//2 + 1))
    n = p * q
    phi = (p-1) * (q-1)
    
    e = random.randrange(1, phi)
    while sympy.gcd(e, phi) != 1:
        e = random.randrange(1, phi)
    
    d = sympy.mod_inverse(e, phi)
    return ((e, n), (d, n))
```

This function generates primes $p$ and $q$, computes $n$ and $\phi(n)$, then selects $e$ and calculates $d$ as the modular inverse of $e$ modulo $\phi(n)$.

### Shor's Algorithm with Cirq

The quantum phase of Shor's algorithm requires two registers: an $n$-qubit target register and a $2n+3$ qubit exponent register. The core operation is modular exponentiation:
```python
class ModularExp(cirq.ArithmeticGate):
    def apply(self, *register_values: int) -> int:
        target, exponent, base, modulus = register_values
        if target >= modulus:
            return target
        return (target * base**exponent) % modulus
```

This gate implements $U_a|y\rangle = |ay \bmod n\rangle$ controlled by the exponent register. The circuit construction combines initialization, superposition, modular exponentiation, and QFT:
```python
def make_order_finding_circuit(x: int, n: int) -> cirq.Circuit:
    L = n.bit_length()
    target = cirq.LineQubit.range(L)
    exponent = cirq.LineQubit.range(L, 3 * L + 3)
    
    mod_exp = ModularExp([2] * L, [2] * (2 * L + 3), x, n)
    
    return cirq.Circuit(
        cirq.X(target[L - 1]),
        cirq.H.on_each(*exponent),
        mod_exp.on(*target, *exponent),
        cirq.qft(*exponent, inverse=True),
        cirq.measure(*exponent, key='exponent'),
    )
```

After measurement, we apply continued fraction expansion to extract the period $r$:
```python
def process_measurement(result: cirq.Result, x: int, n: int) -> Optional[int]:
    exponent_as_integer = result.data["exponent"][0]
    exponent_num_bits = result.measurements["exponent"].shape[1]
    eigenphase = float(exponent_as_integer / 2**exponent_num_bits)
    
    f = fractions.Fraction.from_float(eigenphase).limit_denominator(n)
    
    if f.numerator == 0:
        return None
    
    r = f.denominator
    if x**r % n != 1:
        return None
    return r
```

### Limitations

We initially attempted implementation with Qiskit but encountered circuit optimization issues. Cirq provided better control over gate decomposition. However, simulating quantum systems requires exponentoial classical resurces, representing an $n$-qubit state needs $2^n$ complex amplitudes. Our simulations succeeded for numbers up to five digits (~17 bits) but exhausted memory beyond that. Factoring cryptographically relevant 2048-bit RSA keys would require millions of physical qubits with error rates below $10^{-15}$, far from current $\sim10^{-3}$ rates.

## Results

The implementation successfully demonstrated Shor's algorithm for small integers. For example, factoring $n = 46875$ yielded $p = 3$ and $q = 15625$ using quantum order-finding. While this doesn't approach real-world key sizes, it verified the algorithm's correctness and highlighted the gap between theoretical quantum advantage and practical implementation.

## References

1. Shor, P. W. (1994). "Algorithms for quantum computation: Discrete logarithms and factoring" *FOCS 1994*, pp. 124-134.
2. Nielsen & Chuang (2010). *Quantum Computation and Quantum Information*. Cambridge University Press.

---

*Project completed as part of Master 1 curriculum at Université Clermont Auvergne.*
