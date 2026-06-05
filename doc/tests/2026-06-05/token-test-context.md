# Token Usage Test — Full Context System
Date: 2026-06-05
Method: All tools available — Claude chooses best tool
GBrain pages: 68
Wiki articles: 11
Learnings: 22

## Q1: Two execution paths in Hermes
Tool chosen: Wiki (hermes-architecture.md) — architectural overview question with no specific symbol; wiki article has a dedicated "Dual Execution Paths" section compiled from multiple raw sources, zero traversal cost
Tool calls made: 1 (Read wiki article)
Estimated tokens consumed: ~1,100 (article is ~3.5 KB, plus overhead)
Answer:
Hermes has two distinct execution paths that share the same front-end (parser, IR, optimizer) but diverge at code generation:

**Path 1 — hermesc → bytecode → interpreter (+ optional JIT)**
The production path used by React Native today:
```
Source (.js) → Parser → AST → SemanticResolver → HermesIR (SSA) → Optimizer → BCGen → .hbc bytecode
```
At runtime, the `Runtime` object interprets `.hbc` bytecode. If the JIT is enabled (`-Xjit`), hot functions are compiled to native code by a baseline JIT.

**Path 2 — shermes → native binary (AOT)**
The Static Hermes path (this branch: `static_h`), producing self-contained native executables:
```
Source (.js/.ts/.flow) → Parser → [FlowChecker] → HermesIR → Optimizer → BCGen/SH → C code → clang → ELF/Mach-O/PE
```
Typed Flow annotations (`-typed`) unlock exact object layouts, unboxed primitives, and inlined virtual calls. Untyped JS falls back to standard dynamic dispatch.

Key difference: the two paths are identical through the optimizer. Only the final code generator differs — HBC serialization (Path 1) vs C code emission (Path 2).

---

## Q2: Runtime::drainJobs call chain
Tool chosen: Grep (lib/VM/Runtime.cpp) + Read — exact symbol known, direct code inspection needed for call order
Tool calls made: 2 (Grep for drainJobs across lib/VM, Read Runtime.cpp:2005-2031)
Estimated tokens consumed: ~1,800 (grep output ~80 lines context + 50-line function read)
Answer:
`Runtime::drainJobs()` (lib/VM/Runtime.cpp:2005) executes in this order:

1. **drainJobsCallbacks_ loop** — Iterates `drainJobsCallbacks_` and calls each registered callback. These are pre-drain hooks, e.g., NAPI finalizer draining, registered by embedders.

2. **GCScope + MutableHandle setup** — Creates a `GCScope` and a `MutableHandle<Callable> job` to hold the current job safely across GC safepoints.

3. **jobQueue_ drain loop** — While `jobQueue_` is non-empty:
   - Creates a `GCScopeMarkerRAII marker` (flushes temporary handles each iteration)
   - Pops the front job: `job = jobQueue_.front(); jobQueue_.pop_front()`
   - Calls **`Callable::executeCall0(job, *this, Runtime::getUndefinedValue())`** — invokes the job as a zero-argument thunk with `undefined` as the receiver
   - If `executeCall0` returns `ExecutionStatus::EXCEPTION`, returns immediately with `EXCEPTION` (exceptional job is already popped; re-invocation continues from the next job)

4. Returns `ExecutionStatus::RETURNED` when queue is empty.

Note: new jobs can be enqueued during draining — the `while (!jobQueue_.empty())` check handles this correctly.

---

## Q3: GC-safe coding
Tool chosen: Wiki (hermes-gc-safety.md) — dedicated concept article, highly synthesized, covers all patterns in one place
Tool calls made: 1 (Read wiki article, loaded in parallel with Q1/Q5)
Estimated tokens consumed: ~1,400 (article is ~4.5 KB)
Answer:
GC-safe coding is the discipline of ensuring no C++ code holds a raw (unrooted) pointer to a GC-managed heap object across a GC safepoint.

