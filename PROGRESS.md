# Context Management System — Progress

## Layer Status

| Layer | Tool | Status | Commit |
|-------|------|--------|--------|
| 1 — CLAUDE.md + Docs | /document-generate | DONE | ffaa5c2a6 |
| 2 — Knowledge Graph | gbrain (PGLite) | DONE | see below |
| 3 — Hierarchical Summaries | LLM Wiki + gbrain | PENDING | — |
| 4 — Hybrid RAG | Custom Python | PENDING | — |
| 5 — Agent Memory | /learn + gbrain | IN PROGRESS | see below |

---

## Layer 1 — CLAUDE.md + Documentation (DONE)

**Commit**: ffaa5c2a6  
**Date**: 2026-05-25

Updated/created:
- `README.md` — refreshed
- `doc/Features.md` — updated
- `doc/BuildingAndRunning.md` — updated
- `doc/StaticHermes.md` — created (shermes AOT compiler reference)

Known gaps: hdb debugger undocumented, hbcdump undocumented, no shermes tutorial.

CLAUDE.md trimmed 2026-05-28: removed ~75KB folder listing, now ~5KB. Use `gbrain search` for file lookup.

---

## Layer 2 — Knowledge Graph (DONE)

**Date**: 2026-05-28  
**Tool**: gbrain v0.18.2 with PGLite local engine

### What's indexed

- **135 markdown docs** — all READMEs, design docs, blog posts, build guides  
- **Engine**: PGLite (local, no network, at `~/.gbrain/brain.pglite`)
- **Source pin**: `.gbrain-source` → `gstack-code-hermes`
- **Keyword search**: working via tsvector
- **Vector search**: needs embedding API key (OPENAI_API_KEY or VOYAGE_API_KEY)
- **MCP**: registered as `gbrain` in Claude Code user scope

### Knowledge graph structure (top-level modules)

Based on gbrain sources and doc structure:

```
Hermes knowledge graph (135 pages)
├── API layer
│   ├── doc/design            — Engine architecture overview
│   ├── api/hermes/           — JSI, ABI, NAPI, sandbox APIs
│   └── doc/statichermes      — Static Hermes (shermes) AOT compiler
├── VM + Runtime
│   ├── doc/vm                — VM internals
│   ├── doc/ir                — IR design
│   ├── doc/optimizer         — Optimization passes
│   └── doc/lazyevalcompilation — Lazy function compilation
├── GC subsystem
│   ├── doc/gengc             — Generational GC design
│   └── doc/hades             — Hades concurrent GC
├── Parser + Compiler
│   ├── doc/buildingandrunning — Build system guide
│   ├── doc/features          — Language features
│   └── lib/BCGen/SH/         — Static Hermes bytecode gen
├── Tools
│   ├── tools/test-runner/design — Test runner design
│   ├── tools/secdiff/        — Security diff tool
│   └── tools/hermes-parser/  — Parser tools (JS + Prettier plugin)
├── Blog posts (doc/blog/)
│   ├── 2024-12-23: Compiling JS to WASM
│   ├── 2025-07-15: Static H performance June 2025
│   └── 2025-11-02: Hermes compilation/runtime modes
└── Rust crates (unsupported/)
    ├── hermes_parser
    ├── hermes_estree
    └── hermes_semantic_analysis
```

### How to use gbrain

```bash
# Keyword search
gbrain search "garbage collector write barrier"

# Semantic query (needs embedding API key for vector search)
gbrain query "how does the JIT compiler work"

# Find a doc by slug
gbrain get doc/design

# List all pages
gbrain list
```

### Windows-specific notes

- gbrain bash wrapper: `~/.bun/bin/gbrain` → `bun run ~/gbrain/src/cli.ts`
- The gstack TypeScript orchestrator (`gstack-gbrain-sync.ts`) cannot find gbrain on Windows
  because `execFileSync("gbrain")` doesn't work for bash scripts. Direct `gbrain` CLI usage
  works fine from bash.
