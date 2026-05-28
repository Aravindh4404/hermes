---
id: static-hermes
title: Static Hermes (shermes)
---

# Static Hermes (shermes)

`shermes` is the Static Hermes ahead-of-time (AOT) compiler. It compiles JavaScript
(including typed Flow/JS) directly to native executables via C codegen — no interpreter,
no bytecode. The result is a self-contained binary that runs without a Hermes VM.

This is distinct from `hermesc` (which compiles to Hermes bytecode for the interpreter)
and from the `hermes` REPL (which parses and interprets directly).

## When to use shermes vs. hermes

| Need | Tool |
|---|---|
| Run JS in React Native | Use the pre-built Hermes engine |
| Run JS from a script (interpreted) | `hermes myfile.js` |
| Compile JS to Hermes bytecode | `hermesc -emit-binary -out out.hbc myfile.js` |
| Compile typed JS to a native binary | `shermes myfile.js` |
| Inspect the compiler IR or generated C | `shermes --dump-ir` / `shermes --emit-c` |

Typed mode (`-typed`) unlocks the biggest performance gains: exact object layouts,
no hidden-class polymorphism, inlined method calls, and no boxing for primitives.
Untyped JS is also supported, falling back to standard Hermes semantics.

## Building shermes

`shermes` is built as part of the standard Hermes build. Follow [Building and Running](BuildingAndRunning.md)
to build the whole project; the binary lands at `build/bin/shermes`.

## Quick start

```bash
# Compile and run a JS file (two steps in one)
shermes -exec myfile.js

# Compile to a native binary, then run it separately
shermes -o mybin myfile.js
./mybin

# Compile typed Flow JS
shermes -typed -exec myfile.js

# Strip TypeScript syntax and compile
shermes --transform-ts -exec myfile.ts
```

## Compilation pipeline

```
  Source (.js / .ts / .flow)
        |
        v
  [Parser] → AST
        |
        v (optional: -typed passes type-checker)
  [Sema / FlowChecker] → typed AST
        |
        v
  [IRGen] → Hermes IR
        |
        v
  [Optimizer] → optimized IR
        |
        v
  [BCGen/SH] → generated C code
        |
        v
  [cc / clang] → native binary (.o → ELF/Mach-O/PE)
```

