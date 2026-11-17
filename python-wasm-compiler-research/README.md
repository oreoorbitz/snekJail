# Python → WebAssembly AOT Compiler Research

**Research date:** 2026-11-08  
**Language baseline:** Python 3.15, with Python 3.14 notes where useful  
**Project working name:** `pywasm-aot` (a placeholder, not an existing standard)  
**Status:** Research and proposed design; not a claim of Python or WebAssembly conformance

## Purpose

This research supports an ahead-of-time compiler for a Python subset that produces WebAssembly for executing LLM-generated code inside web servers written in arbitrary languages. The desired deployment avoids a Python installation, OS subprocesses, and per-execution Docker containers.

The central conclusion is:

> Compile Python source directly into a versioned compiler IR and then WebAssembly. Treat the Python language reference as the compatibility contract, not CPython bytecode, CPython code objects, or the C API. Ship a small runtime with the module, expose a narrow host ABI, and enforce security at both the compiler and runtime boundaries.

A WebAssembly runtime is still required. AOT compilation removes the CPython interpreter and the need to create an OS container for each execution; it does not make WebAssembly executable in an environment that has no compatible engine or embedding, and it does not eliminate defense-in-depth isolation for hostile multi-tenant workers.

## Document map

1. [Modern CPython compiler](01-modern-cpython-compiler.md) — the Python 3.15 parser/compiler/interpreter pipeline and what is stable enough to reuse.
2. [Python subset specification](02-python-subset-specification.md) — proposed P0/P1/P2 language tiers, semantics, exclusions, builtins, and diagnostics.
3. [WebAssembly and WASI target](03-webassembly-wasi-target.md) — core Wasm, WASI 0.1/0.2/0.3, the Component Model, runtime features, and alternative Python-on-Wasm approaches.
4. [AOT compiler and runtime architecture](04-aot-compiler-runtime-architecture.md) — IRs, lowering, values, objects, memory, exceptions, generators, imports, optimization, and artifact format.
5. [Security and host integration](05-security-host-integration.md) — the LLM execution protocol, capability policy, resource governance, isolation, observability, and threat model.
6. [Implementation and verification roadmap](06-implementation-verification-roadmap.md) — milestones, differential testing, fuzzing, runtime matrices, and acceptance gates.
7. [Source register](07-source-register.md) — primary references and the claims they support.

## Recommended starting point

Build a **dynamic Python subset first**, not a statically typed Python-like language. Add annotation-guided fast paths only when runtime guards or compiler proofs preserve Python behavior.

### Initial product profile

| Decision | Recommendation |
|---|---|
| Source baseline | Python 3.15 grammar; reject unsupported syntax explicitly |
| Compatibility contract | Python language/library semantics for the accepted subset |
| Front end | Python-compatible lexer/parser → validated subset AST → name resolution |
| Middle end | High-level IR with explicit Python operations → CFG; SSA only where beneficial |
| Initial backend | Core 2.0-oriented `wasm32` module with a small linear-memory runtime |
| Initial object management | Runtime-owned, non-moving objects with precise roots and bounded allocation |
| Initial exceptions | Explicit outcome propagation and compiler-managed cleanup edges |
| Initial host artifact | Core module plus a versioned byte-oriented ABI |
| Primary evolution | Synchronous WASI 0.2/custom component where supported; optional WASI 0.3 async profile |
| Execution isolation | Fresh store/instance and fresh guest memory per untrusted request |
| Compilation trust | Trusted, resource-limited compiler service; validate generated Wasm before caching |
| Host data | Versioned deterministic CBOR envelopes; component calls use no pointers, and core ABI buffers are call-scoped |
| Correctness oracle | Differential execution against the same Python feature version of CPython |
| Portability oracle | A tested runtime/feature matrix, not “runs anywhere Wasm runs” |

### P0 in one paragraph

P0 should support literals, arbitrary-precision integers with resource limits, binary64 floats and complex values, strings, bytes, tuples, lists, dictionaries, sets, expressions with Python evaluation order, functions and closures, built-in control flow, comprehensions, iteration, exceptions and `finally`, and statically resolved imports from a small allowlist. It should exclude classes, generators, async execution, dynamic code evaluation, dynamic imports, reflection that mutates runtime structure, native extensions, threads, processes, and ambient filesystem/network access. These are subset restrictions, not claims that the corresponding Python features are undesirable.

