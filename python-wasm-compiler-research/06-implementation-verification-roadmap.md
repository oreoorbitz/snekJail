# 6. Implementation and Verification Roadmap

## 6.1 Delivery principle

Advance by **semantic vertical slices**, not by accepting a large grammar and filling in behavior later. Every milestone must include positive tests, rejection tests, resource-limit tests, and a validated WebAssembly artifact.

The primary correctness oracle is CPython 3.15 executing the same accepted source in a controlled native test worker. The primary portability oracle is a versioned matrix of WebAssembly engines and host adapters. Neither is optional: matching CPython on one engine does not demonstrate portability, and running on many engines does not demonstrate Python semantics.

## 6.2 Milestones

### M0 — Semantic laboratory and decision records

Build:

- the Python 3.15 feature-version manifest;
- a native CPython differential runner;
- canonical outcome serialization;
- initial semantic test corpus;
- benchmarks for instantiation, execution, memory, and artifact size;
- decision records for values, strings, GC, exceptions, and core/component ABI.

Exit gate:

- the harness distinguishes return, stdout, Python exception class, policy denial, resource termination, and internal error;
- tests run with pinned CPython 3.15.x and pinned Wasm engines;
- all documented non-contractual comparisons are explicit.

### M1 — Front end

Build source decoding, lexer, parser, project AST, source spans, subset validator, scope analysis, and diagnostics.

Exit gate:

- accepted ASTs match CPython structurally for supported constructs;
- whole-block local binding, `global`, `nonlocal`, and closure cells pass focused tests;
- unsupported syntax produces deterministic diagnostics;
- source/token/AST/nesting limits fail without panic or unbounded resource use;
- module top-level code is never executed by the compiler.

### M2 — Runtime value library

Implement and test the runtime independently of generated code:

- tagged values and object headers;
- small integers and arbitrary-precision integers;
- floats and complex values;
- strings and bytes;
- tuples, lists, dicts, and sets;
- generic arithmetic, comparison, truth, indexing, slicing, and iteration;
- bounded allocation and initial precise GC.

Exit gate:

- runtime operations pass native unit tests and Wasm-hosted tests;
- arithmetic includes signs, zero division, huge values, NaN, signed zero, and mixed types;
- container aliasing, mutation, hashing, insertion order, and growth pass differential tests;
- every allocation path checks arithmetic overflow and resource policy;
- GC stress tests retain all live values and reclaim unreachable values without exposing stale references.

### M3 — Straight-line AOT backend

Lower expressions, assignments, branches, loops, and direct builtin calls to HIR/CFG and Wasm. Add module validation and artifact manifests.

Exit gate:

- evaluation order matches CPython with side-effecting test functions;
- loop `else`, short circuiting, chained comparisons, slicing, and augmented assignment pass;
- arbitrary integers never silently wrap;
- generated modules pass independent validation;
- identical inputs/configuration produce identical artifacts;
- all runtime ABI assumptions are versioned.

### M4 — Calls, closures, exceptions, and cleanup

Add function calls, argument binding, recursion limits, frames, closures, Python exceptions, `try`/`except`/`else`/`finally`, and tracebacks.

Exit gate:

- defaults evaluate once at definition time, including mutable defaults;
- argument errors match exception classes;
- late-binding closure tests pass;
- every return/break/continue/error path through `finally` executes the correct cleanup exactly once;
- cleanup can replace pending outcomes correctly;
- policy terminations bypass guest exception handlers;
- recursion and traceback output stay within configured limits and do not expose host internals.

### M5 — Host integration and P0 module surface

Add `run(input)`, deterministic CBOR, captured stdout, static imports, candidate `math`/`json` modules, host-call broker, and the first host SDK.

Exit gate:

- two different host-language adapters pass the same protocol tests;
- no default filesystem, environment, process, network, clock, or random import is linked;
- malformed/oversized/deep CBOR fails before expensive decoding;
- unsupported return values fail without partial success;
- fresh instances cannot observe prior-request values;
- module caching does not become mutable instance caching.

### M6 — Security and portability hardening

Add full resource dimensions, artifact signing hooks, fuzzing, multi-runtime CI, and operational telemetry.

Exit gate:

- every limit has a dedicated exhaustion and boundary-minus-one test;
- parser/compiler fuzzing finds no uncontrolled crash or hang;
- mutated artifacts are rejected by validation or policy checks;
- host-call fuzzing cannot forge capabilities or corrupt host memory;
- cold-start and p99 execution measurements exist for each supported engine;
- the support matrix records exact engine, SDK, target, and feature versions.

### M7 — Extended language and target backends

Prioritize based on measured LLM corpus needs:

1. classes and special-method dispatch;
2. generators as state machines;
3. more standard-library modules;
4. Wasm EH and guarded unboxing if benchmarks justify them;
5. synchronous WASI 0.2/custom Component Model wrapper, then WASI 0.3 async host calls where supported;
6. async Python, metaclasses, and other P2 features only with dedicated specifications.

No performance backend becomes the default until it passes the same semantics and security gates as the portable backend.

## 6.3 Differential test protocol

For each test case:

1. Compile source with the project compiler using a pinned profile and deterministic test policy.
2. Execute source in a fresh native CPython 3.15 worker with the same logical entry point and input.
3. Execute the Wasm artifact in a fresh guest instance on every supported engine.
4. Capture a canonical outcome:
   - returned value;
   - stdout/stderr intended for guest capture;
   - uncaught Python exception class and chaining structure;
   - policy or resource outcome;
   - internal failure.
5. Compare values recursively with explicit type handling.
6. Record pass, expected rejection, documented implementation-defined difference, or failure.

Do not compare:

- CPython bytecode or instruction selection;
- object addresses or `id` values;
- garbage-collection timing;
- unstable hash/set order;
- exact exception message text unless the project intentionally standardizes its own text;
- traceback formatting outside the project’s defined format;
- platform-libm last bits without an explicit numeric policy.

Do compare:

- exception classes and whether control reaches each handler/cleanup;
- return-value types, including `bool` versus `int` and `float` versus `complex`;
- mutation and aliasing visible in results;
- dict insertion order;
- evaluation order and number of side effects;
- resource outcomes under a deterministic test policy.

The test harness may use a versioned typed diagnostic encoding for tuples, sets, complex values, and other guest values so recursive type-preserving comparison is possible. That harness encoding is separate from the deliberately narrower public P0 host-value domain; production rejection of unsupported public return values remains a required test.

A test that mixes supported and unsupported features must be split mechanically or marked expected-reject. Assertions must not be edited merely to make the implementation pass.

## 6.4 Corpus layers

### Curated semantic microtests

Cover one rule per test and combinations known to be risky:

- whole-block local binding and unbound reads;
- closures created inside loops;
- mutable default arguments;
- chained comparisons with side effects;
- negative floor division/modulo and huge shifts;
- bool/int/float/complex interactions;
- Unicode indexing and lone surrogates;
- dict keys with equal cross-type numeric values;
- list/dict mutation during iteration;
- generator-facing `StopIteration` behavior when generators arrive;
- nested `finally` with return, exception, and cleanup replacement.

### CPython-derived tests

