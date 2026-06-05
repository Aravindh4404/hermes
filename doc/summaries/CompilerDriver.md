# CompilerDriver.cpp — LSP Summary

**File:** `lib/CompilerDriver/CompilerDriver.cpp`  
**Module:** `lib/CompilerDriver/` (Compiler Frontend Orchestration)  
**Generated:** LSP documentSymbol + outgoingCalls + incomingCalls

---

## What This File Does

`CompilerDriver.cpp` is the top-level orchestration layer for the Hermes compiler (`hermesc` and `shermes`). It parses command-line flags, reads input files (JS source, bytecode, zip bundles), creates the compilation `Context`, runs the parser → semantic analysis → IR generation → optimization → bytecode generation pipeline, and writes `.hbc` output. It is the glue that wires together all compiler subsystems. The file also provides `compileFromCommandLineOptions` — the single function called by `main` in the compiler tools.

---

## Key Classes Defined Here

| Class | Description |
|---|---|
| `CLFlag` | LLVM `cl::opt` wrapper for boolean yes/no flags (e.g., `--inline` / `--no-inline`). Stores two `opt<bool>` and resolves the final value. |
| `ModuleInSegment` | Holds one module entry within a segment: numeric `id`, source `MemoryBuffer`, and optional source-map `MemoryBuffer`. |

---

## Command-Line Options Defined Here (selection)

The file defines ~60 `cl::opt` / `cl::list` / `CLFlag` objects covering:
- Optimization level (`-O0`, `-Og`, `-OMax`, `-OFixedPoint`)
- Output format (`--dump-ir`, `--dump-lir`, `--dump-bytecode`, `--emit-binary`)
- Language features (`--typed`, `--parse-flow`, `--parse-ts`, `--enable-tdz`)
- Bytecode format (`--bytecode-format=HBC|SH`)
- Debug info (`-g0` through `-g3`)
- Warnings, static require, CommonJS / JSX modes

---

## Key Functions

### `compileFromCommandLineOptions` — Line 2316
Main entry point for the `hermesc` / `shermes` tools. Sequence:
1. `setFlagDefaults` — applies defaults for unset flags
2. `validateFlags` — checks for incompatible flag combinations, returns error
3. Reads input files via `readInputFilenamesFromDirectoryOrZip` or `memoryBufferFromFile`
4. For bytecode input: calls `processBytecodeFile`
5. For source input: calls `createContext` then `processSourceFiles`

**Incoming caller:** `main` in `tools/hbcdump/hbcdump.cpp:451` — also `hermesc/main.cpp` and `shermes/main.cpp` (tools).

---

### `processSourceFiles` — Line 1959
Full source→bytecode pipeline. For each segment:
1. `parseJS` — parse JS source with optional source map
2. Semantic analysis via `sema::SemanticValidator`
3. IR generation via `ESTreeIRGen`
4. Optimization via `runFullOptimizationPasses` / `runDebugOptimizationPasses`
5. `generateBytecodeForSerialization` — bytecode lowering and output

### `parseJS` — Line 800
Parses one JS source buffer into an ESTree AST using `JSParser`. Handles Flow types (`--parse-flow`), TypeScript (`--parse-ts`), and source-map translation. Returns `ESTree::NodePtr`.

### `generateBytecodeForSerialization` — Line 1888
Calls `BytecodeModuleGenerator::generate` to translate the IR module to bytecode, then serializes it to a `raw_ostream` with optional SHA1 delta base.

### `generateBytecodeForExecution` — Line 1860
Variant that compiles source for in-process execution (no file output); used by the REPL and `hermes` tool.

### `createContext` — Line 1154
Constructs a `shared_ptr<Context>` from current flags, setting `CompileFlags` (strict mode, lazy compilation, typed mode, optimization level, feature flags).

### `loadGlobalDefinition` — Line 762
Parses a Flow declaration file (e.g., `globals.d.flow`) and registers its declarations so the semantic checker knows about platform globals.

### `generateIRForSourcesAsCJSModules` — Line 1693
Runs `IRGen` on all CJS modules in a segment and links them under the segment root.

### `validateFlags` — Line 1008
Checks for invalid flag combinations (e.g., `--bytecode-mode` with source-only options). Returns `false` on error to abort compilation.

---

## Who Calls Into This File (Incoming Calls)

| Caller | Call site |
|---|---|
| `tools/hermesc/main.cpp` | Calls `compileFromCommandLineOptions()` after LLVM init |
| `tools/shermes/main.cpp` | Same — Static Hermes compiler entry point |
| `tools/hbcdump/hbcdump.cpp:451` | `main` — uses compiler driver for `.hbc` disassembly workflow |

---

## Related Files in the Same Module

| File | Relationship |
|---|---|
| `include/hermes/CompilerDriver/CompilerDriver.h` | Declares `CompileResult`, `compileFromCommandLineOptions`, `outputFormatFromCommandLineOptions` |
| `lib/IRGen/ESTreeIRGen.cpp` | IR generation called from `processSourceFiles` |
| `lib/Optimizer/PassManager/Pipeline.cpp` | Optimization passes run inside `processSourceFiles` |
| `lib/BCGen/HBC/BytecodeGenerator.cpp` | Bytecode generation called from `generateBytecodeForSerialization` |
| `lib/BCGen/SH/SH.cpp` | Static Hermes backend — alternative to HBC generation |
| `lib/Parser/JSParser.cpp` | Parser used in `parseJS` |
| `lib/Sema/SemanticValidator.cpp` | Semantic analysis between parse and IR gen |
