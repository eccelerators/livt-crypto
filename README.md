# Livt.Crypto

`Livt.Crypto` provides fixed-size cryptographic building blocks for the Livt
base library. Source components live in focused namespaces such as
`Livt.Crypto.Aes`, `Livt.Crypto.Aead`, `Livt.Crypto.Hash`, and
`Livt.Crypto.Mac`; tests mirror that structure below `Livt.Crypto.Tests`.

The implementations are intentionally small, explicit, and vector-driven. They
cover block ciphers, authenticated encryption, hash functions, MACs, key
derivation, and deterministic random generation. Higher-level concerns such as
streaming adapters, padding policy, nonce generation, and key management are
left to callers.

## Package API Shape

The supported package surface is the fixed-size component API documented here:
AES block ciphers and modes, SHA-2/SHA-3 hashes, HMAC, HKDF, CMAC, CTR_DRBG,
ChaCha20, Poly1305, and ChaCha20-Poly1305. Components such as `AesSBox`,
`AesSBoxInverse`, `AesGfMath`, `Ghash`, and `Keccak` live in
`Livt.Crypto.Internal` because they are useful implementation primitives, but
they are lower-level building blocks rather than misuse-resistant application
APIs.

AEAD users should call `FinalizeTag(...)` and then `VerifyTag(expected)`. The
verification helpers compare all tag bytes before returning `1` for a match or
`0` for a mismatch. Plaintext returned by `DecryptBlock()` should not be trusted
until tag verification succeeds.

## Algorithms

### AES

AES block encryption and decryption follow NIST FIPS 197 for 128-bit, 192-bit,
and 256-bit keys. Components process one 16-byte block at a time.

- `EncryptBlock128`, `EncryptBlock192`, `EncryptBlock256`
- `DecryptBlock128`, `DecryptBlock192`, `DecryptBlock256`
- `AesSBox`, `AesSBoxInverse`, `AesGfMath`

AES mode wrappers are also provided:

- CBC mode, NIST SP 800-38A: `Cbc128`, `Cbc192`, `Cbc256`
- CTR mode, NIST SP 800-38A: `Ctr128`, `Ctr192`, `Ctr256`
- GCM mode, NIST SP 800-38D: `Aes128Gcm`, `Aes192Gcm`, `Aes256Gcm`
- CMAC, NIST SP 800-38B: `Cmac128`, `Cmac192`, `Cmac256`

CTR uses NIST Inc32 counter semantics. CBC does not include padding helpers;
callers provide complete 16-byte blocks. GCM exposes block-at-a-time AAD and
data processing with explicit final lengths passed to `FinalizeTag`, plus
`VerifyTag` for authentication checks.

### GHASH

`Ghash` implements the GF(2^128) accumulator used by GCM. It is exposed as a
standalone component for code that needs the authentication primitive directly.

### SHA-2

SHA-256 and SHA-512 follow NIST FIPS 180-4. The components expose a
single-block streaming model: reset state, process complete blocks, then read
the digest.

- `Sha256`
- `Sha512`

### HMAC and HKDF

HMAC follows RFC 2104 and is tested with RFC 4231 vectors. HKDF-SHA-256 follows
RFC 5869 and exposes the extract and expand steps used by the current
single-block message model.

- `HmacSha256`, `HmacSha512`
- `HkdfSha256`

### AES-CTR-DRBG

`Aes256CtrDrbg` implements CTR_DRBG with AES-256 and no derivation function,
following NIST SP 800-90A. It supports instantiate, generate, finalize-generate,
reseed, and key clearing.

### ChaCha20 and Poly1305

ChaCha20, Poly1305, and ChaCha20-Poly1305 follow RFC 8439.

- `ChaCha20`
- `Poly1305`
- `ChaCha20Poly1305`

`ChaCha20Poly1305` uses a 64-byte block-at-a-time API, authenticates ciphertext
with Poly1305, and exposes `VerifyTag` for authentication checks.

### SHA-3

SHA-3 work is present under `src/hash/` and shares the internal Keccak
permutation. These components follow NIST FIPS 202 and are covered by the
configured test suite.

- `Keccak`
- `Sha3256`
- `Sha3512`

## Layout