## Architectural decision summary

### Do not use CPython bytecode as the AOT IR

CPython’s AST can change between releases, and its bytecode is explicitly an implementation detail. Python 3.15 also performs runtime specialization and tracing JIT work that assumes CPython objects, frames, caches, and deoptimization. Reimplementing those mechanisms would import substantial CPython complexity without creating a stable compatibility boundary. Use CPython’s pipeline as a design reference and CPython itself as the semantic test oracle.

### Separate four versioned interfaces

1. **Source compatibility:** Python 3.15 subset and rejected-node diagnostics.
2. **Compiler IR:** private and free to change between compiler releases.
3. **Guest runtime ABI:** private contract between generated code and the bundled runtime.
4. **Host protocol:** stable, versioned module/component imports and exports visible to server applications.

Keeping these separate prevents a Python grammar upgrade, runtime optimization, or host-SDK addition from silently breaking every other layer.

### Core module first, component wrapper second

A core Wasm module with byte-oriented imports/exports gives the project a narrow fallback ABI and direct control over its runtime. The Component Model adds typed interfaces and composition through WIT and the Canonical ABI; a custom synchronous WASI 0.2 world is the first component profile, while WASI 0.3 adds native async primitives for a later optional profile. However, the Component Model is still being standardized as a general proposal, and deployment support varies. Generate component wrappers from the same host protocol rather than making every P0 design decision depend on them.

### Deny ambient authority by default

The guest should receive only the request payload and explicitly granted host operations. It should not inherit the server’s filesystem, environment variables, network, clock, randomness, or process privileges. A Python-level allowlist is not a security boundary; the host must withhold the underlying imports as well.

## Non-goals for the first implementation

- Running arbitrary pip packages or C extension modules.
- Executing CPython `.pyc` files or preserving CPython bytecode compatibility.
- Supporting the full Python data model, including metaclasses and monkey patching.
- Providing a drop-in replacement for every use of Docker.
- Guaranteeing identical `repr`, hash randomization, object addresses, destructor timing, traceback formatting, or platform-libm results unless explicitly tested.
- Claiming confidentiality against every microarchitectural side channel.
- Treating LLM-generated source, generated WebAssembly, compiler plugins, or host-call payloads as trusted input.

## Acceptance criteria for this research package

A third party can use these documents to:

- identify which Python and CPython behaviors are stable contracts and which are implementation details;
- implement a P0 front end and runtime without inventing basic numeric, scope, iteration, or exception semantics;
- explain why the compiler does not target CPython bytecode;
- implement a byte-oriented host protocol with call-scoped core-ABI buffers and a later pointer-free WIT wrapper;
- configure finite limits for compilation and every execution dimension controlled by the project;
- construct differential, negative, fuzz, and multi-runtime tests that reject unsupported behavior rather than silently changing it;
- determine which claims remain proposals or require deployment-specific validation.

## Open decisions requiring prototypes

The research does not pretend that benchmarks have already answered these questions:

- precise mark-sweep GC versus refcounting plus cycle collection for the target workload;
- boxed values versus guarded unboxed `i32`/`i64`/`f64` fast paths;
- explicit exception results versus WebAssembly exception handling across the intended runtimes;
- UTF-32 strings with UTF-8 conversion caches versus multiple internal string representations;
- instantiation cost and memory overhead across server-language embeddings;
- whether WASI 0.3 async host calls improve the workload enough to justify a separate async target profile;
- which CPython pure-Python standard-library modules can be reused legally and with acceptable compatibility and bundle size.

Each becomes an implementation gate only after measurement and semantic tests.

## Terminology and evidence labels

- **Python guarantee:** behavior specified by the Python language or library reference.
- **CPython detail:** behavior of the reference implementation, possibly version-specific.
- **Wasm standard/proposal:** distinguished explicitly; engine support is a separate question.
- **Proposed requirement:** a design rule for `pywasm-aot`, not a Python or Wasm requirement.
- **Deployment policy:** a configurable limit or capability choice, not a language semantic.
- **Open question:** requires a prototype, benchmark, or product decision.

Unless stated otherwise, normative words such as **must**, **must not**, and **should** in the design documents refer to the proposed compiler, not to Python itself.
