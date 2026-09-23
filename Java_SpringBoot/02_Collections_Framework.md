# Collections Framework

## The hierarchy

```
Iterable
  └── Collection                          Map (separate root — NOT a Collection)
       ├── List    → ArrayList, LinkedList, Vector, Stack       ├── HashMap → LinkedHashMap
       ├── Set     → HashSet, LinkedHashSet                     ├── Hashtable
       │            └── SortedSet → TreeSet                     └── SortedMap → TreeMap
       └── Queue   → PriorityQueue
                    └── Deque → ArrayDeque, LinkedList
```

Two points interviewers pull on:

- **`Map` does not extend `Collection`.** A Collection is a group of single elements; a Map is a set of key→value *pairs*. They're deliberately separate hierarchies — you access a Map's contents *as* collections via `keySet()`, `values()`, `entrySet()`.
- **Why doesn't `Collection` extend `Cloneable` or `Serializable`?** Because whether cloning or serializing makes sense — and whether a clone should be deep or shallow — depends entirely on the concrete implementation. Forcing it into the root interface would impose a contract many implementations can't meaningfully honor. Concrete classes (`ArrayList`, `HashMap`) implement both individually.

## The core implementations and when to use each

| Interface | Implementation | Ordering | Duplicates | Notes |
|---|---|---|---|---|
| List | `ArrayList` | Insertion order | Yes | Backed by array, O(1) get, O(n) insert/remove mid-list |
| List | `LinkedList` | Insertion order | Yes | Doubly linked list, O(1) insert/remove at ends, O(n) get; also implements `Deque` |
| Set | `HashSet` | None | No | Backed by `HashMap` internally |
| Set | `LinkedHashSet` | Insertion order | No | HashSet + linked list for order |
| Set | `TreeSet` | Sorted (natural or `Comparator`) | No | Backed by `TreeMap` (red-black tree), O(log n) ops |
| Map | `HashMap` | None | Keys unique | See internals below |
| Map | `LinkedHashMap` | Insertion (or access) order | Keys unique | Great for building an LRU cache with `removeEldestEntry` |
| Map | `TreeMap` | Sorted by key | Keys unique | Red-black tree, O(log n), `NavigableMap` methods (floor/ceiling) |

Default pick when asked "what would you use": `ArrayList` for lists (unless heavy mid-list insertion/removal), `HashMap`/`HashSet` for lookups (unless you need order → `LinkedHashMap`, or sorted order → `TreeMap`).

## The pairwise comparison questions

### Array vs ArrayList

| Array | ArrayList |
|---|---|
| Holds primitives **or** objects | Objects only (primitives are autoboxed) |
| Fixed size, set at creation | Grows dynamically (resizes backing array) |
| `length` field | `size()` method |
| No built-in collection operations | `add`, `remove`, `contains`, iterators, streams |
| Not generic — covariant and unsafe (`Object[] o = new String[1]` compiles, then throws at runtime) | Generic and type-safe at compile time |

Arrays are faster for fixed-size primitive data (no boxing overhead, contiguous memory); ArrayList is the default for everything else.

### ArrayList vs LinkedList

| Operation | ArrayList | LinkedList |
|---|---|---|
| Access by **index** | **O(1)** | O(n) |
| **Search** by value (`contains`/`indexOf`) | O(n) | O(n) |
| Insert/remove at end | Amortized O(1) | O(1) |
| Insert/remove at front | O(n) | O(1) |
| Insert/remove at arbitrary index | O(n) shift | O(n) to traverse, then O(1) relink |
| Memory per element | Low (just the array slot) | Higher (node object + 2 pointers) |

Be precise with wording here — ArrayList's advantage is **random access**, not "search". Searching an unsorted list is O(n) either way.

The honest practical answer: **use `ArrayList` unless you've measured a reason not to.** LinkedList's theoretical insert/delete advantage rarely materializes because you usually traverse to the position first, and ArrayList's contiguous memory is far more cache-friendly. LinkedList is mainly interesting as a `Deque`, and even there `ArrayDeque` is usually faster.

