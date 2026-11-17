# 4. AOT Compiler and Runtime Architecture

## 4.1 System decomposition

```text
                 trusted build / control plane
 ┌───────────────────────────────────────────────────────────┐
 │ source + locked module manifest + compiler policy         │
 │   → lexer/parser → subset validator → scope resolver      │
 │   → Python-aware HIR → analyses → CFG → Wasm backend      │
 │   → runtime linking → Wasm validation → artifact manifest │
 └────────────────────────────┬──────────────────────────────┘
                              │ immutable .wasm + manifest
                              ▼
                untrusted execution / data plane
 ┌───────────────────────────────────────────────────────────┐
 │ host application → host adapter → Wasm engine             │
 │   fresh store/instance → guest runtime + generated code   │
 │   narrow host-call import → capability broker             │
 │   bounded response/logs/metrics → host application        │
 └───────────────────────────────────────────────────────────┘
```

The compiler must not execute module top-level code. The execution artifact contains generated functions and the guest runtime, not a general Python evaluator. The host application still needs a WebAssembly engine, either embedded as a library or supplied by a sidecar/service boundary chosen by the deployment.

## 4.2 Trust boundaries

1. **Source boundary:** LLM source is untrusted compiler input. Parser and compiler limits are mandatory.
2. **Artifact boundary:** even compiler-produced Wasm is validated by an independent validator before caching or execution.
3. **Guest/host boundary:** all external effects pass through explicit imports; guest memory is not host memory.
4. **Tenant/request boundary:** mutable guest state is fresh per untrusted request and destroyed afterward.
5. **Serialization boundary:** request, result, logs, and host-call payloads are decoded with size, depth, and type limits.
6. **Runtime supply boundary:** compiler, runtime library, WASI adapters, and engine versions are pinned and recorded.

## 4.3 Compilation stages

### Stage A — Source bundle and policy

Input includes source files, a locked module manifest, language profile, compiler version, target features, and compile resource policy. The compiler canonicalizes paths into logical module names and never reads arbitrary host paths discovered during compilation.

Initial limits should cover:

- source bytes and number of files;
- tokens, AST nodes, identifiers, literal bytes, and nesting depth;
- functions, closures, imports, constants, and static basic blocks;
- compiler wall time, memory, recursion, and emitted-module size;
- optimization work and generated source-map size.

Exceeding a limit returns `E_RESOURCE_COMPILE`; it is not a guest-catchable Python exception.

### Stage B — Lexing and parsing

