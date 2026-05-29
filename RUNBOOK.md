# Command Runbook — Context Management System
## Every command used, in order, with explanations

---

## SECTION 1 — MACHINE SETUP (already done, for reference only)

### Install Bun (JavaScript runtime needed by GStack)
```powershell
# In PowerShell
powershell -c "irm bun.sh/install.ps1 | iex"

# Add to PATH permanently
[System.Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\Users\aravi\.bun\bin", "User")

# Verify
bun --version
```

### Install GStack (already done)
```bash
# In Git Bash (NOT PowerShell)
cd ~/.claude/skills/gstack
bash setup
```

### Create junctions manually (Windows workaround — already done)
```powershell
# In PowerShell — only needed if setup didn't create junctions
cd C:\Users\aravi\.claude\skills
Get-ChildItem C:\Users\aravi\.claude\skills\gstack -Directory | Where-Object {
    $_.Name -notmatch '^\.' -and
    $_.Name -notin @('node_modules','docs','lib','bin','hosts','contrib','extension','agents')
} | ForEach-Object {
    $dest = "C:\Users\aravi\.claude\skills\$($_.Name)"
    if (!(Test-Path $dest)) {
        New-Item -ItemType Junction -Path $dest -Target $_.FullName | Out-Null
        Write-Host "Created junction: $($_.Name)"
    }
}

# Verify junctions exist
ls C:\Users\aravi\.claude\skills\ | Select-Object Name, LinkType
```

### Install clangd LSP plugin (already done)
```
# Inside Claude Code
/plugin install clangd-lsp@claude-plugins-official
/reload-plugins
```

```powershell
# Install LLVM (already installed on machine)
winget install LLVM.LLVM
```

---

## SECTION 2 — START EVERY SESSION

### Step 1 — Open Claude Code in Hermes folder
```powershell
cd C:\dev\hermes-test\hermes
npx claude
```

### Step 2 — Set PATH for GBrain in Git Bash (if using gbrain commands)
```bash
# Open Git Bash and run this every session
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"

# Verify gbrain works
gbrain --version
# Should show: gbrain 0.18.2
```

### Step 3 — Check git status
```powershell
cd C:\dev\hermes-test\hermes
git log --oneline -5
git status
```

---

## SECTION 3 — GSTACK SLASH COMMANDS (run inside Claude Code)

### Layer 1 — Documentation
```
# Generate/update all documentation from codebase
/document-generate

# Update docs after code changes (run after every significant change)
/document-release
```

### Layer 2 — Knowledge Graph
```
# First time setup of GBrain
/setup-gbrain
# Choose: PGLite local (option 3) — no accounts needed

# Sync/re-index codebase into GBrain
/sync-gbrain
```

### Layer 5 — Memory
```
# Save what was learned this session
/learn

# Weekly retrospective
/retro
```

### Engineering reviews
```
# Architecture review — diagrams, edge cases, test matrix
/plan-eng-review

# Staff engineer code review
/review

# QA — opens real browser, clicks through app
/qa

# Ship — runs tests, opens PR
/ship
```

### Upgrade GStack
```
/gstack-upgrade
```

---

## SECTION 4 — GBRAIN COMMANDS (run in Git Bash)

```bash
# Always set PATH first
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"

# --- SEARCH ---
# Keyword search
gbrain search "garbage collector write barrier"
gbrain search "MicrotaskQueue default"

# Semantic query
gbrain query "how does the Hades garbage collector work"
gbrain query "how to add a new optimization pass"

# Get full content of a specific page
gbrain get doc/hades
gbrain get doc/optimizer
gbrain get doc/statichermes

# List all indexed pages
gbrain list
gbrain list | head -20

# --- SYNC ---
# Incremental sync (picks up new/changed markdown files)
gbrain sync --repo /c/dev/hermes-test/hermes --no-embed

# Full sync
gbrain sync --repo /c/dev/hermes-test/hermes --full --no-embed

# --- STATUS ---
# Check page counts per source
gbrain sources list --json

# Health check
gbrain doctor --json

# --- EMBEDDINGS (needs API key) ---
export OPENAI_API_KEY="sk-..."
# OR
export VOYAGE_API_KEY="..."
gbrain embed --stale

# --- LEARNINGS ---
# View all saved learnings
~/.claude/skills/gstack/bin/gstack-learnings-search --limit 20

# Search learnings for specific topic
~/.claude/skills/gstack/bin/gstack-learnings-search --query "microtask" --limit 5
```

---

## SECTION 5 — LSP COMMANDS (run inside Claude Code)