At each stage you can inspect the intermediate form with a dump flag (see [Inspection flags](#inspection-flags)).

## Flag reference

### Input / output

| Flag | Description |
|---|---|
| `<file1> <file2>...` | Input JavaScript source files |
| `-o <file>` | Output file name (default: `a.out`) |
| `-exec` | Compile and immediately execute the result |
| `-Wx,<arg>,<arg>` | Pass extra arguments (comma-separated) to the runtime at exec time |
| `-exported-unit <name>` | Produce a named SHUnit for linking into other code (no `main` generated) |

### Optimization and code size

| Flag | Description |
|---|---|
| `-O` | Expensive optimizations (default) |
| `-O0` | No optimizations |
| `-Og` | Optimizations suitable for debugging |
| `-Os` | Optimize for size |
| `-Xsmall-c` | Optimize the native code for size, not performance |
| `-fstatic-builtins` | Force static builtin recognition (e.g. `Object.keys`) |
| `-fno-static-builtins` | Disable static builtin recognition |
| `-fauto-detect-static-builtins` | Detect from `'use static builtin'` directive (default) |
| `-finline` / `-fno-inline` | Enable/disable function inlining (default: on) |

### Language / parsing

| Flag | Description |
|---|---|
| `-typed` | Enable typed mode — Flow type annotations are checked and used |
| `-strict` | Enable strict mode for the whole file |
| `-script` | Enable script mode (non-module) |
| `-parse-flow` | Parse Flow type syntax |
| `-parse-ts` | Parse TypeScript syntax (converts to Flow before compilation) |
| `--transform-ts` | Strip erasable TypeScript syntax and compile (implies `--parse-ts`) |
| `-ferror-limit <n>` | Maximum number of errors to emit (0 = unlimited, default: 20) |
| `-w` | Disable all warnings |
| `-Werror[=<category>]` | Treat warnings as errors |

### Debug info

| Flag | Description |
|---|---|
| `-g0` | No debug info (default) |
| `-g1` | Emit location info for backtraces |
| `-g2` / `-g` | Emit location info for all instructions |

### Linking

| Flag | Description |
|---|---|
| `-l<lib>` | Link with the given library |
| `-L<path>` | Add library search path and rpath |
| `-nohermeslibs` | Do not link the standard Hermes libraries |
| `-lean` | Link the lean VM (minimal runtime, no REPL/debugger) |
| `-static-link` | Statically link against the VM |

### Extra CC options

| Flag | Description |
|---|---|
| `-Wc,<arg>,<arg>` | Pass extra arguments (comma-separated) directly to the C compiler |

### Miscellaneous

| Flag | Description |
|---|---|
| `-v` | Verbose output |
| `-keep-temp` | Keep intermediate temporary files (useful for debugging codegen) |
| `-ftime-report` | Print compiler timing for each pass |
| `-source-map <file>` | Specify a source map for the input file |
| `-help-typed` | Print the Typed language documentation and exit |

## Inspection flags

Use these to inspect intermediate representations at any compilation stage.

| Flag | Output |
|---|---|
| `-dump-ast` | AST as JSON |
| `-dump-transpiled-ast` | AST after optional early transpilation |
| `-dump-transformed-ast` | AST after semantic validation |
| `-dump-ir` | Hermes IR |
| `-dump-lir` | Lowered IR |
| `-dump-ra` | Register-allocated IR |
| `-emit-c` | Generated C code |
| `-S` | Assembly |
| `-c` | Object file (no final link) |

Example: inspect the IR for a typed function:

```bash
shermes -typed --dump-ir myfile.js
```

## How-to guides

### Compile and run a typed JS file

1. Write a Flow-typed JavaScript file:

```javascript
// hello.js
'use strict';
function greet(name: string): void {
  print('Hello, ' + name);
}
greet('world');
```

2. Compile and run:

```bash
shermes -typed -exec hello.js
# Hello, world
```

### Produce a standalone binary

```bash
shermes -typed -o hello hello.js
./hello
# Hello, world
```

### Inspect the generated C

```bash
shermes -typed -emit-c hello.js
# Prints generated C to stdout
```

### Cross-compile for a different target

```bash
shermes -typed -Xnative-target aarch64-linux-gnu -o hello-arm hello.js
```

### Produce a library unit for linking

```bash
# Compile foo.js as a named unit with no main function
shermes -exported-unit foo_unit -o foo.o -c foo.js

# Link it into your binary (requires the SHUnit registration API)
cc main.c foo.o -o myapp -lhermes-vm
```

### Add runtime flags at exec time

```bash
# Pass Hermes runtime flags when using -exec
shermes -exec -Wx,-Xgc-sanitize-handles,0 myfile.js
```

## Typed mode overview

`-typed` enables the Flow type checker before codegen. Typed code gets:
- **Exact object layouts** — no hidden-class transitions; field access is a direct offset
- **No boxing** for `number` and `boolean` in typed functions
- **Inlined virtual calls** when methods are marked `@Hermes.final`
- **Compile-time type errors** for mismatched types

Untyped functions called from typed code are treated as `any`-typed and go through
standard dynamic dispatch. You can mix typed and untyped freely in one file.

See [Typed Language](TypedLanguage.md) for the complete type system reference.

## Troubleshooting

**`shermes: command not found`** — Build the project first:
```bash
cmake --build ./build --target shermes
```

**`error: unresolved symbol _SHUnit_...`** — You compiled with `-exported-unit` but
linked without registering the unit. Use the standard build pipeline (`-o` without `-c`)
or register the unit via `SHRuntime_loadUnit`.

**Crash at runtime with `Assertion failed: !allocationWillFail`** — The compiled binary
is hitting GC handle limits inside a hot loop with exceptions. If you are catching exceptions
in a tight loop, ensure you are not mixing SH longjmp-based throws with active `GCScopeMarkerRAII`
instances. See the `putByIndex_RJS` fix in commit `cc7861e6e` for the pattern.

**Output differs from `hermes` interpreter** — `shermes` uses the same JS semantics
for untyped code, but typed mode enforces stricter rules (e.g. exact objects, bounds-checked
arrays). Run with `-O0` to disable optimization and isolate behavior differences.
