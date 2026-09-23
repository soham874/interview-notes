# Concurrency & Multithreading

## Thread vs process

| Process | Thread |
|---|---|
| An executing program instance | A unit of execution *within* a process |
| Own isolated memory space | Shares the process's heap and method area |
| Own address space; IPC needed to communicate | Shares memory directly — cheap communication, but needs synchronization |
| Heavier to create/context-switch | Lighter |
| Crash is isolated to that process | An uncaught error can take down the whole process |

Each thread gets its **own stack** (local variables, call frames) and program counter, but **shares the heap** with every other thread in the process. That single fact is the root of every concurrency bug you'll be asked about: shared heap = shared mutable state = races.

## Thread lifecycle states

`Thread.State` enum — worth memorizing, it's a common quick-fire question:

- **NEW** — created but `start()` not yet called.
- **RUNNABLE** — eligible to run; either running or waiting for CPU time. (Java doesn't distinguish "ready" from "running".)
- **BLOCKED** — waiting to acquire a monitor lock to enter a `synchronized` block.
- **WAITING** — waiting indefinitely for another thread's action: `wait()`, `join()`, `LockSupport.park()`.
- **TIMED_WAITING** — same, but with a deadline: `sleep(ms)`, `wait(ms)`, `join(ms)`, `awaitTermination(...)`.
- **TERMINATED** — `run()` has completed or thrown.

Note **BLOCKED vs WAITING** is a favorite distinction: BLOCKED means contending for a *lock*; WAITING means it has been told to stand down until *signalled*.

## Creating threads (know these, but the answer to "how would you do this in real code" is almost always ExecutorService)

- Extend `Thread` (rare, ties you to inheritance) vs implement `Runnable` (preferred — separates "what to run" from "how it runs", allows extending other classes).
- `Callable<V>` — like `Runnable` but returns a value and can throw checked exceptions; used with `ExecutorService.submit()` which returns a `Future<V>`.
- In real Spring Boot code, you almost never call `new Thread()` directly — you use `ExecutorService`/`TaskExecutor`, `@Async`, or reactive constructs.

## synchronized, volatile, and the memory model

- `synchronized` gives you two things at once: **mutual exclusion** (only one thread holds the monitor/lock at a time) and **visibility** (changes made inside a synchronized block by one thread are guaranteed visible to the next thread that acquires the same lock).
- `volatile` gives you **visibility only**, not atomicity — reads/writes go straight to main memory instead of being cached per-thread/reordered, but `count++` on a volatile field is still a data race (it's read-modify-write, three separate operations).
- Classic gotcha: "is `volatile` enough to make a counter thread-safe?" — no, use `AtomicInteger` or synchronize the increment.
- Instance-level vs class-level locking: `synchronized` instance method locks `this`; `synchronized static` method locks the `Class` object — different locks, so they don't block each other.
- Deadlock recipe: two threads, two locks, acquired in opposite order. Fix: always acquire locks in a consistent global order, or use `tryLock` with a timeout, or reduce lock scope, or use higher-level concurrency utilities instead of manual locking.

### The monitor

Every Java object has an associated **monitor** (an intrinsic lock). `synchronized` acquires it on entry and releases it on exit — including on exceptional exit, which is why you can't "leak" an intrinsic lock the way you can leak a `ReentrantLock` without a `finally`. Only one thread can hold a given object's monitor at a time; others contending for it sit in **BLOCKED**. Monitors are **reentrant** — a thread already holding one can re-acquire it (so a synchronized method can call another synchronized method on the same object without self-deadlocking).

### synchronized method vs synchronized block

```java
public synchronized void method() { ... }        // locks `this` for the whole method
public void method() {
    // ... unsynchronized setup work ...
    synchronized (lockObject) { /* only the critical section */ }
}
```

- A synchronized **method** is coarse-grained: it locks `this` (or the `Class` object for statics) for the method's entire duration.
- A synchronized **block** is fine-grained: you choose *what* to lock on and *how much* code to hold it for. Prefer this — shorter critical sections mean less contention.
- Best practice: lock on a **private final** object (`private final Object lock = new Object();`) rather than on `this`. Locking `this` means any outside code holding a reference to your object can also acquire your lock and interfere with your synchronization.

### N threads, N resources without deadlock

The standard answer: **impose a global lock ordering** and require every thread to acquire locks in that order (e.g., always by increasing resource ID). Deadlock requires a circular wait, and a consistent total order makes a cycle impossible. Alternatives: `tryLock()` with timeout + backoff, or a single coarse lock if contention allows.

## wait() vs sleep() vs join() vs yield()

| | `wait()` | `sleep()` |
|---|---|---|
| Defined on | `Object` (instance method) | `Thread` (static method) |
| Called as | `obj.wait()` | `Thread.sleep(millis)` |
| **Requires a `synchronized` block** | **Yes** — else `IllegalMonitorStateException` | **No** — callable anywhere |
| Releases the monitor? | **Yes** — that's the point | **No** — keeps holding every lock |
| Wakes on | `notify()` / `notifyAll()`, timeout, or interrupt | Timeout or interrupt only |
| Used for | Inter-thread communication, condition waiting | Artificial delay, polling backoff, retries |

The `synchronized` row is the one people get backwards. Reason it out rather than memorizing: **`wait()` releases a lock, so you must be holding one to call it.** `sleep()` has nothing to do with locks, so it has no such requirement.

Two more in the same family:

- **`join()`** — the *calling* thread waits for the target thread to finish (`t.join()` blocks until `t` terminates). Puts the caller in WAITING/TIMED_WAITING.
- **`yield()`** — a hint to the scheduler that this thread is willing to give up its current CPU slice. Non-binding and rarely useful in production code.

**Always wait in a loop, never an `if`:**

```java
synchronized (lock) {
    while (!conditionHolds()) {   // while, not if — guards against spurious wakeups
        lock.wait();
    }
    // condition is now true and we hold the lock
}
```

`notify()` wakes one arbitrary waiting thread; `notifyAll()` wakes all of them (each then re-checks its condition). Prefer `notifyAll()` unless you're certain all waiters are interchangeable — `notify()` with heterogeneous waiters can wake the "wrong" thread and stall the system.

## java.util.concurrent.locks

- `ReentrantLock` — explicit alternative to `synchronized`: same thread can re-acquire without deadlocking itself, supports `tryLock()` (non-blocking or with timeout), fairness policy, and `Condition` objects (like multiple wait/notify queues on one lock). Must unlock in `finally`.
- `ReadWriteLock` (`ReentrantReadWriteLock`) — multiple readers can hold the lock simultaneously, writers get exclusive access. Good when reads vastly outnumber writes.
- `synchronized` is usually enough and simpler; reach for `ReentrantLock` when you need tryLock/timeouts/fairness/multiple conditions.

## Atomic classes

- `AtomicInteger`, `AtomicLong`, `AtomicReference` — lock-free thread safety via CAS (compare-and-swap) at the hardware level. `incrementAndGet()`, `compareAndSet()`, `updateAndGet()`.
- Under high contention, CAS can spin/retry a lot — for very hot counters, `LongAdder` (splits the counter across cells, sums on read) scales better than `AtomicLong`.

## ExecutorService & thread pools

- `Executors.newFixedThreadPool(n)`, `newCachedThreadPool()`, `newSingleThreadExecutor()`, `newScheduledThreadPool(n)` — but in real production code prefer constructing `ThreadPoolExecutor` directly (or configuring Spring's `ThreadPoolTaskExecutor`) so you control the queue and rejection policy explicitly; the `Executors` factory methods can hide unbounded queues (`newFixedThreadPool` uses an unbounded `LinkedBlockingQueue` → risk of OOM under sustained load) or unbounded thread creation (`newCachedThreadPool`).
- `ThreadPoolExecutor` knobs: core pool size, max pool size, keep-alive time, work queue, `RejectedExecutionHandler` (what happens when the queue is full and pool is maxed — abort, caller-runs, discard).
- Sizing rule of thumb: CPU-bound work → core pool size ≈ number of cores; I/O-bound work → higher, since threads spend time blocked/waiting (a common formula: threads ≈ cores × (1 + wait_time/compute_time)).
- Always `shutdown()`/`shutdownNow()` executors you own; in Spring, let the container manage the lifecycle of a `TaskExecutor` bean.

## Future, CompletableFuture

- `Future.get()` blocks (optionally with a timeout) — no easy way to chain or combine without blocking.
- `CompletableFuture` (Java 8+) supports non-blocking composition: `.thenApply()` (transform), `.thenCompose()` (flatMap-style chaining of async calls), `.thenCombine()` (combine two independent futures), `.exceptionally()` / `.handle()` for error handling, `.allOf()` / `.anyOf()` for fan-in.
- Runs on `ForkJoinPool.commonPool()` by default unless you pass an explicit `Executor` — worth calling out, since sharing the common pool with CPU-bound parallel streams can cause contention.

## wait/notify vs higher-level constructs

- `Object.wait()`/`notify()`/`notifyAll()` must be called inside a `synchronized` block on that object, release the lock while waiting, and are notoriously easy to get wrong (spurious wakeups — always wait in a `while` loop re-checking the condition, not `if`).
- In practice, prefer `java.util.concurrent` building blocks over hand-rolled wait/notify: `CountDownLatch` (one-time gate, e.g. wait for N tasks to finish), `CyclicBarrier` (reusable rendezvous point for N threads), `Semaphore` (limit concurrent access to a resource), `BlockingQueue` (producer-consumer without manual coordination).

## Common patterns

- **Producer-consumer**: producer threads put onto a `BlockingQueue`, consumer threads take — queue handles all the blocking/signaling.
- **Thread-safe singleton**: initialization-on-demand holder idiom (static inner class, lazily loaded by classloader guarantees) or `enum` singleton — both avoid double-checked locking pitfalls; if you do need DCL, the field must be `volatile`.

## Coding exercises to be able to write from scratch

These two come up constantly in Java concurrency rounds. Practice writing them without notes.

### 1. N threads printing numbers in sequence

The trick is a shared counter plus a turn condition — thread *i* only prints when `number % N == i % N`, otherwise it waits.

```java
public class SequentialPrinter {
    private static final int MAX = 20;
    private static final int THREADS = 3;
    private final Object lock = new Object();
    private int number = 1;                 // guarded by lock

    public void print(int threadId) {
        synchronized (lock) {
            while (number <= MAX) {
                if (number % THREADS != threadId % THREADS) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                    continue;               // re-check the condition after waking
                }
                System.out.println("Thread-" + threadId + ": " + number);
                number++;
                lock.notifyAll();           // wake everyone; each re-checks its turn
            }
            lock.notifyAll();               // final wake so stragglers can exit
        }
    }

    public static void main(String[] args) {
        SequentialPrinter printer = new SequentialPrinter();
        for (int i = 1; i <= THREADS; i++) {
            int id = i;
            new Thread(() -> printer.print(id), "T" + id).start();
        }
    }
}
```

Points to make out loud while writing it: `while` not `if` around `wait()` (spurious wakeups); `notifyAll()` not `notify()` (waiters aren't interchangeable — waking the wrong one stalls everything); and the counter is read *inside* the lock, never outside — reading shared mutable state outside the monitor is a data race even if it appears to work.

### 2. Producer–consumer with a bounded buffer

Classic version with `wait`/`notifyAll`:

```java
public class BoundedBuffer<T> {
    private final Queue<T> buffer = new LinkedList<>();
    private final int capacity;
    private final Object lock = new Object();

    public BoundedBuffer(int capacity) { this.capacity = capacity; }

    public void produce(T item) throws InterruptedException {
        synchronized (lock) {
            while (buffer.size() == capacity) lock.wait();   // full → wait
            buffer.add(item);
            lock.notifyAll();                                // signal consumers
        }
    }

    public T consume() throws InterruptedException {
        synchronized (lock) {
            while (buffer.isEmpty()) lock.wait();            // empty → wait
            T item = buffer.poll();
            lock.notifyAll();                                // signal producers
            return item;
        }
    }
}
```

Then immediately say what you'd actually ship: **`BlockingQueue` does all of this for you.**

```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);
// producer: queue.put(item);   blocks when full
// consumer: Integer i = queue.take();   blocks when empty
```

Knowing the hand-rolled version proves you understand monitors; reaching for `BlockingQueue` in real code proves judgment. Interviewers like seeing both.

## Commonly asked

- Difference between `synchronized` and `volatile` — what does each actually guarantee?
- Why is `i++` not atomic even for a `volatile int`? How would you fix it?
- Walk through how a deadlock happens and two ways to prevent it.
- Why prefer constructing `ThreadPoolExecutor` explicitly over `Executors.newFixedThreadPool` in production code?
- How would you run three independent API calls in parallel and combine their results once all three finish, using `CompletableFuture`?
- What's the difference between `CountDownLatch` and `CyclicBarrier`?
- Does `wait()` need to be inside a `synchronized` block? Does `sleep()`? Why the difference?
- What's the difference between BLOCKED and WAITING in the thread state enum?
- Why should you `wait()` inside a `while` loop rather than an `if`?
- Why is locking on `this` considered worse practice than locking on a private final object?
- Write code so three threads print 1..20 in strict sequence.
- How do you guarantee N threads can access N resources without deadlock?
