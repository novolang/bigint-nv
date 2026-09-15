# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Five modules.  `bignum` is the magnitude, the signed number and the
  arithmetic over both; `bigfmt` is text and bytes in both directions;
  `bigmod` is arithmetic modulo n, including Montgomery form; `bigfix`
  is the three widths the crypto tier uses; `bigfault` is every way one
  of them refuses.
- **`BigMont` is the load-bearing interface** — a value the caller
  builds once per modulus and passes to every multiplication against
  it.  A C library puts that setup in a context object with an `init`
  and a `free`; here it is a value that can live in a struct beside its
  key, be copied, and never be stale, and that one decision is what
  keeps the package at `core` with no state carried between calls.
- **28-bit limbs, and the reason is published.**  novo-lang's `Int` is
  an `i64` with no widening product, so a schoolbook multiply's inner
  step — product plus accumulator plus carry — has to fit in 63 bits.
  Two 32-bit limbs do not multiply inside one; two 28-bit limbs give
  56 bits and seven bits of headroom, and the bound holds at any
  operand length because the carry propagates at every step.
  `BIG_LIMB_BITS` is the number, and `bigfix`'s limb counts of 10, 74
  and 147 are what it costs at 256, 2048 and 4096 bits.
- **Truncated AND floored signed division, both published.**  C, Rust
  and num-bigint truncate toward zero; Python floors; the two disagree
  on every negative dividend.  `bignum.divmod` truncates and
  `bignum.floordivmod` floors, neither is the default, and the README
  says why a default here would be a trap.
- **Constant time is claimed function by function.**  Three names carry
  `_ct` and the four fixed-width comparisons do; every other function
  is variable time and says so in its own doc comment.  `width` on the
  constant-time exponentiations is an ARGUMENT rather than the
  exponent's measured bit length, because the bit length of a secret is
  itself secret.  What is claimed is the algorithm's operation
  sequence; what is not claimed is the machine, and the README draws
  that line.
- **`std.bigint` is not replaced.**  The stdlib's `Bigint` is a box over
  the runtime's C limb code: add, sub, neg, cmp, decimal text, no
  multiply, no division, no modular arithmetic, every entry point
  marked "not embedded" and "LLVM only".  This package is the same
  subject in novo-lang, which is what makes the missing half writable,
  auditable, interpretable and tier-checkable.  The README has the
  table.
- **No `@tier(embedded)` claim, and no probe.**  A magnitude is a limb
  list and a list literal is a heap allocation the tier refuses.  A
  device that needs 256-bit field arithmetic wants a `@value` type with
  an inline array — ed25519-nv's `Fe25519` is exactly that — and that
  is a different package, named as a missing row in this lane's report.
- **The modules are `bignum` and not `bigint`**, and the signed type is
  `BigNum` and not `BigInt`: `bigint` is a standard library module name
  and `BigInt` a standard library type, and a package may redeclare
  neither.
- Every vector is CPython's own answer, num-bigint's test corpus, or a
  published constant — RFC 8017's RSA-2048 shape, P-256's field prime
  and group order from FIPS 186-4, and Knuth's algorithm D worked
  example.