### HashMap vs Hashtable

| HashMap | Hashtable |
|---|---|
| Not synchronized | Synchronized (every method) |
| One null key, many null values | No nulls at all |
| Fail-fast iterators | Legacy `Enumeration` (not fail-fast); its `keySet()` iterator *is* fail-fast |
| Introduced in Java 1.2 collections framework | Legacy, since Java 1.0 |

Interview framing: Hashtable is **legacy — don't use it.** If asked for a thread-safe map, the answer is `ConcurrentHashMap` (far better throughput than Hashtable's whole-object locking), and `Collections.synchronizedMap()` only as a wrapper for an existing map you can't replace.

### HashSet vs LinkedHashSet vs TreeSet

| | Ordering | add/remove/contains | Backed by |
|---|---|---|---|
| `HashSet` | None | O(1) average | `HashMap` |
| `LinkedHashSet` | Insertion order | O(1) average | `HashMap` + linked list |
| `TreeSet` | Sorted | O(log n) | `TreeMap` (red-black tree) |

`TreeSet` also gives you `first()`, `last()`, `floor()`, `ceiling()`, `headSet()`, `tailSet()` — reach for it when you need ordering or range queries, not just uniqueness. Note it uses `compareTo()`/`Comparator` (not `equals()`) to decide uniqueness, which can surprise you if the two disagree.

### Iterator vs ListIterator

- `Iterator` — works over any `Collection`, forward only, supports `hasNext()`, `next()`, `remove()`.
- `ListIterator` — `List`s only, **bidirectional** (`hasPrevious()`/`previous()`), and can `add()` and `set()` during iteration, plus `nextIndex()`/`previousIndex()`.

### Enumeration vs Iterator

| Enumeration | Iterator |
|---|---|
| Legacy (Java 1.0) — `Vector`, `Hashtable` | Modern (Java 1.2+) — all collections |
| `hasMoreElements()`, `nextElement()` | `hasNext()`, `next()`, `remove()` |
| Read-only — cannot remove | Can remove safely during iteration |
| Not fail-fast | Fail-fast |

## HashMap internals (very commonly asked, go deep here)

