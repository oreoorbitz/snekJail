# 2. Python Subset Specification

## 2.1 Compatibility rule

The subset is defined by **accepted syntax plus preserved semantics plus explicit resource policy**. It is not defined by “whatever the backend can emit.” For every accepted program, behavior must match CPython 3.15 within the documented subset and execution profile, except for explicitly listed implementation-defined or resource-limited behavior.

Primary semantic references are the Python 3.15 [execution model](https://docs.python.org/3.15/reference/executionmodel.html), [data model](https://docs.python.org/3.15/reference/datamodel.html), [expressions](https://docs.python.org/3.15/reference/expressions.html), [simple statements](https://docs.python.org/3.15/reference/simple_stmts.html), [compound statements](https://docs.python.org/3.15/reference/compound_stmts.html), and [built-in types](https://docs.python.org/3.15/library/stdtypes.html).

Three categories must remain separate:

1. **Python semantics:** for example, arbitrary-precision integers and left-to-right expression evaluation.
2. **Subset restriction:** for example, no user-defined classes in P0.
3. **Deployment limit:** for example, a configured maximum integer bit length or output size.

A deployment limit must produce a specified resource error. It must not silently wrap, truncate, reorder, or coerce a value.

## 2.2 Feature tiers

| Area | P0 — executable core | P1 — useful programs | P2 — extended compatibility |
|---|---|---|---|
| Modules | One static module; required `run(input)` entry point | Multiple AOT modules/packages | Broader package metadata |
| Values | `None`, `bool`, `int`, `float`, `complex`, `str`, `bytes`, `tuple`, `list`, `dict`, `set` | `bytearray`, `frozenset`, `range` as a public type | `frozendict`, `memoryview`, `sentinel` |
| Functions | Positional/keyword parameters, defaults, `*args`, `**kwargs`, nested functions, closures, recursion | Lambdas, decorators, keyword reflection within defined limits | Full descriptor/function introspection |
| Operators | Builtin arithmetic, comparison, boolean, bitwise, subscription, slicing | In-place protocol extensions | User-defined numeric and attribute protocols |
| Control flow | `if`, loops and loop `else`, `break`, `continue`, conditional expressions | `with`, `match` subset | Full pattern matching and context-manager protocol |
| Comprehensions | List/set/dict comprehensions | Nested/async-adjacent forms where supported | Generator expressions and 3.15 unpacking extensions |
| Exceptions | Builtin exception hierarchy subset, `try`/`except`/`else`/`finally`, `raise`, `raise from` | Custom exceptions, context managers | Exception groups, full traceback APIs |
| Classes | Excluded | Single-inheritance classes, methods, properties, selected special methods | Multiple inheritance/C3 MRO, descriptors, slots, metaclasses |
| Generators/async | Excluded | Generator functions as explicit state machines | `yield from`, async functions, async iteration/context managers |
| Imports | Eager static imports from an allowlist | Static package graphs and selected pure-Python stdlib | Versioned lazy-import evaluation if needed |
| Dynamic execution | Excluded | Excluded | Still excluded by default for sandbox safety |
| Reflection | `type`, `isinstance`, `len`, `repr` on supported values | Read-only selected attributes | Broader introspection, never ambient mutation by default |
| Strings | Unicode code-point semantics, formatting subset | Full formatting and additional codecs | t-strings and broader Unicode APIs |
| Typing | Annotations parsed for optional static hints; runtime annotation reflection rejected | Typed fast paths with guards | Broader typing syntax as syntax, not proof |

The tiers are cumulative only where stated. A higher tier must not silently weaken a P0 safety rule.

## 2.3 Program and entry-point contract

P0 source is a module containing exactly one public entry point:

```python
def run(input):
    """Receive a host-decoded value and return a host-encodable value."""
    ...
```

Proposed rules:

- `run` must be a synchronous function with one positional parameter.
- Top-level code may define functions, assign constants, and perform allowlisted eager imports. All top-level execution still occurs inside the fresh request instance, before `run` is called.
- The return value crosses the host boundary using the canonical encoding in [security and host integration](05-security-host-integration.md).
- `print` writes to a bounded captured stdout stream, not directly to the server terminal.
- Uncaught Python exceptions become structured guest-error responses. Runtime traps and policy terminations use different response categories.
- Module code must not execute during trusted compilation. Constant folding must not call source-defined functions or perform imports.

A separate script-snippet mode may be added later, but it should lower to the same request-scoped entry-point model rather than introducing persistent ambient globals.

## 2.4 Lexical and syntactic baseline

The lexer follows the [Python lexical rules](https://docs.python.org/3.15/reference/lexical_analysis.html) for source encoding, line joining, indentation, identifiers, literals, and operators. P0 accepts:

- decimal/binary/octal/hexadecimal integer literals, binary64 float literals, and imaginary/complex literals;
- single, double, triple, raw, bytes, and formatted string literals, subject to formatting support;
- list, tuple, dict, and set displays;
- the operators required by the accepted expression forms;
- Unicode identifiers with the same normalization rules as Python.

P0 rejects, rather than reinterprets:

- t-strings;
- `global`-style dynamic code construction through `eval`, `exec`, or `compile`;
- lazy imports, type aliases with runtime typing behavior, and unsupported soft-keyword forms;
- `yield`, `await`, async statements, class definitions, and `with` until their tiers are implemented;
- source constructs whose resource requirements exceed compile policy.

The parser may recognize later-tier syntax solely to issue a precise `E_UNSUPPORTED_FEATURE` diagnostic. It must never discard an unsupported node and continue as though it were a no-op.

## 2.5 Names, binding, and scopes

P0 uses Python’s lexical binding rules for module, function, and comprehension scopes. The [execution model](https://docs.python.org/3.15/reference/executionmodel.html) is the contract.

Required behavior:

- assignment in a function makes the name local unless declared `global` or `nonlocal`;
- reading an unbound local raises `UnboundLocalError`;
- unresolved module/global names raise `NameError`;
- `global` and `nonlocal` affect binding, not value copying;
- closures capture cells, so later mutation is visible through all captured references;
- default arguments are evaluated once when the function definition executes;
- loop targets and comprehension targets follow Python’s scope rules; comprehension implementation inlining must not leak targets;
- builtins are a fixed allowlist shadowed according to Python’s normal name-resolution rules.

P0 may resolve builtins to immutable runtime entries after name resolution. It must preserve explicit user shadowing such as a local variable named `len`.

Prohibited P0 operations include mutation of `globals()` or `locals()`, runtime creation of new bindings through string names, and dynamic modification of the builtins mapping.

## 2.6 Evaluation and side-effect order

The [expression reference](https://docs.python.org/3.15/reference/expressions.html#evaluation-order) specifies left-to-right expression evaluation and right-hand-side-before-left-hand-side assignment evaluation. The compiler must preserve:

- left-to-right evaluation of operands, call arguments, displays, and subexpressions;
- short-circuit `and`/`or`, returning the deciding operand rather than coercing it to `bool`;
- single evaluation of intermediate operands in chained comparisons;
- single evaluation of call targets and subscription/attribute targets where required;
- augmented assignment’s target evaluation, in-place operation, and store ordering;
- RHS evaluation before assignment targets are stored;
- cleanup of active iterators and exception state on every exit path supported by the subset.

Constant folding is legal only when it preserves values, types, side effects, exceptions, and resource behavior visible to the program. For example, folding a huge literal multiplication at compile time can bypass a runtime allocation limit and should be rejected or accounted against a documented compile policy.

## 2.7 Value and object model

Every P0 value has identity, a supported type, and behavior. Variables and container elements hold references to values, not copied structures. Mutation through one alias must be visible through every other alias.

| Type | Required semantics | Important restrictions/policy |
|---|---|---|
| `NoneType` | Singleton `None` | No attributes beyond supported generic operations |
| `bool` | `False`/`True`; subclass-of-int behavior where supported | Do not silently replace all booleans with integers in serialization |
| `int` | Arbitrary precision; signed shifts and Python division/floor/modulo semantics | Configured bit-length, conversion, and allocation limits; never machine-word wrap |
| `float` | IEEE-754 binary64 behavior with Python-level exceptions and conversions | Python division by zero raises; NaN payload reproducibility is not promised |
| `complex` | Real/imaginary binary64 pair with Python numeric conversions and arithmetic | Included so operations such as a negative base with a fractional exponent need not change Python’s result domain |
| `str` | Immutable sequence of Unicode code points, including valid handling of lone surrogates internally | Host encoding may reject values not representable in its string encoding |
| `bytes` | Immutable sequence of integers 0–255 | Bounded length and allocation |
| `tuple` | Immutable, ordered, heterogeneous sequence | Immutability is shallow |
| `list` | Mutable ordered sequence with aliasing and slicing | Growth and total elements bounded |
| `dict` | Insertion-ordered mapping; Python-compatible equality/hash behavior for supported keys | Unhashable keys raise `TypeError`; mutation during iteration follows defined errors |
| `set` | Unordered unique elements using supported hash/equality rules | Iteration order is not a cross-runtime compatibility promise |

Python’s arbitrary-precision integer behavior is described under [numeric types](https://docs.python.org/3.15/library/stdtypes.html#numeric-types-int-float-complex). WebAssembly’s fixed-width integers are implementation building blocks, not a license to introduce overflow.

### Numeric edge cases requiring tests

- `1 / 0`, `1 // 0`, and `1 % 0` raise Python exceptions rather than producing Wasm-style results or traps.
- Floor division and modulo preserve Python’s sign relationships for negative operands.
- `int`/`float` mixed operations apply Python conversion rules and overflow behavior.
- Shifts by negative counts raise; huge shift counts are subject to resource errors rather than unbounded allocation.
- `bool` participates in integer operations as Python specifies.
- `round`, `pow`, and float-to-int conversion need dedicated semantic tests.
- A negative exponent can produce a float, `0.0 ** negative` raises `ZeroDivisionError`, and a negative base with a fractional exponent can produce `complex`; excluding complex results while accepting `**` would silently narrow Python semantics.
- NaN and signed zero comparisons must not be optimized as ordinary ordered numbers.

## 2.8 Truth, equality, identity, hashing, and ordering

- `bool(x)` uses the supported type’s truth rule; empty supported containers and strings are false, zero numbers are false, and `None` is false.
- `x is y` compares identity, not equality.
- `==` and `!=` use value equality for supported builtins, including Python’s cross-numeric behavior.
- Ordered comparisons are defined only for supported operand combinations; invalid combinations raise `TypeError`.
- Dict and set behavior depends on compatible hash/equality pairs. Numeric keys that compare equal must behave as Python specifies.
- Hash values need not equal CPython’s hash values and should be instance-randomized unless determinism policy says otherwise. Persisted protocols must not expose them as stable identifiers.
- Set/dict iteration order must not be used as a substitute for sorted output. Dict insertion order is preserved; set order remains unspecified.

P0 supports `isinstance` and `type` only for supported builtin types and exact, non-mutating queries. `id` is excluded initially; a later implementation may return an instance-unique token but must not expose a host or linear-memory address.

## 2.9 Functions and calls

P0 function definitions support:

- positional-only, positional-or-keyword, and keyword-only parameters;
- defaults and keyword defaults;
- `*args` and `**kwargs`;
- nested definitions and closures;
- recursion subject to a configured frame/stack limit;
- docstrings as static metadata;
- annotations under the restrictions below.

Call binding must detect duplicate, missing, and unexpected arguments with Python-compatible `TypeError` behavior. Argument expressions and unpacking are evaluated in Python order.

Annotations are not proof of runtime type. P0 may use them for optimization only when all dynamic assumptions are guarded or proven. Access to runtime annotation reflection such as `__annotations__` is rejected. This is an explicit subset restriction, particularly important because modern Python uses deferred annotation evaluation.

Decorators and lambdas are P1 to keep the initial call/runtime surface small, although neither is inherently incompatible with AOT.

## 2.10 Iteration and comprehensions

P0 iteration supports builtin sequences, `range`, `dict` views/keys as specified, `enumerate`, `zip`, and `reversed` where implemented. The runtime operation is expressed in terms of iterator acquisition, `next`, and `StopIteration`, even if builtins use optimized internal paths.

Required cases:

- `assert` is always enabled in the P0 execution profile and raises `AssertionError`; optimization must not silently remove it;
- loop `else` executes only when the loop is not terminated by `break`;
- iterator cleanup occurs on exceptional exits where applicable;
- mutating a dict during iteration raises the appropriate runtime error rather than reading invalid storage;
- slicing and iteration remain bounded by resource policy;
- list, set, and dict comprehensions preserve target scope and left-to-right nested iteration;
- comprehension filtering evaluates conditions in order.

User-defined iterator protocols and generator-based iterators arrive with later tiers.

## 2.11 Exceptions and `finally`

P0 includes exceptions from the beginning because ordinary operations can fail. It supports:

- `raise exc` and `raise exc from cause`;
- `try`/`except` with supported exception classes and optional binding;
- `try`/`else` and `try`/`finally`;
- exception propagation through function calls and loops;
- cleanup on normal, return, break, continue, and exceptional exits;
- a curated hierarchy including `Exception`, `ArithmeticError`, `ZeroDivisionError`, `TypeError`, `ValueError`, `OverflowError`, `LookupError`, `IndexError`, `KeyError`, `NameError`, `UnboundLocalError`, `AttributeError`, `RuntimeError`, `StopIteration`, `MemoryError`, and `RecursionError`.

The implementation must distinguish:

1. a Python exception that guest code may catch;
2. a policy denial, which should not be convertible into permission to retry indefinitely;
3. a resource termination such as out-of-fuel;
4. an engine trap or compiler/runtime defect.

Whether a given resource error is guest-catchable is a policy decision, but host wall-clock termination and fatal traps must not be caught by guest code. P0 should make capability denials and execution-budget exhaustion uncatchable control outcomes while reporting them separately from Python exceptions.

Custom exception classes require P1. `except*`, exception groups, and traceback mutation are P2. Syntax that modern Python warns against for exiting a `finally` block should be rejected by the subset rather than assigned new behavior.

## 2.12 Classes and dynamic dispatch roadmap

P1 classes should initially support:

- a fixed class definition evaluated during module initialization;
- one explicit base class plus `object`;
- instance attributes and methods;
- `self`, `super()`, and single-inheritance lookup;
- `__init__`, `__repr__`, selected comparison/operator methods, and properties;
- custom exceptions deriving from an allowlisted exception base.

P1 excludes metaclasses, dynamic base lists, `__getattribute__`, arbitrary monkey patching of builtin types, and runtime mutation of class bases. Multiple inheritance, C3 linearization, full descriptors, slots, `__getattr__`, `__setattr__`, and metaclass hooks belong to P2.

This staging lets P0/P1 optimize exact builtin operations without pretending to implement Python’s complete dynamic object model.

## 2.13 Imports and module graph

P0 imports must be statically discoverable and eager:

```python
import math
from json import dumps, loads
```

The compiler resolves each import against a signed/locked module manifest before code generation. No filesystem search occurs in the guest. Runtime `__import__`, `importlib`, and string-driven module loading are excluded.

P0 candidate modules:

| Module | Initial surface | Notes |
|---|---|---|
| `math` | Pure numeric functions and constants | Cross-libm last-bit equality requires a compatibility policy |
| `json` | `loads`, `dumps` with bounded depth/size and explicit options | Parse/serialize only canonical supported values |
| builtins | Listed below | Fixed allowlist, not the full builtins module |

P1 candidates include `base64`, `binascii`, `collections` subset, `dataclasses` subset after classes, `statistics`, `string`, and a carefully bounded `re` implementation. `random`, `secrets`, `time`, `datetime`, filesystem, and network APIs require explicit host capabilities and determinism policy. `os`, `sys`, `subprocess`, `socket`, `ctypes`, `mmap`, `signal`, `threading`, and `multiprocessing` are not P0 compatibility targets.

Python’s [WebAssembly availability notes](https://docs.python.org/3.15/library/intro.html#webassembly-platforms) reinforce that process, thread, networking, signal, IPC, and filesystem behavior is limited or platform-dependent. The subset should expose purposeful host services rather than emulate ambiguous POSIX stubs.

## 2.14 Builtin allowlist

Initial P0 candidates, each requiring tests:

```text
abs all any bool dict enumerate filter float format int len list map
max min next pow print range repr reversed round set slice sorted str
sum tuple type isinstance zip
```

`iter` and `next` operate only on supported builtin iterators. `print` requires keyword handling and bounded output. `format` and `repr` must be implemented per supported type; they must not route through host-language formatting.

Excluded in P0 include `breakpoint`, `bytearray` until supported, `callable` until broader objects exist, `compile`, `delattr`, `dir`, `eval`, `exec`, `getattr`, `globals`, `hasattr`, `hash` as a public stable value, `help`, `id`, `input`, `locals`, `memoryview`, `object` as a class base, `open`, `setattr`, `super`, `vars`, and `__import__`.

## 2.15 Explicit semantic non-goals

Unless a later profile states otherwise, the subset does not guarantee:

- CPython bytecode, `.pyc`, C API, or native-extension compatibility;
- CPython object addresses, allocation counts, reference counts, or collection timing;
- `__del__` timing or resurrection;
- exact CPython `repr` for every value or exact traceback text;
- stable hash values or set iteration order across runs;
- bit-identical NaN payloads or platform-libm results for every math function;
- wall-clock timing, scheduling, or thread behavior;
- arbitrary package compatibility merely because source syntax is accepted.

Exception message text is not treated as a stable compatibility contract; exception classes, control flow, chaining, and cleanup behavior are. These exclusions do not permit deviations in accepted core semantics such as integer overflow, mutation aliasing, or exception cleanup.

## 2.16 Diagnostics

Compiler diagnostics should be deterministic and structured:

```json
{
  "code": "E_UNSUPPORTED_FEATURE",
  "severity": "error",
  "feature": "async-function",
  "tier": "P2",
  "span": {"start": 42, "end": 61, "line": 4, "column": 1},
  "message": "async def is not supported by the P0 profile"
}
```

Required code families:

- `E_SOURCE_ENCODING`, `E_SYNTAX`, `E_AST_LIMIT`;
- `E_SCOPE`, `E_UNRESOLVED_IMPORT`, `E_UNSUPPORTED_FEATURE`;
- `E_TYPE_ASSUMPTION` for an invalid explicitly typed profile, never for ordinary dynamic Python;
- `E_RESOURCE_COMPILE` for compiler-budget exhaustion;
- `E_INTERNAL` for compiler defects, without exposing sensitive paths or memory data.

Warnings must not silently convert Python into a different language. An accepted program with changed behavior is a compiler bug or a documented profile restriction that was incorrectly accepted.

## 2.17 P0 conformance checklist

P0 is ready for optimization work only when:

- every accepted AST node has a documented lowering and semantic tests;
- numeric edge cases, aliasing, evaluation order, closures, calls, and `finally` pass differential tests against CPython 3.15;
- unsupported syntax and imports fail deterministically before execution;
- arbitrary-precision integers remain correct until a declared resource limit is reached;
- no guest API exposes ambient filesystem, process, network, environment, clock, or randomness access;
- result serialization rejects unsupported values without emitting partial success;
- all compiler and runtime limits have dedicated exhaustion tests;
- generic paths remain available for every speculative optimization that is not fully proven.
