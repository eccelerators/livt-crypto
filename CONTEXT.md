# Livt Agent Context

This file is a project-agnostic, self-contained handoff for agent sessions
working with Livt projects. It covers the Livt language, conventions, build
workflow, and practical style. Keep project-specific goals, component lists,
and milestone status in a repository `AGENTS.md` or `README.md`; keep detailed
compiler failures in `COMPILER.md`.

## What Livt Is

Livt is a programming language for describing FPGA/HDL-oriented behavior with a
software-like component structure. A Livt project usually contains components,
interfaces, functions, processes, tests, and generated VHDL output.

Common use cases:

- hardware application logic
- reusable protocol or peripheral components
- small fixed-function FPGA applications
- tests that compile to VHDL simulations

The goal is not to hide hardware. Good FPGA work still requires clear thinking
about timing, state, signals, reset behavior, and resource usage. Livt raises
the level at which you express that hardware, so more of your attention can go
into the design itself instead of repeated HDL boilerplate.

## Project Files

Typical files:

- `livt.toml`: project configuration, source root, output directory, tests,
  dependencies.
- `src/**/*.lvt`: source components. Files may be flat or grouped in
  subdirectories; namespaces define the logical API.
- `tests/**/*.lvt`: Livt test components.
- `.livt/deps/`: synchronized dependencies.
- `out/`: generated build output.
- `COMPILER.md`: project-local notes about compiler or VHDL generation issues.
- `README.md`: human-facing overview and usage.
- `AGENTS.md`: optional project-local continuation notes for agents, such as
  goals, architecture, API shape, and status.

For agent handoffs:

- Use this `CONTEXT.md` for reusable Livt knowledge.
- Use `README.md` or `AGENTS.md` for project-specific continuation instructions.
- Use `COMPILER.md` only for reproducible current compiler/generation bugs and
  local workarounds. Remove stale warnings once the compiler or verification
  suite proves a pattern is supported.

## Practical Compiler Notes

This file summarizes both the Livt book's intended language model and practical
patterns that have compiled in real projects. When they differ, prefer the
conservative pattern until `livt test` proves a broader form works.

Current conservative defaults:

- Use the Livt types described in the Data Types section. For arithmetic, `int`
  is the safest default, even for algorithms that are conceptually unsigned. Use
  `logic[N]` when explicit hardware width matters.
- Declare component fields with field syntax, described in the Fields section.
  Use `var` only for local variables inside functions, processes, states, and
  tests.
- If a local variable declaration produces `no viable alternative at input
  'var'`, move locals to the top of the function/process/state, give them
  explicit types, inline simple expressions, or reduce the number of locals.
- Treat `for` and `foreach` as convenience syntax, not the safest baseline. For
  bit-serial operations, synthesizable datapaths, and parser-sensitive code,
  prefer a `while` loop with a predeclared index, named states, or manual
  unrolling.
- Mixed arithmetic on `byte`, `logic[N]`, `int`, and `uint` should use explicit
  casts at the operator, not only at the assignment target. Cast bytes to `int`
  before arithmetic, masks, and shifts, then cast back deliberately.

When a pattern fails, reduce it to the smallest reproducer, record the exact
error and workaround in project-local `COMPILER.md`, and then use the workaround
throughout that project.

## Comment Style

Follow the Livt comment style from `livt-book/DESIGN_GUIDE.md`.

Use full aligned `/** */` doc comments immediately before component
declarations, constructors (`new()`), functions, and process declarations, even
when the comment is a single sentence:

```livt
/**
 * Resets the accumulator to zero.
 */
public fn Reset()
```

Use `//` comments for fields, constants, and short inline implementation notes:

```livt
// Maximum signed 16-bit accumulator value.
public const ACC_MAX: int = 32767
```

Prefer self-documenting names over comments that merely explain what the code
does. A comment should usually explain API contracts, ranges, timing
assumptions, byte layouts, or why a non-obvious design choice exists. For
byte-builder functions, include the byte-range layout in the `/** */` comment.

## Basic Livt Shape

### Namespaces

A namespace groups related declarations and prevents name collisions:

```livt
namespace Example.Project
```

Everything declared in that file belongs to `Example.Project`. A file can import
another namespace with `using`:

```livt
namespace Example.Project.Tests

using Example.Project
```

After the `using`, code in `Example.Project.Tests` can refer to components from
`Example.Project` by their short names.

File-scoped and block-scoped namespace forms are both used in verified projects:

```livt
namespace Example.Project;

component A
{
}
```

```livt
namespace Example.Project
{
	component A
	{
	}
}
```

Semicolons are accepted as statement separators in many examples, but the book
style generally omits them. Follow the surrounding project style.

### Components

A component is the main unit of Livt design. It is similar to a hardware module:
it can hold state, expose public behavior, connect to other components, and
generate VHDL.

```livt
namespace Example.Project

component ByteReader
{
	public const OFFSET: int = 14

	public fn GetByte(frame: byte[128], index: int) byte
	{
		if (index == 0) {
			return frame[OFFSET]
		}

		return 0x00
	}
}
```

A component should represent a meaningful design boundary, not a single
primitive operation. Good component names describe a role: `PacketParser`,
`ChecksumVerifier`, `RegisterBank`, `UartTransmitter`.

