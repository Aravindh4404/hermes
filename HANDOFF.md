# Context Management System — Full Handoff Document
## For a new Claude chat to pick up exactly where we left off

---

## WHO IS DOING THIS AND WHY

**Person:** Aravindh (GitHub: Aravindh4404)
**Goal:** Build and test a 5-layer context management system for large codebases using AI tools
**Sample codebase:** Facebook's Hermes JavaScript engine (React Native's JS runtime)
- Location on machine: `C:\dev\hermes-test\hermes`
- Branch: `static_h`
- Size: ~50,000 files, 835 C++ files, C++/JS codebase
- Fork on GitHub: `github.com/Aravindh4404/hermes` (branch: static_h)

**Key clarification:** Hermes is just a test bed. The goal is NOT to contribute to Hermes. The goal is to prove the context management approach works on a real large codebase. Everything done to Hermes is for testing purposes only.

**Supervisor:** Srinivasan Subramaniam — has asked for quantitative metrics (token usage, tool calls, time per query, retrieval accuracy)

---

## THE 5-LAYER PLAN

```
HERMES CODEBASE (50,000 files)
         │
         ▼
┌─────────────────────────────────────────┐
│ Layer 1: CLAUDE.md + Documentation      │
│ Tool: GStack /document-generate         │
│ Status: DONE — commit ffaa5c2a6         │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ Layer 2: Knowledge Graph (docs)         │
│ Tool: GBrain (PGLite local)             │
│ Status: DONE — commit 9e5454ea2         │
│ 136 markdown docs indexed               │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ Layer 3: Hierarchical Summaries         │
│ Tool: LLM Wiki + GBrain /sync-gbrain   │
│ Status: NOT STARTED                     │
│ Needs: embeddings API key first         │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ Layer 4: Hybrid RAG ★ NOVEL             │
│ Tool: Build in Python                   │
│ Status: NOT STARTED                     │
│ Step 1: Vector search → top 20          │
│ Step 2: Graph expand via LSP edges      │
│ Step 3: Rerank → top 5                  │
│ Step 4: Inject into Claude context      │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ Layer 5: Agent Memory                   │
│ Tool: GBrain + GStack /learn            │
│ Status: IN PROGRESS — commit d400cd1f2  │
│ 15 learnings saved                      │
└─────────────────────────────────────────┘
```

---

## WHAT IS INSTALLED ON THE MACHINE