```text
src/aes/       AES block ciphers, CBC/CTR modes, and CMAC
src/aead/      AES-GCM and ChaCha20-Poly1305
src/hash/      SHA-2 and SHA-3 hashes
src/internal/  AES tables/GF math, GHASH, and Keccak
src/kdf/       HKDF-SHA-256
src/mac/       HMAC-SHA-2 and Poly1305
src/random/    AES-256 CTR_DRBG
src/stream/    ChaCha20
tests/<domain>/ matching tests for each source domain
docs/          Supporting notes and independent vector-checking references
```

All production `.lvt` files should use the domain namespace that matches their
folder, for example:

```livt
namespace Livt.Crypto.Aead
```

All test `.lvt` files should mirror that below `Livt.Crypto.Tests`, for
example:

```livt
namespace Livt.Crypto.Tests.Aead

using Livt.Crypto.Aead
```

## Building and Testing

Build the package:

```bash
livt build
```

Run the configured test components:

```bash
livt test
```

The test list is defined in `livt.toml`. Test vectors are embedded in the test
components and are drawn from the relevant NIST, RFC, or known-answer-test
sources. Additional notes for independently checking selected vectors with
OpenSSL live in [`docs/test-vectors.md`](docs/test-vectors.md). Short
call-order examples for common primitive workflows live in
[`docs/usage.md`](docs/usage.md).

## Development Notes

- Preserve the compiler workarounds documented in `COMPILER.md` when editing
  component implementations.
- Keep new primitives in a domain folder under `src/` and place matching tests
  under the same domain folder in `tests/`.
- Use nested `Livt.Crypto.<Domain>` namespaces for source components and
  `Livt.Crypto.Tests.<Domain>` for tests.
- Prefer official standards or RFC vectors for new tests.

## Outlook

This package is intentionally a primitives package, not a batteries-included
application crypto API. The current components expose low-level, fixed-size
building blocks and leave message framing policy to callers.

Future work should add higher-level components around these primitives where
needed, including:

- padding helpers for CBC and other block-oriented modes
- variable-length message adapters for hashes, MACs, AEADs, and stream ciphers
- streaming interfaces with explicit ready/valid or block handshakes
- nonce and counter management helpers with safe defaults
- misuse-resistant wrappers that combine encryption, authentication, and tag
  verification into harder-to-misuse workflows
- clear guidance for key lifecycle, reset, zeroization, and integration into
  larger Livt designs

## Performance

All figures are from GHDL functional simulation at a **5 ns clock period
(200 MHz equivalent)** and cover complete test suites.  "Test throughput"
is computed as *(data bytes processed in the suite) ÷ (total simulation
time)* and includes key-setup, assertions, and test-harness overhead.  It
is therefore a **conservative lower bound**.  For designs that hold the key
constant across many blocks, per-block throughput will be substantially
higher than these numbers suggest; AES-128 CTR streaming, for example,
runs at approximately 10–11 Mbps at 200 MHz once key setup is amortised.

All implementations are iterative (single-block FSM, non-pipelined) and
process exactly one block per `EncryptBlock` / `CryptBlock` / `ProcessBlock`
call.

### AES block ciphers

| Component | Key | Block | Suite cycles | Test throughput |
|-----------|-----|-------|-------------|-----------------|
| `EncryptBlock128` | 128 b | 16 B | 22,161 | 12.7 Mbps |
| `DecryptBlock128` | 128 b | 16 B | 25,637 |  5.0 Mbps |
| `EncryptBlock192` | 192 b | 16 B | 25,657 |  6.0 Mbps |
| `DecryptBlock192` | 192 b | 16 B | 29,641 |  4.3 Mbps |
| `EncryptBlock256` | 256 b | 16 B | 68,091 |  4.1 Mbps |
| `DecryptBlock256` | 256 b | 16 B | 34,675 |  3.7 Mbps |

The variation across key sizes mostly reflects differing key-schedule costs
and unequal test-function counts (seven encrypt tests vs four decrypt tests).

### AES modes of operation

