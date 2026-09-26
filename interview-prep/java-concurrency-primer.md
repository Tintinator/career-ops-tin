# Java Concurrency Primer + Multithreaded LLD Grind

Companion to [mongodb-software-engineer-3.md](mongodb-software-engineer-3.md). A 2026 US SE3 first round at MongoDB was a "thread question," and one reply says "the closer you are to mongoDB internals, the more multi-threading questions you will have" (sources in the prep doc). This file has two parts: Part 1 is a refresher, and Part 2 is a problem set to work through in order.

---

# Part 1 — Primer

## 1. The three things that go wrong

Every concurrency bug comes down to one of these:

| Problem | What happens | Example | Fix |
|---|---|---|---|
| **Atomicity** | A "single" operation is actually several steps, and another thread slips in between them | `count++` is read, add, write, so two threads can both read 5 and both write 6 | Lock, or use an atomic class |
| **Visibility** | One thread writes a value and another thread never sees it, because of CPU caches or compiler reordering | A `while (!stop)` loop that never exits | `volatile`, a lock, or an atomic class |
| **Ordering** | The compiler or CPU reorders instructions, so another thread observes a half-built state | Double-checked locking without `volatile` | `volatile` or a lock (both create *happens-before*) |

**Happens-before** is the rule that ties these together. If action A happens-before action B, B sees everything A wrote. You get happens-before from:
- unlocking a lock, then a later lock of the same lock
- a write to a `volatile` field, then a later read of it
- `Thread.start()` (everything before it is visible to the new thread)
- `Thread.join()` (everything the thread did is visible after `join` returns)
- putting an item into a concurrent collection, then taking it out
- `Future.get()` and `CountDownLatch.await()` returning

Name happens-before in the interview when you explain why a fix works.

## 2. Threads

```java
Thread t = new Thread(() -> System.out.println("hi"), "worker-1");
t.start();      // runs on a NEW thread. t.run() would run on the CURRENT thread (classic bug)
t.join();       // wait for it to finish
```

- **States:** NEW → RUNNABLE → (BLOCKED waiting for a monitor | WAITING / TIMED_WAITING in `wait`, `join`, `park` or `sleep`) → TERMINATED.
- **`Runnable` vs `Callable<V>`:** a `Callable` returns a value and can throw checked exceptions. Use it with an `ExecutorService`.
- **Daemon threads** (`t.setDaemon(true)`) don't keep the JVM alive.
- **Interruption is a cooperative cancellation request**, not a kill:
  - `t.interrupt()` sets a flag. Blocking calls (`sleep`, `wait`, `join`, `BlockingQueue.take`, `lockInterruptibly`, `Condition.await`) throw `InterruptedException` and clear the flag.
  - **Rule:** if you catch `InterruptedException` and can't rethrow it, restore the flag with `Thread.currentThread().interrupt();`. Swallowing it silently is a code-review red flag.
- **Virtual threads** (Java 21): `Thread.ofVirtual().start(r)` or `Executors.newVirtualThreadPerTaskExecutor()`. They're cheap enough to run millions, which suits blocking I/O. Mention them if asked about scaling, but don't reach for them in an LLD answer unless asked.

## 3. `synchronized` and `wait` / `notify`

```java
class Counter {
    private int count;
    public synchronized void inc() { count++; }          // locks on `this`
    public int get() { synchronized (this) { return count; } }
}
```

- Every object has one **intrinsic lock (monitor)**. A `synchronized` instance method locks `this`, and a `static synchronized` method locks `Counter.class`. Those are **different locks**.
- The lock is **reentrant**: a thread that already holds it can enter again.
- The lock is released automatically, even when an exception is thrown.
- **Reads need the lock too.** An unsynchronized `get()` has no visibility guarantee.
- **Prefer a private lock object** (`private final Object lock = new Object();`) so outside code can't lock on your object and deadlock you.

**The guarded-wait pattern.** Memorize it: it's the core of a hand-rolled blocking queue.

```java
synchronized (lock) {
    while (!condition) {   // WHILE, never IF: spurious wakeups + another thread may have consumed it
        lock.wait();       // releases the lock while waiting, reacquires before returning
    }
    // act on condition
    lock.notifyAll();      // wake waiters whose condition may now be true
}
```

- `wait`, `notify` and `notifyAll` must be called **while holding that object's monitor**. Otherwise you get `IllegalMonitorStateException`.
- **`notifyAll` vs `notify`:** `notify` wakes one arbitrary waiter. If producers and consumers wait on the same monitor, it can wake the wrong kind of thread and stall. Default to `notifyAll`, or use `ReentrantLock` with two `Condition`s.

## 4. `volatile`

```java
class Worker implements Runnable {
    private volatile boolean running = true;   // without volatile, this loop may never exit
    public void stop() { running = false; }
    public void run() { while (running) { /* work */ } }
}
```

- **Guarantees:** visibility, plus ordering (no reordering across a volatile read or write).
- **Does NOT guarantee atomicity.** `volatile int count; count++` is still a race.
- **Use it for:** a single writer (or independent writes) with many readers, such as stop flags, status fields, or publishing an immutable object reference.
- **Double-checked locking** (the classic volatile question):

```java
class Config {
    private static volatile Config instance;           // volatile is REQUIRED
    static Config get() {
        Config local = instance;
        if (local == null) {
            synchronized (Config.class) {
                local = instance;
                if (local == null) instance = local = new Config();
            }
        }
        return local;
    }
}
```