### GStack
- Location: `C:\Users\aravi\.claude\skills\gstack\`
- All 59 skill junctions created at: `C:\Users\aravi\.claude\skills\`
- Installed via Git Bash (bash setup script)
- Junction creation was manual via PowerShell (setup script had Windows issues)
- Key Windows workaround: run setup from Git Bash, not PowerShell

### GBrain
- Version: 0.18.2
- Engine: PGLite (local, no network, no accounts needed)
- Location: `~/.gbrain/brain.pglite`
- Config: `~/.gbrain/config.json`
- Bash wrapper at: `~/.bun/bin/gbrain` (calls `bun run ~/gbrain/src/cli.ts`)
- **Critical Windows issue:** compiled `gbrain.exe` has WASM path bug. Use bash wrapper only.
- MCP registered in Claude Code at user scope
- 136 pages indexed (all Hermes markdown docs)
- `.gbrain-source` file in hermes repo pins it to `gstack-code-hermes` source

### Bun
- Location: `C:\Users\aravi\.bun\bin\`
- PATH fix required: `[System.Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\Users\aravi\.bun\bin", "User")`
- In VS Code terminal PATH may not load — may need: `$env:PATH += ";C:\Users\aravi\.bun\bin"`

### clangd LSP
- Plugin installed: `clangd-lsp@claude-plugins-official`
- LLVM already installed on machine
- `compile_commands.json` exists at repo root AND `build/` — 743 entries, built with Clang
- Plugin loaded: confirmed via `/reload-plugins` showing `1 plugin LSP server`

### Claude Code
- Version: 2.1.154
- Run via: `npx claude` from `C:\dev\hermes-test\hermes`

---

## WHAT WAS DONE — COMPLETE HISTORY

### Session 1 — Layer 1
**Command:** `/document-generate`
**What happened:** Claude read the entire Hermes codebase and acted as Technical Writer
**Outputs:**
- `doc/StaticHermes.md` — CREATED FROM SCRATCH — complete reference for shermes AOT compiler
- `doc/Features.md` — FIXED 5 errors (features marked "planned" that were already shipped: FinalizationRegistry, Iterator Helpers, Math.sumPrecise, Float16Array, MicrotaskQueue default)
- `doc/BuildingAndRunning.md` — added shermes tool documentation
- `README.md` — added doc index table covering 8 key documents
**Commit:** `ffaa5c2a6`
**Time taken:** 7 minutes

### Session 2 — Layer 2
**Commands:** `/setup-gbrain` then `/sync-gbrain`
**What happened:** GBrain installed locally, 136 docs indexed
**Key outputs:**
- `.gbrain-source` — created, pins repo to gstack-code-hermes source
- `PROGRESS.md` — created automatically by Claude to track all layers
- `CLAUDE.md` — TRIMMED from 104KB to 5KB (removed giant folder listing)
- GBrain PGLite database created at `~/.gbrain/brain.pglite`
**Commit:** `9e5454ea2`
**Windows issues encountered and fixed:**
- gbrain.exe WASM bug → compiled native binary, then reverted to bash wrapper
- PGLite stale lock → killed background bun process
- execFileSync can't find bash wrappers → use gbrain CLI directly from bash
- gstack-gbrain-detect always reports false on Windows (false negative, ignore it)

### Session 3 — Layer 5
**Command:** `/learn`
**What happened:** 15 architectural learnings saved permanently
**Location:** `~/.gstack/projects/facebook-hermes/learnings.jsonl`
**Commit:** `d400cd1f2`

**The 15 learnings saved:**
- Pitfalls: claude-md-size-bloat, gc-safety-handles-required, putbyindex-handle-leak-fixed, microtask-queue-default-true
- Architecture: hermes-dual-execution-modes, hermes-ir-pipeline, hermes-three-gc-implementations, hermes-lazy-compilation, static-h-branch-purpose
- Tools: hermesc-vs-shermes, hbc-tools, cdp-debugger
- Patterns: hermes-test-structure, shermes-flow-types-required
- Operational: windows-detect-false-negative

### Session 4 — Quality Test
**What happened:** 4 tests run to measure context system effectiveness
**Commit:** `a2f29616f`
**Results stored in:** `PROGRESS.md`

**Test 1** — Without context system
- Question: "How does Hades GC handle write barriers?"
- Score: 6/10 (self-reported by Claude)
- Claude said: "Confidence ~6/10, uncertain about exact implementation details"

**Test 2** — With GBrain context system
- Same question, GBrain retrieved `doc/hades.md` in one query
- Score: 9/10 (our assessment based on 5 verifiable facts added)
- 5 facts found that weren't in training data: 128-element buffer, Write Barrier Mutex, lock fires every 128 writes, flush-to-mark-stack mechanism, 32-bit fallback

**Test 3** — Navigation test
- Question: "Which files to modify for a new IR optimization pass?"
- GBrain only, no grep
- Score: 7/10 — found 4/5 correct locations, missed CompilerDriver.cpp

**Test 4** — Symbol location
- Question: "Where is MicrotaskQueue configured?"
- GBrain keyword search: ZERO results
- Learnings search: immediate hit, correct answer
- Score: 7/10

### Session 5 — LSP Setup and Test 5
**What happened:** clangd LSP plugin installed and tested
**Commands run:**
1. `/plugin install clangd-lsp@claude-plugins-official`
2. `/reload-plugins` — confirmed 1 plugin LSP server loaded
3. `winget install LLVM.LLVM` — already installed
4. Generated compile_commands.json — already existed (743 entries)

**Test 5** — LSP vs GBrain on same MicrotaskQueue question
- GBrain result: ZERO results (Test 4)
- LSP result: Found ALL 8 files in 4 minutes with exact line numbers AND full call graph:
```
RuntimeConfig.h (default=true)
    ↓ withMicrotaskQueue()
RuntimeFlags.cpp / hermes.cpp  ← CLI flag
    ↓ getMicrotaskQueue()
Runtime constructor (Runtime.cpp:304)
    ↓ hasMicrotaskQueue_
hasMicrotaskQueue() [Runtime.h:997]
    ├── ConsoleHost.h:103  → gates drainJobs()
    └── Runtime.cpp:1293   → gates Promise reporting
```

---

## GIT COMMIT TRAIL

```
a2f29616f  test: context system quality test — 4 tests, scores logged
d400cd1f2  chore: /sync-gbrain + /learn — 15 session learnings saved
9e5454ea2  feat: Layer 2 knowledge graph — gbrain PGLite setup, 135 docs indexed
ffaa5c2a6  docs: generate full project documentation pass (Diataxis)
d8001980e  Set microtask queue true by default  ← Facebook's commit, not ours
```

---

## KEY FILES

```
C:\dev\hermes-test\hermes\
├── PROGRESS.md              ← main research log, 14KB, auto-updated by Claude
├── CLAUDE.md                ← 5.3KB (was 104KB), has GBrain guidance block
├── .gbrain-source           ← pins to gstack-code-hermes GBrain source
├── compile_commands.json    ← 743 entries for clangd, already exists
├── doc\
│   ├── StaticHermes.md      ← NEW FILE created by /document-generate
│   ├── Features.md          ← fixed by /document-generate
│   ├── hades.md             ← Facebook's existing doc, used in Test 2
│   └── ...135 other docs    ← all indexed in GBrain
└── build\
    └── compile_commands.json ← same as root, clangd uses root version

C:\Users\aravi\
├── .claude\skills\gstack\   ← GStack installation (59 skills)
├── .claude\skills\          ← 59 junction symlinks pointing into gstack\
├── .gbrain\brain.pglite\    ← GBrain local database (136 pages)
├── .gstack\projects\
│   └── facebook-hermes\
│       └── learnings.jsonl  ← 15 permanent learnings
└── .bun\bin\
    ├── gbrain               ← bash wrapper (use this, not gbrain.exe)
    └── bun.exe
```

---

## KNOWN ISSUES AND WORKAROUNDS

1. **GBrain only indexes markdown** — 835 C++ files are invisible. LSP fills this gap.
2. **No vector embeddings** — needs OPENAI_API_KEY or VOYAGE_API_KEY. All search is keyword (tsvector). Set key then run `gbrain embed --stale` to unlock semantic search.
3. **gstack-gbrain-detect always shows false on Windows** — this is a false negative. gbrain works fine. Ignore the warning.
4. **GStack setup.sh needs Git Bash** — never run from PowerShell directly.
5. **Bun PATH in VS Code** — may need `$env:PATH += ";C:\Users\aravi\.bun\bin"` each session, or use Git Bash.
6. **clangd still indexing on first open** — workspaceSymbol queries may return 0 results for first few minutes. Wait and retry.

---

## WHAT TO DO NEXT

### Immediate next steps
1. **Run Test 5 properly** — run the same 4 quality tests WITH LSP and compare scores. This gives the quantitative metrics the supervisor asked for.
2. **Add metrics to tests** — measure token usage, tool calls, time per query for each test (with and without context system).
3. **Commit Test 5 results to PROGRESS.md** and push to GitHub.

### Layer 3 — Hierarchical Summaries
- Get an embeddings API key (Voyage AI free tier at voyageai.com OR OpenAI)
- Run `gbrain embed --stale` to generate vectors for 136 pages
- Then use LLM Wiki structure for repo → module → file → function summaries

### Layer 4 — Hybrid RAG (the novel contribution)
- Build Python script that does:
  1. Vector search via pgvector → top 20 chunks
  2. Graph expand via LSP edges (clangd call hierarchy)
  3. Rerank → top 5
  4. Inject into Claude context
- Neither GStack nor any existing tool does this
- clangd LSP is now installed and provides the graph edges needed for step 2

### Metrics to implement (supervisor's request)
- Token usage per query (with vs without context system)
- Tool call count per query
- Time per query end to end
- Retrieval precision (was top result correct?)
- First query success rate
- Context window utilisation percentage

---

## USEFUL COMMANDS TO RUN GBRAIN FROM GIT BASH

```bash
# Always set PATH first in Git Bash
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"

# Check gbrain is working
gbrain --version

# Search the knowledge base
gbrain search "garbage collector write barrier"
gbrain query "how does the static compiler work"

# Get a specific doc
gbrain get doc/hades
gbrain get doc/optimizer

# List all indexed pages
gbrain list

# Sync new docs
cd /c/dev/hermes-test/hermes
gbrain sync --repo /c/dev/hermes-test/hermes --no-embed

# Check page counts
gbrain sources list --json

# View learnings
~/.claude/skills/gstack/bin/gstack-learnings-search --limit 20
```

---

## PRESENTATION SUMMARY (for supervisor)

The core finding so far:
- **Without context system:** Claude scores 6/10 on Hermes questions (self-reported)
- **With GBrain docs:** Claude scores 9/10 (+5 specific facts from doc/hades.md)
- **With LSP:** Found all 8 MicrotaskQueue references with full call graph (GBrain got zero)
- **Three tools serve three different needs:** GBrain for design docs, LSP for C++ symbols, Learnings for commit-level facts

The gap that remains is Layer 4 — combining all three into one retrieval pipeline that automatically gives Claude exactly the right context for any question.
