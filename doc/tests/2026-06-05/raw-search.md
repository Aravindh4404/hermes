# Context System Test — Raw Search
Date: 2026-06-05
Method: Raw file search only (Grep, Glob, Read, Bash)
GBrain: NO
Wiki: NO
Learnings: NO

---

## Q1: Two execution paths in Hermes
Files read:
- `doc/blog/2025-11-02-hermes-compilation-runtime-modes.md`
- `doc/StaticHermes.md`
- `doc/summaries/Interpreter.md`

Answer:

Hermes has two primary execution paths: the **bytecode interpreter path** and the **native (AOT) execution path**.

**1. Bytecode interpreter path** (`hermes` / React Native runtime):
Source JS is compiled to Hermes Bytecode (HBC) — either ahead-of-time via `hermesc`, or lazily on the first call to each function. At runtime, `Runtime::run` calls `hbc::createBCProviderFromSrc` which runs the JS through the Parser → Sema → IRGen → Optimizer → BytecodeGenerator pipeline to produce a `BytecodeModule`. That module is handed to `Runtime::runBytecode`, which sets up a call frame and enters `Interpreter::interpretFunction` — a giant template dispatch loop (~170 opcodes, computed-goto or switch) that fetches and executes HBC instructions one at a time. If the JIT is enabled and a function becomes hot enough, the baseline JIT can compile its bytecode to native code without leaving the interpreter path conceptually.

**2. Native (AOT) execution path** — Static Hermes (`shermes`):
The `shermes` AOT compiler runs the same pipeline (Parser → Sema/FlowChecker → IRGen → Optimizer) but instead of producing bytecode it runs BCGen/SH to emit C code, which is then compiled by `clang` into a native binary. The resulting binary runs without a Hermes VM or interpreter — there is no dispatch loop and no bytecode at runtime. Typed mode (`-typed`) enables tighter C code with exact object layouts, no hidden-class polymorphism, and unboxed primitives.

**Key difference:** The interpreter path loads and dispatches instructions at runtime (dynamic, flexible, supports lazy compilation and JIT). The native path compiles all the way to machine code ahead of time (faster startup and peak performance, but requires a full AOT build step and cannot execute dynamically generated code the same way).

The blog post (`2025-11-02-hermes-compilation-runtime-modes.md`) explicitly notes all modes can coexist in one app: some code compiled to native, some to bytecode, some loaded as source — with JIT promoting bytecode to native for hot paths.

---

## Q2: Runtime::drainJobs call chain
Files read:
- `lib/VM/Runtime.cpp` (Grep, lines ~2005–2031)
- `doc/summaries/Runtime.md`

Answer:

`Runtime::drainJobs` (defined at `lib/VM/Runtime.cpp:2005`) drains the Promise microtask queue. Its call chain in order:

1. **`drainJobsCallbacks_` loop** — iterates over any registered drain callbacks (e.g. NAPI finalizer draining) and calls each one before touching the job queue.

2. **GCScope + MutableHandle setup** — creates a `GCScope` and a `MutableHandle<Callable> job` to safely hold each queued job across iterations.

3. **Job queue drain loop** — while `jobQueue_` is not empty:
   a. Creates a `GCScopeMarkerRAII` to flush temporary GC handles at the end of each iteration.
   b. Reads the front entry into `job` and pops it from the deque.
   c. Calls **`Callable::executeCall0(job, *this, Runtime::getUndefinedValue())`** — dispatches the job as a zero-argument thunk, passing `undefined` as the `this` value.
   d. If the call returns `ExecutionStatus::EXCEPTION`, returns `EXCEPTION` immediately (the faulting job is already popped, so re-invoking `drainJobs` picks up from the next job).

4. Returns `ExecutionStatus::RETURNED` when the queue is exhausted.

New jobs enqueued during draining are processed in the same pass because the loop re-checks `jobQueue_` each iteration.

`drainJobs` is called from `HermesRuntimeImpl::drainMicrotasks` (`API/hermes/hermes.cpp:2227`) after every top-level JS evaluation.

---

## Q3: GC-safe coding
Files read:
- `doc/GCSafeCoding.md`

Answer:

GC-safe coding in Hermes is the set of rules C++ code must follow to remain correct in the presence of a garbage collector that can **free** or **move** heap objects at any GC safepoint.

**Why it matters:** If C++ code holds a raw pointer to a GC-managed object and that pointer is not stored in a GC root, two things can go wrong at a safepoint: (1) the GC may consider the object unreachable and free it (use-after-free), or (2) the GC may move it for heap compaction (dangling/stale pointer). These bugs are rare and hard to reproduce because compaction is infrequent — they typically only surface under `HERMESVM_SANITIZE_HANDLES` (enabled in ASAN builds), which deliberately moves the heap after every allocation.

**GC safepoints** occur at: any allocation, and any call that might transitively reach an allocation. Naming conventions encode this: functions taking `Runtime &` or `PointerBase &` are assumed to be safepoints; functions with `_RJS` suffix definitely invoke JS recursively (guaranteed safepoints); functions with `_noalloc`/`_nogc` are explicitly safe.

**Key types for GC-safe code:**

| Type | Role |
|---|---|
| `PinnedHermesValue` | A GC root: a `HermesValue` stored outside the heap in a location the GC knows about |
| `Handle<T>` | Immutable pointer to a `PinnedHermesValue`; the standard way to pass GC values between functions |
| `MutableHandle<T>` | Mutable version of `Handle<T>` |
| `PseudoHandle<T>` | Unrooted GC value (NOT safe across safepoints); move-only; must be stored into a root before the next safepoint |
| `GCScope` (legacy) | RAII dynamic container of `PinnedHermesValue` slots; still used for APIs that return `Handle<>` |
| `Locals` + `PinnedValue<T>` | **Preferred for new code**: fixed-size struct of `PinnedHermesValue` fields registered with the GC via `LocalsRAII` — no dynamic allocation, no loop-growth hazard |

**The fundamental rule:** All live pointers to GC objects must be stored in roots (via `PinnedValue`, `GCScope`/`Handle`, or `MutableHandle`) before any GC safepoint, and reloaded from those roots after.

Common mistakes: holding raw pointers or `PseudoHandle`s across allocations, accumulating handles in a loop without `GCScopeMarkerRAII`, returning a `Handle` from a destroyed `GCScope` or `Locals`.

---

## Q4: Commit d8001980e
Files read:
- `public/hermes/Public/RuntimeConfig.h` (via `git show`)
- `lib/VM/Runtime.cpp` (via `git show`)
- `tools/hdb/hdb.cpp` (via `git show`)
- `unittests/API/APITestFactory.cpp` (via `git show`)

Answer:

Commit `d8001980e` ("Set microtask queue true by default", 2026-05-22, Gang Zhao):

**What changed:**

1. **`public/hermes/Public/RuntimeConfig.h`** — changed the `MicrotaskQueue` field default from `false` to `true`:
   ```diff
   -  F(constexpr, bool, MicrotaskQueue, false)
   +  F(constexpr, bool, MicrotaskQueue, true)
   ```

2. **`lib/VM/Runtime.cpp`** — added an assertion in the `Runtime` constructor:
   ```cpp
   hermes_assert(
       runtimeConfig.getMicrotaskQueue(),
       "MicrotaskQueue must be enabled. Setting it to false is no longer supported.");
   ```
   Constructing a Runtime with `MicrotaskQueue=false` now aborts at startup.

3. **Cleanup** — removed now-redundant `.withMicrotaskQueue(true)` call-sites from:
   - `tools/hdb/hdb.cpp`
   - `unittests/API/APITestFactory.cpp`
   - `unittests/API/SynthTraceTest.cpp` (2 sites)

4. **Test updates** — updated 3 test files in `test/hermes/` that previously relied on MicrotaskQueue being off by default.

**Impact:** MicrotaskQueue (Promise microtask draining) is now unconditionally enabled for all Hermes runtimes. The old opt-in pattern `.withMicrotaskQueue(true)` still compiles but is a no-op; `.withMicrotaskQueue(false)` is now a hard error. This simplifies static_h and removes a common source of configuration drift where code forgot to enable microtasks. 8 files changed: 28 insertions, 57 deletions.

---

