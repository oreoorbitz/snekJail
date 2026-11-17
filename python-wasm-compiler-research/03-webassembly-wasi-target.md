# 3. WebAssembly and WASI Target

## 3.1 Target decision

The recommended initial artifact is a **`wasm32` core WebAssembly module with one linear memory, a bundled Python-subset runtime, and a narrow synchronous byte-buffer ABI**. Its default instruction profile is Core 2.0-oriented: MVP plus bulk memory, multi-value, and reference types where the runtime benefits from them; fixed-width SIMD is optional and separately tested. A Component Model wrapper and WASI interfaces are later, separately validated targets generated from the same versioned host protocol.

This choice separates four questions that are often incorrectly collapsed into “target WebAssembly”:

1. **Python compatibility:** which Python 3.15 subset semantics are implemented?
2. **Core artifact:** which WebAssembly instructions, types, memories, and proposals does the binary require?
3. **Host ABI:** is the artifact a core module or a component, and which exact interface/world does it instantiate?
4. **Engine support:** which pinned runtimes, SDKs, and host languages pass the artifact’s tests?

A module can be valid WebAssembly and still fail to instantiate in a particular embedding because an import, proposal, component feature, resource control, or WASI world is unavailable. Portability is therefore a **tested support matrix**, not an intrinsic property of the `.wasm` suffix.

