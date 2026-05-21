# Livt Compiler Bugs

Bugs discovered while implementing AES-128, AES-192, AES-256, CMAC, SHA-256, ChaCha20, SHA-3/Keccak, and related refactoring.

---

## 31. [BUG] Void inline fn procedure: array field writes use wrong VHDL assignment

**Status**: Active bug. Do not use void `inline fn` with array field writes until fixed.
Tracked as **livt #65** in livt/COMPILER.md.

When a void `inline fn` writes to a component array field (`this.blk[i] = val`), the
VHDL generator emits `p_blk <= val` where `p_blk` is a `logic_array_2d` signal parameter
and `val` is a `std_logic_vector(7 downto 0)`. This is a type mismatch — GHDL rejects it:

```
error: can't match "new3c2" with type array type "logic_array_2d"
```

The correct emission should use `set_logic_array_2d_arity_1`:
```vhdl
set_logic_array_2d_arity_1(p_blk, 11, new3c2);
```

This affects all 6 `EncryptBlock`/`DecryptBlock` components which write to `this.blk[i]`
inside `SubBytes`, `ShiftRows`, `MixColumns`, `InvSubBytes`, `InvShiftRows`, `InvMixColumns`,
and `ApplyRoundKey`.

**Workaround**: Keep these round-op functions as plain `fn` with `state {}` blocks.

---

## 30. [BUG] Void inline fn procedure: class static calls lack package qualifier in VHDL

**Status**: Active bug. Do not use void `inline fn` with class static calls until fixed.
Tracked as **livt #64** in livt/COMPILER.md.

When a void `inline fn` body calls a `public static inline fn` on a class
(e.g. `AesSBox.Substitute(bsb0)`), the VHDL generator emits a bare function call
`substitute((input => bsb0))` inside the generated `procedure` body, without the
package-qualified prefix `livt_crypto_internal_aessbox_package.substitute(...)`.
GHDL rejects this with:

```
error: no declaration for "substitute"
```

This is the same package-qualifier bug that affects component-to-class calls at procedure
scope, but here it manifests inside the generated `procedure` body rather than at the
FSM process level.

**Workaround**: Keep any function that calls class static methods as a plain `fn` with
a `state {}` block, where the class calls are correctly emitted with package qualifiers.

---

## 29. [WORKAROUND] `class` `public const` array fields silently dropped by VHDL generator

**Status**: Compiler bug — workaround applied in `AesSBox` and `AesSBoxInverse`.
Tracked as **livt #63** in livt/COMPILER.md.

`public const` array fields declared inside a `class` are silently omitted from the
generated VHDL package body. The function body still references them by name, causing
GHDL errors:

```
error: no declaration for "table_q0"
error: no declaration for "table_q1"
...
```

Root cause: A Livt `class` has "no state, no constructor, no fields" per the language
spec. The VHDL generator follows this constraint and drops all field declarations,
including `public const` ones.

**Workaround**: Replace array constant lookups with enumerated
`if (idx == 0xNN) { return 0xVV }` chains (same pattern as `AesGfMath.GetRcon`).
Applied in `AesSBox.Substitute` (256 entries) and `AesSBoxInverse.Substitute` (256 entries).

---

 Feature request: void `inline fn` that writes to component fields — emit as VHDL `procedure`

**Status**: Parser now accepts void `inline fn`, and the VHDL generator now emits a proper
VHDL `procedure`. However, two new VHDL generation bugs remain (§30, §31). Do not use
void `inline fn` until §30 and §31 are fixed.

### Observation (May 2026 — confirmed by attempting the §27 migration)

Applying `inline fn` (void) to `SubBytes`, `ShiftRows`, `MixColumns`, and `ApplyRoundKey`
in `EncryptBlock128` (and all other Encrypt/Decrypt variants) revealed that:

- The **Livt parser accepts** void `inline fn` syntax — no compile-time rejection.
- The **VHDL generator silently drops** all `var` declarations from void inline fn bodies.
  Only the write-back statements (`this.blk[N] = ...`) are emitted at the call site.
  Local variables (e.g. `bk0`, `r0`, `c00`, `new0c0`) are referenced in the generated
  VHDL but never declared, producing dozens of GHDL errors:
  ```
  error: no declaration for "bk0"
  error: no declaration for "r0"
  ...
  ```
- **No VHDL `procedure` declarations are emitted** — the generator skips step 4 entirely
  and instead inlines only the statement bodies, discarding the variable declarations.

**Conclusion**: Items 1–2 from "Required generator changes" below are done (parser
accepts void inline fn and allows `this.field` writes), but items 3–6 are not.
The source files were reverted to `fn` + `state {}` until the generator is fixed.

### Motivation

`inline fn` inside a component is currently restricted to:
1. A declared return type (non-void)
2. No `state {}` blocks (purely combinatorial)

This prevents in-place mutation procedures like `SubBytes`, `ShiftRows`, `MixColumns`,
`ApplyRoundKey`, `InvSubBytes`, `InvShiftRows`, `InvMixColumns` from being marked `inline`.
As regular `fn`, each call goes through the run/busy dispatcher (~5 extra FSM cycles).
For AES-128 encryption (10 rounds × 4 round ops + final 2), this is approximately
**38 dispatcher calls × ~5 cycles ≈ 190 wasted cycles per encrypt**.