### Classes and Static Functions

A `class` is a named container for `public static fn` and
`public static inline fn` helpers. Unlike a `component`, a `class` has no
state, no constructor, no fields, and no run/busy dispatcher. It exists solely
to group pure combinatorial functions under a shared namespace.

```livt
namespace Livt.Crypto.Internal

class GfMath
{
    public static inline fn Mul2(a: byte) byte
    {
        var aInt: int = a as int
        var doubled: int = aInt * 2
        var low8: int = 0
        if (aInt >= 128)
        {
            low8 = doubled - 256
            return (low8 as byte) ^ 0x1b
        }
        return doubled as byte
    }

    public static inline fn Mul3(a: byte) byte
    {
        var m2: byte = GfMath.Mul2(a)
        return m2 ^ a
    }
}
```

Static methods are called with the class name as the receiver — no `this.`,
no instance, no dispatcher overhead:

```livt
var b2: byte = GfMath.Mul2(b)
var b3: byte = GfMath.Mul3(b)
```

#### `inline` and `static` are orthogonal

`inline` answers *how is this compiled* (emit as a VHDL combinatorial
`function`, no run/busy FSM). `static` answers *how is this called* (as
`TypeName.Method(args)`, no instance needed). They are independent and can
be combined:

| Declaration | Container | Call site | VHDL emission |
|-------------|-----------|-----------|---------------|
| `fn Foo()` | component | `this.Foo()` | FSM dispatcher in architecture |
| `inline fn Foo()` | component | `this.Foo()` | Combinatorial `function` in architecture declaration region |
| `public static fn Foo()` | class | `MyClass.Foo()` | `function` in namespace package |
| `public static inline fn Foo()` | class | `MyClass.Foo()` | `function` in namespace package (with purity constraint) |

For `static` methods, `inline` and non-`inline` produce identical VHDL — a
package function is always combinatorial. The `inline` modifier on a `static fn`
serves as an explicit purity annotation: no `state {}` blocks, no `this.*`
reads or writes, pure combinatorial only.

#### `inline fn` inside a component

An `inline fn` declared inside a `component` is emitted as a VHDL
`function` in the architecture's declaration region and called directly with
no run/busy dispatcher. Same-component calls have zero dispatch overhead:

```livt
component EncryptBlock128
{
    const SBOX_Q0: byte[64] = [ ... ]

    inline fn SBoxFwd(b: byte) byte
    {
        var idx: int = b as int
        if (idx < 64) { return this.SBOX_Q0[idx] }
        ...
    }

    // Called as this.SBoxFwd(x) — zero dispatcher overhead
}
```

**Constraint**: `inline fn` only eliminates overhead for *same-component*
calls. Calling `this.sub.InlineMethod(x)` from a parent component still goes
through the normal component-call dispatcher.

Use component `inline fn` for pure helpers that return a value. Until the local
compiler notes say otherwise, avoid void `inline fn` procedures that mutate
component fields, especially array fields or code that calls class static
helpers. Use a plain `fn` with `state {}` blocks for those in-place operations.

### Fields

Fields are values owned by a component. Declare fields directly in the component
body with `name: type` forms. The declaration shape tells Livt whether the field
is private state, public state, or a hardware signal.

A field without `public` is private stored state:

```livt
component PacketStatistics
{
	acceptedCount: int
	droppedCount: int

	public fn Accept()
	{
		this.acceptedCount = this.acceptedCount + 1
	}
}
```

A public field without a direction is stored public state — state owned by the
component that callers can read directly:

```livt
public acceptedCount: int
```

A public field with `in` or `out` is a signal field — a hardware port:

```livt
public valid: in logic     // driven by the caller, read by this component
public ready: out logic    // driven by this component, read by the caller
```

Use these component field forms:

```livt
count: int              // private stored state
public count: int       // public stored state
public valid: in logic  // input signal driven by the caller
public ready: out logic // output signal driven by this component
```

Use signal fields at hardware boundaries. Use stored fields for internal state
and observable status values. `var` is local-variable syntax and is not part of
component field declarations.

For values that only internal code mutates, default to a private stored field and
add a public function for reads:

```livt
value: int

public fn GetValue() int
{
	return this.value
}
```

Use directionless `public value: int` only when callers truly need direct stored
state access and the local compiler/VHDL generator has proven that public stored
fields work for that type. For hardware boundary signals, always use explicit
`in` or `out`.

### Constants

Constants are named values that do not change. Use `UPPER_SNAKE_CASE`:

```livt
public const OFFSET: int = 14
public const MAX_BUFFER_SIZE: int = 256
public const MAGIC_HEADER: byte[2] = [0xDE, 0xAD]
```

Constants can be private or public. A public constant is part of the component's
API. Constants can also hold fixed-size data:

```livt
public const CRLF: byte[2] = [0x0D, 0x0A]
public const OK: byte[] = "OK".Encode()
```

### Constructors

A constructor describes how a component is created and wired:

```livt
component StatusOutput
{
	enabled: logic
	public active: out logic

	new(enabled: logic, active: out logic)
	{
		this.enabled = enabled
		active = this.active
	}
}
```

Constructors should wire subcomponents and record input bindings. They should
not contain complex logic. Keep them short.