Primary references for this chapter are the [WebAssembly core specification](https://webassembly.github.io/spec/core/), the [official proposal tracker](https://github.com/WebAssembly/proposals), the [Component Model documentation](https://component-model.bytecodealliance.org/), the [Component Model repository](https://github.com/WebAssembly/component-model), and the [WASI project](https://wasi.dev/).

## 3.2 Core modules, components, WIT, and WASI

### Core module

A core module defines typed functions, tables, memories, globals, imports, and exports. Validation checks such properties as operand types, control-flow structure, and memory instructions. Linear-memory isolation does not supply a Python object system, arbitrary-precision integers, garbage collection, or a standard application ABI.

The project’s P0 runtime implements those missing pieces inside the module. The host sees only bounded request/response buffers and explicitly imported host operations.

### Component

A component is a different binary and interface layer from a core module. The Component Model adds typed interfaces and composition. [WIT](https://component-model.bytecodealliance.org/design/wit.html) describes interface types and worlds; the Canonical ABI defines how those values are represented and transferred at component boundaries.

A component must not be treated as a core module renamed with a new extension. Tooling must validate its binary kind, imports, exports, world, and Canonical ABI requirements. A host that accepts core modules does not necessarily accept components. The Canonical ABI uses a shared-nothing, linear-memory-based boundary for copying component values; it is not a protocol for sharing arbitrary Python/Wasm-GC object graphs with the host.

WIT defines contracts, not behavior or package-resolution policy. Vendor WIT dependencies, pin exact package IDs and versions, and inspect the emitted world with the target toolchain.

### WASI

WASI defines interfaces for capabilities such as filesystems, clocks, randomness, sockets, and other system-oriented services. It is not required merely to execute a core Wasm computation. P0 should avoid ambient WASI capabilities and expose only its own execution interface plus explicitly granted host calls.

WASI 0.3 builds on Component Model async primitives. It does not automatically make Python coroutines work, provide an LLM sandbox policy, or make all older WASI 0.1 APIs interchangeable with 0.3 interfaces.

## 3.3 WebAssembly feature policy

The [core specification](https://webassembly.github.io/spec/core/) is the normative reference for the standard core language. The [proposal tracker](https://github.com/WebAssembly/proposals) is the authority for whether a feature is finished or still an active proposal. Engine and toolchain support must be checked separately; a finished proposal is not a promise of universal deployment support.

The artifact manifest must list the exact emitted features. The compiler must not enable a target feature merely because the build machine supports it.

| Feature/family | Standard status on the research date | Role in this project | P0 policy |
|---|---|---|---|
| `i32`, `i64`, `f32`, `f64` | Core 1.0 | Runtime implementation and numeric fast paths | Use, but implement Python bigint, division, modulo, conversion, and exception semantics in the runtime |
| Structured control and direct calls | Core 1.0 | Lowering HIR/CFG and generated functions | Use |
| One 32-bit linear memory | Core 1.0 | Guest heap, stacks, and buffers | Use with declared initial/maximum limits and a lower runtime heap budget |
| Mutable globals | Standardized core | Runtime instance state where appropriate | Use only with fresh-instance isolation; never imply cross-request persistence |
| Multiple return values | Core 2.0 | Efficient outcome/value returns where supported | Optional optimization; otherwise pack outcomes in runtime storage |
| Bulk memory/reference types | Core 2.0 | Runtime implementation conveniences | Permit only when the reference engine matrix validates them; do not make them accidental dependencies |
| Typed function references | Core 3.0 | Typed closure/vtable/dispatch references | Optional later optimization; ordinary typed tables and direct calls are sufficient initially |
| Fixed-width SIMD | Core 2.0 | Numeric/vector acceleration | Exclude from the minimal P0 profile; permit a separate optional numerical profile after tests |
| Relaxed SIMD | Core 3.0; implementation-dependent/nondeterministic operations | Approximate vector acceleration | Exclude from deterministic P0 execution |
| Threads/shared memory | Active proposal, Phase 4 | Concurrent guest execution | Exclude; P0 has no Python threads or shared mutable instances |
| `memory64` | Core 3.0 | Very large guest address spaces | Exclude from P0; artifact and policy sizes remain bounded well below 4 GiB |
| Multiple memories | Core 3.0 | Separating heaps/buffers | Exclude initially; adds runtime and adapter requirements |
| Wasm GC | Core 3.0 | Engine-managed typed references and heap objects | Prototype later; P0 uses its own linear-memory objects and precise collector |
| Wasm exception handling | Core 3.0 | Native throw/catch lowering | Optional backend after cross-engine tests; P0 uses explicit outcomes |
| Tail calls | Core 3.0 | Call optimization | Never required for Python correctness; use only with a portable fallback |
| Stack switching | Active proposal, Phase 3 | Suspending/resuming execution stacks | Do not depend on the proposal; lower generators/coroutines to state machines |
| Component Model | Active Phase 1 general proposal; also versioned through WASI Developer Preview releases | Typed composition and host interfaces | Separate wrapper target, not the P0 core binary format |

The [Core 3.0 change history](https://webassembly.github.io/spec/core/appendix/changes.html) and [finished-proposal register](https://github.com/WebAssembly/proposals/blob/main/finished-proposals.md) support the release mapping. “Core 3.0” describes standardization, not uniform runtime enablement.

This table is a **project target policy**, not a claim that all listed features have the same standardization status. Before each release, the build must resolve each feature against the pinned core specification/proposal revision and record engine results.

### Deterministic numeric behavior

Core Wasm defines useful numeric behavior, but Python compatibility still requires runtime work:

- Wasm integer overflow wraps; Python integers must grow or reach a declared resource outcome.
- Wasm float division by zero follows IEEE behavior; Python float division by zero raises `ZeroDivisionError`.
- Python `//`, `%`, `pow`, conversions, and complex arithmetic need explicit lowering.
- NaN payload bits and host-libm last bits are not silently promoted to cross-engine compatibility guarantees.
- No optional instruction family with implementation-selectable results belongs in the deterministic P0 profile.

## 3.4 Linear-memory runtime versus Wasm GC

### P0: runtime-owned linear memory

The proposed P0 layout uses tagged values, non-moving heap objects, precise roots, and mark-sweep collection in linear memory. Its advantages are:

- control over object layout and resource accounting;
- no dependency on a particular engine’s Wasm GC implementation;
- direct support for arbitrary-precision integers, Unicode storage, and container data;
- one runtime implementation testable natively and under Wasm;
- predictable teardown by dropping the entire fresh instance.

Its costs are runtime size, manual GC/root correctness, allocation overhead, and potentially slower execution than engine-managed references.

### Later: Wasm GC prototype

Wasm GC may reduce some runtime machinery or improve interoperability, but adopting it requires a new object/representation design and support matrix. It does not by itself implement Python’s numeric tower, container semantics, dynamic dispatch, exception behavior, or host protocol.

A Wasm GC experiment must pass the same differential tests and prove a measured benefit. It must not become an undocumented target requirement introduced by a compiler upgrade.

## 3.5 Exceptions, generators, and suspension

### Exceptions

P0 uses explicit `Outcome` propagation so ordinary Python exceptions remain guest values and cleanup control flow remains visible in generated code. This works without requiring a Wasm exception-handling feature.

A later Wasm EH backend can map protected regions and throws to native constructs. It must still preserve:

- Python exception objects and class matching;
- exception chaining and active-exception state;
- `else` and `finally` behavior;
- cleanup replacement of pending outcomes;
- separation of Python errors from uncatchable policy/resource termination;
- source-level traceback metadata.

The host must not infer that every Wasm trap is a guest `RuntimeError`. Traps and explicit Python errors have different contracts.

### Generators and async Python

Generators and coroutines require suspension of Python frames, expression temporaries, exception state, and delegation state. The portable design lowers them into heap-allocated state machines. WebAssembly stack switching, if available in a future target, is an optimization or alternate implementation—not the semantic specification.

WASI async interfaces similarly address host/component computation. They do not replace Python’s `__await__`, generator `send`/`throw`/`close`, cancellation, or event-loop semantics.

## 3.6 WASI 0.1, 0.2, and 0.3

The three lines are not interchangeable labels for the same ABI.

| WASI line | Integration shape | Project treatment |
|---|---|---|
| WASI 0.1 (`wasi_snapshot_preview1`) | Core-module imports with a POSIX-like system interface | Relevant to existing CPython WASI builds and legacy runtimes; not the desired P0 application ABI |
| WASI 0.2 | Component Model interfaces and WIT worlds | Recommended synchronous component target where supported; pin exact world and tooling |
| WASI 0.3 | Component Model interfaces with native `async func`, `stream<T>`, and `future<T>` primitives | Preferred evolution for deployments with tested support; not mandatory for P0 |

The [WASI roadmap](https://wasi.dev/roadmap) records WASI 0.3.0 as shipped on 2026-06-11 and WASI 0.3.1 as shipped on 2026-08-11. WASI 0.3 adds native async facilities and removes the old `wasi:io` layer in favor of Component Model primitives, making exact WIT/toolchain version matching especially important. The roadmap identifies support in Wasmtime 46 and later and in jco; that is not a claim of universal runtime support. Runtime execution support and embedding-API maturity are also separate: an API such as [Wasmtime’s `p3` embedding module](https://docs.wasmtime.dev/api/wasmtime_wasi/p3/index.html) can carry its own experimental warning. A component built against one world must not be assumed to instantiate against another merely because both are called “WASI 0.3.”

### Capability implications

Linking a broad WASI world can grant capabilities the P0 policy intends to deny. The default artifact should therefore avoid filesystem, environment, socket, process, clock, and random imports. Later profiles may expose a narrow project-specific interface and let the host implement it using WASI internally.

The Python allowlist and the linked world are separate controls: removing `os` from guest builtins does not remove a filesystem import already available to runtime code.

## 3.7 CPython’s WebAssembly targets are a different decision

Python language version and CPython target ABI must be versioned independently.

The [Python WebAssembly platform notes](https://docs.python.org/3.15/library/intro.html#webassembly-platforms), [PEP 11](https://peps.python.org/pep-0011/), and [PEP 816](https://peps.python.org/pep-0816/) show that:

- PEP 11’s release matrix records Python 3.15 with WASI 0.1 and WASI SDK 33;
- WASI 0.2 was skipped in the CPython support plan in favor of moving toward WASI 0.3;
- the same evolving platform document also lists `wasm32-unknown-wasip3` as a Tier 2 target for Python 3.15+, so the matrix and target row must not be collapsed into a claim that every CPython 3.15 WASI build already uses a complete WASI 0.3 ABI; exact release build metadata controls;
- the [CPython 3.15 WASI README](https://github.com/python/cpython/blob/3.15/Platforms/WASI/README.md) identifies WASI as Tier 2, while the [Emscripten README](https://github.com/python/cpython/blob/3.15/Platforms/emscripten/README.md) identifies Emscripten as Tier 3; their host capabilities and platform restrictions differ.

These facts do **not** require `pywasm-aot` to use WASI 0.1 just because its source grammar is Python 3.15. Conversely, adopting WASI 0.3 for the new compiler does not mean CPython 3.15 itself was built against WASI 0.3.

CPython-on-Wasm packages an existing implementation and runtime. This project instead compiles accepted source directly to project-owned Wasm and links a small runtime. The two approaches can share semantic tests but not artifact or ABI assumptions.

## 3.8 Alternative Python-on-Wasm approaches

| Approach | What it provides | Difference from proposed direct AOT |
|---|---|---|
| CPython for WASI | CPython interpreter and standard runtime in a WASI environment | Broad compatibility and a large runtime; host ABI/capabilities follow the chosen WASI build |
| [Pyodide](https://pyodide.org/) | CPython compiled for browser/JavaScript-oriented WebAssembly environments, with package and host integration | Strong ecosystem compatibility, but browser/JS integration and platform limitations differ from a language-neutral minimal server ABI |
| [`componentize-py`](https://github.com/bytecodealliance/componentize-py) | Python applications packaged as components using WIT worlds and a Python runtime/interpreter approach | Useful component baseline; not direct subset-source-to-Wasm compilation without a Python evaluator |
| [RustPython](https://github.com/RustPython/RustPython) | A Rust Python implementation that can target WebAssembly/WASI configurations | An interpreter/runtime architecture with its own completeness and production-readiness constraints |

These are valid engineering alternatives and benchmark baselines. The research does not claim that direct AOT is automatically smaller, faster, or more compatible. Its intended advantage is a smaller accepted semantics/runtime surface and a narrow host protocol; that advantage must be measured.

Build behavior is part of the distinction. `componentize-py` packages and pre-initializes a CPython-based application, so its build pipeline must be treated as capable of executing application/dependency initialization. The proposed direct-AOT compiler does not execute or pre-initialize source during compilation. Neither path makes the resulting artifact inherently trustworthy; both require independent artifact validation.

Choose an interpreter-based distribution instead when broad package compatibility matters more than artifact/runtime minimization, or when maintaining a new Python runtime is outside the project’s capacity.

## 3.9 Host integration profiles

### Profile C0 — synchronous core module

Exports conceptually include allocation, execution, and deallocation functions; the module exports its memory and imports one bounded host-call operation only when enabled.

```text
request bytes → pw_run → response bytes + status
                         ├─ optional synchronous host-call callback
                         └─ bounded logs and execution metadata
```

Advantages: simple core-module embedding and no component requirement. Costs: each SDK must handle guest-memory buffers safely, synchronous reentrancy must be specified, and cancellation/resource controls are engine-specific.

### Profile C1 — synchronous WASI 0.2/custom component

A WIT interface exchanges bounded byte strings and a structured result. The Canonical ABI replaces public guest-memory pointer manipulation with component values. The initial world can be entirely project-defined, with no general WASI filesystem, clock, random, or socket imports.

This is preferable where the host already supports the pinned WASI 0.2/component tooling. The core runtime and Python semantics remain unchanged. A `wasi:http/incoming-handler@0.2.x` adapter can be added for compatible HTTP hosts, but it is not the universal execution ABI.

### Profile C2 — asynchronous component/WASI 0.3

The interface can represent asynchronous host operations, streaming output, and cancellation with explicit component async types. This profile is justified only by measured host-call/streaming needs and tested deployment support. It must not become a prerequisite for basic P0 execution.

All profiles carry the same protocol version and canonical payload semantics. A wrapper must not silently change integer range, Unicode handling, error categories, or resource policy.

## 3.10 Runtime selection and support matrix

Select runtimes by required host languages and verifiable controls, not by brand count. For every advertised engine/SDK combination, record:

- exact engine, embedding SDK, compiler, runtime ABI, and adapter versions;
- core module versus component support;
- accepted target features and validation result;
- imported world/interface versions;
- maximum-memory enforcement;
- deterministic fuel or other instruction accounting;
- epoch/deadline interruption and cancellation behavior;
- stack exhaustion behavior and host-process safety;
- instance/store teardown and peak-memory behavior;
- host-call reentrancy and async behavior;
- known deviations and open security advisories.

At least one reference engine and one additional independent engine should execute the same conformance corpus before “portable” is used as a release claim. Each host-language adapter must pass the same wire-protocol suite.

## 3.11 Artifact validation and compatibility

The execution host must validate the artifact after cache retrieval and before instantiation. Validation includes:

1. binary kind and size limits;
2. core/component structural validation;
3. actual imports/exports and function signatures;
4. memory/table/global limits;
5. target-feature allowlist;
6. runtime ABI and host protocol versions;
7. manifest-to-binary consistency;
8. artifact digest and optional signature/trust policy;
9. instantiation under a restrictive linker containing only granted imports.

The manifest is descriptive, not a security boundary. The binary and linker configuration are authoritative. A module requesting an undeclared import must fail closed.

Distribute portable `.wasm` modules/components. Engine-native precompiled artifacts are optional deployment caches tied to an exact engine version, architecture, compilation strategy, target features, and security configuration; they are not portable distribution artifacts and must never be accepted from an untrusted caller as trusted native input.

## 3.12 Performance and portability expectations

Benchmark separately:

- compilation latency and compiler memory;
- artifact and bundled-runtime size;
- cold module compilation and warm module-cache hits;
- fresh store/instance creation and teardown;
- generic dynamic operations versus guarded specialization;
- GC pauses and peak memory;
- serialization and buffer copies;
- synchronous versus async host calls;
- fuel/epoch instrumentation overhead;
- tail latency after cancellation or resource exhaustion.

Do not compare only a warm function call on one engine. LLM-generated workloads often pay compilation, instantiation, allocation, and serialization costs on every new program. AOT removes a Python interpreter from the execution artifact, but dynamic Python operations can still dominate runtime cost.

## 3.13 Recommended implementation sequence

1. Emit and validate a `wasm32` core module using the Core 2.0-oriented profile, recording which of bulk memory, multi-value, and reference types are actually required.
2. Implement the C0 synchronous buffer protocol in one reference host SDK.
3. Add fuel/deadline, memory, stack, output, and serialization policy enforcement.
4. Add a second engine and second host language to the support matrix.
5. Benchmark instantiation and runtime overhead before enabling additional proposals.
6. Prototype Wasm EH and guarded unboxing independently; retain the generic portable backend.
7. Generate a C1 component wrapper from the same protocol and validate its WIT/world exactly.
8. Add C2 async/streaming only when a real workload and deployment matrix justify it.
9. Reconsider Wasm GC, SIMD, threads, memory64, or stack switching only through separate target profiles and conformance gates.

## 3.14 Open target decisions

The following remain prototype questions rather than settled claims:

- exact minimum core feature set after the first runtime implementation;
- whether multi-value outcomes measurably outperform packed outcomes;
- whether Wasm EH reduces code size and branch overhead across supported engines;
- whether Wasm GC reduces runtime complexity enough to justify its support constraints;
- component instantiation overhead in each required host language;
- whether WASI 0.3 async improves host-call tail latency and cancellation;
- the practical cold-start cost of a fresh store/instance under concurrency;
- whether a callback-based core ABI is safer or simpler than exported-memory buffers for all target SDKs.

Every adopted answer becomes an explicit target/profile revision with tests; none should appear as an invisible compiler default.
