# Interpreter.cpp — LSP Summary

**File:** `lib/VM/Interpreter.cpp`  
**Module:** `lib/VM/` (Virtual Machine)  
**Generated:** LSP documentSymbol

---

## What This File Does

`Interpreter.cpp` implements Hermes's bytecode interpreter — the hot loop that fetches, decodes, and dispatches every HBC instruction during interpreted execution. It contains the main dispatch table for all ~170 opcodes (GetById, PutById, Call, Construct, arithmetic, branches, etc.) along with slow-path helpers invoked when inline caches miss, and generator support. The file is deliberately structured around a single massive template function (`Interpreter::interpretFunction<SingleStep, EnableCrashTrace>`) to enable compile-time specialization for debugger single-step mode and crash tracing.

---

## Key Classes Defined Here

No new classes are defined in this file. It provides implementations of `Runtime` and `Interpreter` methods declared in the headers.

**Performance statistics tracked (HERMES_SLOW_STATISTIC counters):**

| Counter | Measures |
|---|---|
| `NumGetById` | Total property read-by-id accesses |
| `NumGetByIdCacheHits` | Inline cache hits for GetById |
| `NumGetByIdProtoHits` | Prototype cache hits |
| `NumGetByIdDict` | Dictionary-mode property reads |
| `NumGetByIndex` / `NumGetByVal` | Indexed and value-keyed reads |
| `NumPutById` / `NumPutByIdCacheHits` | Property write accesses and cache hits |
| `NumNativeFunctionCalls` | Native function dispatch count |
| `NumBoundFunctionCalls` | Bound function dispatch count |

---

## Key Functions

### `Runtime::interpretFunctionImpl` — Line 393
Sets up the interpreter state (saves/restores IP, validates native stack depth) and forwards to `Interpreter::interpretFunction<>`. This is the main re-entrant entry used from `Runtime::runBytecode`.

### `Runtime::interpretFunction` — Line 453
Thin wrapper around `interpretFunctionImpl` that also checks for interpreter re-entrancy guards.

### `Interpreter::interpretFunction<SingleStep, EnableCrashTrace>` — Line 472
The core dispatch loop (template). Iterates over HBC instructions using a computed-goto or switch dispatch. Contains the inline cache fast paths for `GetById` / `PutById`, type-checked arithmetic, and all control-flow opcodes. Parameterised by `SingleStep` (for CDPDebugger single-step mode) and `EnableCrashTrace`.

### `Interpreter::createGenerator_RJS` — Line 120
Allocates a `JSGeneratorObject` and captures the current execution state for later resumption via `yield`.

### `Interpreter::reifyArgumentsSlowPath` — Line 151
Creates the `arguments` array object when the callee has accessed `arguments` but it hasn't been materialised yet.

### `Interpreter::getArgumentsPropByValSlowPath_RJS` — Line 174
Handles `GetByVal` on the `arguments` object when the index or key isn't a small integer.

### `Interpreter::handleCallSlowPath` — Line 251
Handles function calls when the callee is not a simple `JSFunction` (native functions, bound functions, proxies, etc.).

---

## Who Calls Into This File (Incoming Calls)

`interpretFunctionImpl` / `interpretFunction` are the internal execution entry points; they are called from:

| Caller | Location |
|---|---|
| `Runtime::runBytecode` | `lib/VM/Runtime.cpp:1114` — interprets a loaded bytecode module |
| `interpretFunctionWithRandomStack` | `lib/VM/Runtime.cpp:1063` — entry from `Runtime::run`, sets a random stack base for ASLR |
| `Runtime::loadSegment` | `lib/VM/Runtime.cpp:1229` — loads a lazy segment and re-enters interpreter |

---

## Related Files in the Same Module

| File | Relationship |
|---|---|
| `include/hermes/VM/Interpreter.h` | Declares `Interpreter` and `InterpreterState` |
| `lib/VM/Runtime.cpp` | Owns the `Runtime` that drives the interpreter |
| `lib/VM/JSObject.cpp` | GetById / PutById slow paths call into JSObject property resolution |
| `lib/VM/Operations.cpp` | Arithmetic and comparison slow paths |
| `lib/VM/Callable.cpp` | Function call dispatch |
| `lib/BCGen/HBC/BytecodeGenerator.cpp` | Produces the HBC bytecode consumed here |