Constructor parameters are how components connect to the outside world and to
each other. Top-level components should keep constructor parameters flat and
primitive, then pass those signals into subcomponents through their constructors.
Output constructor parameters can be forwarded to a child constructor, so a
composition root can expose FPGA pins while the child owns the actual driver
fields:

```livt
component Top
{
	device: Device

	new(a: logic, y: out logic)
	{
		this.device = new Device(a, y)
	}
}
```

### Subcomponents

A component can own another component as a field and instantiate it in the
constructor:

```livt
component PacketPipeline
{
	stats: PacketStatistics

	new()
	{
		this.stats = new PacketStatistics()
	}

	public fn AcceptPacket()
	{
		this.stats.Accept()
	}
}
```

This is composition. Larger systems are built by wiring smaller components
together.

Constructors may also call methods on subcomponents after instantiation. The
compiler generates a startup sequence that runs after reset and before ordinary
process behavior:

```livt
component AccumulatorInit
{
	acc: Accumulator

	new()
	{
		this.acc = new Accumulator()
		this.acc.Add(10)
		this.acc.Add(5)
	}
}
```

Keep constructor calls short and deterministic. For heavier computed
initialization, prefer an explicit startup function and call it from a process
or test, as described later in this file.

### Functions And Parameters

Functions are private by default. You may write `private fn` explicitly, but it
is not required. Add `public` when the function is part of the component API:

```livt
fn IsUpperHex(value: byte) bool
{
	return value >= 0x41 && value <= 0x46
}

public fn IsHexDigit(value: byte) bool
{
	if (value >= 0x30 && value <= 0x39)
	{
		return true
	}
	return this.IsUpperHex(value)
}
```

A function with no return type performs an action:

```livt
public fn Reset()
{
	this.count = 0
}
```

Parameters are inputs by default. You may also write `in` explicitly. Use `out`
when the function writes a value back to the caller, and `inout` only when the
function genuinely needs to read and write the same caller-provided value:

```livt
public fn TryPop(value: out byte) bool
{
	value = 0x00
	return false
}
```

### Processes

Processes describe ongoing behavior. A process runs repeatedly and is part of
the component's hardware behavior.

A process with empty brackets is combinational:

```livt
process Update[]()
{
	this.transfer = this.valid & this.ready
}
```

A process without empty brackets uses the component's context (clock and reset)
and is sequential:

```livt
process Count()
{
	this.count = this.count + 1
}
```

Every component has a context for sequential behavior. When a component receives
clock and reset signals explicitly, bind them in the constructor:

```livt
new(clk: clock, rst: reset)
{
	this.context.clk = clk
	this.context.rst = rst
}
```

### Application-Style Process Loop

In Livt, an application "loop" is commonly expressed as a `process Main()` that
runs continuously in hardware. It should check whether input is available and
`continue` when there is no work:

```livt
namespace Example.App

component PollingApp
{
	device: Device

	new(rx: in logic, tx: out logic)
	{
		this.device = new Device(rx, tx)
	}

	process Main()
	{
		var data: byte = 0x00

		if (!this.device.IsDataAvailable())
		{
			continue
		}

		data = this.device.Receive()

		// Decode input, update state, and emit any response.
		this.device.Transmit(data)
	}
}
```

Request/response behavior belongs inside this process or in components called by
it. This is the hardware application analogue of a software event loop.

### Test Components

Tests are written as Livt components marked with `@Test`:

```livt
namespace Example.Project.Tests

using Example.Project

@Test
component ByteReaderTest
{
	reader: ByteReader

	new()
	{
		this.reader = new ByteReader()
	}

	@Test
	fn ReadsByte()
	{
		var frame: byte[128]

		frame[14] = 0x08

		assert this.reader.GetByte(frame, 0) == 0x08
	}
}
```

Common conventions:

- `namespace` declarations carry the logical structure.
- Files may live directly under `src/` and `tests/` or in subdirectories. Keep
  each project's `livt.toml` paths and `[tests].components` list accurate.
- `component` defines state, constants, constructor wiring, and functions.
- `class` defines a named container for `public static fn` / `public static inline fn` helpers with no state or instance.
- `new()` constructs subcomponents or initializes local fields.
- `new[]()` appears in some verification projects for constructors that are
  explicitly context-free. Prefer the local project's existing style; the book
  examples use `new()`.
- `public fn` exposes a standalone public function.
- `inline fn` exposes a same-component combinatorial helper with zero dispatcher overhead.
- `public static inline fn` on a `class` exposes a cross-component combinatorial helper callable as `ClassName.Method(args)`.
- `override fn` implements a function declared in an interface or inherited from
  a parent component.
- Tests are `@Test component` with `@Test fn ...`.

## Data Types

### Primitive Types

| Keyword | Meaning | Typical use |
|---|---|---|
| `bool` | Two-valued condition: `true` or `false` | Decisions, function results, assertions |
| `logic` | Hardware logic value | Signals, ports, bit vectors |
| `byte` | Unsigned 8-bit value | Protocol data, buffers, encoded text |
| `int` | Signed 32-bit integer | Counters, loop variables, arithmetic |
| `uint` | Unsigned 32-bit integer | Non-negative counts and sizes |
| `string` | Text value | Simulation, constants, encoded byte data |
| `clock` | Clock signal | Sequential process context |
| `reset` | Reset signal | Sequential process context |

