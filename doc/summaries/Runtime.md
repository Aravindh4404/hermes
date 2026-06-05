# Runtime.cpp — LSP Summary

**File:** `lib/VM/Runtime.cpp`  
**Module:** `lib/VM/` (Virtual Machine)  
**Generated:** LSP documentSymbol + outgoingCalls + incomingCalls

---

## What This File Does

`Runtime.cpp` implements the central `Runtime` class — the top-level object that owns and coordinates every component of a running Hermes VM instance: the heap, the register stack, the symbol table, predefined strings, built-in objects, JIT context, and the microtask queue. It is constructed once per JS execution context and destroyed when that context ends. Nearly every subsystem in the VM holds a pointer or reference back to `Runtime`.

---

## Key Classes Defined Here

| Class | Description |
|---|---|
| `Runtime::StackRuntime` | RAII helper that allocates a `Runtime` on a dedicated OS thread with its own stack. Holds `startup_` / `shutdown_` promises so the calling thread can await readiness. |
| `Runtime::MarkRootsPhaseTimer` | RAII timer that measures the time spent in each GC root-marking phase and records it in the runtime's stats. |

---

## Key Functions

### `Runtime::Runtime` (constructor) — Line 276
Creates the full runtime. Calls:
- `HadesGC` constructor (`lib/VM/gcs/HadesGC.cpp:1264`) — allocates the GC
- `RuntimeModule::createUninitialized` / `initializeWithoutCJSModulesMayAllocate` — bootstraps the special-runtime bytecode module
- `Runtime::generateSpecialRuntimeBytecode` — creates internal bytecode stubs
- `Runtime::initPredefinedStrings`, `initCharacterStrings`, `initNativeBuiltins`, `initJSBuiltins` — populates the identifier table and global object
- `Runtime::runInternalJavaScript` — executes the JS standard library polyfills
- `JSObject::create`, `HiddenClass::createRoot` — builds the prototype chain roots
- `SamplingProfiler::create` — optional profiling setup

**Who calls it:** `Runtime::StackRuntime::StackRuntime` (line 118) and `Runtime::create` (line 156).

---

### `Runtime::run` — Line 1075 / 1090
Compiles a JS source string or `Buffer` and executes it. Delegates to `runBytecode`.

### `Runtime::runBytecode` — Line 1114
Main execution entry point for already-compiled bytecode. Creates a `RuntimeModule`, sets up the global call frame, and dispatches to `interpretFunctionWithRandomStack`.

### `Runtime::drainJobs` — Line 2005
Drains the Promise microtask queue (when `hasMicrotaskQueue_` is true). Called after every top-level JS evaluation.

### `Runtime::markRoots` — Line 580
Called by the GC to enumerate all GC roots (stack frames, predefined values, symbol table, builtins, cached property accesses). Uses `RootAcceptorWithNames`.

### `Runtime::markWeakRoots` — Line 771
Enumerates weak GC references (WeakRef slots, WeakMap entries, finalizer registries).

### `Runtime::raiseTypeError` / `raiseRangeError` / etc. — Lines 1480–1557
Convenience helpers that allocate a JS `Error` object and set the pending exception on the runtime.

### `Runtime::crashCallback` — Line 2178
Registered with `CrashManager::registerCallback`. On crash, walks the call stack and writes a JSON call trace.

---

## Who Calls Into This File (Incoming Calls)

| Caller | Call site |
|---|---|
| `Runtime::StackRuntime::StackRuntime` | `lib/VM/Runtime.cpp:121` — creates Runtime on a separate thread |
| `Runtime::create` | `lib/VM/Runtime.cpp:175` — public factory used by `HermesRuntimeImpl` |
| `HermesRuntimeImpl::HermesRuntimeImpl` | `API/hermes/hermes.cpp:268` — top-level JSI Runtime creation |

---

## Related Files in the Same Module

| File | Relationship |
|---|---|
| `include/hermes/VM/Runtime.h` | Header: declares `Runtime`, `RuntimeConfig`, stack helpers |
| `lib/VM/Interpreter.cpp` | Contains `Runtime::interpretFunctionImpl` / `interpretFunction` |
| `lib/VM/gcs/HadesGC.cpp` | GC instantiated inside `Runtime::Runtime` constructor |
| `lib/VM/JSLib/GlobalObject.cpp` | `initGlobalObject` called from constructor |
| `lib/VM/RuntimeModule.cpp` | Bytecode module management called from `runBytecode` |
| `lib/VM/Domain.cpp` | `Domain::create` called in constructor |
