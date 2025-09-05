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

The Chang upgrade in Cardano brought support for the BLS12-381 elliptic curve with 
by [CIP-0381](https://cips.cardano.org/cip/CIP-0381)
with a set of operations over $G1$ and $G2$ groups.
However, some cryptographic protocols need to perform extensive arithmetic 
over the _scalar field_ of BLS12-381 curve mostly for polynomial arithmetic in SNARKs or KZG commitments.
The only way to do this in Plutus nowadays is regular arithmetic, 
which is not well-suited for this task.
It may lead to unnecessary use of resources on validating nodes and drain usage limits.
This CIP proposes extending Plutus to provide an efficient implementation for such cases
by introducing an _opaque type_ for Montgomery representation of integers
and a set of primitives to work with them.

## Motivation: why is this CIP necessary?

As a motivating example, let's consider a more efficient and less error-prone replacement 
for what traditionally was done using Merkle trees or similar structures, e.g.
[merkle-patricia-forestry](https://github.com/aiken-lang/merkle-patricia-forestry).

Pairing-based cryptographic accumulators [[SKBP22]](https://dl.acm.org/doi/pdf/10.1145/3548606.3560676) 
provide similar functionality with some additional performance perks and elegance, e.g.
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
and then get the modulus $p$ using `modInteger` built-in. 
This approach is used in Aiken stdlib [Scalar](https://aiken-lang.github.io/stdlib/aiken/crypto/bls12_381/scalar.html#Scalar)
and in the corresponding [module](https://github.com/nau/scalus/blame/master/scalus-core/shared/src/main/scala/scalus/prelude/crypto/bls12_381/Scalar.scala) in Scalus.

2. The step of taking the modulus with `modInteger`
can be skipped since the typical sink for the results is
`bls12_381_g1_scalar_mul` and `bls12_381_g2_scalar_mul` built-ins. 
Those functions take plain integers, and their outputs are the same 
for the whole _congruence class_ of scalars because 
they implicitly reduce integers modulo the field size. 
This allows improving the _model performance_ and lowering the execution costs 
despite the fact Plutus uses bit-size metric for integers 
but drops the _real performance_ by ruining the core benefits of staying within a finite field.
More specifically, in the Aiken implementation of the bilinear accumulator mentioned above
the test case for the maximum number of 45 elements 
(see [PR](https://github.com/perturbing/plutus-accumulator/pull/2) for details)
demonstrates that by omitting the modulus step,
one can drop the budgets roughly by 10% (mem: 140430 vs. 125542, cpu: 44438444 vs. 48159242)
by forcing the validators to operate over numbers up to 10020 bits instead of the regular 255.

[Montgomery multiplication](https://en.wikipedia.org/wiki/Montgomery_modular_multiplication)
seems to be a viable alternative for such and similar use cases when there is a need to multiply
many scalars in one go since it's known to be much faster. 
We did preliminary benchmarks for multiplication of binomials 
to compare an optimized `blst` implementation of Montgomery multiplication
(the library which is already used in Plutus)
with naive implementation.
We used Rust bindings for `blst` and native `num-bigint` library. 
The underlying bindings are the same as used in `cardano-base` for `bslt`, 
so we can expect similar behavior for the Haskell stack.
Each benchmark was executed 1000 times on Intel(R) Core(TM) i7-8550U CPU @ 1.80GHz.
Values show average time and standard deviation.

| Size | Montgomery Avg | Montgomery σ | Naive Avg    | Naive σ      | Speedup |
|-----:|----------------|--------------|--------------|--------------|---------|
|   10 | 101.734 µs     | 22.565 µs    | 685.302 µs   | 159.436 µs   | 6.74x   |
|   15 | 138.966 µs     | 26.496 µs    | 953.121 µs   | 251.909 µs   | 6.86x   |
|   20 | 186.083 µs     | 21.703 µs    | 1.363 ms     | 189.064 µs   | 7.33x   |
|   25 | 241.934 µs     | 30.409 µs    | 2.096 ms     | 370.952 µs   | 8.67x   |
|   30 | 298.265 µs     | 33.504 µs    | 2.926 ms     | 401.267 µs   | 9.81x   |
|   31 | 310.631 µs     | 33.857 µs    | 3.108 ms     | 334.523 µs   | 10.00x  |
|   32 | 328.822 µs     | 45.218 µs    | 3.461 ms     | 521.307 µs   | 10.52x  |
|   35 | 355.684 µs     | 49.508 µs    | 4.142 ms     | 579.682 µs   | 11.65x  |
|   40 | 396.741 µs     | 63.206 µs    | 5.474 ms     | 749.828 µs   | 13.80x  |
|   45 | 472.785 µs     | 85.491 µs    | 7.602 ms     | 1.501 ms     | 16.08x  |
|   50 | 534.158 µs     | 119.238 µs   | 10.226 ms    | 3.898 ms     | 19.14x  |
|  100 | 1.090 ms       | 239.000 µs   | 35.546 ms    | 8.156 ms     | 32.61x  |
|  200 | 3.042173ms     | 483.892µs    | 141.871796ms | 21.302688ms  | 46.64x  |
|  300 | 6.040093ms     | 1.034338ms   | 319.940927ms | 50.53683ms   | 52.97x  |
|  400 | 8.951528ms     | 901.017µs    | 500.398941ms | 49.487447ms  | 55.90x  |
| 1000 | 49.006538ms    | 4.043325ms   | 3.044053658s | 223.957115ms | 62.12x  |

The table shows that the performance improvement rises quickly with the number of binomials.
The provided figures for Montgomery multiplication _include_ time for converting of the initial
vector of coefficients into the Montgomery form and back to integers in the end for the results.  

The availability of built-in functions in the Plutus language will provide a 
more efficient way to perform this important type of computation,
bump the limits of operations that fit into a single transaction,
and reduce costs.  

Implementing the Montgomery multiplication directly in Plutus should be technically possible
with CIP-122, but will hardly bring any improvements mentioned and so is not advisable.

In Cardano, multiplication of scalars in the BLS12-381 scalar field is used 
by cryptographic primitives that need polynomial arithmetic over — most notably:

1. SNARKs (Groth16, PLONK)
2. Polynomial commitments (KZG)
3. Mithril signatures (stake-based aggregation)
4. Other pairing-based protocols

The above-mentioned cryptographic primitives are used in many Cardano products, just to mention a few:

- **Mithril** - utilizes a stake-based threshold multisignature scheme based on elliptic curve pairings
    that technically can be verified onchain.
- **Hydrozoa** (a brand-new layer-2 solution for Cardano) - uses pairing-based cryptographic accumulators to commit to a set of L2 utxos that can be withdrawn once a dispute is completed in the rule-based regime of operation.

In conclusion, the existing approach arguably either leaves developers
with not quite efficient way of multiplying scalars
or pushes them into the suboptimal direction of using infinite integers 
(which is also inefficient in terms of real performance).
Incorporating effective multiplication over the scalar field
directly will streamline such operations, reduce transaction costs, 
thereby advancing the Plutus ecosystem's functionality and dev experience.

## Specification

The various BLS12-381-specific operations for the scalar field $F_r$ including Montgomery multiplication 
are implemented in [blst](https://github.com/supranational/blst/blob/e99f7db0db413e2efefcfd077a4e335766f39c27/bindings/blst.h#L88-L105) library, 
which is already a dependency for the [cardano-base](https://github.com/IntersectMBO/cardano-base/blob/master/cardano-crypto-class/src/Cardano/Crypto/EllipticCurve/). 
It has been used for implementing 
[CIP-381](https://cips.cardano.org/cip/CIP-0381) 
and [CIP-133](https://cips.cardano.org/cip/CIP-0133). 
Basically, we would like to expose several additional functions from this library in Plutus API.

### New types definition

To represent a scalar in $F_r$ stored in the Montgomery form a new opaque type `bls12_381_fr`
(which corresponds to `blst_fr` type) can be used along with introducing and eliminating from/to a _byte array_: 

```
bytestring_to_bls12_381_fr: [bool, 𝚋𝚢𝚝𝚎𝚜𝚝𝚛𝚒𝚗𝚐] -> bls12_381_fr
bls12_381_fr_to_bytestring :: [bool, bls12_381_fr] -> 𝚋𝚢𝚝𝚎𝚜𝚝𝚛𝚒𝚗𝚐
```

The conversion is little-endian if the first arguemn is `false` and big-endian if it is `true`.
We prefer not to choose name `bls12_381_scalar` to avoid name clashes with existing functions
for _scalar multiplication_ like `bls12_381_G1_scalarMul`.

### Function definition

In addition to convertion functions mentioned in the previous section,
we propose to define the only function for Montgomery modular multiplication
**bls12_381_fr_mul** as follows:

```
bls12_381_fr_mul :: [bls12_381_fr, bls12_381_fr] -> bls12_381_fr
```
TBD: Scalar and multi-scalar multiplication in $G1$ and $G2$ are typical downstream functions 
for `bls12_381_fr` values, but in the current Plutus they use integers, i.e., double conversion is needed:
`bls12_381_fr -> bytestring -> integer` to call them.
The underlying `blst` functions take a pointer to raw bytes, i.e. `const byte *scalar`, so probably
we should consider ways of simplifying this by either having scalar multiplication that works with
byte arrays or providing a function to go directly form `bls12_381_fr` to `integer`.

TBD: Additionally, we might consider adding some other functions that `blst` [provides](https://github.com/supranational/blst/blob/e99f7db0db413e2efefcfd077a4e335766f39c27/bindings/blst.h#L88-L105).

### Cost model

The computational impact of Montgomery multiplication is straightforward, since values of
type `bls12_381_fr` are statically limited to 255 bits, so for newly added `bls12_381_fr_mul`
function we can use a static cost model.

TDB: Introducing of this type potentially allows simplifying cost models for some other functions.
Currently, scalars have to be reduced modulo the order of the group before being passed to the `blst` 
functions, see [cardano-base](https://github.com/IntersectMBO/cardano-base/blob/6f9c20abdd3010e5a25356580cc968ba430101ad/cardano-crypto-class/src/Cardano/Crypto/EllipticCurve/BLS12_381/Internal.hs#L521).

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
