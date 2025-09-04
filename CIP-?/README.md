---
CIP: ?
Title: Plutus support for Montgomery multiplication over BLS12-381 scalar field
Status: Proposed
Category: Plutus
Authors:
    - Ilia Rodionov <hey@euony.me>
Implementors: []
Discussions:
    - https://github.com/cardano-foundation/CIPs/pull/?
Created: 2025-09-03
License: CC-BY-4.0
---

## Abstract

The Chang upgrade in Cardano brought the support for BLS12-381 elliptic curve with 
by [CIP-0381](https://cips.cardano.org/cip/CIP-0381)
with a set of operations over $G1$ and $G2$ groups.
However, some cryptographic protocols need to perform extensive arithmetics 
over the _scalar field_ of BLS12-381 curve.
The only way to do this in Plutus nowadays is regular arithmetic, 
which is not well-suited for this task.
This may lead to unnecessary use of resources on validating nodes and drains transaction limits.
This CIP proposes extending Plutus to provide an efficient implementation for such cases
by introducing an _opaque type_ for Montgomery representation of integers
and a set of primitives to work with them.


## Motivation: why is this CIP necessary?

As a motivating example, let's consider a more efficient and less error-prone replacement 
for what traditionally was done using Merkle trees or similar structures, e.g.
[merkle-patricia-forestry](https://github.com/aiken-lang/merkle-patricia-forestry).

Pairing-based cryptographic accumulators [[SKBP22]](https://dl.acm.org/doi/pdf/10.1145/3548606.3560676) 
provide similar functionality with some additional performance perks and elegance, mostly due to 
_batch membership proofs_ that turn out to be new updated accumulator commitments
at the same time.

To check (usually) an offchain-calculated membership proof, the validator needs to calculate
$N+1$ coefficients $c_i$ for a _final polynomial_ $P(x)$ 
by multiplying given $N$ _normalized binomials_ $B_i$:

$$
\begin{align}
B_i(x) = x + a_i \qquad (i=1,\dots,n) \\
P(x) = \prod_{i=1}^N B_i(x) = \sum_{i=0}^{N} c_i \cdot x^i
\end{align}
$$

There are two base algorithms to solve this task.
The simplest is _naive (schoolbook) convolution_ that comes with $O(n^2)$ complexity.
Plutus implementations of
[bilinear accumulators](https://github.com/perturbing/plutus-accumulator)
use this approach in both 
[Haskell](https://github.com/perturbing/plutus-accumulator/blob/main/plutus-accumulator/src/Plutus/Crypto/BlsUtils.hs#L499-L505)
and [Aiken](https://github.com/perturbing/plutus-accumulator/blob/main/aiken-bilinear-accumulator/lib/aiken_bilinear_accumulator/poly.ak#L3-L13).

The other option is 
_[Number Theoretic Transform (NTT)](https://wiki.algo.is/Number%20theoretic%20transform)_, which achieves 
sub-quadratic time complexity of $O(n \cdot log^{2}n$).
It is typically used for a large enough number of products ($\sim 10^4$)
and likely is not the best candidate for the onchain setting, 
at least at the current stage. 

Both algorithms extensively use _modular multiplication_ over 
the _scalar field_ $Z_p$ of BLS12-381, which is a _finite prime field_.

For now, developers have two options:

1. The straightforward way is to use the regular arithmetic to 
multiply integers using `multiplyInteger` built-in
and then get the modulo $p$ using `modInteger` built-in. 
This approach is used in Aiken stdlib [Scalar](https://aiken-lang.github.io/stdlib/aiken/crypto/bls12_381/scalar.html#Scalar)
and in the corresponding [module](https://github.com/nau/scalus/blame/master/scalus-core/shared/src/main/scala/scalus/prelude/crypto/bls12_381/Scalar.scala) in Scalus.

2. Surprisingly, the step of taking the modulo with `modInteger`
can be skipped due to the fact that the typical sink for the results are
`bls12_381_g1_scalar_mul` and `bls12_381_g2_scalar_mul` built-ins. 
Those functions take plain integers, and their outputs are the same 
for the whole _congruence class_ of scalars.
This lows Plutus execution costs but increases the resource usage for validating nodes 
by ruining the core benefits of staying within a finite field.
Concretely, in the Aiken implementation of the bilinear accumulator mentioned above, 
the test case for the maximum number of 45 elements 
(see [PR](https://github.com/perturbing/plutus-accumulator/pull/2) for details)
demonstrates that by omitting the modulo step
one can drop the budgets roughly by 10% (mem: 140430 vs. 125542, cpu: 44438444 vs. 48159242)
by forcing the validators to operate over numbers up to 10020 bits instead of regular 255.

## Specification

TBD

### Function definition

TBD


### Cost model

TBD

## Rationale: how does this CIP achieve its goals?
Integrating these functions directly into Plutus will streamline cryptographic operations, reduce transaction costs, and uphold the integrity of existing cryptographic interfaces. It addresses current inefficiencies and enhances the cryptographic capabilities of the Plutus platform.

It will allow the implementation of complex cryptographic protocols on-chain in Plutus smart contracts, significantly expanding the capabilities of the Cardano blockchain.

## Path to Active

### Acceptance Criteria

We consider the following criteria to be essential for acceptance:

- [ ] The PR for this functionality is merged in the Plutus repository.
- [ ] This PR must include tests, demonstrating that it behaves as the specification requires in this CIP.
- [ ] A benchmarked use case is implemented in the Plutus repository, demonstrating that realistic use of this primitive does, in fact, provide major cost savings.

### Implementation Plan

- [ ] IOG Plutus team consulted and accept the proposal.
- [ ] Authors to provide preliminary benchmarks of naive vs. Montgomery multiplication for use cases in general and in Plutus:
  - https://github.com/euonymos/bench-montgomery

## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
