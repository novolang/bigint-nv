# bigint-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Arbitrary-precision integers written in novo-lang: an unsigned
magnitude over a list of 28-bit limbs, a signed number over that, and
the operations a program that outgrew `Int` needs — add, subtract,
multiply (schoolbook, then Karatsuba above a published threshold),
divide with remainder, shift, compare, the bitwise operations over a
two's-complement reading, decimal and hexadecimal text in both
directions, and big-endian bytes in both directions.

And then the half the crypto tier is here for: arithmetic modulo n,
with modular exponentiation in two forms — one fast and one whose
operation sequence does not depend on the secret — modular inverse,
Montgomery form as a value you build once per modulus, and
width-checked `Big256`, `Big2048` and `Big4096` types for the curve and
RSA sizes.

It is for the program that has a number an `i64` cannot hold: an RSA
verification, a Diffie-Hellman exchange, a database aggregate, a
factorial, a fixed-point scale factor, a notebook doing number theory.

```
novo pkg add bigint-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use bignum
use bigmod
use bigfmt

// RSA signature verification is one modular exponentiation: s^e mod n.
fn rsa_verify_raw(sig: BigNat, e: BigNat, n: BigNat) -> Result<BigNat, BigFault>
    let m = bigmod.montgomery(n)!
    bigmod.mont_pow(m, sig, e)
```

The public exponent is public, so `mont_pow` is the right call and not
`mont_pow_ct`. A signature — the private direction — is
`mont_pow_ct(m, digest, d, 2048)`, and the difference between those two
lines is the whole of what this package has to say about constant time.

## The layer, and why

`core` — no effects at all.

Nothing here is read, written, timed or sampled. A modular inverse is a
loop over limbs; a Montgomery context is arithmetic on a modulus the
caller passed in; the randomness a key generator needs arrives as bytes
the caller already has. The budget is `[]` on every one of the 121
public functions and there was never any pressure on it.

**No `@tier(embedded)` claim, and there is no `tests/embedded_probe.nv`.**
The audit's `core-embedded` row passes and says the package makes no
claim, which is the honest outcome rather than a gap. The reason is the
representation: a magnitude is a limb LIST, list literals are heap
allocations, and `@tier(embedded)` refuses them (SPEC § 14.4). Padding
`bigfix`'s three types out to fixed limb counts does not change that —
they are boxed structs holding lists, because a `@value` struct may not
be a `Result` payload (E2015) and every constructor here is fallible.

That is not a defect in this design; it is the line between two
different packages. A device that needs 256-bit field arithmetic wants
a `@value` type with an inline array and no allocation anywhere, which
is exactly what **ed25519-nv's `Fe25519`** is — five limbs, a `@value`
struct, `[]` throughout, and it builds for a Cortex-M. What bigint-nv
gives a device is nothing; what it gives a host is arbitrary precision.
**If a row is wanted for the device case it is a new one** — a
fixed-width, allocation-free modular arithmetic package over `@value`
types — and this lane's report names it.

## The load-bearing interface

```novo
pub struct BigMont
pub fn montgomery(n: BigNat) -> Result<BigMont, BigFault> []
pub fn mont_mul(m: BigMont, a: BigNat, b: BigNat) -> Result<BigNat, BigFault> []
pub fn mont_pow_ct(m: BigMont, base: BigNat, exponent: BigNat, width: Int) -> Result<BigNat, BigFault> []
```

**`BigMont` is a value the caller builds once per modulus, and that one
decision is what keeps this package at `core`.**

Montgomery form replaces a division by n with a multiply and a shift,
which is what makes modular exponentiation affordable. The price is a
setup — `n'`, the negated inverse of the modulus modulo the limb base,
and `R² mod n` for the conversion in — that costs more than a single
reduction does. So an API where every call re-derived it would be
slower than one that never used Montgomery form at all, and an API that
cached it would need somewhere to cache it.

The C libraries put that somewhere in a context object with an `init`
and a `free`. Here it is a value: `montgomery(n)` answers a `BigMont`,
the caller keeps it as long as the modulus lives, and `mont_mul`,
`mont_sqr`, `mont_pow` and `mont_pow_ct` are pure functions of it. It
can live in a struct beside the key it belongs to, be copied, be used
from two places, and it cannot be stale, because nothing mutates it.
An RSA verification builds one context and does 17 squarings; a
signature builds one and does about 3000 operations; a caller who built
it inside the loop would pay the setup 3000 times and never know why.

