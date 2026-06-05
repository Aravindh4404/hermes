# Context System Test — Full Context
Date: 2026-06-05
Method: All tools available (GBrain, Wiki, Learnings, LSP, docs, git, grep — Claude chooses what fits)
GBrain pages: 50 (indexed at time of session)
Wiki articles: 11 (hermes-codebase wiki, hermes-wiki-summaries tag)
Learnings: active in gstack-brain (count unavailable — source search returned no results)

---

## Q1: Two execution paths in Hermes
Tools used: **Wiki (hermes-architecture.md)** — primary; GBrain search cross-checked; Static Hermes wiki confirmed.
The wiki had a pre-compiled, authoritative "Dual Execution Paths" section that answered the question directly. GBrain search surfaced the wiki pages but not better raw content.

Answer:

Hermes has two distinct compilation and execution paths, both sharing the same front-end (parser, semantic resolver, HermesIR, optimizer):

### Path 1: hermesc → bytecode → interpreter (+ optional JIT)

The traditional path used by React Native today.

```
Source (.js) → Parser → AST → SemanticResolver → HermesIR (SSA) → Optimizer → BCGen → .hbc bytecode
```

At runtime, `Runtime` interprets the `.hbc` bytecode using a register-based interpreter. If JIT is enabled (`-Xjit`), hot functions are compiled to native code by a baseline JIT. Key characteristics:
- Requires the Hermes VM at runtime
- Supports lazy compilation (ship source, compile on first call)
- Can mix AOT-compiled bundles, lazy-compiled code, and JIT'd hot paths simultaneously
- Default until this branch — runtime is the bottleneck

### Path 2: shermes → native binary (AOT)

The Static Hermes path introduced on the `static_h` branch.

```
Source (.js/.ts/.flow) → Parser → [FlowChecker (if -typed)] → HermesIR → Optimizer → BCGen/SH → C code → clang → ELF/Mach-O/PE
```

No interpreter at runtime — the `static_h` runtime is linked directly into the output binary. Typed Flow annotations (via `-typed`) unlock exact object layouts, unboxed primitives, and inlined virtual calls. Untyped JS falls back to standard dynamic dispatch.

**Key difference**: BCGen serializes to `.hbc` (bytecode for the interpreter); BCGen/SH emits C code that is compiled by `clang` to a standalone native binary. Both paths share every stage before that final split.

---

## Q2: Runtime::drainJobs call chain
Tools used: **Grep** (Runtime.cpp:2005–2031, Runtime.h), **Grep** (Callable.cpp:299–305) — these are exact code lookups. GBrain returned no useful results for this specific call chain; raw grep was clearly the right tool.

Answer:

`Runtime::drainJobs()` is defined at [lib/VM/Runtime.cpp:2005](lib/VM/Runtime.cpp#L2005). Call order:

1. **Pre-drain callbacks** (`drainJobsCallbacks_`): Iterates `drainJobsCallbacks_` (a `std::vector<std::function<void()>>`), calling each registered callback. These are used for NAPI finalizer draining and similar pre-microtask cleanup.

2. **Creates a `GCScope`** and a `MutableHandle<Callable> job` to protect the current job from GC during execution.

3. **Main drain loop** — while `jobQueue_` (a `std::deque<Callable *>`) is non-empty:
   - Creates a `GCScopeMarkerRAII marker` to flush handle allocations each iteration
   - Pops the front job from `jobQueue_` into `job`
   - Calls **`Callable::executeCall0(job, *this, Runtime::getUndefinedValue())`** — invokes the job as a thunk (zero arguments, `this = undefined`)
   - If `executeCall0` returns `ExecutionStatus::EXCEPTION`, returns immediately with `EXCEPTION` (the exceptional job has been popped; re-invocation picks up from the next job)

4. Returns `ExecutionStatus::RETURNED` when the queue is empty.

`Callable::executeCall0` ([lib/VM/Callable.cpp:301](lib/VM/Callable.cpp#L301)) is a convenience wrapper that recursively invokes the interpreter to execute a `Callable` with no arguments. New jobs may be enqueued during the drain (the comment at line 2013 calls this out explicitly) — the `while (!jobQueue_.empty())` loop handles this correctly by re-checking after each job.

Summary call chain:
```
Runtime::drainJobs()
  → drainJobsCallbacks_[i]()            (each registered pre-drain cb)
  → Callable::executeCall0(job, ...)    (per microtask, recursive interpreter call)
      → interpreter dispatch (re-entrant)
```

---

## Q3: GC-safe coding
Tools used: **Wiki (hermes-gc-safety.md)** — primary; GBrain surface search confirmed it held the page `gcsafecoding`; wiki had the full compiled article. This is a conceptual/design question so the wiki (synthesized from multiple raw sources) gave the best structured answer.

Answer:

### What it is

GC-safe coding refers to the discipline of ensuring that all live C++ pointers to GC-managed (heap) objects are stored in **GC-known root locations** before any GC safepoint, and reloaded from those locations after it. The Hermes GC (Hades) is a *moving* collector — it can relocate objects to new addresses during collection. Any raw C++ pointer to a GC object held across a safepoint becomes a dangling pointer.

### Why it matters

A GC safepoint is any point where an allocation can occur — directly or transitively. Functions taking `Runtime &` can reach safepoints unless suffixed `_noalloc` or `_nogc`. Functions with `_RJS` suffix (Recursive JavaScript) definitely reach safepoints — they invoke JavaScript code. Without discipline, a pointer obtained before a call to an `_RJS` function silently refers to freed or moved memory after the call returns.

### The mechanism

The GC's root set has four categories:
- **Register stack** — JS call stack, scanned automatically
- **`Locals` structs** — C++ stack structs registered via `LocalsRAII`
- **`GCScope` chains** — legacy dynamically allocated `PinnedHermesValue` slots
- **Runtime fields** — `PinnedHermesValue` fields inside `Runtime` itself

### Preferred API: `Locals` + `PinnedValue<T>`

All new C++ code must use `Locals` + `PinnedValue<T>` + `LocalsRAII` instead of the legacy `GCScope` + `Handle` pattern:

```cpp
struct : public Locals {
  PinnedValue<JSObject> obj;
  PinnedValue<StringPrimitive> str;
} lv;
LocalsRAII lraii(runtime, &lv);
// lv.obj survives any GC safepoint
```

`LocalsRAII` pushes the struct onto `runtime.vmLocals`; `Runtime::markRoots()` walks this list during GC and updates every `PinnedHermesValue`.

`PseudoHandle<T>` is the unsafe counterpart — it holds a GC value *without* protecting it, is move-only, and must be rooted into a `PinnedValue` before any safepoint. Using it after a safepoint is a bug.

### Detection tool

`HERMESVM_SANITIZE_HANDLES` (enabled by default in ASAN builds) moves the heap after *every* allocation, turning silent dangling-pointer bugs into deterministic crashes.

---

## Q4: Commit d8001980e
Tools used: **git show d8001980e** — the only correct tool for a specific commit hash. No semantic search needed; git is authoritative.

Answer:

**Commit**: `d8001980e852cd81b5f84927b8c14f6fa68d8345`
**Author**: Gang Zhao (Meta)
**Date**: Fri May 22 2026
**Title**: "Set microtask queue true by default"

### What changed

1. **`public/hermes/Public/RuntimeConfig.h`**: Changed `MicrotaskQueue` default from `false` to `true` in the `RuntimeConfig` macro table.

2. **`lib/VM/Runtime.cpp`**: Added a `hermes_assert` at `Runtime` construction time that fires if `MicrotaskQueue` is `false` — setting it to false is no longer supported, not merely a different mode.

3. **`tools/hdb/hdb.cpp`** and **`unittests/API/APITestFactory.cpp`**, **`unittests/API/SynthTraceTest.cpp`**: Removed now-redundant `withMicrotaskQueue(true)` call sites that were explicitly opting in.

4. **Three test files** (`function-tostring-builtins-microtask.js`, `promise-jobs-scheduled-in-scripts.js`, `promise-jobs-scheduled-in-tasks.js`): Removed dual `DISABLED`/`ENABLED` check prefixes and the `-Xmicrotask-queue=0` test runs, collapsing each to a single run that assumes the microtask queue is always enabled.

### Impact

- **Behavioral**: Every `Runtime` instance now always uses the microtask queue. There is no longer a path to disable it.
- **API**: Calling `withMicrotaskQueue(false)` on `RuntimeConfig` now triggers a runtime assertion failure at construction.
- **Correctness enforcement**: The `Promise.toString().includes("[native code]")` heuristic used by some React Native versions to detect microtask-queue presence now always returns `true` — aligning with modern React Native expectations.
- **Net change**: 8 files, −57 +28 lines (mostly test simplification from removing the disabled-mode branches).

This commit reflects the decision that the `static_h` branch has stabilized on microtask-queue-enabled semantics and that the pre-microtask-queue mode is no longer a supported configuration.

---

## Q5: Source to execution pipeline
Tools used: **Wiki (hermes-compiler-pipeline.md)** — primary for compilation; **Wiki (hermes-architecture.md)** — for runtime execution path; **Grep** (Callable.cpp) — to verify executeCall0 is the interpreter re-entry point. The wiki articles gave a complete picture of all stages; grep confirmed the final runtime dispatch step.

Answer:

A JavaScript function travels through these stages from source to execution in Hermes:

### Stage 1: Parsing

The `JSParser` runs in one of three passes:
- **PreParse**: Scans token stream, records function boundary offsets into `PreParsedData`. No AST nodes. O(N).
- **FullParse** (AOT): Builds a complete ESTree AST for the entire source (used by `hermesc`/`shermes`).
- **LazyParse** (runtime): On first call to a lazy function, parses only that function's body using the `PreParsedData` offset.

### Stage 2: SemanticResolver

Resolves variable bindings, scope chains, and closure captures into an annotated AST. Builds binding tables. If `-typed` is enabled, `FlowChecker` runs here to type-check Flow annotations.

### Stage 3: IRGen → HermesIR

Generates SSA-form HermesIR from the annotated AST. Each JS function becomes an IR `Function`. Closures reference `Environment` objects linked as parent chains. Lazy functions get `LazyCompilationDataInst` stubs instead of real IR.

### Stage 4: Optimizer

Pass-based optimization over HermesIR:
- Function passes: type inference, inlining, DCE, CSE, constant folding
- Module passes: cross-function analysis

### Stage 5: Register Allocation

Linear-scan register allocation performed *on the IR* (pre-lowering, unusual among compilers). Assigns virtual registers; introduces `MOV` for copies/spills. Arguments at call sites get consecutive register placement.

### Stage 6: Code Generation (diverges here)

**hermesc path**: BCGen lowers IR to Hermes bytecode (`.hbc`), a variable-length, register-based format. Bytecodes are serialized with a string table (greedy superstring packing), function table, and exception handler table.

**shermes path**: BCGen/SH emits C code → `clang` compiles to a native binary (ELF/Mach-O/PE).

### Stage 7: Execution

**Interpreter path**: `Runtime` loads the `.hbc` via `RuntimeModule` (dynamic wrapper) under a `Domain` (GC-managed lifetime owner). Each function body is represented as a `CodeBlock`. When called:
- `Callable::executeCall0` (or `executeCall1` etc.) sets up a new JS stack frame
- The interpreter dispatches bytecodes in a main loop (switch/computed-goto)
- If JIT is enabled and the function is hot, it is compiled to native code in-place

**AOT native path**: The function is already native machine code; the `static_h` runtime is linked in, and the function executes directly with no interpreter.

**Lazy compilation shortcut**: First call to a lazy function triggers `CodeBlock::lazyCompile` → `hbc::compileLazyFunction` → re-runs SemanticResolver + IRGen + BCGen for that function only, modifies the `BCProviderFromSrc` in place, then proceeds to interpretation.

---

## Summary

### Which tools were used most and why

**Wiki (compiled articles)** was used for Q1, Q3, and Q5 — all questions asking for conceptual/architectural understanding. The wiki articles are pre-synthesized from multiple raw sources and organized for exactly this kind of question. They saved several grep/read cycles.

**Grep on raw source** was used for Q2 — a precise "what does function X call internally" question. No amount of semantic indexing beats reading the 26 lines of actual code. GBrain returned nothing useful here.

**git show** was used for Q4 — a commit hash is authoritative ground truth. Git is the only correct tool.

**GBrain search** was used as a routing layer — it reliably surfaced *which* wiki page or summary to read, but rarely provided the answer directly in its snippet output. Its value was navigation, not content.

### Which questions each tool naturally suited best

| Question | Best tool | Why |
|----------|-----------|-----|
| Q1 — Two execution paths | Wiki (hermes-architecture.md) | Pre-compiled architectural overview, exact section |
| Q2 — drainJobs call chain | Grep + Read (Runtime.cpp) | Exact call sequence; 26 lines of code |
| Q3 — GC-safe coding | Wiki (hermes-gc-safety.md) | Synthesized from 3 raw sources; well-structured |
| Q4 — Commit d8001980e | git show | Commit hash → authoritative diff |
| Q5 — Source to execution | Wiki (compiler-pipeline.md + architecture.md) | Multi-stage pipeline; wiki stitched it together |

### Where the context system added clear value over just searching raw files

- **Q1**: The wiki's "Dual Execution Paths" section gave an immediately usable, cross-referenced answer. Raw grep for "execution path" across 835 C++ files would have required significant synthesis effort.
- **Q3**: The GC-safe coding wiki compiled three separate raw source documents (gc-safe-coding.md, vm-overview.md, hades-gc.md) into a single coherent reference. Reading those three files separately would have taken longer with lower coherence.
- **Q5**: The compiler pipeline wiki mapped all stages cleanly. The raw source is spread across Parser/, lib/IR/, lib/BCGen/, lib/VM/ with no single entry point.

### Where it failed or added nothing

- **Q2**: GBrain search for "drainJobs" returned no results. The `summaries/runtime` page in GBrain did not surface the exact call chain. Grep on Runtime.cpp was faster and more accurate than any semantic search.
- **Q4**: GBrain and wiki are irrelevant for a specific commit hash — git is authoritative and instant.
- **GBrain `query` mode**: Returned "No results" for several natural-language queries (drainJobs call chain, interpreter vs static hermes paths). The `search` mode worked better for routing to wiki pages but still rarely surfaced the answer in the snippet itself. GBrain's primary value here was as a page-discovery layer, not a direct answer layer.