Without `volatile`, another thread could see a non-null reference to a `Config` whose constructor hasn't finished. The simpler answer is the holder idiom, a `private static class Holder { static final Config INSTANCE = new Config(); }`, which relies on thread-safe class initialization.

## 5. Atomics (`java.util.concurrent.atomic`)

```java
AtomicInteger hits = new AtomicInteger();
hits.incrementAndGet();
hits.updateAndGet(x -> Math.min(x + 1, 100));        // CAS loop under the hood; the lambda must be side-effect free (it may run more than once)

AtomicReference<State> state = new AtomicReference<>(State.IDLE);
boolean won = state.compareAndSet(State.IDLE, State.RUNNING);   // exactly one thread wins

LongAdder total = new LongAdder();   // better than AtomicLong under heavy write contention (striped cells); sum() is not a snapshot
```

- **CAS (compare-and-swap)** is lock-free: "set to B only if it's still A." Retry on failure.
- An atomic class makes **one variable** atomic. If you have **two** variables that must change together, you need a lock, or an `AtomicReference` to one immutable object that holds both.
- **ABA problem:** A→B→A fools a CAS. `AtomicStampedReference` adds a version stamp. It's worth knowing the name.

## 6. Explicit locks (`java.util.concurrent.locks`)

### ReentrantLock
```java
private final ReentrantLock lock = new ReentrantLock();   // new ReentrantLock(true) = fair (FIFO-ish, slower)

void update() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();          // ALWAYS in finally. Forgetting this is the #1 ReentrantLock bug
    }
}
```

What it adds over `synchronized`:
- **`tryLock()` / `tryLock(timeout, unit)`:** back off instead of deadlocking.
- **`lockInterruptibly()`:** lets the waiting thread be cancelled.
- **Multiple `Condition`s per lock**, such as `notFull` and `notEmpty`. This is the main reason to use it in LLD answers.
- An optional fairness policy.
- Introspection: `isHeldByCurrentThread()`, `getQueueLength()`.

### Condition
```java
private final Condition notEmpty = lock.newCondition();
// waiting side (holding lock):  while (queue.isEmpty()) notEmpty.await();
// signaling side (holding lock): notEmpty.signal();
```

`await`, `signal` and `signalAll` are the `Lock` versions of `wait`, `notify` and `notifyAll`. You still need the `while` loop. **Common bug:** calling `lock.wait()` on a `ReentrantLock` object instead of `condition.await()`.

### ReentrantReadWriteLock
```java
private final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
V get(K k)        { rw.readLock().lock();  try { return map.get(k); } finally { rw.readLock().unlock(); } }
void put(K k, V v){ rw.writeLock().lock(); try { map.put(k, v); }   finally { rw.writeLock().unlock(); } }
```

- Many readers **or** one writer. It pays off only when reads heavily outnumber writes and the critical section isn't tiny.
- **Trap:** a read lock can't be upgraded to a write lock, because it deadlocks. Downgrading (take the write lock, take the read lock, release the write lock) is allowed.
- **Trap:** a `LinkedHashMap` in access order changes structure on `get()`, so an "LRU with a read lock on get" is broken.

### StampedLock (name-drop only)
Optimistic reads (`tryOptimisticRead` / `validate`) with no lock at all. It's **not reentrant**. Mention it only if they push on read-heavy performance.

## 7. Concurrent collections

### ConcurrentHashMap
```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();

// WRONG: check-then-act race. Two threads both see null, one update is lost
if (!counts.containsKey(k)) counts.put(k, 1); else counts.put(k, counts.get(k) + 1);

// RIGHT: each of these is atomic per key
counts.merge(k, 1, Integer::sum);
counts.compute(k, (key, v) -> v == null ? 1 : v + 1);
map.computeIfAbsent(userId, id -> new TokenBucket());      // create-once per key
map.putIfAbsent(k, v);
```

- Reads are lock-free. Writes lock only a single bin. It never throws `ConcurrentModificationException`, and iterators are **weakly consistent** (they may or may not show concurrent updates).
- **No `null` keys or values.** `get` returning null has to mean "absent."
- **Keep `compute` and `computeIfAbsent` lambdas short.** The bin is locked while they run, and they must **never modify the same map** (that can deadlock or throw).
- `size()` is an estimate under concurrency.
- **The value object must be thread-safe too.** `computeIfAbsent(k, x -> new ArrayList<>()).add(v)` is a race on the list. Use `ConcurrentHashMap.newKeySet()` or `CopyOnWriteArrayList`, or do the whole update inside `compute`.
- Old trivia: Java 7 used segment locks, and Java 8+ uses CAS plus per-bin `synchronized`, with bins turning into trees when they fill up.

### Others
| Class | Use when |
|---|---|
| `CopyOnWriteArrayList` / `Set` | Reads vastly outnumber writes, e.g. a listener or subscriber list. Every write copies the array. |
| `ConcurrentLinkedQueue` / `Deque` | Lock-free, unbounded, non-blocking queue |
| `ConcurrentSkipListMap` / `Set` | Sorted and concurrent (a thread-safe `TreeMap`). Use it for time-ordered or range data. |
| `Collections.synchronizedMap(m)` | Legacy. One lock for everything, and you must lock manually while iterating. |

## 8. BlockingQueue: the producer-consumer backbone

| Method | Full or empty behavior |
|---|---|
| `put(e)` / `take()` | **block** |
| `offer(e, timeout, unit)` / `poll(timeout, unit)` | block up to the timeout, then return false or null |
| `offer(e)` / `poll()` | return immediately with false or null |
| `add(e)` / `remove()` | throw an exception |

