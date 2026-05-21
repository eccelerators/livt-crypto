# Design Note - Livt.Crypto

This document records implementation decisions that affect maintainers of the
published package. `COMPILER.md` is the source of truth for active compiler or
VHDL-generation bugs; this file explains why the crypto components are shaped
the way they are.

## Fixed-Size Component APIs

The package intentionally exposes fixed-size block APIs. AES, CBC, CTR, GCM,
CMAC, GHASH, Poly1305, ChaCha20, and the hash/KDF/MAC components all operate on
the block sizes documented in `README.md`. Padding, variable-length message
adapters, nonce generation, and key-management policy belong in higher-level
components.

## Public vs Internal Primitives

`AesSBox`, `AesSBoxInverse`, `AesGfMath`, `Ghash`, and `Keccak` remain in
`Livt.Crypto.Internal` because other package components need them and because
they are useful for low-level verification. They are not intended as
misuse-resistant application APIs.

`AesSBox` and `AesSBoxInverse` are Livt classes with enumerated static inline
returns instead of class array constants. This keeps them compatible with the
current Livt class model and the VHDL generator behavior documented in
`COMPILER.md`.

## Secret State Exposure

Published components should not expose debug methods that return internal key,
counter, or accumulator state. Tests should prefer public known-answer vectors
over state-inspection helpers. `Aes256CtrDrbg` therefore validates the NIST
SP 800-90A known-answer test through generated output, not through public key
or V accessors.

## Compiler Workarounds

Keep workaround comments close to the code that needs them, and keep
`COMPILER.md` focused on current, reproducible issues. When a compiler bug is
fixed and `livt test` confirms the simpler pattern works, remove the stale
warning rather than carrying it forward.
