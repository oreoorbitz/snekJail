# 5. Security and Host Integration

## 5.1 Security objective

Execute untrusted, potentially LLM-generated Python-subset programs as bounded computations with explicitly granted host services. The design must avoid ambient OS authority, per-request containers, and direct exposure of host memory while preserving useful diagnostics and deterministic application behavior.

WebAssembly is one isolation layer, not a complete security program. The [WebAssembly core specification](https://webassembly.github.io/spec/core/) defines validation and execution, while the official [WebAssembly security documentation](https://webassembly.org/docs/security/) explicitly describes the sandbox as complementary to process isolation, syscall interception, and permissions. Runtime hardening, embedding configuration, resource controls, API design, patching, and operational isolation remain deployment responsibilities. [WASI’s security model](https://wasi.dev/security) describes host-granted capability authority, and [Wasmtime’s security documentation](https://docs.wasmtime.dev/security.html) is a useful reference for defense-in-depth concerns; the requirements here apply to every supported engine.

### Security properties intended for P0

A conforming deployment must enforce all of these:

- untrusted source cannot execute during compilation;
- every artifact is validated before caching or execution;
- guest code receives only the linked imports explicitly granted by host policy;
- each request receives fresh mutable guest state and bounded linear memory;
- every controlled resource dimension has a finite limit and termination path;
- host adapters bounds-check every buffer, length, integer conversion, and payload;
- guest Python exceptions, policy denials, resource termination, cancellation, and internal failures remain distinguishable;
- output and logs are captured, bounded, and never written raw to a server terminal;
- one request cannot read or mutate a prior request’s guest state;
- no security claim relies solely on a Python-level import allowlist.

These are necessary project requirements, not a claim of protection against every engine vulnerability or microarchitectural side channel.

## 5.2 Threat model

### Untrusted inputs

Treat all of the following as hostile:

- LLM-generated source and module manifests;
- source produced after prompt injection in another tool or document;
- generated Wasm, even when returned by the project compiler;
- CBOR requests, host-call arguments, results, logs, Unicode, and numeric values;
- artifact metadata, source maps, target-feature declarations, and cache keys;
- guest control flow, recursion, allocation patterns, and exception behavior;
- malicious or compromised compiler/runtime dependencies.

### Assets

Protect:

- host process memory and control flow;
- server credentials, environment variables, filesystem, network identity, and OS privileges;
- other tenants’ inputs, outputs, guest memories, and service quotas;
- artifact-cache integrity and compiler/runtime supply-chain integrity;
- service availability and predictable tail latency;
- capability authority granted to the host application;
- logs and telemetry from injection, secrets, and unbounded growth.

### Representative attacks

| Attack | Required defense |
|---|---|
| Infinite loop or recursion | Fuel/deadline interruption, frame/stack limits, clean teardown |
| Huge integer, shift, string, or container | Checked operation costs, bigint/container limits, hard memory maximum |
| Guest reads server files or environment | No ambient WASI preopens/imports; narrow linker and capability broker |
| Guest calls arbitrary network/OS API | Explicit operation allowlist and per-operation policy; no generic host evaluator |
| Malformed artifact corrupts host | Independent validation, restrictive instantiation, patched engine |
| Buffer length overflow or stale pointer | Call-scoped buffers, checked arithmetic, SDK-generated bounds checks |
| Cross-request data leak | Fresh store/instance/memory/globals; no used-instance pooling |
| Output-terminal injection | Capture bytes, escape/filter on display, never stream raw to terminal |
| Cache poisoning | Content-addressed immutable artifacts, binary validation, authenticated release policy |
| Slow or abusive host call | Per-call timeout, total deadline, payload and call-count quotas, cancellation |
| Error-message information leak | Separate safe guest diagnostics from opaque internal error identifiers |

### Out of scope as an absolute guarantee

P0 does not claim perfect confidentiality against Spectre-like hardware behavior, compromised host kernels, malicious administrators, or a successfully exploited engine. For hostile multi-tenant production service, the recommended deployment runs executions inside **long-lived, unprivileged, OS-sandboxed worker processes or containers**, with worker CPU/RSS/syscall/network controls and stronger separation between trust tiers. This is defense in depth around the Wasm engine, not a requirement to create a Docker container for every program or request. Low-risk single-tenant embeddings may choose a smaller boundary after documenting the residual risk.

## 5.3 Trust and control planes

```text
untrusted source
      │
      ▼
trusted compiler service ── source/AST/compile budgets
      │                       no source execution, no ambient I/O
      ▼
independent artifact validation ── digest, manifest, target policy
      │
      ▼
immutable artifact cache
      │
      ▼
host adapter / execution coordinator
      ├─ resolves tenant policy and allowed operations
      ├─ creates fresh engine store + instance + memory
      ├─ decodes bounded request
      ├─ enforces fuel, deadline, cancellation, memory, output
      ├─ mediates each host call using server-held authority
      └─ destroys request state on every outcome
```

The compiler is trusted to produce artifacts but is not given the guest’s runtime capabilities. The execution engine is trusted as part of the deployment’s TCB but artifacts are still validated. The host adapter is trusted to enforce policy; guest-supplied policy fields never expand its own authority.

## 5.4 Capability model

### Default world

The default P0 world contains no filesystem, directory preopen, environment, socket, process, clock, random, terminal, or dynamic-loader authority. `math`, `json`, builtins, and pure computation are implemented inside the guest runtime and do not imply host capabilities.

A host-call import is linked only when the selected policy permits at least one operation. If no operations are granted, the default artifact should require no host-call import at all, or the import must unconditionally deny calls without exposing host details.

### Server-held grants

A grant is resolved by the host from authenticated application/tenant policy, not from a token invented by the guest:

```json
{
  "operation": "example.kv.get",
  "version": 1,
  "max_calls": 4,
  "max_request_bytes": 4096,
  "max_response_bytes": 16384,
  "timeout_ms": 100,
  "arguments": {
    "namespace": "server-assigned-namespace",
    "key_pattern": "allowed-key-rule"
  }
}
```

Each operation definition must specify:

- stable operation name and interface version;
- request and response schemas with size/depth limits;
- server-side argument constraints and tenant binding;
- call-count, rate, concurrency, and byte quotas;
- timeout and cancellation behavior;
- whether failures are guest-catchable application errors;
- side effects, idempotency, retry rules, and audit classification.

The broker never accepts a guest-provided host pointer, file descriptor, environment name, URL allowlist bypass, module path, or serialized native object as ambient authority.

### Python mapping

An allowed operation may appear through a reviewed Python wrapper. Ordinary operation failure can map to a curated `HostCallError` if the profile allows guest recovery. Capability denial, execution-budget exhaustion, and host-forced cancellation are uncatchable control outcomes. A guest cannot turn denial into an unlimited retry loop.

If a later profile grants HTTP/network access, prefer a typed broker operation over raw sockets. The broker must constrain scheme, DNS resolution, resolved IP ranges, port, redirects, headers, bytes, connections, and time. Loopback, link-local, private, multicast, metadata, and internal addresses are denied unless explicitly authorized. DNS and address checks occur inside the broker so a guest cannot bypass SSRF policy through another API.

## 5.5 Resource policy

No single limit is sufficient. Fuel does not bound time spent inside a host call; a wall deadline does not provide deterministic instruction accounting; an engine memory maximum does not bound response serialization or compiler work.

The following example illustrates the **dimensions**, not universally safe production values:

```json
{
  "policy": "pywasm-policy/v1",
  "compile": {
    "source_bytes_max": 262144,
    "module_files_max": 8,
    "ast_nodes_max": 100000,
    "nesting_depth_max": 100,
    "artifact_bytes_max": 8388608,
    "wall_time_ms_max": 5000,
    "memory_bytes_max": 268435456
  },
  "execute": {
    "fuel_max": 50000000,
    "wall_time_ms_max": 100,
    "cancellation_grace_ms_max": 10,
    "linear_memory_bytes_max": 67108864,
    "guest_heap_bytes_max": 33554432,
    "python_frames_max": 200,
    "bigint_bits_max": 65536,
    "container_elements_max": 100000,
    "string_codepoints_max": 1000000
  },
  "protocol": {
    "request_bytes_max": 1048576,
    "response_bytes_max": 1048576,
    "serialization_depth_max": 64,
    "stdout_bytes_max": 65536,
    "stderr_bytes_max": 16384,
    "host_calls_max": 8,
    "host_call_request_bytes_max": 65536,
    "host_call_response_bytes_max": 262144,
    "host_call_total_time_ms_max": 50
  }
}
```

Every deployment must choose finite values for every dimension it exposes. Adding a new host operation or serializer requires adding its limits to the same reviewed policy.

### Fuel and epoch/deadline interruption

Where supported, use both:

- **Fuel:** deterministic consumption checks for generated code and runtime operations. It supports reproducible resource tests but requires correct accounting in runtime/library paths.
- **Epoch or host deadline interruption:** lower-overhead external interruption that can stop execution not making adequate fuel progress. Its exact interruption point is not a deterministic semantic result.

Engine behavior and APIs differ. The adapter must test whether interruption is synchronous or asynchronous, whether a host call can be interrupted, how traps are classified, and whether the instance remains safe to reuse. P0 destroys it regardless.

### Memory and stack

Set both an engine-visible linear-memory maximum and a lower guest heap budget. Reserve room for runtime structures, stack, host buffers, and error responses. Check every size calculation before allocation. Guest recursion has a Python frame limit in addition to the engine’s native stack protection; stack exhaustion must not crash the host process.

### Expensive operations

Account for work not proportional to generated instruction count:

- bigint multiplication, division, shifts, and decimal conversion;
- hashing and equality over large values;
- container growth and sorting;
- Unicode encoding/decoding and formatting;
- serialization, canonical ordering, and `repr`;
- host calls and response decoding.

Policies may define cost units in addition to fuel. Exceeding a limit returns a resource outcome; it never wraps or truncates the mathematical value and reports success.

## 5.6 Request lifecycle and cache rules

### Per-request sequence

1. Authenticate the application/tenant and resolve a server-side policy.
2. Reject oversized request bodies before CBOR decoding.
3. Look up the immutable artifact by authenticated content digest and target configuration.
4. Validate binary kind, structure, imports/exports, features, memory limits, and manifest consistency.
5. Create a fresh store/context and fresh instance with fresh memory, globals, tables, module state, GC state, and exception state.
6. Link only allowed imports with operation-specific quotas.
7. Arm fuel, deadline, cancellation, and output controls before calling guest code.
8. Execute module initialization and then `run(input)` in that same request instance.
9. Bound, decode, validate, and canonicalize the response.
10. Destroy the instance/store on success, Python error, denial, timeout, trap, cancellation, or adapter failure.

### Cacheable

- immutable source/module-graph digests;
- validated `.wasm` or component artifacts;
- engine-specific compiled code only when keyed by artifact digest, exact engine version, target CPU/feature configuration, runtime ABI, and security-relevant settings;
- immutable compiler/runtime metadata needed to interpret the artifact.

Portable `.wasm`/component bytes are the distribution format. Engine-native precompiled artifacts are runtime/version/architecture/configuration-specific and are accepted only from the trusted build service. Callers must never submit arbitrary native artifacts for deserialization. Authenticate native cache entries before opening them, bind them to the exact engine fingerprint and policy, and invalidate them on runtime or security-configuration upgrades.

### Never reused across untrusted requests

- guest memories, stores, globals, tables containing mutable state, module objects, closure cells, exceptions, or host-call scratch data;
- decoded tenant inputs or output buffers;
- partially initialized or previously executed instances;
- capability sessions or broker quotas unless the host explicitly creates a new request-scoped session.

Pooling pristine, never-executed instances is a later measured optimization. Pooling used instances is prohibited in P0 because perfect reset of every runtime and engine state is a much harder invariant than fresh creation.

## 5.7 Wire protocol

### Encoding rules

Use [CBOR (RFC 8949)](https://www.rfc-editor.org/rfc/rfc8949.html) with a profile based on its deterministic-encoding rules:

- reject trailing data, indefinite-length items, duplicate map keys, non-canonical integers/lengths, and invalid UTF-8 text;
- bound input bytes before allocation and nesting depth before recursive descent;
- encode maps with unique text keys using RFC 8949’s core deterministic ordering: length-first by encoded key bytes, then bytewise lexicographic order;
- use standard CBOR bignum tags 2 and 3 for integers outside the basic 64-bit range, subject to bigint policy;
- encode every protocol float as an eight-byte CBOR binary64 item, preserving finite values, infinities, and signed zero; encode every NaN as the single canonical quiet-NaN binary64 bit pattern `0x7ff8000000000000`, because NaN payload preservation is not a compatibility promise;
- treat canonical bytes as the cache/signature representation, not a re-serialization performed with host-language defaults.

Canonicalization is validated by encoding a decoded value again and comparing bytes where appropriate. P0 rejects every unknown field in a versioned envelope and never ignores a malformed field silently. Extension fields require a new explicitly negotiated protocol/profile version.

### P0 host value domain

The initial cross-language value domain is deliberately narrower than all guest runtime values:

| Host value | CBOR representation | Guest mapping |
|---|---|---|
| null | `null` | `None` |
| boolean | `true`/`false` | `bool` |
| integer | CBOR integer or bignum tag | arbitrary-precision `int`, bounded |
| binary | byte string | `bytes` |
| text | valid UTF-8 text string | `str` without lone surrogates |
| float | profile-defined CBOR float | binary64 `float` |
| array | array of host values | `list` |
| object | map with unique text keys | `dict[str, value]` |

A returned tuple, set, complex value, non-text dict key, lone-surrogate string, or other unsupported host value produces `E_ENCODE_UNSUPPORTED_VALUE` unless a later typed-value extension defines it. The host never silently converts these to a different type. Guest code can explicitly convert a tuple to a list or a complex result to an object before returning it.

A later `pywasm-values/v2` extension can add explicit tagged tuple, set, complex, and typed-mapping representations without changing the execution control envelope.

### Request envelope

```json
{
  "protocol": "pywasm-exec/v1",
  "request_id": "host-generated-correlation-id",
  "entrypoint": "run",
  "input": {"example": [1, 2, 3]},
  "policy_id": "server-selected-policy-identifier"
}
```

The JSON shape documents fields; production bytes are deterministic CBOR. `policy_id` is a correlation/reference field checked against server-resolved policy. It does not authorize an artifact or guest to select stronger capabilities.

### Success envelope

```json
{
  "protocol": "pywasm-exec/v1",
  "request_id": "same-host-generated-correlation-id",
  "status": "ok",
  "result": {"total": 6},
  "stdout": "",
  "stderr": "",
  "truncated": {"stdout": false, "stderr": false},
  "metrics": {
    "fuel_used": 12345,
    "guest_duration_us": 800,
    "peak_linear_memory_bytes": 2097152,
    "host_calls": 0
  }
}
```

Metrics are host-generated, bounded, and informational. They are not guest-authoritative evidence. Output truncation must set a flag; exceeding the hard output policy may instead terminate the request according to policy, but cannot produce unbounded data.

`stdout` and `stderr` are valid protocol text. The guest adapter defines UTF-8 conversion and escaping for otherwise unencodable code points. Raw guest output is never sent directly to a terminal or log collector without the host’s escaping and structured-field handling.

### Error envelope

```json
{
  "protocol": "pywasm-exec/v1",
  "request_id": "same-host-generated-correlation-id",
  "status": "python_error",
  "error": {
    "code": "E_PYTHON_EXCEPTION",
    "class": "ValueError",
    "message": "bounded guest-safe message",
    "frames": [
      {"function": "run", "line": 4, "column": 9}
    ]
  },
  "stdout": "",
  "stderr": "",
  "truncated": {"stdout": false, "stderr": false}
}
```

Exception chaining can be represented as a bounded list of guest-safe exception records. Host paths, native addresses, engine internals, secrets, and arbitrary core dumps are not included.

### Component-world equivalent

The C1 component target wraps the same canonical envelopes rather than inventing a second value semantics:

```wit
package pywasm:exec@1.0.0;

interface sandbox {
    // Ok contains a success envelope; Err contains an error envelope.
    run: func(request-cbor: list<u8>) -> result<list<u8>, list<u8>>;
}

interface host-call {
    call: func(request-cbor: list<u8>) -> result<list<u8>, list<u8>>;
}

world executor {
    export sandbox;
}

world executor-with-host-call {
    import host-call;
    export sandbox;
}
```

The default `executor` world has no WASI imports. `executor-with-host-call` is used only with a server-held grant. WIT `result` distinguishes expected execution outcomes from traps; the CBOR payload retains the detailed stable error code. Size, time, and quota limits remain policy-enforced because WIT types alone do not bound them.

## 5.8 Error taxonomy

| Status | Example codes | Guest-catchable? | Meaning |
|---|---|---:|---|
| `invalid_request` | `E_PROTOCOL_DECODE`, `E_PROTOCOL_NONCANONICAL`, `E_REQUEST_TOO_LARGE` | No execution occurs | Host rejected the envelope/value before guest execution |
| `python_error` | `E_PYTHON_EXCEPTION`, `E_ENCODE_UNSUPPORTED_VALUE` | The original Python exception may be; encoding failure occurs after return | Accepted guest semantics failed or result cannot cross profile |
| `policy_denied` | `E_CAPABILITY_DENIED`, `E_OPERATION_DENIED`, `E_HOST_QUOTA` | No in P0 | Guest requested authority not granted by policy |
| `resource_exhausted` | `E_RESOURCE_FUEL`, `E_RESOURCE_MEMORY`, `E_RESOURCE_STACK`, `E_RESOURCE_BIGINT`, `E_RESOURCE_OUTPUT`, `E_RESOURCE_SERIALIZATION` | No for host-forced termination | A declared finite limit was reached |
| `cancelled` | `E_CANCELLED`, `E_DEADLINE` | No | Host/application cancelled or deadline expired |
| `host_error` | `E_HOST_OPERATION`, `E_HOST_TIMEOUT`, `E_HOST_RESPONSE` | Only if operation contract explicitly maps it to `HostCallError` | A granted operation failed without implying guest compromise |
| `internal_error` | `E_ENGINE_TRAP`, `E_RUNTIME_INTERNAL`, `E_ADAPTER_INTERNAL` | No | Engine, runtime, or adapter invariant failed |

A generic Wasm trap is not automatically `E_RESOURCE_MEMORY` or `E_PYTHON_EXCEPTION`. The adapter classifies only traps it can attribute reliably; everything else is `E_ENGINE_TRAP` with an internal diagnostic correlation ID.

Compiler diagnostics use the separate `E_SOURCE_*`, `E_SYNTAX`, `E_SCOPE`, `E_UNSUPPORTED_FEATURE`, and `E_RESOURCE_COMPILE` families defined in the subset specification.

## 5.9 Host-call subprotocol

A synchronous core-module call uses a request-scoped sequence number:

```json
{
  "protocol": "pywasm-hostcall/v1",
  "sequence": 0,
  "operation": "example.kv.get",
  "version": 1,
  "payload": {"key": "example"}
}
```

The broker checks the expected sequence, operation grant, version, schema, payload size, current total deadline, and remaining quota before dispatch. The response is either a bounded success payload or a structured operation error. It contains no host pointer or native object reference.

Core ABI guest-memory buffers are valid only for the documented synchronous call sequence. The host must validate:

- pointer/length addition and conversion overflow;
- that the complete range lies within the current exported memory;
- output pointers and lengths after every guest write;
- maximum request/response sizes before copying;
- status codes and protocol versions;
- reentrancy state and expected call sequence;
- memory growth during or around the call according to the ABI contract.

The component wrapper exchanges WIT-level byte strings/records and removes public guest-memory pointer manipulation, but retains all schema, quota, timeout, and capability checks.

## 5.10 Observability and audit

Record structured, bounded events for compilation and execution:

- request/tenant correlation ID according to the application’s privacy policy;
- artifact and module-graph digest;
- compiler version, source profile, runtime ABI, and host protocol;
- engine/SDK/target versions;
- policy identifier and granted operation names, not secret grant contents;
- outcome code, fuel used, duration, peak memory, output sizes, and host-call counts;
- cache hit/miss and validation result;
- cancellation, termination, and teardown completion.

Do not log source, payloads, returned values, environment, credentials, or full guest tracebacks by default. Guest text is untrusted log data: bound it, escape control characters, and prevent it from forging structured log fields.

Metrics and logs must not become an unbounded side channel. Debug source maps and core dumps require separate access control and retention policy.

## 5.11 Supply chain and operational hardening

- Run the compiler in an unprivileged, disposable, resource-limited build worker with no production secrets and no default network. The proposed direct-AOT pipeline never pre-initializes source; alternative packaging tools must be reviewed separately because some do execute/pre-initialize application code during builds.
- Pin compiler, runtime library, Unicode data, WASI/WIT definitions, engine, SDK, and adapter versions.
- Generate an SBOM and provenance record containing source, dependency, toolchain, target, policy, and output digests; prohibit mutable `latest` dependencies.
- Generate reproducible artifacts and compare independent builds in release tests.
- Authenticate release artifacts with the deployment’s signing/trust policy; a plain SHA-256 digest identifies but does not authenticate.
- Track engine and dependency security advisories and define an emergency artifact/cache invalidation process.
- Restrict compiler-service filesystem/network access as well as guest access.
- Validate artifacts again after cache retrieval; cache presence is not proof of trust.
- Apply host OS resource limits and monitoring to the whole engine process as defense in depth.
- Test host survival under repeated traps, cancellation, memory pressure, and malicious concurrency.
- Treat optional native CPU features as part of the compiled-artifact cache key and portability profile.

## 5.12 Testable security invariants

A third-party conformance suite must be able to reject a deployment that violates any clause:

1. **No compile-time execution:** rejected or compiled source performs no import, host call, output, or source-defined function execution in the compiler process.
2. **Closed linker:** instantiation fails when the artifact requests any import absent from the exact granted world.
3. **Fresh state:** two requests with the same artifact cannot observe each other’s globals, memory, module initialization, files, output, exceptions, or capabilities.
4. **Finite execution:** every test infinite loop terminates within the configured fuel/deadline policy and the next request succeeds.
5. **Finite memory:** excessive bigint, string, container, recursion, and response growth produce defined resource outcomes without host failure.
6. **Bounded protocol:** malformed, non-canonical, duplicate-key, deep, and oversized CBOR fails before uncontrolled allocation or recursion.
7. **No forged authority:** changing guest operation names, policy IDs, sequences, payloads, or pointers cannot expand the server-held grant.
8. **Buffer safety:** every out-of-bounds/overflowing core ABI buffer is rejected without host memory access or partial success.
9. **Clean teardown:** success, Python exception, host error, trap, cancellation, and resource exhaustion all release request state and quotas.
10. **Cache separation:** mutable instances are never stored in artifact caches; engine compiled-code keys include all compatibility/security inputs.
11. **Output containment:** terminal escapes and log-forging text remain inert bounded data.
12. **Error separation:** Python exceptions, denials, resource termination, cancellation, host operation errors, and internal traps have distinct outcomes.
13. **No silent coercion:** unsupported result types and unencodable Unicode fail explicitly rather than changing value type or reporting partial success.
14. **Independent validation:** mutated binaries, manifests, imports, feature declarations, and memory limits are rejected before execution.
15. **Host survival:** fuzzing and repeated malicious requests do not crash, hang, or unboundedly grow the host process.

These invariants complement the semantic and multi-engine gates in [the implementation roadmap](06-implementation-verification-roadmap.md). A fast execution path that violates any of them is not a supported optimization.