| Implementation | Notes |
|---|---|
| `ArrayBlockingQueue(cap)` | Bounded, one lock, optional fairness. **The default choice for backpressure.** |
| `LinkedBlockingQueue(cap)` | Separate put and take locks, so throughput is higher. **Unbounded if you omit the capacity**, which risks running out of memory. |
| `PriorityBlockingQueue` | Unbounded and ordered by `Comparator`. `take()` blocks only when it's empty. |
| `DelayQueue<E extends Delayed>` | An element becomes available only after its delay expires. Good for schedulers. |
| `SynchronousQueue` | Capacity zero: every put waits for a take (a direct handoff). `newCachedThreadPool` uses it. |
| `LinkedTransferQueue` | `transfer()` waits until a consumer receives the item |

**Shutdown with a poison pill:** the producer puts a sentinel object. Each consumer exits when it takes the sentinel. With N consumers, put N pills, or have each consumer put the pill back before exiting.

## 9. Executors and futures

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> f = pool.submit(() -> compute());      // Callable
Integer result = f.get(2, TimeUnit.SECONDS);           // blocks; throws ExecutionException wrapping the task's exception
pool.shutdown();                                        // no new tasks; running ones finish
if (!pool.awaitTermination(5, TimeUnit.SECONDS)) pool.shutdownNow();   // interrupts workers
```

- **`execute(Runnable)` vs `submit`:** with `submit`, an exception is **captured in the Future** and silently lost if you never call `get()`. This is a classic "why didn't my error show up" bug.
- **`ThreadPoolExecutor(core, max, keepAlive, unit, workQueue, threadFactory, rejectionHandler)`** is what's behind the factory methods. Know what each parameter does:
  - The pool grows past `core` only when the **queue is full**, so with an unbounded queue `max` is never used.
  - Rejection policies: `AbortPolicy` (throw, the default), `CallerRunsPolicy` (the submitting thread runs the task, a natural backpressure), `DiscardPolicy`, and `DiscardOldestPolicy`.
  - Pitfall: `newFixedThreadPool` and `newSingleThreadExecutor` use an **unbounded** `LinkedBlockingQueue`. In production, build a `ThreadPoolExecutor` with a bounded queue.
- **`ScheduledExecutorService`:** `schedule`, `scheduleAtFixedRate` (starts a run every period; if one run takes longer than the period, the next starts late, never concurrently) and `scheduleWithFixedDelay` (waits the delay after each run ends).
- **`CompletableFuture`:** `supplyAsync(supplier, executor).thenApply(...).thenCombine(other, fn).exceptionally(...)`, plus `allOf` and `anyOf`. Always pass your own executor; the default is the shared `ForkJoinPool.commonPool()`.
- **`ForkJoinPool` / parallel streams:** work-stealing for CPU-bound divide-and-conquer. Just name them.

## 10. Synchronizers

```java
CountDownLatch ready = new CountDownLatch(3);   // one-shot: N countDown() calls release all await()ers
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("phase done"));  // reusable: N threads wait for each other
Semaphore permits = new Semaphore(10);          // N permits: acquire() blocks at 0, release() returns one
```

| Tool | Mental model | Typical LLD use |
|---|---|---|
| `CountDownLatch` | A gate that opens once | Start all workers at the same moment, or wait for N tasks to finish |
| `CyclicBarrier` | A meeting point that resets after each round | Rounds or phases where all threads sync (the H2O problem) |
| `Semaphore` | A counter of permits | Connection pool, concurrency cap, turn-taking (odd/even printing) |
| `Phaser` | A dynamic-party barrier | Name-drop only |
| `Exchanger` | Two threads swap objects | Rare |

**Semaphore gotcha:** `release()` doesn't check ownership, so any thread can release. That makes it good for signaling between threads, and dangerous when used as a mutex.

## 11. ThreadLocal

`ThreadLocal<SimpleDateFormat> fmt = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));` gives each thread its own copy of a non-thread-safe object. **In a thread pool, call `remove()` when you're done**, because pool threads live forever. Otherwise you leak memory and stale values carry over into the next task.

## 12. Liveness hazards

- **Deadlock** requires all four of: mutual exclusion, hold-and-wait, no preemption, and circular wait. The interview fix:
  - Break circular wait with a **global lock ordering**. For example, a bank transfer locks the account with the lower ID first.
  - Or break hold-and-wait with **`tryLock` plus a timeout, then back off**.
- **Livelock:** threads keep backing off and retrying in lockstep. Fix it with randomized backoff.
- **Starvation:** some thread never gets the lock. Fix it with a fair lock, or a writer-preference RW lock so writers aren't starved by readers.
- **Lost wakeup:** you signal before the waiter starts waiting, or you use `if` instead of `while`. The guarded-wait pattern above prevents both.
- **Calling foreign code while holding a lock** (listener callbacks, user lambdas) invites deadlock. Copy the listener list under the lock, then call the listeners outside it.

## 13. Which tool? (decision table)

| Situation | Reach for |
|---|---|
| One counter or flag | `AtomicInteger` / `AtomicBoolean` / `LongAdder` |
| Publish a config or stop flag, single writer | `volatile` |
| Per-key state in a map | `ConcurrentHashMap` + `compute` / `merge` / `computeIfAbsent` |
| Several fields must change together | `synchronized` or `ReentrantLock` |
| "Wait until X" with two kinds of waiters | `ReentrantLock` + two `Condition`s |
| Producer-consumer handoff | `BlockingQueue` (bounded) |
| Cap concurrent access to N resources | `Semaphore` |
| Wait for N things to finish | `CountDownLatch` or `invokeAll` / `CompletableFuture.allOf` |
| Run tasks later or periodically | `ScheduledExecutorService` / `DelayQueue` |
| Read-mostly map, writes rare | `ConcurrentHashMap` (usually wins), or `ReentrantReadWriteLock` around a plain map |
| Read-mostly list of listeners | `CopyOnWriteArrayList` |

## 14. How to talk through a concurrency LLD problem (script)

1. **Clarify:** how many producers and consumers? Should callers block, time out, or fail fast when full or empty? Bounded or unbounded? Fairness required? Is shutdown in scope?
2. **Write the single-threaded version of the state first**, naming the fields and **the invariant** (e.g. "0 ≤ count ≤ capacity").
3. **Name the shared state and pick one lock that guards all of it.** Say it out loud: "`lock` guards `items`, `head`, `tail` and `count`."
4. **For each method:** what does it wait for (a `while` loop), what does it change, and whom does it wake?
5. **Walk a two-thread interleaving** to show there's no lost update and no lost wakeup.
6. **Name the trade-offs:** coarse lock vs striping, `notifyAll` vs `Condition`s, fairness vs throughput, blocking vs non-blocking.
7. **Offer a way to test it:** a stress harness (below), and assert the invariant at the end.

### Stress-test harness (reuse for every problem)
```java
static void hammer(int threads, int opsPerThread, Runnable op) throws InterruptedException {
    ExecutorService pool = Executors.newFixedThreadPool(threads);
    CountDownLatch start = new CountDownLatch(1);          // release all threads at once to maximize contention
    CountDownLatch done = new CountDownLatch(threads);
    for (int t = 0; t < threads; t++) {
        pool.execute(() -> {                               // execute, not submit, so exceptions aren't swallowed
            try {
                start.await();
                for (int i = 0; i < opsPerThread; i++) op.run();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                done.countDown();
            }
        });
    }
    start.countDown();
    done.await();
    pool.shutdown();
}
// e.g. hammer(8, 100_000, counter::inc); assert counter.get() == 800_000;
```

---

# Part 2 — Multithreaded LLD grind list

**How to use this list.** Write each solution in a plain editor, without autocomplete, in Java. Time yourself. Run it under `hammer`. Only then open the reference solution. The target times assume a 60-minute screen where the coding block is about 40 minutes.

Legend: ⭐ = directly tied to MongoDB US intel. 🔒 = LeetCode Premium.

## Tier 1 — Warm-ups (15 min each)

These rebuild muscle memory for the primitives.

| # | Problem | Practice | Tool to use |
|---|---|---|---|
| 1 | **Thread-safe counter, three ways** | `synchronized`, `AtomicInteger`, `ReentrantLock`. Prove the unsafe version fails under `hammer`. | All three |
| 2 | **Print in Order** ([LC 1114](https://leetcode.com/problems/print-in-order/)) | Enforcing order across threads | `CountDownLatch` ×2 or `Semaphore` |
| 3 | **FooBar alternately** ([LC 1115](https://leetcode.com/problems/print-foobar-alternately/)) | Taking turns | Two `Semaphore`s |
| 4 | **Odd/even printer:** two threads print 1..N in order | Taking turns with shared state | `synchronized` + `wait`/`notifyAll`, then redo it with `Semaphore`s |
| 5 | **Print Zero Even Odd** ([LC 1116](https://leetcode.com/problems/print-zero-even-odd/)) | Three-way turn-taking | `Semaphore` ×3 |
| 6 | **Fizz Buzz Multithreaded** ([LC 1195](https://leetcode.com/problems/fizz-buzz-multithreaded/)) | Many waiters, one shared counter | `synchronized` + `while` + `notifyAll` |

## Tier 2 — Core LLD (25–35 min each). This is the interview tier.

### 7. ⭐ Bounded blocking queue ([LC 1188](https://leetcode.com/problems/design-bounded-blocking-queue/) 🔒)
- **Build:** `put` (blocks when full), `take` (blocks when empty), `size`. Do it **twice**: once with `synchronized` + `wait`/`notifyAll`, and once with `ReentrantLock` + `notFull`/`notEmpty` Conditions.
- **Follow-ups:**
  - Add `offer(e, timeout)`, using `awaitNanos` in a loop.
  - Why `while` and not `if`?
  - Why two Conditions beat `notifyAll`.
  - Graceful shutdown.
- This is the most common concurrency LLD problem. If you drill only one problem, drill this one.

<details><summary>Walkthroughs</summary>

- [Jenkov: Blocking Queues](https://jenkov.com/tutorials/java-concurrency/blocking-queues.html) builds a bounded queue step by step with `synchronized` and `wait`/`notifyAll`.
- [Oracle `Condition` javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/Condition.html): its `BoundedBuffer` example is the canonical `ReentrantLock` + `notFull`/`notEmpty` solution.
- [Baeldung: Producer-Consumer Problem](https://www.baeldung.com/java-producer-consumer-problem) covers the bounded-buffer framing with `wait`/`notifyAll`, then shows the `BlockingQueue` version.
- [Baeldung: wait() and notify()](https://www.baeldung.com/java-wait-notify) explains why the wait sits in a `while` loop and how the monitor is released.
- [Implementing Blocking Queue (LLD)](https://programmingappliedai.substack.com/p/implementing-blocking-queuelld) is written as an interview walkthrough.
- [Java ArrayBlockingQueue internals](https://topdeveloperacademy.com/articles/java-arrayblockingqueue-a-thread-safe-bound-size-queue) shows how the JDK builds the same thing with a circular array.
</details>

<details><summary>Reference: synchronized version</summary>

```java
class BoundedBlockingQueue<T> {
    private final Object[] items;
    private int head, tail, count;           // guarded by `this`

