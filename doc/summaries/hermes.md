# hermes.cpp — LSP Summary

**File:** `API/hermes/hermes.cpp`  
**Module:** `API/hermes/` (Public JSI API)  
**Generated:** LSP documentSymbol + outgoingCalls + incomingCalls

---

## What This File Does

`hermes.cpp` is the JSI (JavaScript Interface) adapter — it bridges the Hermes VM (`hermes::vm::Runtime`) to React Native's `jsi::Runtime` abstract interface. `HermesRuntimeImpl` implements every method of `jsi::Runtime` (property access, function calls, string marshalling, ArrayBuffer, WeakObject, Symbols, BigInts, Typed Arrays, etc.) by forwarding to internal VM operations. It also implements `HermesRootAPI` — the factory interface that creates `HermesRuntime` instances and provides bytecode inspection utilities. This is the file that React Native and other embedders link against.

---

## Key Classes Defined Here

| Class | Description |
|---|---|
| `HermesRuntimeImpl` | The full JSI `Runtime` implementation (~3500 lines). Holds a `shared_ptr<vm::Runtime>`, manages `ManagedValues` for rooted HermesValues and WeakRoots, and dispatches all JSI calls to the VM. |
| `HermesRootAPI` | Implements the `ICast` factory interface: `makeHermesRuntime`, bytecode inspection (`isHermesBytecode`, `hermesBytecodeSanityCheck`, `getBytecodeVersion`), profiler control (`enableSamplingProfiler`). |
| `JsiProxy` | `vm::HostObject` adapter: wraps a `jsi::HostObject` so JS code can call `get`/`set` on it; translates between JSI values and HermesValues. |
| `HFContext` | Wraps a `jsi::HostFunctionType` as a native Hermes callable. `func()` calls the host function and converts arguments/return value between JSI and HermesValue. |
| `NativeStateContext` | Stores a `shared_ptr<jsi::NativeState>` alongside a JS object; finalizer deletes the native state when the object is collected. |
| `ManagedValue<T>` | Reference-counted slot (either live or on a freelist) for a `PinnedHermesValue` or `WeakRoot<JSObject>`. |
| `ManagedValues<T>` | Pool of `ManagedValue<T>` slots; registers a custom GC roots callback so the GC scans all live slots. |
| `HermesPreparedJavaScript` | Holds a compiled bytecode buffer, optional source map, and source URL, implementing `jsi::PreparedJavaScript`. |
| `BufferAdapter` | Adapts a `jsi::Buffer` to `hermes::Buffer` for the VM's bytecode consumer. |

---

## Key Functions

### `HermesRuntimeImpl::HermesRuntimeImpl` (constructor) — Line 265
Creates the internal `vm::Runtime` via `Runtime::create`, sets up GC roots callbacks for `hermesValues_` and `weakHermesValues_` pools, and initialises compile flags from `RuntimeConfig`.

**Incoming caller:** `HermesRootAPI::makeHermesRuntime` (line 1447)

---

### `HermesRootAPI::makeHermesRuntime` — Line 1447
Public factory function: constructs `HermesRuntimeImpl(runtimeConfig)`. Called by:
- `makeHermesRuntime` (free function, line 3758) — the primary public entry point
- `makeHermesRuntimeNoThrow` (line 3764)
- `makeThreadSafeHermesRuntime` (line 3773)

---

### `HermesRuntimeImpl::evaluateJavaScript` — Line 734
Evaluates a JS `Buffer` (source or bytecode). Delegates to `evaluatePreparedJavaScript`.

### `HermesRuntimeImpl::evaluatePreparedJavaScript` — Line 2168
Prepares bytecode (compiling from source if needed) and calls `vm::Runtime::runBytecode`.

### `HermesRuntimeImpl::prepareJavaScriptWithSourceMap` — Line 2073
Compiles source to bytecode using `hermesc` pipeline, stores result as `HermesPreparedJavaScript`.

### `HermesRuntimeImpl::call` / `callAsConstructor` — Lines 3244 / 3280
Calls a JS function / constructor from native code. Marshals JSI `Value` args to HermesValues, invokes `vm::Callable::executeCall`, then converts the result back.

### `HermesRuntimeImpl::queueMicrotask` / `drainMicrotasks` — Lines 2215 / 2227
Enqueues a microtask function and drains the microtask queue via `vm::Runtime::drainJobs`.

### `HermesRuntimeImpl::createObject` / `getProperty` / `setPropertyValue` — Lines 2559 / 2779 / 2851
Core JSI object manipulation: delegates to `vm::JSObject` property accessors.

### `makeHermesRuntime` (free function) — Line 3758
The canonical public entry point used by React Native. Constructs `HermesRootAPI` and calls `makeHermesRuntime` on it.

---

## Who Calls Into This File (Incoming Calls)

This file is the external API surface. It is called by:

| Caller | Context |
|---|---|
| React Native JSI layer | Via `makeHermesRuntime` — creates the runtime for RN app execution |
| `tools/hermes/hermes.cpp` | CLI tool creates a `HermesRuntime` for interactive use |
| `unittests/API/` | JSI unit tests |
| `API/jsi/jsi/test/testlib.cpp` | Cross-runtime JSI compliance tests |

---

## Related Files in the Same Module

| File | Relationship |
|---|---|
| `include/hermes/hermes.h` / `API/hermes/hermes.h` | Public header declaring `HermesRuntime`, `makeHermesRuntime` |
| `API/jsi/jsi/jsi.h` | JSI abstract interface implemented by `HermesRuntimeImpl` |
| `lib/VM/Runtime.cpp` | The internal VM runtime wrapped by `HermesRuntimeImpl` |
| `API/hermes/SynthTrace.h` | Trace recording for replay-based testing |
| `API/hermes/HermesRootAPI.h` | `HermesRootAPI` interface declaration |
