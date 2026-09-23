# JVM, Memory & Garbage Collection

## Memory areas (know the diagram)

- **Heap** — all objects and arrays live here, shared across threads. Split into:
  - **Young generation** (Eden + two Survivor spaces S0/S1) — new objects allocated here, collected frequently (minor GC).
  - **Old/tenured generation** — objects that survive enough minor GCs get promoted here, collected less often but more expensively (major/full GC).
- **Metaspace** (replaced PermGen in Java 8) — class metadata, method info; grows into native memory instead of a fixed-size heap region, so "PermGen OOM" from over-loading classes (common with old app servers doing lots of redeployment) is much rarer now.
- **Stack** — one per thread, holds stack frames (local variables, method call state). `StackOverflowError` when it's exceeded (usually unbounded/incorrect recursion).
- **PC register, native method stacks** — lower-level detail, rarely probed beyond acknowledging they exist.

**Careful with the "method area" question.** The method area is a *specification* concept (class structures, method bytecode, static variables, runtime constant pool). Its *implementation* changed in Java 8: it used to be **PermGen**, a fixed-size region **inside the heap**; it is now **Metaspace**, which lives in **native memory outside the heap** and grows dynamically. So "the method area is part of the heap" was true through Java 7 and is wrong from Java 8 onward. Practical consequences: `OutOfMemoryError: PermGen space` no longer exists, class-loading leaks now show up as unbounded native memory growth, and you cap it with `-XX:MaxMetaspaceSize` rather than `-XX:MaxPermSize`.

### Heap vs stack

| Heap | Stack |
|---|---|
| Objects, arrays, instance fields | Method frames, local variables, parameters, return addresses |
| One per JVM, **shared by all threads** | **One per thread**, private to it |
| Managed by GC | Frames popped automatically on method return |
| Slower access, larger | Faster access (LIFO, cache-friendly), smaller |
| `OutOfMemoryError: Java heap space` | `StackOverflowError` |
| Objects live until unreachable | Locals die when the method returns |

The connection worth stating: a local variable holding an object lives on the **stack**, but the object it points to lives on the **heap**. That's why passing a reference into another thread shares the object — and why locals themselves are never shared.

## Object lifecycle & generational hypothesis

- Most objects die young (generational hypothesis) — this is *why* the heap is split: young gen collection is cheap because it only scans a small, mostly-garbage region.
- New objects → Eden. Minor GC survivors → a Survivor space, age incremented each cycle. After surviving enough cycles (tenuring threshold), promoted to old gen.
- Full GC (collecting old gen, sometimes with young gen too) is much more expensive — this is usually what shows up as an application-visible pause.

### How collection actually works: mark–sweep–compact

