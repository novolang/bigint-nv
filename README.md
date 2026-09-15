# bigint-nv

An arbitrary-precision integer is a whole number stored as a list of
digits, so its size is bounded by memory rather than by a machine word.
This package brings arbitrary-precision integers to novo-lang, together
with arithmetic modulo a number, which is what public-key cryptography
is made of. Its references are the Rust crate
[num-bigint](https://docs.rs/num-bigint) and
[CPython's `int`](https://docs.python.org/3/library/stdtypes.html#numeric-types-int-float-complex),
and the Montgomery multiplication it uses is algorithm 14.36 of the
[Handbook of Applied Cryptography](https://cacr.uwaterloo.ca/hac/)
chapter 14.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the numbers here are

A **limb** is one digit of a large number, held in a machine integer.
This package uses 28 bits per limb. A **magnitude** is a list of limbs,
least significant first, and it is always non-negative. The type is
`BigNat`. A **signed number** is a sign together with a magnitude, which
is what num-bigint and CPython both store. The type is `BigNum`.

A magnitude is **canonical**: it carries no leading zero limbs, so zero
is the empty list. Two equal magnitudes are equal limb for limb.

Arithmetic **modulo n** is arithmetic in which every result is reduced
into the range 0 up to n, not including n. Public-key cryptography is
almost entirely this arithmetic, because an RSA operation is one
exponentiation modulo a 2048-bit number.

**Montgomery form** is a different representation of a number modulo n
in which the reduction after a multiply is a shift instead of a
division. Converting into and out of it costs a setup per modulus: the
negated inverse of the modulus with respect to the limb base, and a
conversion constant. `BigMont` is that setup, as a value you build once
and keep for as long as the modulus lives.

A **constant-time** operation performs the same sequence of steps
whatever the secret it is given, so its running time says nothing about
that secret. See "Timing behaviour".

The sizes the package fixes are these.

| Quantity | Value |
| --- | --- |
| Bits per limb | 28 |
| Largest limb value | 268,435,455 |
| Limbs in a `Big256` | 10 |
| Limbs in a `Big2048` | 74 |
| Limbs in a `Big4096` | 147 |
| Bytes in a `Big256`, `Big2048`, `Big4096` | 32, 256, 512 |
| Limb count at which multiplication switches to Karatsuba | 32 |

Twenty-eight bits is the largest limb whose product fits in novo-lang's
`Int`, which is a signed 64-bit integer with no widening multiply. The
inner step of a schoolbook multiply is one product plus an accumulator
plus a carry, and with 28-bit limbs that stays below 2 to the 56th
power whatever the operands' length.

## Install

```
novo pkg add bigint-nv
```

## Example

```novo
use bignum
use bigfmt
use bigmod

fn main() [io]
    // A number too large for an Int, read from its decimal text.
    match bigfmt.nat_parse_decimal("18446744073709551616")
        Err(e) => println(e.message())
        Ok(n) =>
            // Multiplying two magnitudes cannot refuse. The value grows to fit.
            println(bigfmt.nat_to_decimal(bignum.nat_mul(n, n)))

            // A prime modulus, written as one 28-bit limb.
            let p = bignum.nat_from_limbs([1000003])

            // The Montgomery setup is built once and reused for every
            // operation against this modulus.
            match bigmod.montgomery(p)
                Err(e) => println(e.message())
                Ok(ctx) =>
                    // 65537 is a public RSA exponent, so the plain ladder
                    // is correct here. A secret exponent needs mont_pow_ct.
                    match bigmod.mont_pow(ctx, n, bignum.nat_from_limbs([65537]))
                        Err(e) => println(e.message())
                        Ok(r)  => println(bigfmt.nat_to_hex(r))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: bigint-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `bignum` | The magnitude and the signed number, and the arithmetic over them: add, subtract, multiply, square, divide with remainder, power, greatest common divisor, shifts, the bitwise operations, and the comparisons. |
| `bigmod` | Arithmetic modulo a number: reduce, add, subtract, multiply, square, negate, exponentiate, invert, the greatest common divisor, the constant-time select and comparison, and the Montgomery context with its four operations. |
| `bigfmt` | A number as text and as bytes, both ways: decimal, hexadecimal, any radix from 2 to 36, and big-endian bytes in a shortest and a fixed-length form. |
| `bigfix` | Three width-checked types, `Big256`, `Big2048` and `Big4096`, with their conversions to and from a magnitude, bytes and hexadecimal, and a constant-time comparison each. |
| `bigfault` | Every reason an operation refuses, as one enum, plus the two questions a caller asks of one: is this a fault in some text, and at which character. |

## How to choose an entry point

**Use `BigNat` when the number cannot be negative** and `BigNum` when it
can. Every function over magnitudes has a `nat_` prefix, so the two
surfaces do not collide. `bignum.of_sign_mag` and `bignum.magnitude`
convert between them.

**Use `bigmod`'s plain functions for a single operation and a `BigMont`
for a loop.** `bigmod.mul` and `bigmod.pow` reduce by division and need
no setup. A `BigMont` costs a setup and then makes every multiply a
shift, so one exponentiation against a 2048-bit modulus pays for it many
times over. An RSA verification builds one context and squares
seventeen times; an RSA signature builds one and performs about three
thousand operations.

**Use `bigfix` when a protocol fixes the width.** A P-256 scalar is 256
bits and an RSA-2048 modulus is 2048 bits. Constructing a `Big256`
checks the width once, and every function downstream of that
construction has the width as a type rather than as a comment.

**Use `bigfmt.nat_to_hex` rather than `nat_to_decimal` in a hot path.**
Hexadecimal walks the limbs once. Decimal divides the whole number once
per output chunk, so it costs a few hundred divisions on a 4096-bit
value.

## The rules a user needs

1. **Signed division comes in two kinds, and neither is the default.**
   `bignum.divmod` truncates toward zero, as C, Rust and num-bigint do,
   so −7 divided by 2 is −3 with remainder −1. `bignum.floordivmod`
   floors, as Python does, so the same division is −4 with remainder 1.
   Both are correct and they disagree, so both are published under
   names that say which.
2. **`bignum.nat_divmod` always answers a remainder from 0 up to the
   divisor, not including it.** A magnitude has no sign, so the
   Euclidean answer is the only one.
3. **The bitwise operations on a negative number use the infinite
   two's-complement reading**, which is Python's rule and num-bigint's.
   `bit_and(-1, x)` is `x`, and `bit_not(x)` is `-x - 1`. The
   representation here is a sign and a magnitude, so the rule is stated
   rather than read off the bits.
4. **Add, subtract, multiply, square, compare and left shift on a
   magnitude cannot refuse.** Their signatures answer a bare `BigNat`.
   The value grows until memory runs out, which is a panic and not an
   error value.
5. **Bytes are big-endian.** Every format this package meets is:
   PKCS #1, SEC 1, an ECDSA `r` and `s`, a Diffie-Hellman public value.
   There is no little-endian conversion.
6. **`nat_to_be_bytes` strips leading zeroes and
   `nat_to_be_bytes_padded` does not.** RFC 8017 section 4.1 requires an
   RSA signature to be exactly as many bytes as the modulus, leading
   zeroes included. Use the padded form whenever a protocol fixes a
   length. A stripped leading zero is a signature that verifies nowhere.
7. **`width` on the constant-time exponentiations is an argument, not
   the exponent's bit length.** The bit length of a secret is itself
   secret. An RSA private exponent is 2048 bits because the key is, not
   because of where its top set bit lands. Pass the protocol's width.
8. **A `BigMont` refuses an even modulus.** Montgomery reduction needs
   the modulus to be odd. Every modulus in RSA and in elliptic-curve
   cryptography is.
9. **`bigmod.inverse` refuses when the value and the modulus share a
   factor.** The error does not carry the factor. `bigmod.gcd` is the
   same walk run for that other answer.
10. **A magnitude you build by hand must be canonical.** Leave a zero
    limb on top and this package's own comparisons will disagree with
    it. Every function here answers a canonical value.
11. **`pow` with a negative exponent is refused.** There are no
    rationals here.

## Timing behaviour

- **Seven functions are constant-time by construction.**
  `bigmod.pow_ct`, `bigmod.mont_pow_ct` and `bigmod.select_ct`, plus the
  four comparisons `bigmod.eq_ct`, `bigfix.b256_eq_ct`,
  `.b2048_eq_ct` and `.b4096_eq_ct`. The exponentiation ladder squares
  and multiplies at every bit position and then selects, the select
  reads both of its arms, and each comparison reads every limb.
- **Every other function here is variable time**, and each says so in
  its own documentation comment. `pow` skips clear exponent bits.
  `inverse` and `gcd` branch on the values they reduce. `reduce`
  divides. A secret exponent goes through a `_ct` function or it leaks.
- **The claim is about the algorithm, not about the machine.** Nothing
  in a high-level language stops a compiler from turning a branchless
  select back into a branch, or keeps a limb out of a particular cache
  line. What a library can be responsible for is the sequence of
  operations, and that is what is claimed here.

## What is not included

- **Primality testing and key generation.** Miller-Rabin needs random
  witnesses, and no function in this package draws entropy. The
  randomness would have to arrive as an argument, which belongs in the
  package that also knows what a key is, built over `mont_pow_ct`.
- **Rational numbers and arbitrary-precision decimals.** A different
  subject. `pow` with a negative exponent refuses rather than answering
  a fraction.
- **Arithmetic on the fixed-width types.** `bigfix` converts and
  compares. Three more copies of modular exponentiation, one per width,
  would be three surfaces that drift apart.
- **Little-endian byte conversion.** See rule 5. A caller who needs it
  reverses the list.
- **Operator overloading.** `+` and `*` on a `BigNum` are a language
  feature rather than a package's to add.
- **A microcontroller build.** A magnitude is a list, a list literal is
  a heap allocation, and a build for a device with no heap allocator
  refuses one (SPEC section 14.4). The fixed-width types in `bigfix` do
  not change that: they are boxed structs holding lists, because a
  `@value` struct may not be the payload of a `Result` and every
  constructor here can refuse. A device that needs 256-bit field arithmetic
  wants a value type with an inline array, which is a different package.
  Nothing here is claimed to build for a device with no heap allocator, and
  there is no `tests/embedded_probe.nv`.
- **A module named `bigint`, and a type named `BigInt`.** Both names
  belong to the standard library, and a package may neither ship a
  module named after a standard library module nor redeclare a standard
  library type. `use bigint` reaches the standard library. The modules
  here are `bignum`, `bigmod`, `bigfmt`, `bigfix` and `bigfault`, and
  the signed type is `BigNum`.

## Related packages

- `std.bigint` in the standard library is a different implementation of
  a smaller subject. It is a handle over limb code written in the
  runtime's C, and it covers construction from an integer and from text,
  rendering to text, add, subtract, negate, compare and sign. It has no
  multiply, no division and no modular arithmetic. Every one of its
  entry points is host-only and needs the native compiler, so it does
  not run under the interpreter. Take it for a large sum in a program
  that already links the runtime. Take this package for arithmetic, for
  code that must run under the interpreter, and for anything modular.

  | | `std.bigint` | bigint-nv |
  | --- | --- | --- |
  | Multiply, divide, shift, bitwise | absent | present |
  | Modular arithmetic | absent | present |
  | Runs under `novo run` | no | yes |
  | Needs a dependency | no | yes |

- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is hashes and
  HMAC. An RSA or Diffie-Hellman implementation needs both packages: a
  digest from there and a modular exponentiation from here.
- [ed25519-nv](https://novo-lang.org/packages/ed25519-nv) is
  fixed-width field arithmetic for one curve, over a value type with an
  inline array and no allocation. It is what a microcontroller uses
  where this package does not fit.

## Tests

```bash
novo test tests/bigint_tests.nv       # 43 tests: the arithmetic and the constants
```

Integer arithmetic has exactly one answer, so there is no tolerance
anywhere in this suite and every assertion is an equality. That is the
one thing that separates a big-number suite from every other numeric
one.

The expected values come from CPython and from num-bigint's own
documented examples: 2 to the 64th power, 17 divided by 5, −7 divided by
2 under both rounding rules, and the inverse of 3 modulo 11. The large
ones are published constants: the field prime and the group order of
P-256, from FIPS 186-4 appendix D.1.2.3, which every ECDSA
implementation carries, and the shape of an RSA-2048 modulus from
RFC 8017.

The suite multiplies the same pairs of numbers above and below the
32-limb Karatsuba threshold, because the two algorithms must agree
exactly. It also asserts the canonical form, the two signed division
rules against each other, the two's-complement reading of the bitwise
operations, and the fixed-length byte encoding.

The tests compile today and fail at run, each on the
`not implemented: bigint-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `bignum.BIG_LIMB_BITS`, `.BIG_LIMB_MASK`, `.BIG_KARATSUBA_LIMBS` | yes (they are constants) |
| `bigfix`'s nine width, limb-count and byte-count constants | yes (they are constants) |
| `bignum.BigNat`, `.BigNum`, `.BigDivision`, `.BigNumDivision` | declared |
| `bigmod.BigMont`, `bigfix.Big256`, `.Big2048`, `.Big4096`, `bigfault.BigFault` | declared |
| `bignum`'s magnitude construction, inspection and comparison | no |
| `bignum.nat_add`, `.nat_sub`, `.nat_mul`, `.nat_sqr`, `.nat_divmod`, `.nat_div`, `.nat_rem`, `.nat_pow`, `.nat_gcd` | no |
| `bignum.nat_shl`, `.nat_shr`, `.nat_bit_and`, `.nat_bit_or`, `.nat_bit_xor`, `.nat_popcount`, `.nat_trailing_zeros` | no |
| `bignum`'s signed construction, arithmetic, `divmod`, `floordivmod`, `pow`, comparison, bitwise and shifts | no |
| `bigmod.reduce`, `.of_num`, `.add`, `.sub`, `.mul`, `.sqr`, `.neg`, `.pow`, `.inverse`, `.gcd` | no |
| `bigmod.pow_ct`, `.select_ct`, `.eq_ct` | no |
| `bigmod.montgomery`, `.to_mont`, `.from_mont`, `.mont_mul`, `.mont_sqr`, `.mont_pow`, `.mont_pow_ct` | no |
| `bigfmt`'s seven magnitude text functions and four signed text functions | no |
| `bigfmt.nat_from_be_bytes`, `.nat_to_be_bytes`, `.nat_to_be_bytes_padded`, `.nat_byte_len`, `.from_sign_be_bytes`, `.to_be_bytes` | no |
| `bigfix`'s thirty conversions, comparisons and width changes | no |
| `bigfault.is_text_fault`, `.text_offset`, and the `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