    BoundedBlockingQueue(int capacity) { items = new Object[capacity]; }

    public synchronized void put(T item) throws InterruptedException {
        while (count == items.length) wait();
        items[tail] = item;
        tail = (tail + 1) % items.length;
        count++;
        notifyAll();                          // wake takers (and, unavoidably, other putters)
    }

    @SuppressWarnings("unchecked")
    public synchronized T take() throws InterruptedException {
        while (count == 0) wait();
        T item = (T) items[head];
        items[head] = null;                   // let GC reclaim it
        head = (head + 1) % items.length;
        count--;
        notifyAll();
        return item;
    }

    public synchronized int size() { return count; }
}
```
</details>

<details><summary>Reference: ReentrantLock + Conditions version</summary>

```java
class BoundedBlockingQueue<T> {
    private final Object[] items;
    private int head, tail, count;                 // guarded by lock
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    BoundedBlockingQueue(int capacity) { items = new Object[capacity]; }

    public void put(T item) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == items.length) notFull.await();
            items[tail] = item;
            tail = (tail + 1) % items.length;
            count++;
            notEmpty.signal();                     // wake exactly one taker; no wasted wakeups
        } finally {
            lock.unlock();
        }
    }

    public boolean offer(T item, long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                if (nanos <= 0) return false;
                nanos = notFull.awaitNanos(nanos);  // returns remaining time
            }
            items[tail] = item;
            tail = (tail + 1) % items.length;
            count++;
            notEmpty.signal();
            return true;
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == 0) notEmpty.await();
            T item = (T) items[head];
            items[head] = null;
            head = (head + 1) % items.length;
            count--;
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try { return count; } finally { lock.unlock(); }
    }
}
```
</details>

### 8. ⭐ Thread-safe inverted index
The MongoDB US phone-screen problem, made concurrent.
- **Build:** `insert(docId, text)`, `search(term)`, `delete(docId)`, `andSearch(t1, t2)`, safe under concurrent readers and writers.
- **Discuss:**
  - `ConcurrentHashMap<String, Set<Integer>>` with `ConcurrentHashMap.newKeySet()` values. Each operation is atomic on its own, but `insert` spanning many terms is **not atomic as a whole**, so a reader may see a half-indexed doc. Is that acceptable? If not, use a `ReentrantReadWriteLock` over the whole index.
  - Delete must remove empty term sets without racing an insert that's adding to the same term. Do the check-and-remove inside `compute`.
  - `andSearch` over weakly consistent sets.

<details><summary>Walkthroughs</summary>

- [How to Implement Inverted Index Data Structure in Java (Tarun Telang)](https://medium.com/practice-programming/how-to-implement-inverted-index-data-structure-in-java-14067093acd4) is the single-threaded build: term map, indexing, search.
- [A Deep Dive into the Inverted Index (Jatin Mamtora)](https://medium.com/@jatinumamtora/the-unsung-hero-of-search-a-deep-dive-into-the-inverted-index-be452d15a5d6) uses `Map<String, Set<Integer>>` and walks through AND queries as a posting-list intersection.
- [USF CS 212: Project 3, Multithreading](https://sites.google.com/a/cs.usfca.edu/cs-212-01-2013-spring/projects/project-3-multithreading) is a university project spec that makes an inverted index thread-safe with a custom read/write lock. It's the same shape as this problem.
- [Jenkov: Read / Write Locks](https://jenkov.com/tutorials/java-concurrency/read-write-locks.html) is the lock you'd wrap around the whole index if a reader must never see a half-indexed doc.
- [Baeldung: A Guide to ConcurrentMap](https://www.baeldung.com/java-concurrent-map) covers the per-key atomic operations (`compute`, `merge`) you'd use for the lock-free version.
</details>

### 9. Thread-safe LRU cache
- **Build:** `get` and `put` with capacity eviction.
- **Discuss:**
  - Why a read-write lock **doesn't** work: `get` changes the recency order.
  - Coarse lock vs **lock striping** (N independent segments, each its own LRU; approximate global LRU).
  - Mention Caffeine's approach: buffer reads and replay them asynchronously.

<details><summary>Walkthroughs</summary>

- [Baeldung: How to Implement LRU Cache in Java](https://www.baeldung.com/java-lru-cache) builds it with a hash map plus a doubly linked list, then covers thread safety.
- [Implement an LRU Cache in Java: Multithreaded Design with Concurrent Collections](https://www.javaspring.net/blog/how-would-you-implement-an-lru-cache-in-java/) goes beyond `LinkedHashMap` into concurrent designs.
- [Implement thread-safe LRU cache: LinkedHashMap vs. ConcurrentHashMap (Alok Maurya)](https://medium.com/@alokkmaurya7/implement-thread-safe-lru-cache-5be57022d9a4) compares the coarse-lock design with a concurrent one.
- [Thread-Safe LRU Cache in Java: Design and Implementation (Shubham Sharma)](https://medium.com/@shubhamsharma935154/thread-safe-lru-cache-in-java-design-and-implementation-a5e00c4d8e54) is a hash map + doubly linked list + `ReentrantLock` walkthrough.
- [Blind: Thread Safe LRU Cache](https://www.teamblind.com/post/thread-safe-lru-cache-yf6hq20j) is an interview thread showing what interviewers push on.
</details>

<details><summary>Reference: coarse-lock version (say "then I'd stripe it" as the follow-up)</summary>

```java
class LruCache<K, V> {
    private final int capacity;
    private final LinkedHashMap<K, V> map;          // guarded by `this`

