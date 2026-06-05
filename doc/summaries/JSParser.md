# JSParser / JSLexer — LSP Summary

**Files:** `lib/Parser/JSParser.cpp`, `lib/Parser/JSLexer.cpp`  
**Module:** `lib/Parser/` (Frontend Parser)  
**Generated:** source read + include/hermes/Parser/JSParser.h

---

## What This Module Does

`lib/Parser/` implements the two-phase Hermes JavaScript parser:
- **JSLexer** tokenises the raw UTF-8 source stream into a token stream.
- **JSParser** (backed by `JSParserImpl`) runs over the token stream and builds an ESTree-compatible AST.

The parser supports three modes (see `enum ParserPass`), enabling Hermes's lazy compilation strategy: expensive AST construction is deferred until a function is actually called.

---

## The Three Parser Passes

Defined in `include/hermes/Parser/JSParser.h:26`:

```cpp
enum ParserPass {
  PreParse,   // Index functions; no AST built
  LazyParse,  // Parse one function using the PreParse index to skip others
  FullParse   // Build the full AST for AOT / hermesc
};
```

| Pass | What it does | When used |
|------|-------------|-----------|
| `PreParse` | Walks the token stream and records function boundaries (start/end offsets) into a `PreParsedData` table. No AST nodes are allocated. O(N) in source length. | First pass for all lazy-compiled modules. |
| `LazyParse` | Given a specific function offset from the `PreParsedData` table, parses only that function body to a full AST. Skips all other function bodies. | Triggered by `CodeBlock::compileLazyFunction()` at first call. |
| `FullParse` | Parses the entire source and builds a complete ESTree AST. Used for AOT compilation (`hermesc`, `shermes`) where all code must be compiled upfront. | Compiler-driver AOT path. |

**Why three passes?** PreParse + LazyParse gives Hermes its fast startup: top-level code and eagerly-called functions are compiled, everything else is a stub until needed. FullParse is used only when the entire module must be compiled ahead of time.

---

## JSLexer

**File:** `lib/Parser/JSLexer.cpp`

### `JSLexer::advance(GrammarContext)` — Line 255

The main tokenizer entry point. Called by the parser to consume the next token from the source buffer.

**What it does:**
1. Skips whitespace and line terminators (calls `optimisticSkipWhitespace` for fast common path)
2. Reads the next character and dispatches on it:
   - Identifiers / keywords → scans word, looks up in keyword table
   - Numeric literals → `scanNumber()`
   - String literals → `scanString()`
   - Punctuators → single-char or multi-char lookahead (`/` disambiguation: `/regex/` vs `/` division)
   - Template literals → handled separately
3. Returns a pointer to the current `Token` (owned by the lexer)

**Grammar context** (`AllowRegExp`, `AllowDiv`, `AllowJSXIdentifier`, `Type`) controls how `/` is disambiguated and whether Flow/TS type syntax is tokenized.

### `JSLexer::advanceInJSXChild()` — Line 749
Variant used inside JSX children — `<`, `{`, and text runs have different tokenization rules.

### `JSLexer::lookahead1()` / `lookahead2()` — Lines 1038, 1101
Non-destructive one- and two-token lookahead. Used by the parser for grammatical disambiguation (e.g. `async function` vs `async` as identifier, `let [` as destructuring vs `let` as identifier).

---

## JSParser / JSParserImpl

**File:** `lib/Parser/JSParser.cpp` + `lib/Parser/JSParserImpl.h` + `lib/Parser/JSParserImpl.cpp`

`JSParser` is a thin public wrapper that holds a `shared_ptr<detail::JSParserImpl>`. All real work is in `JSParserImpl`.

### Parser construction

```cpp
// Used by CompilerDriver for AOT
JSParser(Context &context, std::unique_ptr<llvh::MemoryBuffer> input)

// Used for lazy / pre-parse with a specific pass mode
JSParser(Context &context, uint32_t bufferId, ParserPass pass)
```

### Key methods

| Method | What it does |
|--------|-------------|
| `JSParser::parse()` | Entry point — runs the selected `ParserPass` and returns the root ESTree node (or `llvh::None` on error) |
| `JSParser::isStrictMode()` | Returns whether strict mode was detected (from `"use strict"` directive) |
| `JSParser::getSourceURL()` | Returns the URL from a `//# sourceURL=` magic comment |

---

## Data flow from CompilerDriver

```
CompilerDriver::parseJS()            CompilerDriver.cpp:800
  └─→ JSParser(context, bufferId, pass)
         └─→ JSParserImpl::parse()
               ├─ [PreParse]  scan function boundaries → PreParsedData
               ├─ [LazyParse] parse one function using PreParsedData
               └─ [FullParse] full recursive descent → ESTree AST
                    └─ returns ESTree::NodePtr to CompilerDriver
                          └─→ generateIRFromESTree()   IRGen.cpp:21
```

---

## Related Files

| File | Relationship |
|------|-------------|
| `include/hermes/Parser/JSParser.h` | Public API; `ParserPass` enum; `JSParser` class |
| `include/hermes/Parser/JSLexer.h` | `JSLexer` class; `Token`, `TokenKind`, `GrammarContext` |
| `lib/Parser/JSParserImpl.h` | Full `JSParserImpl` class with recursive-descent methods |
| `lib/CompilerDriver/CompilerDriver.cpp:800` | `parseJS()` — calls JSParser, feeds result to IRGen |
| `lib/VM/CodeBlock.cpp:225` | `compileLazyFunction()` — triggers LazyParse on first call |
| `lib/AST/` | ESTree node types produced by the parser |
| `lib/Sema/SemanticValidator.cpp` | Semantic analysis runs on the ESTree AST after FullParse/LazyParse |