### Proposed VHDL emission: `procedure` instead of `function`

A VHDL `procedure` accepts `inout` signal parameters and can read and write them.
Execution is still combinatorial (no `wait` statements), so the FSM caller advances by
exactly **one clock cycle** per call — identical to the current dispatcher overhead but
without the handshake overhead.

**Example**: `SubBytes` as a void inline fn:

```livt
inline fn SubBytes()
{
    var b0:  byte = this.blk[0]
    ...
    var b15: byte = this.blk[15]
    this.blk[0]  = this.sbox.Substitute(b0)
    ...
    this.blk[15] = this.sbox.Substitute(b15)
}
```

Would emit in the architecture declaration region as:

```vhdl
procedure sub_bytes(signal blk : inout t_blk_type) is
begin
    blk(0)  <= livt_crypto_internal_aessbox_package.substitute((input => blk(0)));
    blk(1)  <= livt_crypto_internal_aessbox_package.substitute((input => blk(1)));
    ...
    blk(15) <= livt_crypto_internal_aessbox_package.substitute((input => blk(15)));
end procedure;
```

Call site in the FSM (one FSM state, no dispatcher):

```vhdl
sub_bytes(this_blk);
```

### Required generator changes

1. **Parser / type-checker**: Allow `inline fn` with no return type (void).
2. **Parser / type-checker**: Allow `this.field[i] = ...` inside an `inline fn` body.
3. **`state {}` handling**: The `state {}` blocks inside void inline fn dissolve — the
   procedure body executes within a single FSM step at the **caller**, so the
   "all writes in one cycle" semantics of `state {}` are already satisfied by the
   caller's clock edge. Strip `state {}` wrappers during inline fn code generation.
4. **Code generator — function vs procedure selection**: When an `inline fn` has void
   return type OR writes to any `this.*` field, emit a VHDL `procedure` instead of a
   VHDL `function`.
5. **Parameter inference**: Analyse the function body to collect:
   - All `this.field` reads → `in` parameter (or `inout` if also written)
   - All `this.field` writes → `out` parameter (or `inout` if also read)
   - Cross-component inline calls (e.g. `this.sbox.Substitute(b)`) → already emit as
     package-qualified function calls; no change needed.
6. **Call site emission**: Replace the dispatcher handshake with a single procedure call
   statement passing the field signals: `sub_bytes(this_blk);`

### Impact on this project

After this feature is implemented, the following functions in all six AES encrypt/decrypt
components can be marked `inline`:

| Function | Components |
|----------|------------|
| `SubBytes` / `InvSubBytes` | `EncryptBlock128/192/256`, `DecryptBlock128/192/256` |
| `ShiftRows` / `InvShiftRows` | same |
| `MixColumns` / `InvMixColumns` | same |
| `ApplyRoundKey` | same |

No changes to test files are needed — the observable behaviour is identical.

---

## 27. `inline fn` inside a component: current constraints

1. **Void `inline fn` is now parser-accepted** — `inline fn Foo()` (no return type)
   is no longer rejected at parse time. However, the VHDL generator is broken for this
   case (§28 observation). Do **not** use void `inline fn` until §28 is fully fixed.

2. **Must not contain `state {}` blocks** — still rejected with:
   `ERROR: Inline functions must not contain 'state {}' blocks — they must be purely combinatorial.`

**Consequence for round-op functions** (`SubBytes`, `ShiftRows`, `MixColumns`,
`InvSubBytes`, `InvShiftRows`, `InvMixColumns`, `ApplyRoundKey`):

Keep these as plain `fn` with `state {}` blocks until §28 VHDL generator bug is fixed.

---

## 16. Variable declarations are valid inside a `state {}` scope

**Status**: Supported (confirmed May 2026).

Local `var` declarations are valid inside a `state {}` block. All statements within a
single `state {}` execute in one clock cycle, so variables declared there are resolved
combinatorially within that cycle and are in scope for the remainder of the block.

```livt
state {
    var lo: byte = this.blk[0]
    var hi: byte = this.blk[1]
    this.blk[0] = hi
    this.blk[1] = lo
}
```

This is safe as long as:
- The declared variable is only used within the same `state {}` block (it does not
  persist across clock boundaries).
- No cross-component calls appear inside the same `state {}` block (cross-component
  calls require their own FSM handshake and cannot be fused into a `state {}`).

**Practical use**: Intermediate XOR chains in `MixColumns` and the read-then-write
patterns in `ShiftRows`/`InvShiftRows` can fold their intermediate `var` declarations
into a `state {}` block, collapsing multiple sequential cycles into one.

---

## 6. VHDL reserved words are not escaped

**Symptom**: Using `state` or `block` as a Livt variable or field name causes GHDL
analysis errors because those identifiers are reserved in VHDL-2008.

**Workaround**: Avoid VHDL reserved words as Livt identifiers (e.g., use `blk` instead
of `state` or `block`).