    LruCache(int capacity) {
        this.capacity = capacity;
        this.map = new LinkedHashMap<>(16, 0.75f, true) {   // accessOrder = true
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > LruCache.this.capacity;
            }
        };
    }

    public synchronized V get(K key) { return map.get(key); }       // get mutates order, so it needs the exclusive lock
    public synchronized void put(K key, V value) { map.put(key, value); }
}
```
Expect a follow-up asking you to write it without `LinkedHashMap`: a `HashMap<K, Node>` plus a doubly linked list with sentinel head and tail, all under one lock.
</details>

### 10. Per-user rate limiter (token bucket)
- **Build:** `boolean allow(String userId)`, with N requests per second per user, bursts up to capacity, and many threads calling at once.
- **Discuss:**
  - `computeIfAbsent` creates each bucket exactly once.
  - The per-bucket lock (not a global lock) keeps users independent.
  - Lazy refill instead of a refill thread.
  - Evicting idle users.
  - The distributed version, if they ask "what about 10 servers": Redis plus a Lua script, or sticky routing.

<details><summary>Walkthroughs</summary>

- [Designing a Thread-Safe Rate Limiter in Java (LLD Interview Guide)](https://medium.com/@abhishekverman3459/designing-a-thread-safe-rate-limiter-in-java-lld-interview-guide-a6490a1ced72) is written as an interview walkthrough, with the race in the unsafe version explained.
- [Designing a Rate Limiter in Java: The Token Bucket Algorithm (Paras Jain)](https://medium.com/@parasjain21389/designing-a-rate-limiter-in-java-the-token-bucket-algorithm-18a4af18c4e0) covers a token bucket with lazy refill.
- [Implementing Rate Limiting in Java from Scratch: Leaky Bucket and Token Bucket (Deven Chen)](https://medium.com/@devenchan/implementing-rate-limiting-in-java-from-scratch-leaky-bucket-and-tokenn-bucket-implementation-63a944ba93aa) compares the two algorithms side by side.
- [Redis: Token bucket rate limiter with Java](https://redis.io/docs/latest/develop/use-cases/rate-limiter/java-lettuce/) answers the "10 servers" follow-up with an atomic Lua script.
- [Baeldung: Rate Limiting with Bucket4j](https://www.baeldung.com/spring-bucket4j) shows the production library, if they ask "what would you use in real life".
</details>

<details><summary>Reference</summary>

```java
class TokenBucket {
    private final long capacity;
    private final double refillPerNano;
    private double tokens;          // guarded by `this`
    private long lastRefillNanos;   // guarded by `this`

