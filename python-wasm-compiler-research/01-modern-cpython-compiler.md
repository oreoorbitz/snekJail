# 1. Modern CPython Compiler

## 1.1 Scope and version

This chapter describes CPython 3.15 as the current implementation reference. Python 3.15 was released on 2026-10-09 and is documented as the latest stable feature series in the [Python 3.15 release notes](https://docs.python.org/3.15/whatsnew/3.15.html). CPython is one implementation of Python; its internal files, bytecode, code-object layout, inline caches, and JIT are not the language’s portable ABI.

The most useful primary sources are CPython’s [3.15 compiler design document](https://raw.githubusercontent.com/python/cpython/3.15/InternalDocs/compiler.md), [interpreter design document](https://raw.githubusercontent.com/python/cpython/3.15/InternalDocs/interpreter.md), [JIT design document](https://raw.githubusercontent.com/python/cpython/3.15/InternalDocs/jit.md), and the [3.15 `compile.c` source](https://github.com/python/cpython/blob/3.15/Python/compile.c).

## 1.2 End-to-end pipeline

```text
source bytes
  → source decoding and tokenization
  → token-driven PEG parser
  → abstract syntax tree (AST)
  → future-statement handling, AST validation/preprocessing
  → symbol-table construction and scope analysis
  → bytecode-like pseudo-instruction sequence
  → control-flow graph (CFG) and optimizations
  → assembled bytecode, metadata, exception/location tables
  → PyCodeObject
  → adaptive bytecode interpreter
  → optional hot-trace recording → micro-op optimization → experimental JIT
```

CPython’s compiler document summarizes the main path as tokenize, parse to AST, transform AST into an instruction sequence, construct and optimize a CFG, and emit bytecode. The 3.15 `compile.c` entry point additionally shows future handling, AST preprocessing, symbol-table construction, calls into `codegen.c`, and final assembly. When prose and source differ in detail, the versioned source is the more precise implementation reference.

### Design lesson for a new AOT compiler

Reuse the **stage boundaries**, not CPython’s concrete bytecode:

```text
Python source → Python AST → subset validation → scope resolution
              → Python-aware HIR → CFG/optional SSA → Wasm IR → binary
```

A Python-aware HIR should retain operations such as `PyBinaryAdd`, `PyGetIter`, `PyCall`, and `PyRaise` until their dynamic dispatch and exceptional behavior can be lowered correctly. Lowering `a + b` immediately to an integer instruction would confuse Python syntax with a machine-level type assumption.

## 1.3 Lexing and parsing

### Source decoding and tokens

The [lexical-analysis reference](https://docs.python.org/3.15/reference/lexical_analysis.html) specifies source decoding, logical and physical lines, indentation-generated `INDENT`/`DEDENT` tokens, identifiers, literals, operators, and delimiters. UTF-8 is the default source encoding. Python 3.15 also makes UTF-8 the default encoding generally; this does not eliminate explicit source-encoding declarations.

Important front-end requirements include:

- normalize physical line endings as specified;
- process explicit and implicit line joining;
- maintain the indentation stack and reject inconsistent dedents;
- handle identifiers and NFKC normalization;
- implement numeric and string literal rules, including f-strings;
- recognize soft keywords contextually rather than as globally reserved words.

Python 3.15 adds `lazy` as a soft keyword for explicit lazy imports. The P0 subset can reject lazy imports, but its parser should not accidentally make `lazy` an ordinary import form with different behavior.

### PEG parser

Since Python 3.9, CPython uses a token-driven PEG parser. The grammar is in [`Grammar/python.gram`](https://github.com/python/cpython/blob/3.15/Grammar/python.gram), literal token definitions are in [`Grammar/Tokens`](https://github.com/python/cpython/blob/3.15/Grammar/Tokens), and generated parser code is derived from those definitions. Selected grammar rules are memoized to improve backtracking performance.

A new implementation may use a PEG parser generator, a handwritten equivalent, or another parsing strategy that accepts exactly the intended grammar. It should compare parsed structure against CPython on the accepted subset rather than comparing parser-generator internals.

### Concrete versus abstract syntax

CPython’s parser constructs an AST directly; the public `ast` tree is not a lossless concrete syntax tree. Comments, exact whitespace, and all original formatting are not preserved. The [`ast` documentation](https://docs.python.org/3.15/library/ast.html) explicitly points users interested in formatting preservation toward concrete-syntax tooling.

For the AOT compiler:

- use a Python-shaped AST for semantic analysis;
- retain byte-accurate source spans for diagnostics;
- do not require round-trip source formatting from this AST;
- keep a token/CST layer only if editor tooling or source rewriting requires it.

## 1.4 AST contract and limitations

The AST is specified using ASDL in [`Parser/Python.asdl`](https://github.com/python/cpython/blob/3.15/Parser/Python.asdl). The public [`ast` module](https://docs.python.org/3.15/library/ast.html) exposes corresponding Python objects and helpers.

The documentation states that the abstract syntax itself may change with each Python release. Consequently:

- `ast` is a useful versioned interchange for tools running in the same Python feature series;
- it is not a permanent cross-version compiler IR;
- a serialized CPython AST is not a safe or stable untrusted artifact format;
- parsing into an AST does not prove that the code is compilable—scope and other checks happen later.

Expression and statement nodes carry line numbers and UTF-8 byte offsets. End positions are optional for compilation. A new compiler should define its own stable source-span type and convert from CPython locations if it uses CPython as a front-end oracle.

Notable 3.15 details include lazy-import fields in the abstract grammar and stricter AST-node construction behavior. Python 3.14 added template-string AST nodes. These examples illustrate why a compiler should pin a feature version and validate every node rather than assuming AST stability.

## 1.5 Preprocessing and symbol tables

Before code generation, CPython handles future statements and preprocesses the AST. `_PySymtable_Build()` in [`Python/symtable.c`](https://github.com/python/cpython/blob/3.15/Python/symtable.c) traverses code blocks and records binding and scope information. The public [`symtable` module](https://docs.python.org/3.15/library/symtable.html) exposes a read-only view of some of this analysis.

The analysis distinguishes, among other cases:

- local bindings;
- explicit and implicit globals;
- nonlocal/free variables;
- cell variables captured by nested scopes;
- module, function, class, comprehension, annotation, and other specialized scopes.

This pass is essential. Textual name lookup is insufficient for Python because assignment anywhere in a function affects local binding throughout that function unless declared `global` or `nonlocal`. Closures require captured cells rather than simple copying of an enclosing stack slot.

The new compiler should emit a resolved binding for every name:

```text
Local(slot) | Cell(slot) | Free(slot) | ModuleGlobal(name)
Builtin(name) | HostBuiltin(name) | Error(unresolved-or-invalid)
```

Class scopes and comprehensions have special behavior and should be implemented only with focused conformance tests. P0 avoids classes but still needs correct function, module, closure, and comprehension scoping.

## 1.6 Instruction sequence, CFG, optimization, and assembly

CPython’s code generation creates a sequence of pseudo-instructions. These are more abstract than final bytecode in some cases. `_PyCfg_FromInstructionSequence()` constructs basic blocks, `_PyCfg_OptimizeCodeUnit()` applies optimizations, and the optimized CFG is converted back to an instruction sequence before assembly.

A basic block has a single entry and sequential execution until its exit. Explicit CFG construction is valuable for:

- short-circuit boolean expressions and chained comparisons;
- loop exits and `else` clauses;
- exceptional edges and `finally` cleanup;
- jump threading and unreachable-code removal;
- constant propagation where Python semantics permit it;
- stack-depth or local-slot analysis.

Final assembly resolves abstract instructions and labels into actual bytecode and relative offsets, and constructs exception and source-location tables. `_PyAssemble_MakeCodeObject()` combines these with constants, names, variable information, flags, filename data, and other metadata into a `PyCodeObject`.

For the Wasm backend, equivalent metadata should include:

- function type and local declarations;
- source-span mapping for generated instructions;
- runtime function identifiers;
- closure/environment layouts;
- exception-cleanup regions in the compiler IR;
- static module and import descriptors;
- deterministic constant encoding;
- optional stack maps for precise garbage collection.

## 1.7 Code objects and bytecode are not a portable Python ABI

The [`dis` documentation](https://docs.python.org/3.15/library/dis.html) describes CPython bytecode as a CPython implementation detail and warns that bytecode can change between Python releases. CPython uses a magic number for bytecode versioning, but that supports CPython’s own artifact validation; it does not turn bytecode into a cross-implementation standard.

A code object couples bytecode to CPython’s object model, frame semantics, constants, exception table, inline caches, and interpreter behavior. Targeting it would require implementing a CPython-compatible interpreter before AOT specialization could begin.

**Proposed requirement:** the project must not consume `.pyc`, CPython adaptive bytecode, or serialized `PyCodeObject` values as its public input. Its own IR and artifact manifest must be explicitly versioned.

## 1.8 Adaptive interpreter and specialization

CPython’s adaptive interpreter executes bytecode and specializes selected instruction families using runtime observations and inline caches. Specialized operations use guards; when assumptions fail, execution returns to a more general path. The [interpreter internals](https://raw.githubusercontent.com/python/cpython/3.15/InternalDocs/interpreter.md) describe code units, extended arguments, inline cache entries, the evaluation stack, specialization, and interpreter dispatch choices.

The transferable idea is **guarded specialization**:

```text
if exact-small-int(a) and exact-small-int(b):
    use checked integer fast path
else:
    call generic Python addition
```

The non-transferable part is assuming that Python annotations alone justify removing the guard. AOT type inference may eliminate checks only when it proves all relevant dynamic properties. Otherwise it must retain a generic fallback.

## 1.9 The Python 3.15 JIT is not an AOT compiler

CPython’s [JIT design](https://raw.githubusercontent.com/python/cpython/3.15/InternalDocs/jit.md) describes hot-region detection, trace recording, translation into micro-ops, optimization, executor objects, side exits/deoptimization, and copy-and-patch generation from precompiled stencils. The JIT operates at runtime on hot traces and remains integrated with the adaptive interpreter and CPython runtime.

This differs from the proposed project:

| Property | CPython 3.15 JIT | Proposed `pywasm-aot` |
|---|---|---|
| Compilation time | During execution, after hot detection | Before deployment/execution |
| Input | Executed CPython bytecode trace | Accepted Python-subset AST/IR |
| Output | Native executor for a trace | WebAssembly module/component |
| Runtime assumptions | CPython frames, objects, caches, deopt | Project-owned Python-subset runtime |
| Main portability goal | CPython performance | Language-neutral sandboxed hosting |

CPython 3.15 describes its JIT as significantly upgraded but still experimental. The [3.15 release notes](https://docs.python.org/3.15/whatsnew/3.15.html) report an LLVM 21 build-time dependency, a tracing frontend, basic register allocation, and additional optimizations; these are runtime JIT improvements, not an AOT interface.

The feature’s longer-term status also changed during this research period. [PEP 836](https://peps.python.org/pep-0836/), which proposed commitments for continuing JIT work, was withdrawn on 2026-10-19. Contemporary [LWN reporting](https://lwn.net/Articles/1102102/) says removal from CPython’s `main` branch was expected early in the 3.16 development cycle. This is a current-development report, not a claim that Python 3.13–3.15 removed their experimental JIT or that the final 3.16 source state has already been verified here.

Accordingly, the JIT must not be presented as a stable roadmap dependency. Its existence in 3.15 does not provide a supported Python-source-to-Wasm AOT path, and its trace/micro-op interfaces should not become this project’s compatibility contract.

## 1.10 Runtime model relevant to compiler writers

The [execution model](https://docs.python.org/3.15/reference/executionmodel.html) and [data model](https://docs.python.org/3.15/reference/datamodel.html), not bytecode, define the important semantic targets:

- names bind to objects;
- objects have identity, type, and value;
- mutability and aliasing are observable;
- calls, attribute access, subscription, iteration, and operators may execute user-defined behavior in full Python;
- exceptions are part of normal control flow;
- modules and code blocks establish naming environments.

A restricted compiler can omit user-defined dispatch by excluding the relevant class/metaclass features, but it must specify the remaining behavior precisely. For example, P0 list and integer addition can use direct runtime paths because user-defined operand classes are outside that execution profile; mixed builtin operations still need Python-compatible coercion and error behavior.

## 1.11 What to reuse and what not to reuse

| CPython element | Use in this project |
|---|---|
| Language reference | Primary semantic specification, pinned by feature version |
| Grammar and token rules | Reference for a compatible front end and parser tests |
| Public AST | Structural oracle for accepted source; not a stable long-term IR |
| Symbol-table design | Conceptual model for binding and closure analysis |
| Pseudo-instructions and CFG | Architectural precedent for a private Python-aware IR |
| Code objects | Metadata checklist only; not a compatibility target |
| Bytecode and opcode numbers | Reject as unstable implementation details |
| Inline caches/adaptive specialization | Inspiration for guarded fast paths and later profiling |
| Tracing JIT and stencils | Not needed for the initial AOT backend |
| CPython standard library | Selective source review/reuse only after license, dependency, and semantic analysis |
| CPython test suite | Curated semantic corpus, adapted to subset restrictions |

## 1.12 Front-end implementation requirements

The proposed front end must:

1. pin `source-python = "3.15"` in the artifact manifest;
2. bound source bytes, tokens, AST nodes, nesting depth, literals, and compile time before expensive work;
3. parse all syntax it claims to recognize and reject unsupported nodes with stable diagnostic codes;
4. perform scope validation before type inference or optimization;
5. preserve left-to-right evaluation and exceptional edges in HIR;
6. keep annotations separate from runtime type proof;
7. produce deterministic IR and artifacts from identical inputs and compiler configuration;
8. compare accepted ASTs and execution behavior against CPython 3.15 in CI.

A smaller accepted grammar is preferable to accepting full Python syntax and then guessing at unsupported runtime behavior.