Livt's primitive vocabulary is small. It has booleans (`bool`), hardware logic
bits and vectors (`logic`, `logic[N]`), bytes (`byte`), integer arithmetic types
(`int`, `uint`), text (`string`), and timing signals (`clock`, `reset`).
Fixed-size arrays are written by adding dimensions, such as `byte[64]` or
`logic[32]`. Width-suffixed integer families are not separate Livt types; use
`int` for arithmetic and `logic[N]` when the bit width is the important part.

For arithmetic, use `int` as the default even when the algorithm is conceptually
unsigned. Use explicit masks, range checks, and casts to document the intended
low-bit behavior. Use `uint` only after the local compiler version has proven
the exact pattern works.

For a CRC32-style value, choose the representation by intent:

- `int` for the accumulator and polynomial when arithmetic or shifts are needed.
- `logic[32]` for a 32-bit hardware register or bit vector at a signal boundary.
- `byte[4]` when byte layout and serialization matter more than word arithmetic.

### `bool`

Use `bool` for decisions. Comparisons produce `bool`:

```livt
var enabled: bool = true
var small: bool = value < 10
```

Keep `bool` separate from `logic`. A `bool` answers a language-level question.
A `logic` value represents a hardware signal.

### `logic`

Use `logic` for hardware signals. Common literals: `0b0`, `0b1`, `0bU`, `0bX`,
`0bZ`.

`logic[N]` is an N-bit vector:

```livt
var nibble: logic[4] = 0b1010
var word: logic[32] = 0x0000002A
```

### `byte`

`byte` is an unsigned 8-bit value with range 0 through 255:

```livt
var zero: byte = 0x00
var letterA: byte = 0x41
```

Bytes are natural for packet data, UART payloads, memory contents, encoded text,
and protocol fields.

### `int` and `uint`

Use `int` for integer arithmetic and loop counters. The book presents `uint` for
counts or values where negatives do not make sense:

```livt
var offset: int = -4
var length: uint = 1500
```

In current compiler-sensitive code, `int` is usually the safest arithmetic type,
even for logically unsigned algorithms such as CRCs. Use `logic[N]` for explicit
hardware width at ports or registers, but cast to `int` before numeric arithmetic
or shifts when the operator does not accept the vector form. Prefer this:

```livt
var poly: int = 0x04C11DB7
var acc: int = 0
```

Avoid this shape for arithmetic-heavy code:

```livt
var poly: logic[32] = 0x04C11DB7  // good register shape, awkward arithmetic
```

### `string`

Strings are text values, especially useful in tests, simulation reports, and
fixed text that should become byte data:

```livt
Simulation.Report("starting test")
```

A string can be encoded into bytes:

```livt
const GREETING: byte[] = "Hello".Encode()
```

Common escapes work in encoded strings, for example `"\r\n".Encode()` for CRLF
and `"\t".Encode()` for a tab.

Treat strings carefully in synthesizable code. Text is most often used at
compile time, in constants, or in simulation-only APIs.

### Fixed-Size Arrays

Hardware resources are statically allocated, so arrays usually have fixed sizes.
The size is part of the type:

```livt
var payload: byte[64]
var table: int[16]
var matrix: byte[2, 3]
```

Array literals:

```livt
var header: byte[4] = [0xDE, 0xAD, 0xBE, 0xEF]
var offsets: int[3] = [0, 14, 34]
```

Prefer explicit array types on literals. Hardware review and VHDL generation are
clearer when the element type and fixed length are visible:

```livt
var bytes: byte[3] = [0x10, 0x20, 0x30]
```

Multi-dimensional arrays use nested literals:

```livt
var matrix: byte[2, 3] = [
	[0x01, 0x02, 0x03],
	[0x04, 0x05, 0x06]
]
```

Indexing is zero-based:

```livt
var first = header[0]
header[1] = 0xAA
matrix[1, 2] = 0xFF
```

Function parameters can use unconstrained array types when the function should
accept multiple fixed lengths:

```livt
public fn GetFirst(values: int[]) int
{
	return values[0]
}

fn GetLength(bits: logic[]) int
{
	return bits.Length()
}
```

Use `.Length()` when a helper needs the caller-provided array length.

### Slices

Use Python-style `start:stop` slices to read or write a sub-range of a
`logic[N]` vector or byte array. `start` is inclusive and `stop` is exclusive,
so the result width is `stop - start`.

```livt
var word: logic[8] = 0b10110010
var lo: logic[4] = word[0:4]     // bits 3..0
var hi: logic[4] = word[4:8]     // bits 7..4
```

Variable bounds are supported when the slice width is inferable:

```livt
var i: int = offset as int
return word[i:i+4]
```

Slices can also be assignment targets on `logic[N]` fields:

```livt
this.word[4:8] = nibble
```

Slices do not use omitted bounds, negative indices, or step syntax. Write
explicit non-negative bounds:

```livt
word[0:4]      // supported
word[4:8]      // supported
word[:4]       // unsupported
word[4:]       // unsupported
word[::-1]     // unsupported
```

### Logic Vectors vs Array Dimensions

`logic[N]` is a vector, not a list of N separate Livt values. Initialize it with
a binary or hex literal:

```livt
var flags: logic[4] = 0b1010
```