    TokenBucket(long capacity, double refillPerSecond) {
        this.capacity = capacity;
        this.refillPerNano = refillPerSecond / 1_000_000_000.0;
        this.tokens = capacity;
        this.lastRefillNanos = System.nanoTime();
    }

    synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        tokens = Math.min(capacity, tokens + (now - lastRefillNanos) * refillPerNano);
        lastRefillNanos = now;
        if (tokens >= 1) {
            tokens -= 1;
            return true;
        }
        return false;
    }
}

class RateLimiter {
    private final ConcurrentHashMap<String, TokenBucket> buckets = new ConcurrentHashMap<>();
    private final long capacity;
    private final double refillPerSecond;

    RateLimiter(long capacity, double refillPerSecond) {
        this.capacity = capacity;
        this.refillPerSecond = refillPerSecond;
    }

    boolean allow(String userId) {
        return buckets.computeIfAbsent(userId, id -> new TokenBucket(capacity, refillPerSecond)).tryAcquire();
    }
}
```
Variants to try next: a sliding-window log (a `Deque<Long>` of timestamps per user) and a fixed-window counter (`AtomicLong` plus a window ID). Compare how accurate each is at window boundaries.
</details>

### 11. Read-write lock from scratch
- **Build:** `lockRead`, `unlockRead`, `lockWrite`, `unlockWrite`, using only `synchronized`/`wait`/`notifyAll` (or one `ReentrantLock`).
- **Discuss:**
  - Reader preference vs **writer preference** (so writers aren't starved).
  - Reentrancy isn't supported, and supporting it would take per-thread hold counts.

<details><summary>Walkthroughs</summary>

- [Jenkov: Read / Write Locks in Java](https://jenkov.com/tutorials/java-concurrency/read-write-locks.html) is the best from-scratch walkthrough: the writer-preference version, why it uses `notifyAll`, and then read, write and read-to-write reentrance.
- [Jenkov: ReadWriteLock (java.util.concurrent)](https://jenkov.com/tutorials/java-util-concurrent/readwritelock.html) covers the JDK version's rules, for comparison.
- [Baeldung: Guide to java.util.concurrent.Locks](https://www.baeldung.com/java-concurrent-locks) covers `ReentrantReadWriteLock` and `StampedLock` usage.
</details>

<details><summary>Reference: writer-preferring</summary>

```java
class SimpleReadWriteLock {
    private int readers;          // active readers
    private boolean writing;      // a writer holds the lock
    private int waitingWriters;   // writers queued; blocks new readers so writers don't starve

    public synchronized void lockRead() throws InterruptedException {
        while (writing || waitingWriters > 0) wait();
        readers++;
    }