- Embeddings require `OPENAI_API_KEY` or `VOYAGE_API_KEY` — not yet configured.
- PGLite does not support `pgvector` extension, so vector search needs the embedding API + pgvector.

### Next steps for Layer 2

1. Set `OPENAI_API_KEY` or `VOYAGE_API_KEY` to enable vector embeddings
2. Run `gbrain embed --stale` to generate 135 page embeddings
3. Integrate `lsp_cpp.py` for symbol-level edges (function calls, class hierarchies)
4. Store LSP edges in gbrain alongside doc-level knowledge

---

## Layer 3 — Hierarchical Summaries (PENDING)

**Planned approach**:
1. Use gbrain + LLM to generate repo → module → file → function level summaries
2. Store as gbrain pages with links to the knowledge graph
3. Structure: `summary/repo`, `summary/lib/vm`, `summary/lib/bcgen`, etc.

**Prerequisite**: Layer 2 embeddings working (need API key)

---

## Layer 4 — Hybrid RAG (PENDING)

**Novel contribution**: Graph-expanded retrieval pipeline

```
Query → Vector search (pgvector) → top 20 chunks
      → Graph expand via LSP edges → pull in callers/callees
      → Rerank (cross-encoder) → top 5
      → Inject into Claude context
```

**Implementation**: Python script in `agent-perf/tools/` or new `scripts/context/`  
**Prerequisite**: gbrain embeddings + lsp_cpp.py symbol edges

---

## Layer 5 — Agent Memory (IN PROGRESS)

**Date**: 2026-05-28  
**Tool**: /learn (gstack-learnings-log at `~/.gstack/projects/facebook-hermes/learnings.jsonl`)

### What's saved (15 learnings)

**Pitfalls (4)**
- `claude-md-size-bloat` — CLAUDE.md auto-gen includes 75KB folder listing; keep under 10KB
- `gc-safety-handles-required` — All GC objects need Handle<T>, never raw pointers across GC points
- `putbyindex-handle-leak-fixed` — commit cc7861e6e fixed handle exhaustion in putByIndex_RJS()
- `microtask-queue-default-true` — commit d8001980e changed MicrotaskQueue default; embedders may break

**Architecture (5)**
- `hermes-dual-execution-modes` — hermesc→HBC interpreter vs shermes→C AOT (separate pipelines)
- `hermes-ir-pipeline` — JS→Parser→AST→SemaResolver→HermesIR(SSA)→Optimizer→BCGen
- `hermes-three-gc-implementations` — GenGC (legacy), Hades (concurrent, default), MallocGC (test)
- `hermes-lazy-compilation` — top-level eager, inner functions compiled on first call
- `static-h-branch-purpose` — static_h adds FlowChecker, typed IR, BCGen/SH C codegen

**Tools (3)**
- `hermesc-vs-shermes` — different flags, different outputs; hermesc for RN, shermes for native
- `hbc-tools` — hbcdump/hbc-diff/hbc-attribute for bytecode debugging; undocumented
- `cdp-debugger` — CDP debugger in API/hermes/cdp/; hdb CLI debugger; both undocumented

**Patterns (2)**
- `hermes-test-structure` — lit tests in test/, gtest in unittests/, shermes in test/shermes/
- `shermes-flow-types-required` — untyped JS works but falls back to dynamic dispatch

**Operational (1)**
- `windows-detect-false-negative` — gstack detect reports no-cli on Windows; use gbrain CLI directly

### Sessions run so far (gstack skills)
- /document-generate → Layer 1 docs
- /setup-gbrain → Layer 2 brain
- /sync-gbrain → incremental refresh
- /learn → Layer 5 patterns

### Next session
- /retro → retrospective on what worked and what to fix
- /plan-eng-review → architecture diagram + test matrix for Hermes
- /review → code review on recent commits