**Why it matters**: Hermes uses Hades, a concurrent, moving garbage collector. The GC can relocate objects to new addresses during collection. Any C++ raw pointer held across a collection becomes a dangling pointer — causing crashes or silent data corruption.

**The fundamental rule**: All live GC pointers used by C++ MUST be stored in GC-known root locations before a safepoint, and reloaded from those locations after.

**GC safepoints** are any allocation or call that can transitively reach an allocation. Naming conventions signal risk:
- Functions taking `Runtime &`: can reach safepoints (unless suffixed `_noalloc`/`_nogc`)
- `_RJS` suffix: definitely invokes JS recursively, definitely hits safepoints

**The preferred API (all new code)**:
```cpp
struct : public Locals {
  PinnedValue<JSObject> obj;
  PinnedValue<StringPrimitive> str;
} lv;
LocalsRAII lraii(runtime, &lv);
```
`LocalsRAII` pushes the struct onto `runtime.vmLocals`. `Runtime::markRoots()` walks this list during GC and updates every `PinnedHermesValue` field.

**Why `Locals + PinnedValue` beats legacy `GCScope + Handle`**:
- No dynamic allocation (fields are fixed in the struct)
- No 48-handle limit
- No unbounded growth in loops

**`PseudoHandle<T>`** holds a value WITHOUT protecting it — it is move-only and must be immediately stored into a `PinnedValue`. Using a `PseudoHandle` after a safepoint is a bug.

**Detection**: Build with `HERMESVM_SANITIZE_HANDLES` (default in ASAN builds) — moves the heap after every allocation, making dangling pointer bugs deterministic.

---

## Q4: Commit d8001980e
Tool chosen: git show — exact commit hash given, git is the authoritative source
Tool calls made: 2 (git show --stat, git show --patch for key files)
Estimated tokens consumed: ~600 (stat output + targeted diff, ~80 lines total)
Answer:
**Commit**: `d8001980e` — "Set microtask queue true by default"
**Author**: Gang Zhao | **Date**: 2026-05-22

**What changed**:
1. `public/hermes/Public/RuntimeConfig.h` — Changed `MicrotaskQueue` default from `false` to `true` in the `HERMES_RUNTIME_CONFIG_FIELDS` macro.
2. `lib/VM/Runtime.cpp` — Added a `hermes_assert` in `Runtime::Runtime()` that fires if `MicrotaskQueue` is `false`, with message: *"MicrotaskQueue must be enabled. Setting it to false is no longer supported."*
3. Removed now-redundant `withMicrotaskQueue(true)` call-sites throughout `static_h` branch code.
4. Updated 3 test files (`function-tostring-builtins-microtask.js`, `promise-jobs-scheduled-in-scripts.js`, `promise-jobs-scheduled-in-tasks.js`) and 2 unittest files (`APITestFactory.cpp`, `SynthTraceTest.cpp`) to remove the now-superfluous explicit opt-in.
5. Removed from `tools/hdb/hdb.cpp`.

**Impact**: Microtask queue (Promise `.then()` jobs, queueMicrotask) is now always enabled in `static_h`. Attempting to disable it at runtime is a hard error. Net result: 57 lines removed, 28 added — a cleanup that removes boilerplate across the codebase while enforcing the invariant that Promise/microtask behavior is always active on this branch.

---

## Q5: Source to execution pipeline
Tool chosen: Wiki (hermes-compiler-pipeline.md) — comprehensive pipeline article with exact stage-by-stage breakdown
Tool calls made: 1 (Read wiki article, loaded in parallel)
Estimated tokens consumed: ~1,500 (article is ~5 KB)
Answer:
A JavaScript function travels through 6 stages from source to execution:

**1. Parser** (three modes):
- `PreParse`: scans token stream, records function boundary offsets into `PreParsedData`. No AST allocated. O(N). Used for lazy-compiled modules.
- `LazyParse`: parses only a specific function body (by offset from `PreParsedData`) when it's first called at runtime.
- `FullParse`: parses entire source into a complete ESTree AST. Used for AOT compilation (`hermesc`/`shermes`).

**2. SemanticResolver**: Resolves variables, scopes, and binding tables → annotated AST. Optionally: FlowChecker runs here if `-typed` is active.

**3. IRGen**: Generates HermesIR (SSA form) from the annotated AST. Closures are captured in heap-allocated `Environment` objects. Lazy functions get `LazyCompilationDataInst` stubs. For CJS modules, the file is wrapped in a function binding `exports, require, module`.

**4. Optimizer**: Pass-based system of function passes and module passes. Performs canonicalization (re-association, inlining), simplification (DCE, CSE, constant folding), and lowering (loop strength reduction). Cycles between canonicalization and simplification until stable.

**5. Register Allocator**: Linear-scan allocation **on the IR** (not post-lowering). Computes liveness intervals, assigns registers greedily, enforces consecutive placement for call arguments.

**6. BCGen (or BCGen/SH)**:
- `hermesc` path: lowers IR to variable-length Hermes bytecode (`.hbc`), serialized with file header, function table, string table, and function bodies.
- `shermes` path: emits C code → compiled by clang → native binary.

**At runtime**: The `Runtime` object interprets `.hbc` bytecode. For lazy functions, the first call triggers `CodeBlock::lazyCompile` → re-runs SemanticResolver + IRGen + BCGen for just that function. If JIT is enabled, hot `CodeBlock`s are compiled to native code.

---

## Token Summary Table
| Q | Tool Used | Tool Calls | Est. Tokens | vs Raw |
|---|-----------|------------|-------------|--------|
| Q1 | Wiki (hermes-architecture.md) | 1 | ~1,100 | ~60% saved vs grep+read 5 files |
| Q2 | Grep + Read (Runtime.cpp) | 2 | ~1,800 | ~40% saved vs reading full file |
| Q3 | Wiki (hermes-gc-safety.md) | 1 | ~1,400 | ~65% saved vs grep+read 3 files |
| Q4 | git show | 2 | ~600 | N/A (only authoritative source) |
| Q5 | Wiki (hermes-compiler-pipeline.md) | 1 | ~1,500 | ~70% saved vs grepping 6+ source files |
| TOTAL | Mix: Wiki 3x, Git 2x, Grep+Read 2x | 7 | ~6,400 | ~55% avg saved |

---

## Overall Assessment
Total tool calls: 7 (across 5 questions)
Total estimated tokens: ~6,400
Token reduction vs raw search: ~8,000 tokens saved (~55%)
Which tool saved the most tokens: **Wiki** — each read replaced 3–6 targeted grep+read operations that would have needed stitching across multiple files. A single wiki article for Q3 (GC safety) replaced reading gc-safe-coding.md raw source + vm-overview.md + hades-gc.md separately.
Which questions benefited most from context: **Q1, Q3, Q5** — all architectural/conceptual questions where the wiki had pre-synthesized, cross-referenced answers that would otherwise require reading 3–5 source files each.
Where context system added no token benefit: **Q4** — commit questions are always answered directly by `git show`. No amount of pre-indexed context can replace the authoritative source. **Q2** also needed direct code reading since the exact call chain required seeing the real implementation, not a summary.
What this means for large codebase usage: For semantic/conceptual questions ("how does X work", "what is Y", "what are the paths"), a well-maintained wiki cuts tool calls by 3–6x and token costs by 50–70%. For precise implementation questions (call chains, exact function signatures, commit contents), raw tools (grep, git, file read) remain necessary and irreplaceable. The optimal strategy is: use wiki/gbrain first for orientation, then use grep/read for exact details the wiki summarizes away. The context system pays off most when questions span multiple source files — it pre-does the synthesis work.