**`width` on the constant-time calls is an argument and not
`bit_len(exponent)`**, and that is the second load-bearing decision.
The bit length of a secret is itself secret: an RSA private exponent is
2048 bits because the key is, not because of where its top set bit
happens to land. A function that derived the ladder length from the
value would leak the one thing the ladder exists to hide.

## What std.bigint keeps, and why both exist

The standard library has `std.bigint`, and it is **not** what this
package replaces. Both stay, and the split is clean:

| | `std.bigint` | bigint-nv |
| --- | --- | --- |
| representation | a `void*` box over the runtime's C limb code | a novo-lang list of 28-bit limbs |
| operations | `from_int`, `from_str`, `to_str`, `add`, `sub`, `neg`, `eq`, `cmp`, `sign` | those, plus multiply, divmod, shifts, bitwise, gcd, pow, radix text, bytes, and all of modular arithmetic |
| runs under `novo run` | no — every entry point is "LLVM only" | yes |
| checks at a tier | no — every entry point is "not embedded" | yes at `wasm` and `system` |
| available without a dependency | yes, auto-prepended to every program | no, `novo pkg add bigint-nv` |
| a `core` package may use it | yes, and inherits "LLVM only" | yes |

**`std.bigint` keeps the case it is good at**: a program that already
links the runtime and wants a 20-digit sum out of a database
aggregate types `bigint.add` and is finished. Nothing is added, nothing
is imported, and the C implementation is faster than a limb list in
novo-lang will be for a long time.

**bigint-nv exists because the missing half cannot be written where
`std.bigint` lives.** There is no multiply, no division, no modular
anything, and the stdlib's own page lists all of those under "Roadmap —
none of the following exists". Adding them means writing them in the
runtime's C, which is the one place a package cannot reach and a
reviewer of this project's own effect rules cannot audit. Writing them
in novo-lang gives a package the effect checker measures, the
interpreter can run, the tier checker can refuse, and the crypto tier
can depend on at `core`.

So: **`std.bigint` for a big sum, bigint-nv for arithmetic.** If the
stdlib's `Bigint` is ever reimplemented over this package, its surface
does not change and no caller notices; that is a possibility this
design leaves open and does not schedule.

## The reference implementations, and what is specification

num-bigint and CPython's `int` are the references, and the distinction
matters because it decides what a test may assert.

**Specification, and binding on this package**

- The arithmetic itself. `a * b`, `a / b`, `a mod n` have one answer
  each and there is no tolerance anywhere in this package — unlike
  every other package this lane wrote, where a float comparison needs
  an epsilon. A test here asserts equality.
- **Euclidean remainder**: `nat_divmod` answers a remainder in
  `0 ..= divisor - 1`, always.
- **Truncated vs floored signed division.** C, Rust and num-bigint
  truncate toward zero (`-7 / 2 == -3`, remainder `-1`); Python floors
  (`-7 // 2 == -4`, remainder `1`). Both are correct and they disagree,
  so this package publishes **both under names that say which**:
  `bignum.divmod` truncates, `bignum.floordivmod` floors. Neither is
  the default, because a default here is a trap.
- **The bitwise operations over negative numbers** are defined by the
  infinite two's-complement reading — `bit_and(-1, x) == x`,
  `bit_not(x) == -x - 1` — which is what Python and num-bigint both do.
  A sign-and-magnitude representation has no sign bits to hand, so the
  rule is stated rather than derived.
- **Big-endian bytes**, because every format this package will meet is:
  PKCS#1, SEC 1, an ECDSA `r` and `s`, a Diffie-Hellman public value.
- **RFC 8017 § 4.1's length rule**: an RSA signature is as many bytes
  as the modulus, leading zeros included. `bigfix`'s `to_be_bytes` is
  total for exactly that reason.

**num-bigint's own choices, which this package follows and a test may
not treat as correctness**

- The Karatsuba threshold of 32 limbs. It is a speed decision measured
  on somebody else's machine, the two algorithms agree exactly, and the
  suite multiplies the same pairs on both sides of it.