Use the [CPython 3.15 `Lib/test` tree](https://github.com/python/cpython/tree/3.15/Lib/test) as a source of focused cases. Useful starting files include:

- `test_grammar`, `test_syntax`, `test_scope`, `test_global`, `test_nonlocal`;
- `test_bool`, `test_int`, `test_long`, `test_float`, `test_numeric_tower`, `test_pow`;
- `test_str`, `test_string_literals`, `test_fstring`, `test_bytes`;
- `test_list`, `test_tuple`, `test_dict`, `test_set`, `test_dictviews`;
- `test_compare`, `test_binop`, `test_operator`, `test_augassign`;
- `test_iter`, `test_enumerate`, `test_unpack`, `test_unpack_ex`;
- `test_call`, `test_extcall`, `test_keywordonlyarg`, `test_named_expressions`;
- `test_exceptions`, `test_exception_variations`, `test_builtin` filtered to the subset;
- class/generator/async files only at their corresponding tiers.

Retain provenance for every imported or split test and comply with CPython’s license notice requirements.

### Generated programs

Generate small type-directed and dynamically typed programs:

- literal/operator matrices;
- nested displays and subscripts;
- side-effect traces for evaluation order;
- function/closure combinations;
- control-flow graphs with exceptional exits;
- allocation/collection stress;
- import graphs and repeated initialization.

The generator itself must have a complexity budget so tests do not spend most time generating pathological input unintentionally.

## 6.5 Negative and rejection testing

Every excluded feature needs a fixture proving deterministic rejection:

- `eval`, `exec`, `compile`, dynamic imports, and native extension loading;
- unsupported class/async/metaclass syntax for the active profile;
- ambient modules such as `os`, `subprocess`, and `socket`;
- runtime annotation reflection when rejected by the profile;
- lazy imports and unsupported 3.15-specific syntax;
- imports absent from the locked manifest;
- malformed source encoding, indentation, and AST nesting;
- result values outside the host encoding domain.

Also verify that rejected code produces no executable artifact and no host side effect.

## 6.6 Resource-limit tests

For each limit, test below, at, and above the boundary:

| Dimension | Example tests |
|---|---|
| Source size | One byte below/above maximum; many tiny files; one huge literal |
| AST complexity | Deep expressions, nested comprehensions, many bindings/functions |
| Compile budget | Pathological grammar/backtracking and optimization inputs |
| Wasm artifact | Excessive functions/constants/data/source maps |
| Execution | Infinite loop, recursion, huge exponent/shift, repeated host calls |
| Memory | Container growth, bigint growth, GC pressure, response allocation |
| Stack | Deep guest recursion and deeply nested cleanup |
| Serialization | Deep CBOR, huge maps/lists/strings, duplicate/hostile keys |
| Output/logs | Endless `print`, huge `repr`, terminal control characters |
| Host calls | Too many calls, large payload, slow response, cancellation |

A limit failure must leave the host able to serve the next request and must not poison a shared artifact cache.

## 6.7 Fuzzing and malformed-input testing

### Compiler fuzzing

Feed arbitrary bytes, valid-prefix mutations, deep nesting, huge numeric/string literals, unusual Unicode, and token/AST mutations. Invariants:

- success or structured failure within compile budget;
- no host filesystem/network access;
- no panic, memory corruption, or unbounded recursion;
- no artifact unless all validation succeeds.

### Runtime and serializer fuzzing

Generate and mutate CBOR, host-call payloads, container shapes, numeric values, Unicode, and log streams. Decode twice where useful and compare deterministic canonicalization. Reject non-canonical or trailing data according to protocol policy.

### Artifact fuzzing

Mutate binary sections, type declarations, imports, memory limits, function bodies, data segments, and manifests. The host must validate before execution. A malformed artifact must not be cached as valid.

### Engine differential fuzzing

Run the same valid artifact on all supported engines. Different outcomes indicate a project bug, an engine bug, an unspecified Wasm behavior, or an accidental feature assumption. Minimize and classify rather than selecting whichever engine appears convenient.

## 6.8 WebAssembly conformance and support matrix

The [WebAssembly 3.0 core specification](https://webassembly.github.io/spec/core/) defines validation and execution; the [official proposal tracker](https://github.com/WebAssembly/proposals) distinguishes active proposals from finished work. Test each emitted feature independently.

Record at minimum:

| Field | Purpose |
|---|---|
| Engine and exact version | Reproduce validation/execution behavior |
| Host SDK/language version | Reproduce adapter behavior |
| Core Wasm feature set | Detect accidental proposal dependency |
| WASI/component version | Prevent world/ABI mismatch |
| Runtime ABI and host protocol | Prevent generated-code or serializer mismatch |
| Resource-control capabilities | Confirm memory/fuel/epoch/stack support |
| Async/cancellation behavior | Verify termination and host-call behavior |
| Test result and known deviations | Make support claims auditable |

Start with one reference engine and one additional independent engine as soon as the core ABI stabilizes. Add runtimes based on required host languages, not on a desire for a long logo list.

## 6.9 Performance methodology

Measure after correctness, using realistic LLM workloads:

- compile latency and compiler peak memory;
- artifact and runtime size;
- cold and warm module instantiation;
- p50/p95/p99 request latency;
- guest peak and retained memory;
- GC count, pause time, and reclaimed bytes;
- serialization and host-call overhead;
- fuel/epoch instrumentation overhead;
- throughput at concurrency and tenant isolation levels;
- generic versus guarded-specialized execution;
- engine-specific tail latency after policy termination.

Compare against meaningful alternatives: embedded CPython where permitted, CPython WASI/component packaging, and an interpreter-based Wasm approach. Do not claim that AOT is automatically faster: dynamic dispatch, boxing, GC, and instantiation can dominate.

Publish workload source, configuration, hardware, engine version, warm-up, sample count, and failure/termination counts with results.

## 6.10 Release acceptance gate

A P0 release candidate must satisfy all of these:

- [ ] No known silent semantic divergence for accepted programs.
- [ ] Every excluded feature has an expected-reject test.
- [ ] All documented resource dimensions have enforced limits and exhaustion tests.
- [ ] Differential tests pass on the pinned CPython feature version.
- [ ] Artifacts validate on every advertised engine.
- [ ] At least two host-language adapters pass the protocol suite.
- [ ] Fresh-request isolation and cache-separation tests pass.
- [ ] No ambient capability is present in the default linker configuration.
- [ ] Fuzzing meets a time/input budget without unresolved critical findings.
- [ ] Build artifacts and manifests are deterministic and authenticated by the chosen release process.
- [ ] Known implementation-defined differences are listed in user-facing documentation.
- [ ] Performance claims are linked to reproducible measurements.

A failure in a correctness or isolation gate blocks optimization work and release; it is not converted into a warning merely to preserve the milestone date.