1. **Mark** — starting from the **GC roots**, traverse every reachable object and mark it live. GC roots are the entry points into the object graph: local variables and parameters on thread stacks, active threads themselves, static fields, and JNI references.
2. **Sweep** — everything not marked is unreachable and its space is reclaimed.
3. **Compact** — surviving objects are moved together so free memory is one contiguous block. Without this you get fragmentation: plenty of total free space but no single gap large enough for a new allocation. (Fragmentation is exactly what made CMS problematic and motivated G1's region-based design.)

Two things this framing lets you say correctly:

- **Reachability, not reference counting.** Java's GC doesn't count references — it asks "is this reachable from a root?" That's why **circular references are collected fine** in Java: two objects pointing at each other with nothing else pointing at them are both unreachable, so both go. (Reference-counting runtimes leak on exactly this case — a good contrast to draw.)
- **GC only manages the heap.** Stack frames pop automatically on method return; Metaspace is native memory with its own (much rarer) reclamation, tied to classloader unloading.

Related: `finalize()` — historically a hook the GC called before reclaiming an object. **Deprecated since Java 9**: it isn't guaranteed to ever run, it delays collection by at least one extra cycle, and it can resurrect the object being finalized. Never the right answer for cleanup — use `try-with-resources`/`AutoCloseable`, or `java.lang.ref.Cleaner` for native resources.

## Garbage collectors (know names + one distinguishing trait each, not full algorithms)

| Collector | Trait |
|---|---|
| Serial | Single-threaded, stop-the-world; fine for small heaps/single-core |
| Parallel (throughput collector) | Multi-threaded stop-the-world; optimizes for throughput over pause time; was the Java 8 default |
| CMS (Concurrent Mark Sweep) | Deprecated/removed in newer JDKs; did most work concurrently with app threads to reduce pauses, but suffered fragmentation |
| **G1 (Garbage First)** | Default since Java 9; heap split into regions, concurrent + incremental, targets a configurable max pause time — most likely one to discuss in a modern interview |
| ZGC / Shenandoah | Low-latency collectors aiming for sub-millisecond pauses even on very large heaps, mostly concurrent |

For a mid-level interview, being able to say "G1 is the default, it divides the heap into regions and prioritizes collecting regions with the most garbage first, aiming to hit a target pause time" covers most of what's asked.

## Class loading

- Loaders form a hierarchy with delegation: Bootstrap (core JDK classes) → Platform/Extension → Application (your classpath). A load request delegates *up* first — parent tries to load before the child does — which is why you can't accidentally shadow `java.lang.String` with your own class on the app classpath.
- Phases: **loading** (find bytecode, create `Class` object) → **linking** (verify bytecode, prepare static fields with default values, resolve symbolic references) → **initialization** (run static initializers and static field assignments, in source order).
- Classes are loaded lazily, typically on first active use.

## JIT compilation (just enough to sound informed)

- The JVM starts by interpreting bytecode, profiles which methods run hot, and compiles those to native machine code at runtime (JIT) — this is why JVM apps often get *faster* after warming up, and why benchmark code needs a warm-up phase to be meaningful.
- HotSpot has a tiered compilation strategy (C1 for fast-but-less-optimized compiles, C2 for slower-but-highly-optimized compiles of the hottest methods).

## Common OutOfMemoryError scenarios (good to be able to name-drop)

- `java.lang.OutOfMemoryError: Java heap space` — too many live objects retained (memory leak via static collections, unclosed resources holding references, caches without eviction) or heap sized too small for the workload.
- `OutOfMemoryError: Metaspace` — too many classes loaded, often from classloader leaks (e.g., repeated hot redeployment without releasing old classloaders).
- `StackOverflowError` — not technically an OOM but often discussed alongside it — runaway/unbounded recursion.
- `OutOfMemoryError: GC overhead limit exceeded` — JVM is spending almost all its time doing GC and reclaiming very little each time — a strong signal of an actual leak, not just heap sizing.
- How you'd actually debug one in practice: heap dump on OOM (`-XX:+HeapDumpOnOutOfMemoryError`), analyze with a tool (Eclipse MAT, VisualVM), look for dominator objects and retained size, check for unbounded caches/collections and unclosed resources.

## Debugging a memory leak (good structured answer for "how would you investigate...")

**Symptoms that say "leak" rather than "undersized heap":**

- Memory usage shows a **sawtooth pattern whose troughs keep rising** — each GC reclaims less than the last, so the baseline creeps up. A healthy app's troughs return to roughly the same level.
- GC runs more and more frequently, and the app spends an increasing share of time paused (`OutOfMemoryError: GC overhead limit exceeded` is the endgame of this).
- Old-gen occupancy after a full GC keeps climbing.

**Investigation steps:**

1. Confirm from GC logs / JFR / a metrics dashboard that post-GC live-set size is genuinely trending upward — not just heap sized too small for a spiky workload.
2. Capture a heap dump (`jmap -dump:live,format=b,file=heap.hprof <pid>`, or automatically via `-XX:+HeapDumpOnOutOfMemoryError`). Ideally take two, some time apart, and compare.
3. Open in Eclipse MAT (or VisualVM/JProfiler). Use the **Leak Suspects** report; sort by **retained size**, not shallow size — retained size is what would actually be freed if the object went away.
4. Walk the **path to GC roots** for the biggest retainer. That path names the thing holding the reference, which is the bug.

**Usual culprits, in rough order of frequency:**

- A `static` collection (map, list, cache) that only ever grows — no eviction, no TTL, no bound.
- Unclosed resources: DB connections, `InputStream`s, HTTP clients, `ResultSet`s — always `try-with-resources`.
- **`ThreadLocal`s not removed on thread-pool threads.** Pool threads are long-lived and reused, so a `ThreadLocal` set on a request and never `remove()`d survives forever and pins whatever it holds. A very common one in Spring apps.
- Broken `equals()`/`hashCode()` on map keys — every put "adds" instead of replacing, so the map grows unboundedly with logically-duplicate entries.
- Listener/callback registration without a matching deregistration.
- Classloader leaks (repeated hot redeploys) — these show as Metaspace growth rather than heap growth.

**Prevention:** bound every cache (size or TTL — or use Caffeine/`LinkedHashMap` with `removeEldestEntry`), `try-with-resources` everywhere, `ThreadLocal.remove()` in a `finally`, and load-test at ≥2× expected volume for long enough that a slow leak becomes visible.

## Commonly asked

- Why is the heap split into young and old generations — what's the underlying assumption that motivates it?
- What's the default garbage collector today and roughly how does it work?
- Explain class loader delegation — why does it prevent classpath spoofing of core classes?
- What causes `OutOfMemoryError: GC overhead limit exceeded`, and how is that different from a plain heap space OOM?
- What's the difference between the interpreter and JIT compilation, and why do JVM apps benefit from warm-up?
- Is the method area part of the heap? (Careful — the answer changed in Java 8.)
- Does Java's garbage collector handle circular references? Why or why not?
- What are GC roots, and why does "retained size" matter more than "shallow size" in a heap dump?
- Walk me through how you'd investigate a service whose memory keeps climbing over days.
- Why is `finalize()` deprecated, and what should you use instead?
- Where do a local variable and the object it points to each live?
