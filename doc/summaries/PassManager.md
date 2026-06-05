# PassManager.cpp — LSP Summary

**File:** `lib/Optimizer/PassManager/PassManager.cpp`  
**Module:** `lib/Optimizer/PassManager/` (IR Optimizer)  
**Generated:** LSP documentSymbol + outgoingCalls + incomingCalls

---

## What This File Does

`PassManager.cpp` implements the IR optimizer's pass orchestration layer. It drives two levels of pass execution: per-`Function` passes (running each IR transformation on every function in the module) and per-`Module` passes (module-wide analyses and transforms). It also implements `FixedPointLoopPass` — a meta-pass that repeatedly runs a group of sub-passes until no further changes occur, which is how Hermes achieves iterative optimization (e.g., inlining enabling further constant folding).

---

## Key Classes Defined Here

| Class | Description |
|---|---|
| `PassManager::DynamicInfo` | Runtime state for a pass run: owns the `TimerGroup` for per-pass timing, `deque<Timer>` for individual pass timers, and `dumpBetweenPasses` / `verifyBetweenPasses` flags. |
| `FixedPointLoopPass` | A `Pass` subclass that wraps a `PassSeq` and iterates it until the module hash stops changing (up to `maxIters_` iterations). Used by the optimizer pipeline for fixed-point optimization. |

---

## Key Functions

### `PassManager::run(Function*)` — Line 96
Runs all registered function-level passes on a single `Function`. For each non-lazy function:
1. Calls `Pass::runOnFunction` for each pass in the sequence
2. If `dumpBetweenPasses`, calls `Function::dump` via `IR.cpp`
3. Validates IR integrity if `verifyBetweenPasses`

**Incoming callers:**
- `lib/BCGen/SH/SH.cpp::lowerAllocatedFunctionIR` (line 2690)
- `lib/BCGen/HBC/LoweringPipelines.cpp::lowerAllocatedFunctionIR` (line 126)

---

### `PassManager::run(Module*)` — Line 218
Runs all passes at module scope. Sequence:
1. Asserts the pass list is non-empty
2. For each pass, calls `runPassOnModule`
3. Optionally dumps the module after each pass (debug builds)

**Incoming callers (called from `Pipeline.cpp`):**

| Caller | Location |
|---|---|
| `runFullOptimizationPasses` | `lib/Optimizer/PassManager/Pipeline.cpp:111` |
| `runDebugOptimizationPasses` | `lib/Optimizer/PassManager/Pipeline.cpp:178` |
| `runCustomOptimizationPasses` | `lib/Optimizer/PassManager/Pipeline.cpp:35` |
| `runNativeBackendOptimizationPasses` | `lib/Optimizer/PassManager/Pipeline.cpp:191` |
| `runOptimizationPassesToFixedPoint` | `lib/Optimizer/PassManager/Pipeline.cpp:165` |
| `lowerModuleIR` (SH backend) | `lib/BCGen/SH/SH.cpp:2674` |
| `lowerModuleIR` (HBC backend) | `lib/BCGen/HBC/LoweringPipelines.cpp:89` |

---

### `PassManager::runPassOnModule` — Line 167
Iterates all functions in the module, calling `Pass::runOnFunction` for function-level passes. For module-level passes, calls `Pass::runOnModule`. Handles timer start/stop and dumps.

### `PassManager::addPass` — Line 67
Appends a `Pass*` to the current pass sequence (either the top-level sequence or the current `FixedPointLoopPass` sequence).

### `PassManager::beginFixedPointLoop` / `endFixedPointLoop` — Lines 74 / 81
Push/pop a `FixedPointLoopPass` onto the pass stack, causing subsequent `addPass` calls to add passes inside the loop.

### `FixedPointLoopPass::run` — Line 142
Repeatedly calls `PassManager::run(Module*)` on the wrapped pass sequence. Computes a hash of the module IR after each iteration; stops when the hash stabilises or `maxIters_` is reached.

### `PassManager::getModuleHash` — Line 132
Computes a structural hash of the module IR (function count × instruction count product) used by `FixedPointLoopPass` to detect convergence.

