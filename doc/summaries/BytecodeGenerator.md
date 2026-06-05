# BytecodeGenerator.cpp — LSP Summary

**File:** `lib/BCGen/HBC/BytecodeGenerator.cpp`  
**Module:** `lib/BCGen/HBC/` (HBC Bytecode Generation)  
**Generated:** LSP documentSymbol + outgoingCalls + incomingCalls

---

## What This File Does

`BytecodeGenerator.cpp` translates Hermes IR (the SSA intermediate representation produced by the optimizer) into HBC bytecode. It operates at two levels: `BytecodeFunctionGenerator` translates one IR `Function` into a `BytecodeFunction` (calling the instruction selector in `ISel.cpp`), while `BytecodeModuleGenerator` orchestrates the full module — collecting strings, assigning function IDs, running the lazy-compilation pipeline, and serializing the final `BytecodeModule`. This is the last compiler stage before bytecode is either executed or written to a `.hbc` file.

---

## Key Classes Defined Here

| Class | Description |
|---|---|
| `FixupEnvIDVisitor` | Dominator-tree walk that assigns environment slot IDs to `GetEnvironment` / `StoreNPToEnvironment` instructions. Ensures each environment level has a unique, stable ID for the runtime. |
| `FixupEnvIDStackNode` | Per-basic-block node used by `FixupEnvIDVisitor` during the dominator traversal. |

---

## Key Functions

### `BytecodeFunctionGenerator::generateBytecodeFunction` — Line 87
Translates one IR `Function` into a `BytecodeFunction`. Key calls:
- `runHBCISel` (`lib/BCGen/HBC/ISel.cpp:2738`) — the instruction selector; maps IR instructions to HBC opcodes
- `bytecodeGenerationComplete` (line 198) — finalises jump targets and resolves forward branches
- Reads register allocation results (`getMaxHVMRegisterUsage`, `getMaxRegisterUsage`) to populate the `FunctionHeader`
- Appends debug source locations from `DebugInfoGenerator`

**Incoming callers:**
- `BytecodeModuleGenerator::generateAddedFunctions` (`lib/BCGen/HBC/BytecodeGenerator.cpp:704`) — called once per function added to the module

---

### `BytecodeModuleGenerator::generate` — Line 723
Main entry for full (non-lazy) module bytecode generation. Sequence:
1. Calls `collectStrings` to build the string literal table
2. Calls `generateAddedFunctions` to translate all IR functions
3. Serializes the `BytecodeModule`

**Incoming callers (via `generateBytecodeForSerialization` in CompilerDriver):**
- `lib/CompilerDriver/CompilerDriver.cpp` — called from `generateBytecodeForSerialization` (line 1888)

---

### `BytecodeModuleGenerator::generateLazyFunctions` — Line 790
Variant of `generate` for lazy compilation: re-enters the compiler for a single function on first call, using the saved IR stub.

### `BytecodeModuleGenerator::generateForEval` — Line 857
Generates bytecode for `eval()` input at runtime.

### `BytecodeModuleGenerator::collectStrings` — Line 349
Walks all functions and instructions to collect every string literal, assigning indices in the string table. Handles Unicode escapes via `appendUnicodeToStorage`.

### `fixupEnvironmentIDs` — Line 615
Runs `FixupEnvIDVisitor` on a `Function` to assign deterministic environment slot numbers before ISel.

### `BytecodeFunctionGenerator::shrinkJump` / `updateJumpTarget` — Lines 165 / 174
Post-ISel fixups: replaces long jumps with short jumps where the offset fits, and patches forward-branch placeholders.

### `BytecodeModuleGenerator::addFunction` — Line 233
Registers an IR `Function` with the module generator, assigning it a bytecode function ID.

---

## Who Calls Into This File (Incoming Calls)

| Caller | Call site |
|---|---|
| `lib/CompilerDriver/CompilerDriver.cpp` | `generateBytecodeForSerialization` calls `BytecodeModuleGenerator::generate` |
| `lib/BCGen/HBC/LoweringPipelines.cpp` | Calls `generateBytecodeFunction` after running the HBC lowering passes |
| `lib/VM/Runtime.cpp` | Lazy compilation re-enters via `generateLazyFunctions` through `RuntimeModule` |

---

## Related Files in the Same Module

| File | Relationship |
|---|---|
| `lib/BCGen/HBC/ISel.cpp` | HBC instruction selector — called from `generateBytecodeFunction` |
| `lib/BCGen/HBC/BytecodeGenerator.h` | `BytecodeFunctionGenerator` class definition |
| `lib/BCGen/HBC/LoweringPipelines.cpp` | Runs lowering passes then calls into this file |
| `lib/BCGen/HBC/DebugInfo.cpp` | `appendSourceLocations` used for debug info generation |
| `lib/CompilerDriver/CompilerDriver.cpp` | Drives the full pipeline including bytecode generation |
| `include/hermes/BCGen/HBC/Bytecode.h` | `BytecodeFunction` and `BytecodeModule` data structures |