Produce a Python 3.15-shaped AST with source spans. Compare accepted AST structure with CPython’s [`ast`](https://docs.python.org/3.15/library/ast.html) in tests, but convert immediately to a project-owned AST schema. Do not serialize CPython AST objects into artifacts.

### Stage C — Subset validation

Walk every node and either:

- map it to a supported semantic operation; or
- emit a stable unsupported-feature diagnostic.

This pass also validates closed-world imports, entry-point shape, forbidden reflection, annotation policy, and compile-domain restrictions. No later pass should encounter an unrecognized node.

### Stage D — Name and scope resolution

Resolve each binding to local, cell, free, global, builtin, module, or error. Build closure conversion data and determine which locals require heap cells. Perform whole-block assignment analysis before type inference.

### Stage E — Python-aware high-level IR

HIR preserves Python operations and control flow:

```text
%0 = load_local %x
%1 = load_local %y
%2 = py_add %0, %1                // may dispatch, allocate, or raise
%3 = py_truth %2                  // may call behavior in later tiers
if_true %3 then %then else %else
```

HIR nodes carry value/exception behavior, allocation possibility, source span, and supported-type information. Expressions that can raise have explicit exceptional exits by the time CFG lowering finishes.

### Stage F — Analyses and optimization

Initial analyses:

- exact builtin type inference where sound;
- local liveness and reference roots;
- closure capture and escape analysis;
- constant propagation for immutable literals;
- CFG reachability and cleanup-region construction;
- allocation and host-call safepoints;
- runtime stack maps;
- resource instrumentation points.

Later analyses may include SSA, scalar replacement of non-escaping containers, inline caches keyed by exact runtime type, and guarded unboxing. There is no interpreter to deoptimize into: every speculative path branches to an already compiled generic AOT path.

### Stage G — Lowering to WebAssembly

Lower generic Python operations to runtime calls and proven operations to Wasm instructions. Emit structured control flow or explicit CFG branches, function types, locals, linear-memory data, tables for internal indirect calls, and exports for the host adapter. Validate every operand type and exceptional path.

### Stage H — Verification and packaging

- validate the core module with an independent parser/validator;
- verify that imports/exports match the requested target world;
- disallow unrequested Wasm proposals;
- run deterministic-build comparison;
- record hashes and feature requirements;
- execute smoke and differential tests before publishing to the artifact cache.

## 4.4 IR contracts

A useful middle-level instruction schema is:

```text
Instruction {
    opcode,
    operands: [ValueId],
    result: Option<ValueId>,
    normal_successors: [BlockId],
    exceptional_successor: Option<BlockId>,
    effects: EffectSet,
    source_span: SourceSpan,
    roots_required: RootSet
}
```

`EffectSet` should distinguish allocation, arbitrary guest call, host call, mutation, iteration, Python exception, resource check, and potential nontermination. Optimization passes use these flags rather than guessing from instruction names.

Example:

```python
def run(input):
    values = [input, 2]
    try:
        return sum(values)
    finally:
        print("done")
```

Conceptual lowering:

```text
construct list                         → normal or resource failure
call builtin sum                       → value or Python exception
enter finally cleanup                  → on return and exceptional paths
call captured print                    → may produce Python/host error
merge cleanup outcome with pending outcome
encode return value or propagate exception
```

A `finally` block is not merely a normal successor. The compiler must merge normal return, loop-control, and exception outcomes while preserving which pending outcome cleanup may replace.

## 4.5 Runtime value representation

The public semantics must not depend on a particular tagging scheme. A practical initial `wasm32` representation is a 64-bit internal value:

- low bits distinguish small integer, immediate singleton, and heap reference;
- small signed integers use a checked immediate range;
- heap references contain aligned linear-memory offsets or stable object handles;
- `None`, `False`, and `True` have dedicated immediate values;
- all `float` and `complex` values are heap-boxed initially so every binary64 bit pattern can be preserved without NaN-boxing conflicts;
- oversized small-int results promote to arbitrary-precision heap integers.

Unboxed `i32`, `i64`, and `f64` values are later local optimizations, with boxing at generic calls, container stores, closure cells, and host serialization.

The compiler and runtime share a private `runtime-abi-version`. Generated code must not execute with a different object layout or runtime calling convention.

## 4.6 Object layout and memory management

Every heap object has a runtime header conceptually containing:

```text
type descriptor / tag
object flags and GC metadata
identity-stable storage
payload or payload pointer
optional type-specific length/capacity
```

Object addresses are never exposed through `id` or serialization. Identity is the lifetime-stable object reference, not a host pointer.

### Recommended P0 collector

Use a **non-moving, precise mark-sweep collector inside guest linear memory**, with:

- compiler-generated stack maps or an explicit guest shadow stack for live references;
- roots for module globals, frames, closure cells, runtime temporaries, pending exceptions, and host-call scratch values;
- collection only at defined allocation safepoints;
- free lists or segregated size classes;
- a hard linear-memory maximum and lower runtime heap budget;
- full instance teardown after each request.

Non-moving collection simplifies references held in generated Wasm locals and runtime data. Collection safepoints must not occur while an unregistered temporary is live across a runtime call. P0 excludes `__del__` and resurrection, avoiding finalizer-driven semantics. Garbage-collection timing is implementation latitude, not a promise to mimic CPython.

### Alternatives to prototype

| Strategy | Advantage | Cost/risk |
|---|---|---|
| Request arena only | Very simple and fast allocation | No reclamation during execution; easy memory exhaustion |
| Refcount + cycle collector | Prompt ordinary reclamation | Write barriers/refcount overhead; cycles and root discipline remain |
| Non-moving mark-sweep | Precise, no finalizer requirement in P0, stable addresses | Pauses and free-list fragmentation |
| Moving/compacting GC | Better locality and fragmentation behavior | Requires handles or updating every root/reference |
| Wasm GC types | Engine-managed references and typed heap | Runtime support/feature constraints; host and toolchain dependency |

Do not make P0 portability depend on Wasm GC. Revisit it after the runtime matrix and benchmarks demonstrate a supported advantage.

## 4.7 Arbitrary-precision integers

Use sign-and-magnitude limbs, for example 32-bit limbs with `i64` intermediates for addition/subtraction and widened multiplication. Required runtime operations include normalization, comparison, shifts, bitwise operations on signed values, division/modulo, conversion to/from binary64 and decimal, and checked allocation.

Fast path:

```text
if both values are small ints:
    perform widened/checked operation
    if result fits small-int range: return immediate
    else: allocate normalized bigint
else:
    generic bigint/bool/numeric operation
```

Policy checks apply to limbs allocated, result bit length, shift counts, conversion input/output length, and operation cost. Exceeding policy produces a defined resource outcome. It never returns a wrapped fixed-width value.

Multiplication/division algorithms should start simple and correct. Asymptotically faster algorithms can be added only with crossover benchmarks and differential tests. Compile-time integer folding uses the same checked implementation or a proven equivalent arbitrary-precision library.

## 4.8 Strings, bytes, and text encoding

A straightforward P0 representation is immutable UTF-32 code units plus an optional cached UTF-8 encoding:

- indexing and length count Python Unicode code points rather than UTF-8 bytes or UTF-16 code units;
- lone surrogate code points can be represented internally;
- comparisons and slicing are simple and deterministic;
- host conversion validates whether the target protocol can represent the value;
- Unicode methods use a pinned Unicode data version and are implemented only when their semantics are included.

The memory cost is significant. Later profiles may use Latin-1/UTF-32 variants or ropes, but every representation must preserve indexing, equality, slicing, and encoding behavior. Do not silently substitute the host language’s UTF-16 string semantics.

Bytes and bytearray use separate byte storage. Host CBOR byte strings avoid lossy text conversion.

## 4.9 Containers

### Lists and tuples

Store a contiguous array of runtime values. Lists maintain length/capacity and an iteration/mutation version where required. Tuples use fixed immutable storage. Growth checks multiply and add with overflow detection before allocation.

### Dicts

A practical design separates an insertion-ordered entry array from an open-addressed index table:

```text
entries: [(hash, key, value, state)] in insertion order
index:   [entry-position-or-empty] using per-instance hash seed
```

Deletion and mutation-during-iteration behavior require careful entry-state rules. Equality and hash dispatch are runtime operations for supported keys. Numeric cross-type equality receives dedicated tests. Rehashing must not change insertion-order iteration.

### Sets

Reuse the hash/index machinery without a value payload. Set iteration order is unspecified. Deterministic builds may use a fixed test seed, but production code must not serialize sets by unsorted internal order when reproducible output is required.

## 4.10 Functions, frames, and closures

Generated functions use native Wasm calls for the ordinary path plus explicit Python-frame metadata where tracebacks, recursion limits, or GC roots require it. A frame contains or references:

- function/code descriptor;
- arguments and live locals or root slots;
- closure environment;
- current source region for traceback mapping;
- pending exception/outcome state where needed.

A closure is a heap object containing a function descriptor and captured cells. A cell is heap storage shared by the defining and nested functions. Loop or closure optimizations must not copy a cell value prematurely.

Direct calls bypass dynamic lookup only when the callee is proven stable. Indirect tables, if used, are internal and type-checked by Wasm; they are not guest-writable host pointers.

## 4.11 Exceptions and cleanup

### Portable P0 mechanism

Use explicit outcomes:

```text
Outcome = Ok(Value) | PythonError(ExceptionRef)
        | PolicyTermination(PolicyCode) | InternalTrap(InternalCode)
```

Operations that can fail return or otherwise produce an outcome. Generated code branches on it. `PythonError` participates in `try`/`except`; policy termination and internal traps bypass guest handlers.

For performance, ok values and error tags may be returned in multiple Wasm values or packed registers rather than allocated result objects.

### `try`/`except`/`finally` lowering

The compiler constructs cleanup regions and a pending-outcome state:

1. execute the protected body;
2. route a Python error through matching handlers;
3. preserve a pending return/break/continue/error while running `finally`;
4. run `else` only on the specified normal path;
5. let cleanup replace the pending outcome if cleanup returns or raises;
6. rethrow an unhandled saved exception after cleanup;
7. clear exception-target bindings as Python requires.

WebAssembly traps are reserved for invalid internal states, unreachable generated code, memory exhaustion that cannot be reported safely, or engine-level termination. Ordinary `TypeError`, `KeyError`, and `ZeroDivisionError` must not begin as unrecoverable traps.

### Later Wasm exception handling

A Wasm EH backend can reduce explicit outcome checks where supported. It must preserve Python exception objects, cleanup, and host-policy outcomes. It should be a separately tested target feature, not an assumption hidden inside the IR.

## 4.12 Iterators, generators, and async

P0 builtin iterators use runtime state objects and `StopIteration` outcomes. P1 generators lower each function into a heap frame and explicit state machine:

```text
state = entry or yield label
saved locals, expression temporaries, exception state, and delegation state
resume action = next(value) | send(value) | throw(exc) | close()
result = yielded(value) | returned(value) | raised(exc) | closed
```

`yield from`, `send`, `throw`, `close`, and `GeneratorExit` require explicit transitions. This stackless approach does not depend on WebAssembly stack switching, which remains a separate proposal in the [official proposal tracker](https://github.com/WebAssembly/proposals).

Async functions can reuse the state-machine approach later, with host futures/streams and cancellation policy. WASI 0.3 async may simplify the host boundary but does not implement Python coroutine semantics automatically.

## 4.13 Static imports and initialization

The locked module graph is compiled into the artifact. Each module has an initialization function and a request-local module object. Import statements resolve through a runtime module table, preserving one initialization per fresh instance. Circular imports, if supported later, require partially initialized module entries and focused tests.

The P0 guest has no filesystem finder, package installer, import hook, dynamic loader, or source compiler. Standard-library modules may be:

- runtime intrinsics;
- compiler-builtins;
- reviewed pure-Python modules compiled through the same pipeline;
- explicit host capabilities exposed through a Python wrapper.

Reuse of CPython library source must retain license notices and pass dependency, semantic, and resource review.

## 4.14 Host ABI layers

### Private generated-code/runtime ABI

This ABI may use tagged values, direct object references, internal tables, guest allocation functions, and runtime calls. It is versioned with the generated module and is never exposed to application SDKs.

### Public component ABI

The preferred component boundary exchanges byte strings and structured execution errors through WIT. The Canonical ABI handles component-level strings/lists; guest object references never cross it.

### Portable core-module ABI

A core fallback may export guest memory and synchronous buffer functions:

```text
pw_alloc(size) -> pointer
pw_free(pointer, length)
pw_run(input_pointer, input_length,
       output_pointer_out, output_length_out) -> status
```

The host writes a bounded request buffer, invokes `pw_run`, reads the bounded response, and frees buffers before dropping the instance. These pointers are valid only during the documented call sequence; they are not long-lived host values or capabilities. Host adapters must validate lengths, overflows, status codes, and output limits on every call.

An alternative callback-based core ABI can avoid exported-memory manipulation, but it adds state-machine and reentrancy complexity. Choose one and conformance-test it rather than supporting accidental mixtures.

## 4.15 Artifact manifest

The immutable module should have an external or custom-section manifest containing at least:

```json
{
  "format": "pywasm-aot-artifact/v1",
  "source-python": "3.15",
  "subset-profile": "P0",
  "compiler-version": "pinned-version",
  "runtime-abi-version": 1,
  "host-protocol": "pywasm-exec/v1",
  "target-profile": "C0",
  "module-sha256": "hex-encoded digest",
  "module-graph-sha256": "hex-encoded digest",
  "wasm-features": ["bulk-memory", "multi-value", "reference-types"],
  "imports": [],
  "exports": ["memory", "pw_alloc", "pw_run", "pw_free"],
  "unicode-version": "pinned-at-build",
  "source-map": "omitted-or-separately-referenced"
}
```

The example is a C0 artifact with no host-call grant. If host calls are enabled, the exact versioned import appears in `imports`; a component artifact instead names its validated world and exports. List only features actually emitted, not every feature the compiler can support.

The manifest is descriptive until the host verifies it against the actual binary. The hash identifies the artifact; it is not a signature. Production distribution should add an authenticated signing and trust policy.

## 4.16 Optimization policy

Safe initial optimizations:

- direct local slot access after scope resolution;
- exact small-int checked arithmetic with bigint fallback;
- direct calls to statically bound functions;
- builtin length/index operations after exact-type guards;
- branch and unreachable-code cleanup after exceptional CFG construction;
- constant encoding and deduplication that preserve identity/equality semantics;
- removal of redundant root registrations only with precise liveness;
- inlining only with stack, traceback, and closure semantics preserved.

Unsafe without additional proof:

- treating annotations as runtime types;
- wrapping integers at 32/64 bits;
- replacing Python `/`, `//`, `%`, or `pow` with raw Wasm operations;
- reordering calls, display construction, or container stores;
- removing `finally` cleanup on an exception path;
- constant-folding closure reads or mutable defaults;
- omitting allocation/resource checks after inlining;
- using host-language string length or comparison as Python semantics.

## 4.17 Reference implementation sequence

1. Build parser, subset validator, scope analysis, and structured diagnostics.
2. Implement runtime values, arbitrary integers, strings, tuples/lists, and generic operations in a host-testable runtime library.
3. Add HIR/CFG and direct Wasm emission for straight-line expressions.
4. Add outcome-based exceptions and cleanup before broad syntax support.
5. Add calls, frames, closures, containers, and precise GC roots.
6. Add the host protocol and fresh-instance execution adapter.
7. Establish differential tests and multi-runtime validation.
8. Add comprehensions, static modules, and P0 standard-library surface.
9. Benchmark before adding unboxing, inlining, alternative GC, Wasm EH, or Wasm GC.
10. Treat classes, generators, async, and richer Component Model support as separate semantic/backend milestones.