    public synchronized void unlockRead() {
        readers--;
        if (readers == 0) notifyAll();
    }

    public synchronized void lockWrite() throws InterruptedException {
        waitingWriters++;
        try {
            while (writing || readers > 0) wait();
        } catch (InterruptedException e) {
            waitingWriters--;
            notifyAll();          // readers may have been waiting only because of us
            throw e;
        }
        waitingWriters--;
        writing = true;
    }

    public synchronized void unlockWrite() {
        writing = false;
        notifyAll();
    }
}
```
</details>

### 12. Delayed task scheduler
- **Build:** `schedule(Runnable, delay, unit)`, with a dispatcher that runs each task on a worker pool when it becomes due.
- **Discuss:**
  - Why the dispatcher uses `awaitNanos(untilHead)` and why `schedule` must `signal`: a new task might be due sooner than the current head.
  - Alternatively, a `DelayQueue` does this for you.
  - Cancellation.
  - Periodic tasks (re-enqueue after each run).

<details><summary>Walkthroughs</summary>

- [How to design a delayed scheduler in Java? (Yihang's blog)](https://silhding.github.io/2021/03/13/How-to-design-a-delayed-scheduler-in-Java/) is an interview-style build with a priority queue plus wait/signal. Closest to the reference solution.
- [Java DelayQueue internals (Deepak Vadgama)](https://deepakvadgama.com/blog/delayed-queue-internals/) explains how the JDK does it, including the leader-follower trick where only one thread does a timed `awaitNanos`. It's a strong follow-up talking point.
- [Java's DelayedWorkQueue](https://loadingmunn.substack.com/p/javas-delayedworkqueue) shows the queue behind `ScheduledThreadPoolExecutor`.
- [Baeldung: Guide to DelayQueue](https://www.baeldung.com/java-delay-queue) is the "just use the JDK" version with a `Delayed` element.
- [System design: delay queue](https://github.com/l-u-k-e-mw-g/system-design/blob/master/delayQueue.md) covers the distributed version, if they scale the question up.
</details>

<details><summary>Reference</summary>

```java
class DelayedScheduler {
    private record Task(long runAtNanos, Runnable job) {}

    private final PriorityQueue<Task> queue =
            new PriorityQueue<>(Comparator.comparingLong(Task::runAtNanos));   // guarded by lock
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition changed = lock.newCondition();
    private final ExecutorService workers = Executors.newFixedThreadPool(4);
    private final Thread dispatcher = new Thread(this::dispatchLoop, "scheduler-dispatcher");

    public void start() { dispatcher.start(); }    // don't start threads in the constructor (leaks `this`)

    public void schedule(Runnable job, long delay, TimeUnit unit) {
        lock.lock();
        try {
            queue.add(new Task(System.nanoTime() + unit.toNanos(delay), job));
            changed.signal();                      // new task may be earlier than the current head
        } finally {
            lock.unlock();
        }
    }

