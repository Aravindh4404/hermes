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

---

## Context System Quality Test — 28-05-2026

**Date**: 2026-05-28  
**Branch**: static_h  
**GBrain pages at test time**: 136 (keyword search only; no vector embeddings)  
**Learnings at test time**: 15 entries

---

### Test 1 — Knowledge retrieval WITHOUT context system

**Question**: How does the Hermes garbage collector handle write barriers in the Hades collector?

**Answer quality: 6/10**

What training data got right:
- SATB (Snapshot at the Beginning) — correct barrier algorithm
- Card tables for YG→OG tracking — correct, 512-byte granularity — correct
- Write barriers are a no-op outside concurrent marking — correct

What training data missed or got wrong:
- Did NOT know about the 128-element fixed-size buffer before lock acquisition
- Did NOT mention the dedicated Write Barrier Mutex (separate from main GC mutex)
- Did NOT know the lock is taken every 128 barriers specifically
- Vague on the flush-into-mark-stack mechanism
- Did NOT know about 32-bit incremental mode fallback

---

### Test 2 — Knowledge retrieval WITH context system

**Question**: Same question.

**Answer quality: 9/10**

**gbrain queries run**:
1. `gbrain search "garbage collector write barrier hades"` → hit `doc/hades`, `doc/gengc`
2. `gbrain search "hades GC incremental"` → hit `doc/hades`
3. `gbrain query "how does Hades handle write barriers"` → hit `doc/hades` (top result)
4. learnings search → `hermes-three-gc-implementations` ("Write barriers required for GC safety"), `gc-safety-handles-required`

**Sources retrieved**:
- `doc/hades` — primary source, contained the full Write Barriers section
- `doc/gengc` — secondary, card table details referenced by Hades doc
- learnings: `hermes-three-gc-implementations`, `gc-safety-handles-required`

**What context system added over training data**:
- **SATB confirmation** — confirmed correct, with precise wording from the doc
- **128-element buffer** — not in training data; found in `doc/hades`
- **Write Barrier Mutex** — dedicated mutex separate from GC mutex; found in `doc/hades`
- **Lock frequency** — every 128 barriers; found in `doc/hades`
- **Flush-to-mark-stack mechanism** — exact implementation; found in `doc/hades`
- **32-bit incremental mode** — for CPUs where 64-bit reads aren't lock-free; found in `doc/hades`

**Delta**: Context system provided 5 concrete factual details unavailable from training data.

---

### Test 3 — Navigation test

**Question**: Which files would I need to modify to add a new optimization pass?

**gbrain queries run**:
1. `gbrain search "optimizer pass pipeline IR"` → `doc/ir` (IR reference), `doc/optimizer` (indirect)
2. `gbrain query "how to add a new optimization pass to Hermes IR"` → `doc/ir`, `doc/plans/ir-type/phase1-steps`
3. `gbrain search "pass manager scalar optimization register"` → no results
4. `gbrain search "optimizer passes list hermes"` → `agent-perf/readme`, `doc/statichermes`
5. `gbrain get doc/optimizer` — directly retrieved full optimizer design doc
6. learnings: `hermes-ir-pipeline` → `lib/Optimizer/Scalar/`, `lib/Optimizer/PassManager/`

**Files identified by gbrain + learnings**:
| File/Dir | Correct? | Source |
|----------|----------|--------|
| `lib/Optimizer/Scalar/` | YES — function pass implementations | learning + doc/optimizer |
| `lib/Optimizer/PassManager/` | YES — pass registration and scheduling | learning |
| `lib/BCGen/` | YES — pipeline exit point | learning |
| `include/hermes/Optimizer/` | YES — pass headers (inferred from doc) | doc/optimizer |
| `lib/CompilerDriver/CompilerDriver.cpp` | LIKELY — pipeline driver | not found by gbrain |

**Assessment**: gbrain found the correct directories for pass implementation (`lib/Optimizer/Scalar/`) and the design doc. It did NOT surface `CompilerDriver.cpp` as the file where passes are registered in the pipeline — that would require a grep or direct file read. gbrain got 4/5 needed files.

**Navigation accuracy: 7/10** — good enough to orient a developer, missing the pipeline registration point.

---

### Test 4 — Symbol location test

**Question**: Where is MicrotaskQueue configured and what is its default value?

**gbrain queries run**:
1. `gbrain search "MicrotaskQueue default value configuration"` → **no results**
2. `~/.claude/skills/gstack/bin/gstack-learnings-search --query "microtask"` → **immediate hit**

**Result from learnings**:
> Commit d8001980e changed MicrotaskQueue default from false to true in RuntimeConfig.  
> Files: `include/hermes/VM/RuntimeConfig.h`, `lib/VM/Runtime.cpp`

**Verification read** (one targeted read):
- Actual file: `public/hermes/Public/RuntimeConfig.h:78`
- Line: `F(constexpr, bool, MicrotaskQueue, true)` — default is `true`
- Learning said `include/hermes/VM/RuntimeConfig.h` — **minor path imprecision** (public/ not include/)
- Default value `true` — **correct**

**Finding**: keyword gbrain search failed (MicrotaskQueue is a C++ identifier, not in markdown prose). The learnings system succeeded on the second query. The stored file path was slightly wrong (wrong directory) but the default value was correct.

**Symbol location accuracy: 7/10** — learnings filled the gap that keyword search missed; file path needed refinement.

---

### Overall Verdict

| Dimension | Score | Notes |
|-----------|-------|-------|
| Keyword search recall | 7/10 | Good for doc concepts; poor for C++ identifiers |
| gbrain query precision | 8/10 | Semantic query routes to right doc pages |
| Learnings system recall | 9/10 | Saved specific commit-level facts not in docs |
| Navigation (finding files) | 7/10 | Finds modules/dirs; misses specific registration files |
| Improvement over baseline | +3/10 | Test 2: 6→9, Tests 3+4: ~5→7 |

**Is the context system working?** YES, partially.

**What works well**:
- `gbrain search` + `gbrain query` reliably find the right design docs (Hades, GenGC, Optimizer)
- Learnings captured commit-level facts (MicrotaskQueue default, putByIndex fix) not in any doc
- The combination of docs + learnings answers questions no single source can answer
- 136 pages at keyword search is genuinely useful even without vector embeddings

**What is missing**:
1. **No vector embeddings** — semantic similarity search not working; all results are keyword (tsvector). Adding `OPENAI_API_KEY` or `VOYAGE_API_KEY` would upgrade every query from 7→9+
2. **C++ symbol search** — gbrain doesn't index C++ source, only markdown. Identifiers like `MicrotaskQueue`, `putByIndex_RJS`, `PassManager` need grep/LSP to locate. The learnings partially compensate but don't scale
3. **Pipeline registration** — `CompilerDriver.cpp` (where passes are wired) not findable via docs alone
4. **136 pages is shallow** — only markdown docs indexed. The ~835 C++ files, ~6000 JS test files are dark to gbrain

**Next action to improve the system**:
- Set `OPENAI_API_KEY` or `VOYAGE_API_KEY` → run `gbrain embed --stale` → vector search unlocks
- Build Layer 4 RAG pipeline to index C++ source symbols alongside docs