- Backed by an array of buckets (`Node<K,V>[] table`). `hash(key)` (spread of `hashCode()`) determines the bucket index via `(n - 1) & hash`.
- Collisions within a bucket are handled as a linked list; **since Java 8**, if a bucket's chain grows beyond a threshold (8) *and* the table has at least 64 buckets, it treeifies into a red-black tree for that bucket — worst-case lookup goes from O(n) to O(log n).
- Default initial capacity 16, load factor 0.75 — resizes (doubles) when `size > capacity * loadFactor`. Resizing rehashes everything, so if you know the approximate size up front, pass an initial capacity to the constructor to avoid repeated resizes.
- Not thread-safe — concurrent structural modification (put during iteration, or concurrent puts from multiple threads) can corrupt the internal structure or infinite-loop in older JDKs. Use `ConcurrentHashMap`, not `Collections.synchronizedMap`, for concurrent access with good throughput (see below).
- Keys should be immutable (or at least: hashCode/equals-relevant fields shouldn't change after insertion) — mutating a key after it's in the map can make it unfindable, since it now hashes to a different bucket than where it's stored.

## ConcurrentHashMap

- Java 8+: no more segment locking (that was Java 7) — uses a combination of CAS operations and synchronized blocks scoped to individual bins, so concurrent reads are lock-free and writes only lock the bin being modified. Much better throughput than `Collections.synchronizedMap` under contention.
- Iterators are weakly consistent (see fail-fast vs fail-safe below) — no `ConcurrentModificationException`, but iteration may or may not reflect concurrent updates.
- `null` keys/values are **not allowed** (unlike `HashMap`) — this is deliberate, to avoid ambiguity between "key absent" and "key mapped to null" in a concurrent context (`get()` returning null could mean either).
- Useful atomic methods: `putIfAbsent`, `computeIfAbsent`, `merge` — these are how you do read-modify-write safely without external locking.

## Comparable vs Comparator

- `Comparable<T>` — one natural ordering, defined *inside* the class via `compareTo()`. E.g. `Integer`, `String` implement this.
- `Comparator<T>` — external, as many orderings as you want, doesn't require modifying the class. `Comparator.comparing(Person::getAge).thenComparing(Person::getName)`, `.reversed()`.
- `Collections.sort(list)` uses `Comparable`; `Collections.sort(list, comparator)` or `list.sort(comparator)` uses `Comparator`.

## Fail-fast vs fail-safe iterators

- **Fail-fast** (`ArrayList`, `HashMap`, most `java.util` collections): keeps a `modCount`; structural modification during iteration (other than via the iterator's own `remove()`) throws `ConcurrentModificationException` on next `next()` call. This is a best-effort detection, not a guarantee.
- **Fail-safe** — an umbrella term covering two genuinely different mechanisms. Getting this distinction right is what separates a memorized answer from an understood one:
  - **Snapshot iterators** (`CopyOnWriteArrayList`, `CopyOnWriteArraySet`) — the iterator really does hold a frozen copy of the backing array taken at creation time. Later writes create a *new* array and never touch the one being iterated, so the iterator is completely immune to modification — and completely blind to it.
  - **Weakly consistent iterators** (`ConcurrentHashMap`, `ConcurrentLinkedQueue`, `ConcurrentSkipListMap`) — **no copy is made.** The iterator traverses the live structure, never throws `ConcurrentModificationException`, and *may or may not* reflect updates made after it was created. It's guaranteed only to return each element that existed for the whole traversal, at most once.
  - Saying "fail-safe iterators work on a clone of the collection" is only true for the copy-on-write family. `ConcurrentHashMap` clones nothing — say **weakly consistent** for it.
- `CopyOnWriteArrayList` copies the whole backing array on every write — great for read-heavy, write-rare scenarios (e.g., listener lists), terrible for write-heavy ones.

### How to remove elements safely while iterating

The follow-up to "why does this throw `ConcurrentModificationException`":

```java
// Throws CME — structural modification behind the iterator's back
for (String s : list) { if (s.isEmpty()) list.remove(s); }

// Correct — the iterator's own remove() updates modCount in step
Iterator<String> it = list.iterator();
while (it.hasNext()) { if (it.next().isEmpty()) it.remove(); }

// Cleanest for this case (Java 8+)
list.removeIf(String::isEmpty);
```

Note the enhanced for-loop *is* an `Iterator` under the hood — you just don't have a handle on it, which is why you can't call its `remove()`.

## Queue/Deque family

- `ArrayDeque` — resizable array, use it instead of `Stack` (legacy, synchronized, slow) and generally preferred over `LinkedList` as a stack/queue.
- `PriorityQueue` — binary heap, natural ordering or `Comparator`, O(log n) insert/poll, **not thread-safe** (use `PriorityBlockingQueue` for concurrent use).
- `BlockingQueue` implementations (`LinkedBlockingQueue`, `ArrayBlockingQueue`) — the backbone of producer-consumer patterns and thread pool work queues.

## Commonly asked

- Walk through what happens internally on `map.put(key, value)` for a `HashMap`, including a collision and a resize.
- Why can't you safely mutate a key's fields after inserting it into a `HashMap`?
- How does `ConcurrentHashMap` achieve thread safety without locking the whole map, and why does it disallow null keys/values?
- Difference between `Comparable` and `Comparator`, and when would you need both on the same class?
- Why does `ArrayList` throw `ConcurrentModificationException` when modified during a for-each loop, and how do you remove elements safely while iterating?
- When would you reach for `LinkedHashMap` over `HashMap`, or `TreeMap` over both?
- Why is `Map` not part of the `Collection` hierarchy?
- Why doesn't `Collection` extend `Cloneable` or `Serializable`?
- `ArrayList` vs `LinkedList` — which would you actually use in production, and why is the textbook answer misleading?
- What's the real difference between a snapshot iterator and a weakly consistent one?
- Someone asks for a "thread-safe map" — why is `Hashtable` the wrong answer in 2026?