Individual bits can be read or written with zero-based indexing:

```livt
var bit0: logic = flags[0]
this.status[3] = 0b1
```

Use bracket-list literals when the type is an array of elements:

```livt
var bytes: byte[3] = [0x10, 0x20, 0x30]
```

The distinction matters because it maps to different VHDL shapes.

### Type Inference

Livt can infer many local variable types from their initializer:

```livt
var value = 42
var ok = true
```

Use an explicit type when the width, signedness, or hardware shape matters:

```livt
var marker: byte = 0xFF
var flags: logic[8] = 0b00001111
```

For public fields, function parameters, constants, and interfaces, prefer
explicit types.

### Casts

The `as` operator converts a value at a specific point. Use it when a
conversion is intentional, especially when changing signedness or narrowing a
value:

```livt
var word: logic[32] = this.GetWord()
var high: byte = (word[24:32]) as byte
var sum: int = high as int
var low: byte = sum as byte
var bits: logic[8] = sum as logic[8]
```

Common verified conversions:

| Conversion | Meaning |
|---|---|
| `byte as int` / `byte as uint` | Unsigned widening, 0..255 |
| `logic[N] as uint` | Unsigned interpretation |
| `logic[N] as int` | Unsigned integer value of the bit pattern |
| `logic[8] as byte` | Bit-for-bit 8-bit data reinterpretation |
| `byte as logic[8]` | Bit-for-bit 8-bit vector reinterpretation |
| `byte as logic[]` | Open-ended logic vector; width deduced from target, commonly 8 bits |
| `int as byte` / `uint as byte` | Low 8 bits; values outside 0..255 truncate/wrap |
| `int as logic[N]` | Signed two's-complement encoding into N bits |
| `uint as logic[N]` | Unsigned encoding into N bits |

Do not use `as` for `string` conversions. Keep `bool` separate from `logic`;
use comparisons for control flow (`x != 0`, `flag == 0b1`) rather than relying
on casts.

Widening conversions such as assigning `byte` to `int` or `uint` may be
accepted implicitly. Narrowing, reinterpretation, or meaning-changing
conversions should be explicit:

```livt
var b: byte = n as byte
if (b == (0 as byte))
{
	return true
}
```

Arithmetic operators require operands with compatible types. Cast each
sub-expression where the mix happens:

```livt
var sum: int = 0
var item: byte = 0x10
sum = sum + (item as int)
```

This is especially important for bytes. A `byte` is 8-bit data, not an
automatically promoted integer. Promote before arithmetic, masking, or shifting:

```livt
var data: byte = 0x80
var dataInt: int = data as int
var shifted: int = dataInt << 1
var low: byte = shifted as byte
```

The cast belongs where the operator sees it. This can still fail:

```livt
var shifted: int = data << 1     // wrong: operator still sees byte << int
```

Parentheses around casts make mixed expressions easier to read, especially with
slices, masks, and arithmetic.

### Operators

Arithmetic: `+`, `-`, `*`, `/`, `%`

Comparison (return `bool`): `==`, `!=`, `<`, `>`, `<=`, `>=`

Logical (on `bool`): `&&`, `||`, `!`

Bitwise (on `logic`, `logic[N]`, and bytes where appropriate): `&`, `|`, `^`,
`~`

Shifts: `<<`, `>>`

Compound assignment forms are supported for common arithmetic and bitwise
operators:

```livt
result += value
result -= value
result &= mask
result |= flags
result ^= flags
result *= scale
result /= divisor
result %= modulus
```

Increment and decrement are supported as standalone statements and in
expressions:

```livt
index++
remaining--
var next = base + index++
```

When checking `logic`, compare it explicitly:

```livt
if (this.valid == 0b1)
{
	this.Accept()
}
```

## Control Flow

### Variables

Declare local variables with `var`:

```livt
var count: int = 0
var valid: bool = true
```

Variables are local to the function, process, or block where they are declared.
Fields belong to the component; variables belong to the current piece of code.

In the book, `var` appears freely in function bodies and loop headers. Some
current compiler versions are more fragile. For production code, prefer explicit
types and declare locals near the top of the function, process, or state before
loops and nested blocks:

```livt
public fn Sum(values: byte[4]) int
{
	var sum: int = 0
	var index: int = 0

	while (index < 4)
	{
		sum = sum + (values[index] as int)
		index++
	}

	return sum
}
```

If `for (var index = 0; ...)` or `foreach (var item in ...)` triggers
`no viable alternative at input 'var'`, use the predeclared-`while` form above.
For very small fixed mappings, a sequence of direct assignments or guarded
branches is often clearer and more compiler-friendly than a loop.

When the parser is still unhappy with local declarations, reduce local variables
rather than fighting the grammar:

- Inline simple one-use expressions directly into assignments or returns.
- Move repeated logic into a small helper function.
- Use component fields for persistent state, declared with field syntax, not
  `var`.
- Manually unroll a tiny fixed sequence when that is clearer than a loop and
  compiles more reliably.

### Conditions

Conditions should be `bool` expressions:

```livt
if (count == 0)
{
	return true
}
else
{
	return false
}
```

### Guard Clauses

A guard clause handles a special case early and returns immediately:

```livt
public fn IsAsciiDigit(value: byte) bool
{
	if (value < 0x30)
	{
		return false
	}

	if (value > 0x39)
	{
		return false
	}

	return true
}
```

Guard clauses keep the main path of a function less nested.

### Branching

Livt does not have `else if`. Use `elif` for chained branches:

```livt
if (index == 0) {
	return 0x12
} elif (index == 1) {
	return 0x34
} else {
	return 0x00
}
```

In practice, independent guarded returns (as shown above) have proven more
reliable in functions. Use `elif` only where a chained branch is clearly
appropriate and the local project has confirmed it compiles correctly.

### `while` Loops

```livt
var index = 0

while (index < length)
{
	index++
}
```

Use `while` when the stopping condition is the clearest way to express the loop.

### `for` Loops

```livt
var sum = 0

for (var index = 0; index < 4; index++)
{
	sum = sum + values[index]
}
```

Use `for` when iterating over a fixed range, array, or known number of cycles
only after the project has shown that loop form compiles in the current context.
If the `var` in the loop header is rejected by the parser, predeclare the index
and use a `while` loop instead. For bit-serial transforms such as CRC update
steps, manual unrolling is an acceptable fallback:

```livt
acc = this.UpdateBit(acc, bit0)
acc = this.UpdateBit(acc, bit1)
acc = this.UpdateBit(acc, bit2)
acc = this.UpdateBit(acc, bit3)
```

For synthesizable components, also consider explicit named states when each
iteration should happen over time rather than in one combinational expression.

### `foreach` Loops

Use `foreach` to iterate over arrays or iterator components:

```livt
public fn SumBytes(data: logic[4, 8]) logic[8]
{
	var result: logic[8] = 0b00000000
	foreach (var item in data)
	{
		result += item
	}
	return result
}
```

For iterator components, the component should provide the iterator contract used
by the base library (`Init`, `HasNext`, and `Next` in the book; some older
verification examples may use `Reset` instead of `Init`). The loop owns the
traversal for that cursor:

```livt
foreach (var item in this.iter)
{
	sum = sum + item
}
```

Treat `foreach` as a convenience form rather than the safest baseline. If the
compiler rejects `foreach (var item in ...)`, rewrite it as an explicit index
loop over `.Length()` or a fixed bound.

### `break` and `continue`

`break` exits a loop early. `continue` skips the rest of the current iteration.

Inside a `for` loop, `continue` advances through the increment step before the
next condition check. Inside a `while` loop, it jumps back to the condition.

### Process-Level `continue`

Processes run repeatedly. In a process body, `continue` has an additional
meaning: restart the process on the next cycle. This is useful for polling:

```livt
process Main()
{
	if (this.inputAvailable == false)
	{
		continue
	}

	this.HandleInput()
}
```

### State Blocks

Some hardware behavior is naturally multi-step. Livt uses `state` blocks to make
those steps explicit.

An anonymous state groups several statements into one step:

```livt
state {
	this.valid = 0b1
	this.data = 0x41
}
```

A named state can be used as a local jump target:

```livt
process Main()
{
	state Idle
	{
		if (this.start == 0b1)
		{
			goto Load
		}

		goto Idle
	}

	state Load
	{
		this.value = this.input
		goto Done
	}

	state Done
	{
		this.ready = 0b1
		goto Idle
	}
}
```

`goto` is deliberately local. It jumps only to named states in the current
process. It is not a general-purpose cross-function jump.

Use named states when the design has real sequencing over time: protocol
handshakes, multi-cycle algorithms, waiting for external input, staged reads or
writes.

### Cycle Cost and `state` as a Performance Tool

In a **sequential process or function**, every Livt statement compiles to its
own FSM state — one clock cycle per statement. This default is safe and
predictable, but can be slow for compute-heavy code.

A `state {}` block groups all of its statements into a **single** FSM state,
executing them all in the same clock cycle:

```livt
// Without state {}: 4 separate clock cycles
this.a = x ^ y
this.b = x & y
this.c = this.a ^ this.b
this.out = this.c

// With state {}: all 4 assignments in one clock cycle
state {
	this.a = x ^ y
	this.b = x & y
	this.c = this.a ^ this.b
	this.out = this.c
}
```

This is the primary tool for reducing latency in performance-critical functions.
For example, a transform that processes 16 bytes sequentially can take 16+
cycles; wrapping the byte operations in a single `state {}` can reduce that to
one cycle at the cost of a larger combinational path.

### Combinational Processes and Zero-Latency Outputs

A **combinational process** (`process Name[]()`) contains no FSM states at all.
All statements in its body are evaluated combinationally — output updates
instantly when inputs change, with no clock cycles consumed:

```livt
process ComputeTag[]()
{
	this.tag = this.a ^ this.b ^ this.c
}
```

This is useful for derived outputs that are pure functions of stored fields.
Combining a combinational process with `state {}` in sequential functions is
the main strategy for achieving high-throughput designs in Livt.

### Summary: Latency Reduction Strategies

| Approach | Effect |
|---|---|
| Default (no `state {}`) | One clock cycle per statement |
| `state { stmt1; stmt2; ... }` | All statements in one cycle (wider combinational path) |
| `process Name[]()` | Zero-latency combinational output |
| Named states + `goto` | Explicit multi-cycle FSM with reuse of states |

## Interfaces And Inheritance