| Component | Mode | Key | Block | Suite cycles | Test throughput |
|-----------|------|-----|-------|-------------|-----------------|
| `Cbc128` | CBC | 128 b | 16 B | 29,841 | 3.4 Mbps |
| `Cbc192` | CBC | 192 b | 16 B | 34,337 | 3.0 Mbps |
| `Cbc256` | CBC | 256 b | 16 B | 54,169 | 2.8 Mbps |
| `Ctr128` | CTR | 128 b | 16 B | 18,065 | 7.1 Mbps |
| `Ctr192` | CTR | 192 b | 16 B | 13,511 | 5.7 Mbps |
| `Ctr256` | CTR | 256 b | 16 B | 24,067 | 5.3 Mbps |
| `Aes128Gcm` | GCM | 128 b | 16 B | 96,965 | 0.79 Mbps |
| `Aes192Gcm` | GCM | 192 b | 16 B | 100,013 | 0.77 Mbps |
| `Aes256Gcm` | GCM | 256 b | 16 B | 107,873 | 0.71 Mbps |
| `Cmac128` | CMAC | 128 b | 16 B | 23,889 | ~3.2 Mbps |
| `Cmac192` | CMAC | 192 b | 16 B | 26,861 | ~2.9 Mbps |
| `Cmac256` | CMAC | 256 b | 16 B | 30,451 | ~2.5 Mbps |

GCM figures include GHASH authentication per block.  CMAC throughput is
approximate (test suite byte count is small).

### Hash functions

| Component | Block | Output | Suite cycles | Test throughput |
|-----------|-------|--------|-------------|-----------------|
| `Sha256`  |  64 B | 32 B | 693,957 | 0.59 Mbps |
| `Sha512`  | 128 B | 64 B | 1,480,965 | 0.55 Mbps |
| `Sha3256` | 136 B | 32 B | 917,825 | 0.47 Mbps |
| `Sha3512` |  72 B | 64 B | 916,929 | 0.25 Mbps |

SHA-3-512's lower byte throughput versus SHA-3-256 reflects the narrower
absorption rate (72 B vs 136 B) over the same Keccak-f[1600] permutation.

### MACs, AEAD stream cipher, KDF, and DRBG

| Component | Block | Suite cycles | Test throughput | Notes |
|-----------|-------|-------------|-----------------|-------|
| `HmacSha256` | 64 B | 1,389,101 | ~0.15 Mbps | Two RFC 4231 vectors; two SHA-256 passes per MAC |
| `HmacSha512` | 128 B | 2,964,397 | ~0.14 Mbps | Two RFC 4231 vectors; two SHA-512 passes per MAC |
| `HkdfSha256` | 64 B | 4,166,969 | — | Extract + ExpandT1 + ExpandTN per TC; chains HMAC |
| `Poly1305` | 16 B | 14,605 | ~5.3 Mbps | 16-byte block one-time MAC |
| `ChaCha20` | 64 B | 240,167 | 1.72 Mbps | — |
| `ChaCha20Poly1305` | 64 B | 386,243 | 0.79 Mbps | Includes Poly1305 auth per block |
| `Aes256CtrDrbg` | 16 B | 49,629 | ~4.1 Mbps | 8 generated blocks (two requests) in KAT suite |

### Resource estimates

Rough order-of-magnitude figures for an iterative implementation on a
mid-range FPGA (Xilinx 7-series equivalent).  **Do not rely on these for
resource planning — synthesise and report for your actual target.**

| Component group | Est. LUTs | Est. FFs | Dominant source of FFs |
|----------------|-----------|----------|------------------------|
| AES-128 block cipher (enc or dec) | 400–900 | 1,500–1,700 | Round-key schedule (176 B) |
| AES-192 block cipher | 450–1,000 | 1,700–2,000 | Round-key schedule (208 B) |
| AES-256 block cipher | 500–1,100 | 2,000–2,300 | Round-key schedule (240 B) |
| CBC / CTR / CMAC wrapper | 100–300 | 150–300 | IV / counter registers |
| GCM wrapper + GHASH | 300–600 | 300–500 | GHASH accumulator (128 b) |
| SHA-256 | 1,500–3,000 | 2,500–3,500 | Message schedule + state (8×32 b) |
| SHA-512 | 2,500–5,000 | 4,000–5,500 | Message schedule + state (8×64 b) |
| SHA-3 / Keccak | 3,000–7,000 | 1,600–2,200 | Keccak state (1600 b) |
| HMAC-SHA-256 | 1,600–3,200 | 2,600–3,700 | Includes SHA-256 twice |
| ChaCha20 | 800–1,500 | 1,500–2,000 | 16-word 32-bit state |
| Poly1305 | 300–700 | 400–600 | 130-bit accumulator |
| ChaCha20-Poly1305 | 1,100–2,200 | 2,000–2,700 | ChaCha20 + Poly1305 |
| AES-256 CTR DRBG | 600–1,300 | 2,100–2,500 | AES-256 block cipher |

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