    private void dispatchLoop() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                Runnable due;
                lock.lock();
                try {
                    while (true) {
                        Task head = queue.peek();
                        if (head == null) {
                            changed.await();
                            continue;
                        }
                        long waitNanos = head.runAtNanos() - System.nanoTime();
                        if (waitNanos <= 0) {
                            due = queue.poll().job();
                            break;
                        }
                        changed.awaitNanos(waitNanos);
                    }
                } finally {
                    lock.unlock();
                }
                workers.execute(due);              // run outside the lock
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public void shutdown() {
        dispatcher.interrupt();
        workers.shutdown();
    }
}
```
</details>

### 13. Simple thread pool
- **Build:** a fixed pool of N worker threads pulling `Runnable`s from a `BlockingQueue`. Include `submit` and `shutdown`: stop accepting new tasks, drain the queue, then stop the workers.
- **Discuss:**
  - A worker that dies from an exception (catch `Throwable` per task, keep looping).
  - Shutdown using poison pills vs a `volatile` flag plus `poll(timeout)`.
  - What to do when the queue is full (the rejection policies from §9).

<details><summary>Walkthroughs</summary>

- [Jenkov: Thread Pools](https://jenkov.com/tutorials/java-concurrency/thread-pools.html) is a minimal pool built on `BlockingQueue`, including `stop()`.
- [Custom Thread Pool (jojozhuang)](https://jojozhuang.github.io/algorithm/problem-custom-thread-pool/) is written as an interview problem.
- [Building a Thread Pool from Scratch in Java (HackerNoon)](https://hackernoon.com/building-a-thread-pool-from-scratch-in-java-understanding-concurrency-by-rebuilding-the-core-of-jvm) goes deeper: worker lifecycle, shutdown, and comparison with `ThreadPoolExecutor`.
</details>

### 14. Connection pool
- **Build:** `acquire(timeout)` / `release(conn)` over N connections.
- **Discuss:**
  - A `Semaphore(N)` caps concurrency, and a `ConcurrentLinkedQueue` holds the idle connections.
  - What if the caller never releases? (try-with-resources wrapper, leak detection.)
  - Validating a connection before handing it out.
  - Why `release` must reject connections that didn't come from the pool.

<details><summary>Walkthroughs</summary>

- [Javamex: Controlling resources with Semaphore](https://www.javamex.com/tutorials/synchronization_concurrency_semaphore.shtml) builds a resource pool with a `Semaphore`. This is exactly the connection-pool pattern.
- [Jenkov: Semaphores](https://jenkov.com/tutorials/java-concurrency/semaphores.html) covers how a semaphore works internally, and bounded semaphores.
- [Baeldung: Semaphores in Java](https://www.baeldung.com/java-semaphore) covers `tryAcquire` with a timeout, which you'll need for `acquire(timeout)`.
- [Javarevisited: Counting Semaphore example](https://javarevisited.blogspot.com/2012/05/counting-semaphore-example-in-java-5.html) uses a connection pool as its example.
</details>

### 15. Producer-consumer log pipeline
- **Build:** M producers write log lines, and one async writer batches them to "disk" every 100 lines or 50 ms, whichever comes first.
- **Discuss:**
  - `BlockingQueue.poll(timeout)` combined with `drainTo(batch, max)`.
  - Flushing on shutdown with a poison pill.
  - Backpressure when the disk is slow: block the producers, or drop logs?

<details><summary>Walkthroughs</summary>

- [Asynchronous Producer-Consumer with BlockingQueue in Java](https://looksok.wordpress.com/2015/12/19/asynchronous-producer-consumer-with-blockingqueue-in-java/) is the basic fire-and-forget pipeline.
- [Java Logging in Production: Sync, Async, Flush, and Durability (HackerNoon)](https://hackernoon.com/java-logging-in-production-sync-async-flush-and-durability-explained) covers the trade-offs to discuss: block, drop, or grow when full; flush timing; losing logs on a crash.
- [Oracle `BlockingQueue` javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/BlockingQueue.html) has the producer-consumer usage example, plus `drainTo(collection, max)` for batching.
</details>

### 16. Thread-safe versioned KV store
- **Build:** `put(key, value, ts)` and `get(key, ts)` returning the latest version at or before `ts`, under concurrency.
- **Discuss:**
  - `ConcurrentHashMap<K, ConcurrentSkipListMap<Long, V>>` with `floorEntry`.
  - Why a `TreeMap` inside a CHM value isn't safe.
  - Snapshot reads.
- This ties to MongoDB's world: MVCC and WiredTiger timestamps.

<details><summary>Walkthroughs</summary>

- [NeetCode: Time Based Key-Value Store (LC 981)](https://neetcode.io/solutions/time-based-key-value-store) walks through the single-threaded version, with a video.
- [AlgoMonster: 981 In-Depth Explanation](https://algo.monster/liteproblems/981) walks through the binary-search and `TreeMap.floorKey` approaches.
- [Baeldung: Guide to the ConcurrentSkipListMap](https://www.baeldung.com/java-concurrent-skip-list-map) is the thread-safe sorted map to swap in for `TreeMap`, including `floorEntry` usage.
</details>

## Tier 3 — Stretch (35–45 min). Do these after Tiers 1 and 2 are clean.

| # | Problem | What it tests |
|---|---|---|
| 17 | **Bank transfers without deadlock:** `transfer(from, to, amt)` across many accounts | Lock ordering by account ID, or `tryLock` with backoff. Keep total balance invariant under `hammer`. |
| 18 | **Dining Philosophers** ([LC 1226](https://leetcode.com/problems/the-dining-philosophers/)) | Deadlock avoidance: resource ordering, or a `Semaphore(4)` "waiter" |
| 19 | **Building H2O** ([LC 1117](https://leetcode.com/problems/building-h2o/)) | `Semaphore` + `CyclicBarrier` grouping |
| 20 | **Multithreaded web crawler** ([LC 1242](https://leetcode.com/problems/web-crawler-multithreaded/) 🔒) | `ConcurrentHashMap.newKeySet()` for visited URLs, an executor, and **termination detection** (a pending-task counter or `Phaser`). Knowing when the crawl is done is the hard part. |
| 21 | **In-memory pub/sub broker:** topics, multiple subscribers, each with its own queue | `CopyOnWriteArrayList` of subscribers, a per-subscriber `BlockingQueue`, how a slow subscriber is isolated, delivering outside locks |
| 22 | **Concurrent hit counter:** hits in the last 300 s at high QPS | A ring buffer of `AtomicLong` buckets with timestamps. Compare with `LongAdder` plus rotation. |
| 23 | **Parking lot with concurrent entry gates** | Spot allocation with `ConcurrentHashMap` / `AtomicReferenceArray` CAS on spots, or a per-floor lock |
| 24 | **Traffic Light Controlled Intersection** ([LC 1279](https://leetcode.com/problems/traffic-light-controlled-intersection/)) | A simple mutual-exclusion warm-down |

## Suggested grind order

1. **Day 1:** Primer §1–§7, then Tier 1 #1–#4.
2. **Day 2:** #7 (both versions, twice each), #9, #10.
3. **Day 3:** #8 ⭐, #11, #12.
4. **Day 4:** #13, #14, #16. Reread §12 and §14.
5. **If time remains:** #17, #20, #21.

Before the call, redo #7 (Lock + Conditions) and #10 from a blank page in under 20 minutes each.

## Pre-call cheat sheet (30 seconds to skim)

- `while (!cond) wait()/await()`, **never `if`**.
- `lock.lock(); try { … } finally { lock.unlock(); }`
- `volatile` gives visibility, **not** atomicity.
- Replace check-then-act on a CHM with `merge`, `compute`, `computeIfAbsent` or `putIfAbsent`.
- Restore the flag on interrupt: `Thread.currentThread().interrupt();`
- `submit` swallows exceptions until you call `get()`.
- Bound your queues.
- Name the shared state and the lock that guards it, out loud.
- Prevent deadlock with lock ordering or `tryLock` with a timeout.
