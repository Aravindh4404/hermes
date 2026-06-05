# HadesGC.cpp — LSP Summary

**File:** `lib/VM/gcs/HadesGC.cpp`  
**Module:** `lib/VM/gcs/` (Garbage Collector)  
**Generated:** LSP documentSymbol + outgoingCalls + incomingCalls

---

## What This File Does

`HadesGC.cpp` is the implementation of Hermes's production garbage collector — a concurrent, generational, tri-color mark-and-sweep GC with optional compaction. "Hades" runs old-generation (OG) marking on a background thread to minimize main-thread pause times, while young-generation (YG) collections are stop-the-world but very fast. The file is ~3600 lines and contains the complete lifecycle of objects from allocation to finalization, including the write-barrier logic that keeps the concurrent marker consistent.

---

## Key Classes Defined Here

| Class | Description |
|---|---|
| `HadesGC::OldGen` | Manages the OG heap: multiple `FixedSizeHeapSegment`s + jumbo segments; owns the freelist, sweep logic, and alloc fast/slow paths. |
| `HadesGC::CollectionStats` | RAII struct that records timing, allocated bytes before/after, swept bytes, CPU time, and emits an `InternalAnalyticsEvent` on destruction. |
| `HadesGC::EvacAcceptor<CompactionEnabled>` | Template GC root acceptor used during YG and compacting OG collection; forwards live objects from from-space to to-space. Maintains a `copyListHead` for the Cheney scan. |
| `HadesGC::MarkAcceptor` | Tri-color mark acceptor for concurrent OG marking. Pushes grey objects onto the mark stack; handles compressed pointers and HermesValues. |
| `HadesGC::MarkWeakRootsAcceptor` | Sweeps weak roots after marking completes; clears references to unmarked objects. |
| `HadesGC::Executor` | Single background thread with a `deque<function<void()>>` work queue; used for concurrent OG collection. |

---

## Key Functions

### `HadesGC::youngGenCollection` — Line 2627
YG stop-the-world collection. Key calls:
- `youngGenEvacuateImpl<>` — evacuates live objects from YG to OG
- `finalizeYoungGenObjects` — runs JS finalizers for collected YG objects
- `promoteYoungGenToOldGen` — optionally promotes entire YG to OG when survival rate is high
- `transferExternalMemoryToOldGen` — moves external memory accounting
- `updateYoungGenSizeFactor` — adjusts YG size based on survival ratio
- `checkTripwireAndSubmitStats` — fires analytics events
- `oldGenCollection` — optionally triggers a background OG collection if OG threshold exceeded

**Incoming callers:**
- `HadesGC::allocSlow` (`lib/VM/gcs/HadesGC.cpp:2319/2323`) — allocation failure triggers YG GC
- `HadesGC::collect` (`lib/VM/gcs/HadesGC.cpp:1472/1481`) — explicit GC request
- `HadesGC::makeAImpl` (`include/hermes/VM/HadesGC.h:1652`) — inline fast-path allocation fallback

---

### `HadesGC::oldGenCollection` — Line 1515
Triggers a concurrent OG collection. Submits work to `Executor` via `collectOGInBackground`.

### `HadesGC::incrementalMark` — Line 910
Performs a bounded increment of concurrent marking (drains the mark stack up to `size` bytes). Called from the mutator thread during allocation to yield marking work.

### `HadesGC::completeMarking` — Line 1882
STW final marking phase: drains the remaining mark stack, marks weak roots, clears weak references, and marks weak map entries. Called at the end of concurrent marking before sweep.

### `HadesGC::youngGenEvacuateImpl<CompactionEnabled>` — Line 2576
Template evacuator. Scans YG dirty cards, copies live objects, updates all pointers. `CompactionEnabled=true` path also compacts an OG segment.

### `HadesGC::snapshotWriteBarrierInternal` — Lines 2103–2146
Write barrier implementation (overloaded for `GCCell*`, `CompressedPointer`, `HermesValue`, `SmallHermesValue`, `SymbolID`). Enqueues the old value onto the barrier worklist when the OG marking phase is active (SATB — snapshot-at-the-beginning semantics).

### `HadesGC::barrierEnqueue` / `handleFullBarrierChunk` — Lines 656 / 666
Implements the 128-element write-barrier chunk buffer. When the chunk fills, `handleFullBarrierChunk` flushes it to the mark stack, acquiring the Write Barrier Mutex. This is the mechanism described in the Hades architecture documentation.

### `HadesGC::OldGen::alloc` / `allocSlow` — Lines 2403 / 2417
OG allocation fast path (freelist search) and slow path (expand heap or trigger collection).

---

## Who Calls Into This File (Incoming Calls)

HadesGC is instantiated by `Runtime::Runtime` (`lib/VM/Runtime.cpp:292`). The public GC interface (`GCBase`) routes all calls here:

| Entry point | Caller context |
|---|---|
| `youngGenCollection` | `allocSlow`, `collect`, `makeAImpl` (see above) |
| `oldGenCollection` | `youngGenCollection`, explicit `collect` |
| `snapshotWriteBarrierInternal` | Every GC write barrier site in the VM (JSObject, Arrays, etc.) |
| `HadesGC` constructor | `Runtime::Runtime` constructor |

---

## Related Files in the Same Module

| File | Relationship |
|---|---|
| `include/hermes/VM/HadesGC.h` | Header: class declarations, inline alloc fast path, segment accessors |
| `lib/VM/gcs/AlignedHeapSegment.cpp` | Segment reset/level operations called from YG collection |
| `lib/VM/GCBase.cpp` | Base GC stats, `GCCycle` RAII, `recordGCStats` |
| `lib/VM/Runtime.cpp` | Instantiates HadesGC; drives collection via `drainJobs` |
| `doc/hades.md` | Architecture description (write barriers, 128-element buffer, mutex) |