### Defining an Interface

Interfaces declare contracts without implementation. They can contain function
signatures, constants, and fields:

```livt
interface IMemory
{
	fn Read(address: logic[16]) logic[8]
	fn Write(address: logic[16], data: logic[8])
}
```

### Implementing an Interface

Use `:` to implement an interface. Each contract function must use `override fn`,
not `public fn`:

```livt
component SimpleRam : IMemory
{
	storage: byte[65536]

	override fn Read(address: logic[16]) logic[8]
	{
		return this.storage[address]
	}

	override fn Write(address: logic[16], data: logic[8])
	{
		this.storage[address] = data
	}
}
```

The `override` keyword is required whenever a function fulfils an interface
contract or overrides an inherited function. Using `public fn` for an interface
function is incorrect and will not satisfy the contract.

### Interface Fields (Automatic Contract Fields)

Interfaces may declare contract fields. For signal contracts, write explicit
directions:

```livt
interface ICounter
{
	value: out logic[8]
	fn Increment()
	fn Reset()
}

component UpCounter : ICounter
{
	// 'value' is provided automatically — do NOT redeclare it.

	override fn Increment() { this.value = this.value + 1 }
	override fn Reset()     { this.value = 0 }
}
```

Redeclaring an interface field in the implementing component is a compiler error.

Directionless interface fields such as `value: logic[8]` are intended to model
stored public contract fields, but the book's design notes mark that behavior as
an open implementation gap. Avoid directionless interface fields in production
examples unless a local project has already verified them. Use explicit
`in`/`out` signal fields in interfaces, or use functions for stored state.

### Interface Constants In Scope

Constants declared in an interface are automatically in scope inside the bodies
of functions in any implementing component:

```livt
interface IBus
{
	const DATA_WIDTH: int = 32
	fn Read() logic[DATA_WIDTH]
}

component MyBus : IBus
{
	override fn Read() logic[DATA_WIDTH]
	{
		// DATA_WIDTH is in scope here without qualification.
		return 0
	}
}
```

### Interface Inheritance

Interfaces can extend other interfaces with `:`. An implementing component must
provide `override fn` for all functions in the entire chain:

```livt
interface IBase    { fn GetBase() int }
interface IDerived : IBase { fn GetDerived() int }

component MyComponent : IDerived
{
	override fn GetBase()    int { return 1 }
	override fn GetDerived() int { return 2 }
}
```

### Multiple Interfaces

A component can implement more than one interface by listing them
comma-separated:

```livt
component Combo : IFoo, IBar
{
	override fn GetFoo() int { return 1 }
	override fn GetBar() int { return 2 }
}
```

If two implemented interfaces declare the same function name, one `override fn`
body satisfies both.

### Component Inheritance

One component can extend another with `:`. The child component inherits all
`public fn` declarations from the parent and can call them directly:

```livt
component Animal
{
	public fn GetKind() int { return 1 }
}

component Dog : Animal
{
	public fn GetName() int { return 2 }
	// GetKind() is inherited and callable on Dog instances.
}
```

A child may also override a parent function using `override fn`. Chain
inheritance (`Puppy : Dog : Animal`) is supported.

### Signal Bundles (Interface as Signal Group)

Interfaces are also useful for grouping signals:

```livt
interface IByteStream
{
	valid: in logic
	ready: out logic
	data: in byte
}
```

This keeps protocol signals together. Instead of passing `valid`, `ready`, and
`data` through every constructor separately, a component can accept one stream
interface:

```livt
component StreamConsumer
{
	stream: IByteStream

	new(stream: IByteStream)
	{
		this.stream = stream
	}
}
```

### Interface Direction and `flip`

Declare signal interfaces from one consistent point of view (usually the
consumer). A producer is on the opposite side. Use `flip` when a component
should drive the fields that the consumer normally receives:

```livt
component StreamProducer
{
	stream: IByteStream

	new(stream: flip IByteStream)
	{
		this.stream = stream
	}

	process Produce[]()
	{
		this.stream.valid = 0b1
		this.stream.data = 0x41
	}
}
```

`flip IByteStream` reverses the field directions for this constructor binding.

### Pass-Through Wiring

A parent component can receive an interface and forward it to a subcomponent:

```livt
component StreamStage
{
	consumer: StreamConsumer

	new(stream: IByteStream)
	{
		this.consumer = new StreamConsumer(stream)
	}
}
```

### Test Doubles

Interfaces make tests easier. A test can provide a small implementation of an
interface instead of instantiating a full subsystem:

```livt
component TestByteSource : IByteSource
{
	override fn HasData() bool { return true }
	override fn Read() byte    { return 0x42 }
}
```

### Hierarchy vs Composition

Use hierarchy (interfaces + inheritance) when you want substitutability: several
components can stand in for the same role, and callers should not need to know
which implementation they receive.

Use composition (owning subcomponents) when you want ownership and structure:
one component contains and coordinates other components. Each subcomponent can
be tested independently.

In real Livt systems, both approaches are valid and often appear together.

## Build And Test

Normal test command:

```sh
livt test
```

Compile without running simulations:

```sh
livt build
```

Run a single configured test component or test method:

```sh
livt test -r MyComponentTest
livt test -r MyComponentTest:TestSomething
```

Dependency sync, if needed:

```sh
livt sync
```

Some environments may print Log4j errors because the Livt CLI tries to write a
log file outside the workspace, for example:

```text
/home/.../.livt/logs/livt.log (Read-only file system)
```

Treat this as noise only if the command exits successfully.

Useful CLI commands:

- `livt new Name`: create a new project directory.
- `livt init`: initialize a project in the current directory.
- `livt build`: parse, type-check, and generate VHDL.
- `livt test`: build and run configured test components using GHDL.
- `livt test -r Component[:Method]`: run a filtered test.
- `livt sync`: install dependencies into `.livt/deps/`.
- `livt add`, `livt remove`, `livt lock`: manage dependencies.
- `livt vendor`: create vendor-tool integration files.

When adding new source or test files, incremental generation may miss new
entities. A useful forced-regeneration pattern is:

```sh
rm -rf out .livt/src.json .livt/ghdl
livt test
```

Avoid `livt clean` in projects with synced dependencies unless you know how it
behaves for that project. In at least one project it removed `.livt/deps`, which
then required `livt sync`.

## Practical Livt Style

Naming:

- Namespaces, components, interfaces, and functions use `PascalCase`.
- Interfaces start with `I`.
- Fields and parameters use `camelCase`.
- Constants use `UPPER_SNAKE_CASE`.

Design:

- Give each component one clear responsibility and a small public API.
- Default fields to private; add `public` only for real API state or signals.
- Keep constructors for wiring. Put behavior in functions and processes.
- Prefer pure helpers where possible. For shared stateless helpers, use a
  `class` with `public static inline fn`.
- Bind nested subcomponent-call results to a local before passing them onward,
  unless the direct expression is simple and known to compile.
- Move heavy startup work into an explicit startup function, called from a
  process with an initialization guard or at the start of a test.

## Dependencies

Dependencies appear in `livt.toml`, for example:

```toml
[dependencies]
Ram="1.0.0"
```

Synced dependencies live under `.livt/deps/`. Inspect their source before using
them; dependency APIs are often small and explicit.

## Base Library

The Livt base library provides common types, annotations, and utilities:

- **Annotations**: `@Test` on components and functions
- **Simulation utilities**: `Simulation.Report(...)` for test output and
  `Simulation.Wait(...)` for simulation time in tests
- **String encoding**: `.Encode()` to convert strings to byte arrays
- **Context**: `this.context.clk` / `this.context.rst` for clocked processes

Keep simulation helpers in tests. Use the base library for universal building
blocks; use packages for domain-specific functionality.

Key base library namespaces include:

- `Livt.Bits`: bit manipulation helpers such as extract, insert, count ones,
  parity, reverse, rotate, zero/sign extend
- `Livt.Ascii`: ASCII classification and conversion helpers
- `Livt.Convert`: explicit clamped, wrapping, checked, or truncating conversions
- `Livt.Diagnostics`: test and simulation reporting helpers
- `Livt.String`: string helpers for simulation and tests
- `Livt.Array`: fixed-size array helpers

## Project Lifecycle Notes

- Extract packages only for stable, reusable APIs with tests and documentation.
- Keep synthesizable source separate from tests and simulation-only helpers.
- For vendor integration, keep the top-level component hardware-facing and wrap
  vendor primitives behind stable Livt interfaces.
- In CI, run `livt build` and `livt test`; archive generated VHDL when changes
  to backend output matter.
- For migrations, start with small, well-tested pieces and preserve behavior
  against the existing VHDL/Verilog design.

## Testing Strategy

Add tests immediately after each small component:

1. constants/offset helpers
2. parser field extraction
3. classifier predicates
4. builder byte emission
5. checksum or arithmetic helpers
6. small application composition
7. end-to-end byte-array scenarios
8. platform adapters last

Prefer several small tests over one monolithic integration test. When a test
fails due to compiler/VHDL generation rather than logic, reduce the pattern and
document it.

Use explicit boolean comparisons in assertions:

```livt
assert component.IsReady() == true
assert component.IsError() == false
```

Some compiler versions have generated invalid VHDL for bare boolean assertions
such as `assert component.IsReady()`.

### Testing Stateful Components

Stateful components need tests that perform actions in order:

```livt
@Test
fn CountsAcceptedPackets()
{
	this.stats.Accept()
	this.stats.Accept()

	assert this.stats.acceptedCount == 2
}
```

### Testing Processes

Processes may need time to pass. Use states to give the design cycles:

```livt
@Test
fn UpdatesAfterCycles()
{
	state {}
	state {}

	assert this.counter.count == 2
}
```

For tests and simulation-only components, `Simulation.Wait(n)` is also
available when a time-based wait is clearer than spelling out many empty states:

```livt
@Test
fn CompletesAfterStartup()
{
	Simulation.Wait(100)
	assert this.component.IsDone() == true
}
```

## Recommended Project-Specific Files

For each Livt example project, use:

- `CONTEXT.md`: this reusable Livt agent guide.
- `README.md`: human-facing overview and run instructions.
- `AGENTS.md`: optional project-specific agent notes: goal, scope,
  architecture, current status, next steps, and local quirks.
- `COMPILER.md`: compiler/generation issues found in that project.

Future prompt pattern:

```text
Please read CONTEXT.md and the project README first, then continue the project.
```