- 28-bit limbs. num-bigint uses 32 and CPython 30; this package uses 28
  because novo-lang's `Int` is an `i64` with no widening product, so
  the whole inner step of a schoolbook multiply — product plus
  accumulator plus carry — has to fit in 63 bits. `bignum`'s module
  comment has the arithmetic. The limb size is visible in
  `BIG_LIMB_BITS` and in `bigfix`'s limb counts, and nowhere else.
- Binary GCD rather than Euclid's, binary extended Euclid for the
  inverse, HAC § 14.36's interleaved form for `mont_mul`. All three are
  implementation choices with one right answer.

## Constant time: what is claimed and what is not

Three functions carry `_ct` — `bigmod.pow_ct`, `bigmod.mont_pow_ct`,
`bigmod.select_ct` — plus the four `eq_ct` comparisons. Everything else
in this package is variable time and says so in its own doc comment:
`pow` skips clear exponent bits, `inverse` and `gcd` branch on the
values they reduce, `reduce` divides.

**What is claimed**: the algorithm performs the same operations in the
same order whatever the secret bits are. The ladder squares and
multiplies at every bit position and selects; the select reads both
arms; the comparison reads every limb.

**What is not claimed, and cannot be from here**: that the machine
executes it that way. Nothing in a high-level language stops a compiler
from turning a branchless select back into a branch, or keeps a limb
out of a cache line, or controls what a JIT does. A constant-time claim
in novo-lang is a claim about the algorithm, which is the part a
library can be responsible for.

**What the implementation lane owes**: a test that counts operations
rather than measuring a clock — the same exponent width with different
values must issue the same sequence of multiplies — and a note in the
CHANGELOG saying so. A timing test on a shared machine measures the
machine.

## Module names, and the two the standard library owns

The modules are `bignum`, `bigmod`, `bigfmt`, `bigfix` and `bigfault`.

**Not `bigint`.** `bigint` is a standard library module, and no package
may ship a module named after one — `novo pkg init --interface
bigint-nv` scaffolds `bigint_nv.nv` for exactly that reason, and the
build refuses the bare name. The same rule is why the signed type is
`BigNum` and not `BigInt`: the standard library declares `struct BigInt`
and a package may not redeclare a stdlib type at all.

So the two names a reader would reach for first are both taken by the
thing this package sits beside, and that is worth saying plainly rather
than leaving them to wonder why `use bigint` does not work.

## Deliberately not here

- **Primality testing and key generation.** Miller-Rabin needs random
  witnesses; a `core` package has no entropy, so the randomness would
  arrive as an argument and the API would become "give me 64 random
  bytes per round". That is a real design and it belongs in the package
  that also knows what a key is — crypto-nv or an rsa-nv — with this
  package's `mont_pow_ct` underneath. **A missing row**, named in this
  lane's report.
- **Rational numbers and arbitrary-precision decimals.** `pow` with a
  negative exponent refuses rather than returning a rational, and
  `std.bigint`'s page lists `Decimal` under its own roadmap. Different
  subject, different package.
- **Arithmetic on the fixed-width types.** `bigfix` converts and
  compares and does nothing else; see its module comment for why three
  copies of modular exponentiation would be three times the surface and
  would drift.
- **Little-endian byte conversion.** One call, in one direction that
  nothing this package serves uses. A caller who needs it reverses the
  list, which is honest about what it is doing.
- **Operator overloading.** `+` and `*` on a `BigNum` would read
  better and are not this package's to add; `std.bigint`'s page lists
  them under its roadmap for the language.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: bigint-nv.<module>.<fn>` —
which is the expected result until the bodies land, and is what makes
the suite a description of the interface rather than of nothing.
`novo test --isolate` is the readable form: one verdict per test,
naming the function it stopped at.

| module | public types | functions | constants | implemented |
| --- | --- | --- | --- | --- |
| `bigfault` | 1 | 2 | 0 | no |
| `bigfix` | 3 | 30 | 9 | no |
| `bigfmt` | 0 | 17 | 0 | no |
| `bigmod` | 1 | 20 | 0 | no |
| `bignum` | 4 | 52 | 3 | no |
| **total** | **9** | **121** | **12** | **no** |