---

## `runFullOptimizationPasses` Pass Sequence (`-O` / `-O3`)

**Source:** `lib/Optimizer/PassManager/Pipeline.cpp:39`

The full pass sequence executed at optimization level `-O`. Passes are listed in order.
Some passes (`Mem2Reg` / `SimpleMem2Reg`) are conditional on the `useLegacyMem2Reg` flag.

```
Phase 1 — Lowering + static require resolution
  LowerGeneratorFunction          lower generator functions to state machines
  InstSimplify                    constant folding + algebraic identities
  ResolveStaticRequire            resolve require() calls to known module IDs
  DCE                             dead code elimination (removes dead frame loads after ResolveStaticRequire)
  LowerBuiltinCallsOptimized      replace known built-in calls with faster intrinsics (once only)

Phase 2 — Frame promotion + scope cleanup (round 1)
  SimplifyCFG                     merge basic blocks, remove unreachable
  SimpleStackPromotion            promote frame-allocated vars to stack slots
  FrameLoadStoreOpts              eliminate redundant frame loads/stores
  Mem2Reg / SimpleMem2Reg         promote stack slots to SSA registers
  SimpleStackPromotion            (second pass — catches new opportunities)
  ScopeElimination                eliminate unused captured-variable scopes

Phase 3 — Function analysis + inlining (round 1)
  FunctionAnalysis                compute per-function attributes (pure, side-effect-free, etc.)
  Inlining                        inline small / hot functions
  DCE                             clean up after inlining
  ObjectMergeNewStores            merge redundant object construction stores
  ObjectStackPromotion            promote short-lived objects to the stack
  TypeInference                   infer types from value flow
  SimpleStackPromotion
  InstSimplify
  DCE
  Mem2Reg / SimpleMem2Reg

Phase 4 — Metro require + inlining (round 2)
  FunctionAnalysis
  MetroRequire                    handle Metro bundler require() semantics
  Inlining                        (second inlining round — Metro may enable new sites)
  DCE
  SimpleStackPromotion
  FrameLoadStoreOpts
  Mem2Reg / SimpleMem2Reg
  ScopeElimination
  FunctionAnalysis
  ScopeHoisting                   hoist closed-over variables to outer scopes
  ObjectStackPromotion

Phase 5 — Type-aware optimisations
  TypeInference                   (second pass)
  CSE                             common subexpression elimination
  PrivateBrandCheckDedup          deduplicate repeated private brand checks
  TDZDedup                        deduplicate temporal dead zone checks
  SimplifyCFG
  InstSimplify
  FuncSigOpts                     specialise call sites to known function signatures
  DCE
  SimplifyCFG
  FrameLoadStoreOpts
  Mem2Reg / SimpleMem2Reg
  Auditor                         (debug verification pass — no-op in release)
  TypeInference                   (final pass — feeds BCGen type annotations)
```

**Key design pattern:** DCE + Mem2Reg appear after every major transform phase because inlining, constant folding, and type inference all create dead code and redundant loads. Running them early enables subsequent passes to see a cleaner graph.

---

## Related Files in the Same Module

| File | Relationship |
|---|---|
| `include/hermes/Optimizer/PassManager/PassManager.h` | Class declaration, `PassSeq` type, `getName` |
| `include/hermes/Optimizer/PassManager/Pass.h` | `Pass` base class with `runOnFunction` / `runOnModule` virtual methods |
| `lib/Optimizer/PassManager/Pipeline.cpp` | Pre-built pass pipelines (`runFullOptimizationPasses`, etc.) that call `PassManager::run(Module*)` |
| `lib/BCGen/HBC/LoweringPipelines.cpp` | HBC-specific lowering pipelines; calls both `run(Function*)` and `run(Module*)` |
| `lib/BCGen/SH/SH.cpp` | Static Hermes backend; calls `run(Function*)` and `run(Module*)` |
| `lib/Optimizer/` | Contains all individual passes (Inliner, SimplifyCFG, TypeInference, etc.) |