## Q5: Source to execution pipeline
Files read:
- `lib/BCGen/HBC/BCProviderFromSrc.cpp` (lines 91–260)
- `lib/BCGen/HBC/HBC.cpp` (lines 13–55)
- `lib/VM/Runtime.cpp` (Grep, lines ~1075–1113)
- `doc/summaries/Interpreter.md`
- `doc/summaries/Runtime.md`
- `doc/summaries/hermes.md`
- `doc/StaticHermes.md`

Answer:

**Interpreter path (the standard runtime path for React Native / `hermes` CLI):**

```
JS source string
  │
  ▼ Runtime::run  (lib/VM/Runtime.cpp:1075)
hbc::createBCProviderFromSrc  →  BCProviderFromSrc::create
  │
  ├─ 1. parser::JSParser::parse()
  │      → ESTree AST
  │
  ├─ 2. hermes::transformASTForCompilation()
  │      → normalized AST (desugared)
  │
  ├─ 3. hermes::sema::resolveAST()
  │      → semantic analysis (scope resolution, variable binding)
  │
  ├─ 4. hermes::generateIRFromESTree()
  │      → Hermes IR (Module)
  │
  ├─ 5. runOptimizationPasses()  (if enabled)
  │      → optimized IR
  │
  └─ 6. hbc::generateBytecodeModule()
         → BytecodeModule (HBC bytecode)
  │
  ▼ Runtime::runBytecode  (lib/VM/Runtime.cpp:1114)
     Creates RuntimeModule, sets up global call frame
  │
  ▼ interpretFunctionWithRandomStack  (lib/VM/Runtime.cpp:1063)
     Randomizes stack base for ASLR
  │
  ▼ Runtime::interpretFunction  →  Interpreter::interpretFunction<>
     Core dispatch loop: fetches and executes HBC opcodes one by one
     (~170 opcodes, computed-goto dispatch, inline caches for GetById/PutById)
```

**Static Hermes / AOT native path (`shermes`):**

```
JS/TS/Flow source
  → Parser → Sema/FlowChecker (optional typed mode)
  → IRGen → Optimizer
  → BCGen/SH (emits C code)
  → clang  → native binary
```

No interpreter loop, no bytecode at runtime. The binary is self-contained.

**Lazy compilation variant:** If `compileFlags.lazy` is set and the file is large, `BCProviderFromSrc::create` runs a lightweight pre-parser first and sets `LazyParse` mode — individual functions are compiled to bytecode only when first called, rather than all upfront.

---

## Summary
Total files read across all questions: 14
(doc/blog/2025-11-02-hermes-compilation-runtime-modes.md, doc/StaticHermes.md,
doc/summaries/Interpreter.md, doc/summaries/Runtime.md, doc/summaries/hermes.md,
doc/GCSafeCoding.md, lib/VM/Runtime.cpp, lib/BCGen/HBC/BCProviderFromSrc.cpp,
lib/BCGen/HBC/HBC.cpp, public/hermes/Public/RuntimeConfig.h,
tools/hdb/hdb.cpp, unittests/API/APITestFactory.cpp — via git show,
plus 2 doc summary files re-read across questions)

Hardest question to answer: **Q1 (two execution paths)**. The blog post names four runtime modes and two compilation modes — it's not a clean "two paths" binary. Deciding which two to call out required judgment: interpreter vs native-AOT is the clearest split, but the JIT (bytecode→native inside the runtime) blurs the line. The answer required reading three files and synthesizing across them rather than finding a single authoritative source.

What was missing or unclear:
- **Q1**: No single file said "there are exactly two execution paths." The blog post lists ~6 modes; the Design.md focuses only on bytecode generation; the Interpreter summary only covers the interpreter side. Had to triangulate.
- **Q2**: `Callable::executeCall0` is called but not followed further — the call chain ends there without reading `Callable.cpp`. The full depth (into the interpreter re-entry) was not traced.
- **Q4**: The test file diffs (`test/hermes/*.js`) were not read in detail — the commit message described them as "update tests" which was taken at face value.
- **Q5**: The lazy compilation sub-path (where individual functions compile on first call) is visible in `BCProviderFromSrc.cpp` but the runtime-side mechanism (deferred bytecode generation triggered by a call opcode) was not traced into `Interpreter.cpp`.
