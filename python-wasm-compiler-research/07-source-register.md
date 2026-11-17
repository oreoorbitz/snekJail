# 7. Source Register

## 7.1 How to read this register

**Research date:** 2026-11-08. This register maps external references to the factual claims they support. It does not convert the proposed `pywasm-aot` design into a Python, WebAssembly, WASI, or runtime standard.

Evidence labels used throughout the package:

- **Language guarantee:** specified by the Python language/library reference or an accepted standards-track PEP.
- **CPython detail:** observed in versioned CPython source or internal documentation; not a cross-implementation contract.
- **WebAssembly standard/proposal:** status must be distinguished using the core specification and official proposal tracker.
- **Runtime documentation:** describes a particular engine/version and is not automatically portable to another engine.
- **Secondary current-events source:** used only where official primary documentation does not yet record the complete development outcome.
- **Proposed requirement:** a design decision for this project, justified by the research but not sourced as an external obligation.

When documentation branches such as `3.15`, `main`, or unversioned runtime pages change, reproduce the result by recording the exact release/commit used for implementation. URLs alone do not freeze content.

## 7.2 Python language references

| Source | Claims supported in this package | Evidence class |
|---|---|---|
| [Python 3.15: What’s New](https://docs.python.org/3.15/whatsnew/3.15.html) | 3.15 feature baseline; explicit lazy imports, `frozendict`, UTF-8 default I/O, JIT improvements and release-note performance figures | Official release documentation |
| [Full grammar specification](https://docs.python.org/3.15/reference/grammar.html) | Versioned Python grammar and PEG notation; basis for accepted/rejected syntax | Language reference |
| [Lexical analysis](https://docs.python.org/3.15/reference/lexical_analysis.html) | UTF-8 source default, encoding declarations, lines, indentation, identifiers, literals, operators | Language reference |
| [Execution model](https://docs.python.org/3.15/reference/executionmodel.html) | Blocks, naming and binding, whole-block local rules, `global`/`nonlocal`, resolution and exceptions | Language guarantee |
| [Data model](https://docs.python.org/3.15/reference/datamodel.html) | Object identity/type/value, special methods, iteration, calls, context managers, awaitables; implementation-detail labels | Language guarantee plus marked CPython details |
| [Expressions](https://docs.python.org/3.15/reference/expressions.html) | Evaluation order, short circuiting, comparisons, arithmetic, subscriptions, calls, displays, comprehensions | Language guarantee |
| [Simple statements](https://docs.python.org/3.15/reference/simple_stmts.html) | Assignment, augmented assignment, `assert`, `raise`, `global`, `nonlocal`, imports | Language guarantee |
| [Compound statements](https://docs.python.org/3.15/reference/compound_stmts.html) | Functions, loops, `else`, `try`/`except`/`finally`, class/generator/async syntax for later tiers | Language guarantee |
| [Built-in types](https://docs.python.org/3.15/library/stdtypes.html) | Arbitrary-precision integers, binary64/complex numeric behavior, strings, bytes, containers, hashing and order | Library/language reference |
| [`ast`](https://docs.python.org/3.15/library/ast.html) | Public AST API, locations and helpers; explicit warning that abstract syntax may change each release | Public but version-specific CPython API |
| [`symtable`](https://docs.python.org/3.15/library/symtable.html) | Symbol tables are generated from AST before bytecode and determine identifier scopes | Public CPython inspection API |
| [`dis`](https://docs.python.org/3.15/library/dis.html) | Explicit warning that bytecode is a CPython implementation detail with no cross-release/VM stability | CPython detail |
| [`tokenize`](https://docs.python.org/3.15/library/tokenize.html) | Tokenization API and source-encoding detection | CPython library reference |
| [`importlib`](https://docs.python.org/3.15/library/importlib.html) | Bytecode magic/cache concepts and import implementation context | CPython/library detail |
| [Python WebAssembly platform notes](https://docs.python.org/3.15/library/intro.html#webassembly-platforms) | WebAssembly platform availability and restricted process/thread/socket/filesystem behavior | Official platform documentation |

## 7.3 Python Enhancement Proposals

| PEP | Claims supported | Status-sensitive note |
|---|---|---|
| [PEP 263](https://peps.python.org/pep-0263/) | Source encoding declaration syntax | Historical/final source-encoding contract |
| [PEP 3120](https://peps.python.org/pep-3120/) | UTF-8 as default Python 3 source encoding | Final |
| [PEP 552](https://peps.python.org/pep-0552/) | Hash-based `.pyc` validation concept | CPython artifact detail, not portable IR |
| [PEP 617](https://peps.python.org/pep-0617/) | CPython PEG parser, direct AST generation, removal of old CST path | Final; implementation architecture |
| [PEP 657](https://peps.python.org/pep-0657/) | Fine-grained source locations | Final; metadata design reference |
| [PEP 659](https://peps.python.org/pep-0659/) | Specializing adaptive interpreter, local guards/deoptimization concept | Final; not an AOT specification |
| [PEP 649](https://peps.python.org/pep-0649/) | Deferred/lazy annotation evaluation | Important reason annotations are not eager runtime type proof |
| [PEP 686](https://peps.python.org/pep-0686/) | UTF-8 default for general I/O in Python 3.15 | Distinct from the older UTF-8 source default |
| [PEP 744](https://peps.python.org/pep-0744/) | Copy-and-patch JIT background | Informational, not a stability commitment |
| [PEP 750](https://peps.python.org/pep-0750/) | Template strings and related modern AST/language features | Later-tier/rejected P0 syntax |
| [PEP 810](https://peps.python.org/pep-0810/) | Explicit lazy imports and 3.15 grammar changes | Rejected by P0 static import profile |
| [PEP 814](https://peps.python.org/pep-0814/) | `frozendict` in Python 3.15 | Later-tier value |
| [PEP 836](https://peps.python.org/pep-0836/) | Proposed JIT maintenance/compatibility commitments; withdrawn on 2026-10-19 | Withdrawn; do not describe it as accepted |
| [PEP 11](https://peps.python.org/pep-0011/) | CPython platform support tiers and current target listing | Changes over time; pin checked revision |
| [PEP 816](https://peps.python.org/pep-0816/) | WASI support planning, WASI SDK/version distinctions and move toward WASI 0.3 | Target support is separate from Python language version |

## 7.4 CPython versioned implementation sources

| Source | Claims supported | Stability conclusion |
|---|---|---|
| [`InternalDocs/compiler.md` on `3.15`](https://github.com/python/cpython/blob/3.15/InternalDocs/compiler.md) | Tokenize → AST → instruction sequence → CFG optimization → bytecode assembly; AST/symbol/codegen details | Architectural reference only |
| [`Python/compile.c` on `3.15`](https://github.com/python/cpython/blob/3.15/Python/compile.c) | Versioned compile entry, future/AST preprocessing, symbol/codegen/assembly calls | CPython detail |
| [`Grammar/python.gram` on `3.15`](https://github.com/python/cpython/blob/3.15/Grammar/python.gram) | Concrete PEG grammar and 3.15 lazy-import forms | Front-end reference, not generated-parser ABI |
| [`Grammar/Tokens` on `3.15`](https://github.com/python/cpython/blob/3.15/Grammar/Tokens) | Token definitions | Token numbering not a project ABI |
| [`Parser/Python.asdl` on `3.15`](https://github.com/python/cpython/blob/3.15/Parser/Python.asdl) | ASDL node schema and version-specific fields | Project-owned AST still required |
| [`Python/symtable.c` on `3.15`](https://github.com/python/cpython/blob/3.15/Python/symtable.c) | Static scope analysis and block traversal | Reuse concepts, not C layouts |
| [`InternalDocs/interpreter.md` on `3.15`](https://github.com/python/cpython/blob/3.15/InternalDocs/interpreter.md) | Code units, stack discipline, inline caches, specialization, interpreter variants | Not portable bytecode semantics |
| [`InternalDocs/jit.md` on `3.15`](https://github.com/python/cpython/blob/3.15/InternalDocs/jit.md) | Runtime traces, micro-ops, copy-and-patch/stencils, deoptimization | Runtime JIT, not AOT |
| [`InternalDocs/exception_handling.md` on `3.15`](https://github.com/python/cpython/blob/3.15/InternalDocs/exception_handling.md) | Zero-cost/table-driven CPython exception implementation | Conceptual comparison only |
| [`InternalDocs/code_objects.md` on `3.15`](https://github.com/python/cpython/blob/3.15/InternalDocs/code_objects.md) | Code-object metadata, bytecode and location-table context | Not an artifact compatibility target |
| [`InternalDocs/frames.md` on `3.15`](https://github.com/python/cpython/blob/3.15/InternalDocs/frames.md) | CPython frame/evaluation-stack organization | CPython detail |
| [CPython 3.15 `Lib/test`](https://github.com/python/cpython/tree/3.15/Lib/test) | Differential/curated semantic corpus and expected-reject cases | Preserve provenance and license notices |
| [CPython 3.15 WASI platform README](https://github.com/python/cpython/blob/3.15/Platforms/WASI/README.md) | WASI Tier 2 support and build-platform context | Does not prove every 3.15 build uses WASI 0.3 |
| [CPython 3.15 Emscripten platform README](https://github.com/python/cpython/blob/3.15/Platforms/emscripten/README.md) | Emscripten Tier 3 support and browser/JS/platform restrictions | Not a language-neutral server ABI recommendation |

**Unresolved documentation detail:** CPython internal documentation has linked to `Objects/object_layout.md`, but that path was not retrievable on the checked `3.15`/`main` branches during research. The package therefore makes no claim based on that file and treats C-level object layout as unstable implementation detail regardless.

## 7.5 WebAssembly and Component Model

| Source | Claims supported | Use limitation |
|---|---|---|
| [WebAssembly core specification](https://webassembly.github.io/spec/core/) | Normative core syntax, validation and execution reference | Does not establish host-engine support or Python semantics |
| [Core 3.0 change history](https://webassembly.github.io/spec/core/appendix/changes.html) | Core 2.0/3.0 feature mapping and relaxed-SIMD implementation-dependent behavior | Standardization is not runtime enablement |
| [Official WebAssembly proposal tracker](https://github.com/WebAssembly/proposals) | Finished versus active-proposal distinctions, including Component Model Phase 1, threads Phase 4 and stack switching Phase 3 | A finished proposal still needs engine/toolchain tests |
| [Finished-proposal register](https://github.com/WebAssembly/proposals/blob/main/finished-proposals.md) | Reference types, bulk memory, multi-value and fixed-width SIMD in 2.0; EH, GC, memory64 and related features in 3.0 | Pin the checked revision |
| [Stack-switching explainer](https://github.com/WebAssembly/stack-switching/blob/main/proposals/stack-switching/Explainer.md) | Coroutines, async/await and generators are primary use cases | Active proposal; not the P0 implementation dependency |
| [Component Model documentation](https://component-model.bytecodealliance.org/) | Components, typed interfaces, composition, WIT and Canonical ABI concepts | Documentation/tooling versions must be pinned |
| [Component Model repository](https://github.com/WebAssembly/component-model) | Proposal development, binary/model context and WASI preview tooling | Do not equate ecosystem tooling stability with completed core-standard status |
| [WIT design documentation](https://component-model.bytecodealliance.org/design/wit.html) | Interfaces, types and worlds used by the proposed wrapper; WIT does not define behavior or package resolution | Exact world/interface versions and vendored dependencies remain mandatory |
| [Canonical ABI explainer](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md) | Shared-nothing, linear-memory-based ABI for copying component values | Not arbitrary guest-object graph sharing |
| [WASI project](https://wasi.dev/) | WASI purpose, interface families and capability-oriented system APIs | WASI is optional for pure core computation |
| [WASI repository](https://github.com/WebAssembly/WASI) | Versioned WASI interface/release development and “eventual standardization” framing | Check exact release/world rather than relying on `main` |
| [WASI roadmap](https://wasi.dev/roadmap) | WASI 0.3.0/0.3.1 dates, async features, and identified Wasmtime 46+/jco support | Not universal runtime or embedding-API maturity |
| [WASI releases](https://wasi.dev/releases) | Current release positioning and continued WASI 0.2 toolchain relevance | Pin exact release and target world |

## 7.6 Runtime security and resource control

| Source | Claims supported | Use limitation |
|---|---|---|
| [WebAssembly security](https://webassembly.org/docs/security/) | Sandbox is complementary to process isolation, syscall interception and permissions | Not a claim of complete multi-tenant/security-side-channel protection |
| [WASI security](https://wasi.dev/security) | Host-granted, no-ambient-authority capability model | Exact granted imports/preopens still determine authority |
| [Wasmtime security](https://docs.wasmtime.dev/security.html) | Defense in depth, Wasm isolation context, untrusted output risks, engine security considerations, deterministic execution/resource-control discussion | Wasmtime-specific APIs are not portable requirements |
| [Wasmtime embedding/API documentation](https://docs.wasmtime.dev/) | Fuel, epoch interruption, memory/configuration and embedding concepts | Verify against exact engine/SDK version |
| [Wasmtime `Config`](https://docs.wasmtime.dev/api/wasmtime/struct.Config.html) | Feature flags, fuel, epoch, stack and deterministic numeric/runtime configuration examples | Implementation-specific |
| [Wasmtime `Module`](https://docs.wasmtime.dev/api/wasmtime/struct.Module.html) | Native module serialization/deserialization context | Native artifacts are trusted-input-only deployment caches |
| [Wasmtime WASI 0.3 embedding module](https://docs.wasmtime.dev/api/wasmtime_wasi/p3/index.html) | Experimental/unstable/incomplete embedding warning despite WASI 0.3 runtime release support | Runtime release and embedding API maturity are separate |
| [RFC 8949: CBOR](https://www.rfc-editor.org/rfc/rfc8949.html) | CBOR data model, bignum tags and deterministic-encoding rules | Project profile still defines value domain, float profile, limits, errors and unknown fields |

The requirements for fresh instances, no ambient authority, finite limits, validated caches, bounded protocols, and error separation are **project security requirements derived from the threat model**. No external source is cited as though it mandates this exact architecture.

## 7.7 Alternative Python-on-Wasm implementations

| Source | Comparison supported | Not claimed |
|---|---|---|
| [Pyodide](https://pyodide.org/) | CPython/package ecosystem for browser/JavaScript-oriented Wasm environments and documented platform constraints | Not equivalent to a minimal language-neutral AOT runtime |
| [`componentize-py`](https://github.com/bytecodealliance/componentize-py) | Packaging Python applications as components using WIT worlds and a Python runtime/interpreter approach | Not direct subset-source-to-Wasm AOT without an evaluator |
| [Component Model Python build guide](https://component-model.bytecodealliance.org/language-support/building-a-simple-component/python.html) | Official component-generation workflow and explicit host linker requirements | Toolchain/version-specific |
| [`componentize-py` architecture overview](https://www.fermyon.com/blog/introducing-componentize-py) | Bundling/linking CPython and pre-initialization architecture | Vendor technical source; builds must be reviewed for code execution |
| [RustPython](https://github.com/RustPython/RustPython) | Rust Python implementation with WebAssembly/WASI build paths and its own readiness constraints | Not evidence of direct-AOT semantics or universal production support |

Performance, bundle-size, cold-start, and compatibility comparisons require benchmarks on pinned versions. This package intentionally makes no unsupported ranking.

## 7.8 Secondary source for JIT development status

| Source | Claim supported | Qualification |
|---|---|---|
| [LWN: “Python’s experimental JIT to be removed”](https://lwn.net/Articles/1102102/) | Reporting on 2026-10-26 that removal from CPython `main` was expected early in the 3.16 cycle | Secondary current-events evidence; verify final 3.16 source before making a completed-removal claim |

The official withdrawn status comes from PEP 836 itself. The package does not say Python 3.13–3.15 lack their experimental JIT, nor does it claim that the final 3.16 removal has already been verified.

## 7.9 Claim-to-design traceability

| Design conclusion | Evidence inputs | Project inference |
|---|---|---|
| Do not target CPython bytecode | `dis` instability warning, versioned code-object/interpreter/JIT sources | Use project-owned HIR/CFG and Wasm artifact |
| Use AST only as a versioned front-end oracle | `ast` release-variability warning, ASDL and PEP 617 | Convert immediately to project AST and pin Python feature version |
| Preserve dynamic semantics with guarded optimization | Python execution/data model and PEP 659’s specialization concept | Generic AOT fallback instead of interpreter deoptimization |
| Start with linear-memory runtime | Core Wasm supplies memory/validation, not Python objects/GC | Implement tagged values, bigint, Unicode, containers and precise GC |
| Avoid P0 dependence on Wasm EH/stack switching | Proposal/status separation and Python cleanup/suspension requirements | Explicit outcomes and later state-machine generators |
| Core module first, component wrapper second | Core/component binary distinction and variable deployment support | C0 byte ABI plus generated C1/C2 targets |
| Do not inherit CPython’s WASI ABI automatically | PEP 11/816 target versions differ from Python feature versions | Version source profile and host ABI independently |
| Deny ambient authority | Threat model and capability-oriented WASI/runtime model | Narrow linker plus server-held operation grants |
| Separate fuel, deadline, memory, protocol and output limits | Runtime resource controls measure different dimensions | Multi-dimensional policy and exhaustion tests |
| Use deterministic CBOR with a restricted value domain | RFC 8949 plus cross-language protocol needs | Versioned envelopes, canonical bytes, explicit encoding failures |
| Test on CPython and multiple engines | Language semantics and engine support are independent oracles | Differential corpus plus versioned support matrix |

## 7.10 Claims intentionally left unmade

The package does not claim that:

- the proposed P0/P1/P2 tiers are an existing Python standard;
- every Python 3.15 package or C extension can compile;
- CPython bytecode, AST, marshal, or code objects are stable ABIs;
- WebAssembly alone provides complete sandboxing or side-channel confidentiality;
- every engine supports every finished proposal, component, or WASI release;
- WASI 0.3 is required for P0 or automatically implements Python async semantics;
- direct AOT is inherently faster, smaller, or more compatible than interpreter-based alternatives;
- example policy numbers are safe defaults for every deployment;
- a digest authenticates an artifact without a signature/trust process;
- prototype-open questions have been resolved by benchmarks.

These limitations are part of the specification. A later implementation report should add pinned versions, measurements, test results, and any deviations rather than silently removing them.