```
# Find all references to a C++ symbol (do not use grep)
Using the LSP, find all references to MicrotaskQueue in the Hermes codebase.
Do not use grep. Use only the clangd language server.

# Go to definition
Using LSP, find the definition of HermesValue::encodeObjectValue

# Find all callers of a function
Using LSP call hierarchy, find all callers of hasMicrotaskQueue()

# Get diagnostics for a file
Using LSP, show all diagnostics for lib/VM/Runtime.cpp

# Find all symbols in a file
Using LSP documentSymbol, list all symbols in include/hermes/VM/Runtime.h
```

### LSP operations available
```
workspaceSymbol    — search symbols across whole codebase
documentSymbol     — list all symbols in one file
findReferences     — find all uses of a symbol
prepareCallHierarchy — set up call graph for a symbol
incomingCalls      — who calls this function
outgoingCalls      — what does this function call
hover              — get type info and docs for a symbol
```

---

## SECTION 6 — GIT COMMANDS

```powershell
# Check what commits exist
git log --oneline -10

# See what changed in a specific commit
git show ffaa5c2a6 --stat
git show ffaa5c2a6

# Check current status
git status
git diff

# Push to your GitHub fork
git push origin static_h

# Force push (if rejected due to diverged history)
git push origin static_h --force

# See PROGRESS.md
cat PROGRESS.md

# See the new doc Claude wrote
cat doc\StaticHermes.md
```

---

## SECTION 7 — QUALITY TEST COMMANDS (to re-run tests)

### Test 1 — Without context system
```
# In Claude Code — tell it explicitly not to use any tools
Answer this question using ONLY your training data, no tools, no file access:
"How does the Hermes garbage collector handle write barriers in the Hades collector?"
Give yourself a confidence score out of 10.
```

### Test 2 — With GBrain
```
# In Claude Code
Answer this question using gbrain search and gbrain query only:
"How does the Hermes garbage collector handle write barriers in the Hades collector?"
Show exactly which gbrain queries you ran and what they returned.
```

### Test 3 — Navigation
```
# In Claude Code
Using only gbrain, no grep, no file browsing:
"Which files would I need to modify to add a new optimization pass to the Hermes IR pipeline?"
Show every gbrain query you ran.
```

### Test 4 — Symbol location via GBrain
```
# In Claude Code
Using only gbrain search and learnings, no file reading:
"Where is MicrotaskQueue configured and what is its default value?"
```

### Test 5 — Symbol location via LSP
```
# In Claude Code
Using the LSP only, no grep, no gbrain:
"Find all references to MicrotaskQueue in the Hermes codebase.
Show exact files, line numbers, and what each one does."
```

### Generate quality report
```
# In Claude Code after running all tests
Write a markdown report to PROGRESS.md under a new section:
## Context System Quality Test — [today's date]

Include:
- Each test question
- The answer given
- Score out of 10
- Token count if available
- Number of tool calls made
- Time taken
- Which files/pages were retrieved

Commit the report when done.
```

---

## SECTION 8 — METRICS TO COLLECT (supervisor's request)

For each test run, record:
```
- Time taken (seconds)
- Token usage (Claude Code shows this)
- Number of tool calls made
- Number of files read
- Was first query result correct? (yes/no)
- How many queries needed to find answer?
- Score out of 10 (for answer quality)
```

Prompt to add to each test:
```
After answering, report:
1. How many tool calls did you make?
2. How many files did you read?
3. How long did it take?
4. Was the first search result the correct one?
```

---

## SECTION 9 — WHAT TO DO WHEN THINGS BREAK

### GBrain says "no-cli" or gbrain not found
```bash
# In Git Bash
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
gbrain --version
# If this works, gbrain is fine — the gstack detector has a Windows false negative
```

### PGLite lock timeout
```bash
# Kill stale bun processes
ps aux | grep bun | grep -v grep | awk '{print $1}' | xargs kill
sleep 3
gbrain list  # should work now
```

### GStack skills not found in Claude Code
```powershell
# Check junctions exist
ls C:\Users\aravi\.claude\skills\ | Select-Object Name, LinkType
# All should show LinkType = Junction

# If missing, re-run junction creation from Section 1
```

### Claude Code not found
```powershell
# Use npx instead
npx claude
```

### Bun not found in VS Code terminal
```powershell
$env:PATH += ";C:\Users\aravi\.bun\bin"
bun --version
```

---

## SECTION 10 — KEY COMMITS AND FILES TO CHECK

```
Commits:
ffaa5c2a6  Layer 1 — documentation (doc/StaticHermes.md created)
9e5454ea2  Layer 2 — GBrain setup (136 docs indexed)
d400cd1f2  Layer 5 — 15 learnings saved
a2f29616f  Quality test report

Files to check:
PROGRESS.md                          — full research log
CLAUDE.md                            — 5.3KB context file
doc/StaticHermes.md                  — new doc Claude wrote
~/.gstack/projects/facebook-hermes/learnings.jsonl  — 15 learnings
~/.gbrain/brain.pglite/              — GBrain database

GitHub:
github.com/Aravindh4404/hermes/tree/static_h
```
