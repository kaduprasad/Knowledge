# Multithreading in Java

**Multithreading** is a feature in Java that **allows concurrent execution of two or more parts** of a program to maximize CPU utilization.

---

## What is a Thread?

A **thread** is a lightweight sub-process, the smallest unit of processing. It allows a program to perform multiple tasks concurrently. Java's multithreading enables concurrent/parallel execution of two or more parts of a program for maximum CPU utilization.

---

## Ways to Create a Thread

### 1. Extending `Thread` Class

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running via Thread class");
    }
}

public class Main {
    public static void main(String[] args) {
        MyThread t = new MyThread();
        t.start();
    }
}
```

### 2. Implementing `Runnable` Interface

```java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Thread running via Runnable");
    }
}

public class Main {
    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable());
        t.start();
    }
}
```

### 3. Using Lambda Expression (Java 8+)

```java
Thread t = new Thread(() -> System.out.println("Thread running via Lambda"));
t.start();
```

### 4. Using `Callable` + `ExecutorService`

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(() -> 42);
System.out.println(future.get()); // 42
executor.shutdown();
```

---

## Thread Lifecycle

A thread in Java exists in one of these states at any point:

| State | Description |
|-------|-------------|
| **New** | Thread created but `start()` not yet called |
| **Runnable** | Thread is running or ready to run |
| **Blocked** | Waiting to acquire a lock held by another thread |
| **Waiting** | Waiting indefinitely for another thread to notify |
| **Timed Waiting** | Waiting for a specified time (`sleep()`, `wait(timeout)`) |
| **Terminated** | Thread completed execution or was terminated |

![][image1]

---

## `synchronized` Keyword

The `synchronized` keyword **controls access of multiple threads to shared resources.** It prevents thread interference and ensures **only one thread** can access a shared resource at a time to avoid **race conditions**.

### Class-Level Lock vs Instance-Level Lock

| Type | Syntax | Scope |
|------|--------|-------|
| Instance Lock | `synchronized (this)` | Locks the current object instance |
| Class Lock | `synchronized (MyClass.class)` | Locks the entire class (all instances) |

### Example: Double-Checked Locking Singleton

```java
public class SingletonClass {
    private static volatile SingletonClass instance;

    private SingletonClass() {}

    public static SingletonClass getInstance() {
        if (instance == null) {                    // First check (no lock)
            synchronized (SingletonClass.class) {  // Class-level lock
                if (instance == null) {            // Second check (inside lock)
                    instance = new SingletonClass();
                }
            }
        }
        return instance;
    }
}
```

> **Why `SingletonClass.class` instead of `this`?**
> - `this` only works in **instance methods**, but `getInstance()` is **static**.
> - `SingletonClass.class` ensures thread safety across **all instances**.

---

## Lock vs `synchronized`

| Feature | `synchronized` | `Lock` (ReentrantLock) |
|---------|---------------|----------------------|
| **Type** | Keyword (implicit) | Interface (explicit) |
| **Flexibility** | Limited | High (tryLock, timed lock) |
| **Fairness** | No guarantee | Can be fair |
| **Unlock** | Automatic on block exit | Manual (`unlock()` in finally) |
| **Interruptible** | No | Yes (`lockInterruptibly()`) |
| **Try Lock** | Not possible | `tryLock()` returns boolean |

### Simple Example

```java
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    // Using synchronized
    public synchronized void incrementSync() {
        count++;
    }

    // Using Lock
    public void incrementLock() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock(); // Always unlock in finally!
        }
    }
}
```

---

## Deadlock

A **deadlock** occurs when two or more threads are **waiting for each other's resources** indefinitely, preventing further execution.

### Example

```java
public class DeadlockExample {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {
                System.out.println("Thread 1: Holding Lock A...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (lockB) {
                    System.out.println("Thread 1: Holding Lock A & B");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (lockB) {
                System.out.println("Thread 2: Holding Lock B...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (lockA) {
                    System.out.println("Thread 2: Holding Lock B & A");
                }
            }
        });

        t1.start();
        t2.start();
    }
}
```

### Why Deadlock?
1. **Thread 1 locks A** -> waits for B
2. **Thread 2 locks B** -> waits for A
3. Both wait **indefinitely** -> **Deadlock!**

### How to Avoid
1. **Consistent Lock Ordering** - Always acquire locks in the same order
2. **Use `tryLock()` with timeout** - Avoid indefinite waiting
3. **Avoid nested locks** - Minimize lock scope
4. **Use higher-level concurrency utilities** - `java.util.concurrent`

---

## Read / Write Lock

`ReadWriteLock` allows **multiple readers OR one writer** at a time, improving performance for read-heavy workloads.

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

class SharedData {
    private int data = 0;
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public int readData() {
        rwLock.readLock().lock();  // Multiple threads can read simultaneously
        try {
            return data;
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void writeData(int value) {
        rwLock.writeLock().lock(); // Only one thread can write
        try {
            data = value;
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

| Feature | `synchronized` / `ReentrantLock` | `ReadWriteLock` |
|---------|----------------------------------|-----------------|
| Multiple Readers | No (only 1 thread) | Yes |
| Writer Exclusion | Yes | Yes |
| Best For | Write-heavy | Read-heavy workloads |

---

## `sleep()` vs `wait()` vs `notify()` vs `notifyAll()`

### Quick Summary

| Method | Lock Released? | Class | Wake Condition |
|--------|---------------|-------|----------------|
| `sleep(ms)` | **No** | `Thread` | After time elapses |
| `wait()` | **Yes** | `Object` | `notify()` / `notifyAll()` |
| `notify()` | - | `Object` | Wakes **one** waiting thread |
| `notifyAll()` | - | `Object` | Wakes **all** waiting threads |

![][image2]

### Simple Example: Producer-Consumer

```java
class SharedResource {
    private boolean dataReady = false;

    public synchronized void produce() throws InterruptedException {
        System.out.println("Producing data...");
        Thread.sleep(1000); // sleep does NOT release lock
        dataReady = true;
        notify(); // wakes up one waiting consumer
        System.out.println("Data produced & notified!");
    }

    public synchronized void consume() throws InterruptedException {
        while (!dataReady) {
            System.out.println("Waiting for data...");
            wait(); // releases lock and waits
        }
        System.out.println("Data consumed!");
    }
}

public class Main {
    public static void main(String[] args) {
        SharedResource resource = new SharedResource();

        new Thread(() -> { try { resource.consume(); } catch (InterruptedException e) {} }).start();
        new Thread(() -> { try { resource.produce(); } catch (InterruptedException e) {} }).start();
    }
}
```

### Key Difference: `sleep()` vs `wait()`

| Feature | `sleep()` | `wait()` |
|---------|-----------|----------|
| Lock | Does **NOT** release | **Releases** lock |
| Class | `Thread` | `Object` |
| Usage | Anywhere | Inside `synchronized` block only |
| Wake up | After time expires | On `notify()` / `notifyAll()` |
| Purpose | Pause execution | Inter-thread communication |

![][image4]

---

## What is Immutable Class? (Thread Safety)

An **immutable class** is a class whose instances **cannot be modified** after creation. They are inherently **thread-safe** because no thread can change their state.

### Rules to Create Immutable Class
1. Declare class as `final`
2. Make all fields `private` and `final`
3. No setter methods
4. Initialize all fields via constructor
5. Return deep copies of mutable objects (defensive copy)

### Example

```java
public final class ImmutablePerson {
    private final String name;
    private final int age;
    private final List<String> hobbies;

    public ImmutablePerson(String name, int age, List<String> hobbies) {
        this.name = name;
        this.age = age;
        this.hobbies = Collections.unmodifiableList(new ArrayList<>(hobbies)); // defensive copy
    }

    public String getName() { return name; }
    public int getAge() { return age; }
    public List<String> getHobbies() { return hobbies; } // already unmodifiable
}
```

> **Why Thread-Safe?** Since state can't change after construction, multiple threads can read without synchronization. Examples: `String`, `Integer`, `LocalDate`.

---

## `volatile` Keyword

The `volatile` keyword ensures **visibility** of variable changes across threads:
- Every **read** comes from main memory (not CPU cache)
- Every **write** goes to main memory immediately
- Prevents **instruction reordering**

```java
class StoppableThread extends Thread {
    private volatile boolean running = true;

    public void run() {
        while (running) {  // Always reads fresh value from main memory
            // do work
        }
        System.out.println("Thread stopped.");
    }

    public void stopThread() {
        running = false;  // Immediately visible to other threads
    }
}
```

> **Note:** `volatile` guarantees **visibility** but NOT **atomicity**. For compound operations (like `count++`), use `AtomicInteger` or `synchronized`.

---

## What is `ExecutorService`?

`ExecutorService` is a framework for **managing and reusing threads** via a thread pool. It decouples task submission from thread management.

### Key Methods

| Method | Description |
|--------|-------------|
| `submit(Callable/Runnable)` | Submits a task, returns `Future` |
| `execute(Runnable)` | Executes task, no return value |
| `shutdown()` | Graceful shutdown (finishes running tasks) |
| `shutdownNow()` | Forceful shutdown (interrupts tasks) |
| `invokeAll(tasks)` | Submits all tasks, waits for all to complete |

### Types of Thread Pools

```java
// Fixed thread pool - exactly N threads
ExecutorService fixed = Executors.newFixedThreadPool(10);

// Cached thread pool - creates threads as needed, reuses idle ones
ExecutorService cached = Executors.newCachedThreadPool();

// Single thread executor - only 1 thread
ExecutorService single = Executors.newSingleThreadExecutor();

// Scheduled thread pool - for delayed/periodic tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);
```

### Example

```java
ExecutorService executor = Executors.newFixedThreadPool(5);

for (int i = 0; i < 20; i++) {
    final int taskId = i;
    executor.submit(() -> {
        System.out.println("Task " + taskId + " by " + Thread.currentThread().getName());
    });
}

executor.shutdown();
```

---

## `Runnable` vs `Callable`

| Feature | `Runnable` | `Callable<T>` |
|---------|-----------|--------------|
| Return Type | `void` (no return) | Generic `<T>` (returns result) |
| Method | `run()` | `call()` |
| Exceptions | Cannot throw checked exceptions | Can throw checked exceptions |
| Result | Cannot get result | Result via `Future<T>` |

### When to use `Callable`?
- Need to **return a result**
- Need to **handle checked exceptions**
- Need **asynchronous computation with output**

### Example

```java
import java.util.concurrent.*;

class MyCallable implements Callable<Integer> {
    public Integer call() throws Exception {
        return 10 * 2; // Returns 20
    }
}

public class CallableExample {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<Integer> future = executor.submit(new MyCallable());

        System.out.println("Result: " + future.get()); // Blocks until result available -> 20
        executor.shutdown();
    }
}
```

---

## What is a Race Condition?

A **race condition** occurs when **multiple threads access shared data simultaneously** and the **final outcome depends on execution order** (non-deterministic).

![][image3]

### Example

```java
class Counter {
    private int count = 0;

    public void increment() {
        count++; // NOT atomic! (read -> modify -> write)
    }

    public int getCount() { return count; }
}
```

If two threads call `increment()` simultaneously, `count` may be 1 instead of 2.

### How to Prevent

**1. Use `synchronized`**
```java
public synchronized void increment() {
    count++;
}
```

**2. Use `AtomicInteger` (Lock-Free)**
```java
private AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet();
}
```

**3. Use `ReentrantLock`**
```java
private final ReentrantLock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try { count++; }
    finally { lock.unlock(); }
}
```

---

## What is `Future`?

- `Future` represents the **result of an async computation** that will be available *sometime in the future*.
- You submit a task to `ExecutorService`, it returns a `Future` — you can check later if it's done or get the result.
- **Problem:** `future.get()` is **blocking** — your thread just sits and waits. No chaining, no callbacks.

### Simple Example

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Future<String> future = executor.submit(() -> {
    Thread.sleep(2000);
    return "Result ready!";
});

// Do other work here...

String result = future.get(); // BLOCKS until result is available
System.out.println(result);
executor.shutdown();
```

### Limitations of `Future` (Why `CompletableFuture` was created)

| Problem | Description |
|---------|-------------|
| **Blocking** | `get()` blocks the calling thread |
| **No chaining** | Can't do "when done, do this next" |
| **No combining** | Can't easily merge results of multiple futures |
| **No exception handling** | No `exceptionally()` or fallback mechanism |
| **Can't complete manually** | Once submitted, you can't set the result yourself |

**Real-life analogy:** `Future` is like ordering food and **standing at the counter** until it's ready. `CompletableFuture` is like getting a **buzzer** — you go do other things and it notifies you.

---

## What is `CompletableFuture`?

- It's an **upgrade over `Future`** — introduced in Java 8
- With plain `Future`, you call `future.get()` which **blocks** the thread. No chaining, no callbacks.
- `CompletableFuture` solves this — you can **chain tasks**, **combine results**, and **handle errors** without blocking.
- Think of it as **Promises in JavaScript** but for Java.
- Internally uses **ForkJoinPool** (common pool) for async execution.

**When to use:** Calling multiple APIs in parallel, non-blocking I/O, async pipelines.

### Real-Life Scenarios

| Scenario | Why CompletableFuture? |
|----------|----------------------|
| REST API calling 3 microservices | Call all 3 in parallel, combine results — response time = max of 3, not sum |
| E-commerce checkout | Validate payment + check inventory + apply coupon — all async |
| Notification system | Send email + SMS + push notification — don't wait for each |
| Report generation | Fetch data from multiple DBs in parallel, merge into one report |

### Simple Example

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // runs in ForkJoinPool (separate thread)
    return "Hello from async!";
});

// Chain: transform result -> consume it
future.thenApply(result -> result.toUpperCase())
      .thenAccept(System.out::println); // HELLO FROM ASYNC!

future.join(); // non-blocking wait (unlike Future.get(), doesn't throw checked exceptions)
```

### Combining Multiple Futures

```java
CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> 20);

f1.thenCombine(f2, (a, b) -> a + b)
  .thenAccept(System.out::println); // 30
```

### Error Handling

```java
CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("Something broke!");
}).exceptionally(ex -> {
    return "Fallback value"; // graceful recovery
}).thenAccept(System.out::println); // Fallback value
```

### Key Methods Cheat Sheet

| Method | What it does |
|--------|-------------|
| `supplyAsync(() -> ...)` | Run task async, returns result |
| `thenApply(fn)` | Transform result (like `.map()`) |
| `thenAccept(fn)` | Consume result (no return) |
| `thenCombine(other, fn)` | Combine 2 futures |
| `allOf(f1, f2, f3)` | Wait for ALL to complete |
| `anyOf(f1, f2, f3)` | Wait for FIRST to complete |
| `exceptionally(fn)` | Handle errors with fallback |
| `join()` | Block and get result (no checked exception) |

### Tricky Follow-up Questions

> **Q: What thread pool does `supplyAsync()` use by default?**
> ForkJoinPool.commonPool(). You can pass a custom executor as 2nd argument.

> **Q: Difference between `thenApply()` vs `thenApplyAsync()`?**
> `thenApply()` runs on the **same thread** that completed the previous stage. `thenApplyAsync()` runs on a **different thread** from the pool.

> **Q: What happens if you don't call `join()` or `get()`?**
> The main thread may exit before the async task finishes. In a web server (Spring Boot), this isn't an issue because the server keeps running.

> **Q: `Future.get()` vs `CompletableFuture.join()` ?**
> `get()` throws **checked** `InterruptedException` + `ExecutionException`. `join()` throws **unchecked** `CompletionException`. Use `join()` in lambdas/streams.

---

## Limiting Threads: 1000 Tasks with Only 100 Threads

**Q: I have 1000 tasks but want to limit concurrent execution to 100 threads. How?**

- The idea is simple: **don't create 1000 threads** — create a **pool of 100** and queue the rest.
- The pool picks up new tasks as threads finish their current work.
- This is a very common interview question for **backend/microservice roles**.

### Solution 1: `ExecutorService` with Fixed Thread Pool (Recommended)

```java
ExecutorService executor = Executors.newFixedThreadPool(100); // only 100 threads

for (int i = 0; i < 1000; i++) {
    final int taskId = i;
    executor.submit(() -> {
        System.out.println("Task " + taskId + " on " + Thread.currentThread().getName());
    });
}

executor.shutdown();
executor.awaitTermination(1, TimeUnit.HOURS);
```

### Solution 2: Using `Semaphore` (When you need finer control)

```java
Semaphore semaphore = new Semaphore(100); // max 100 concurrent
ExecutorService executor = Executors.newCachedThreadPool();

for (int i = 0; i < 1000; i++) {
    executor.submit(() -> {
        semaphore.acquire();  // blocks if 100 already running
        try {
            // do task
        } finally {
            semaphore.release(); // free permit for next task
        }
    });
}
executor.shutdown();
```

### Solution 3: `ThreadPoolExecutor` (Custom control)

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    100, 100,                      // core & max pool size
    60L, TimeUnit.SECONDS,         // idle thread keep-alive
    new LinkedBlockingQueue<>(900) // queue for remaining tasks
);
```

| Approach | Best When |
|----------|-----------|
| `FixedThreadPool` | Simple, most common solution |
| `Semaphore` | Need to limit concurrency in specific code sections |
| `ThreadPoolExecutor` | Need fine-grained control (queue size, rejection policy) |

### Tricky Follow-up Questions

> **Q: What happens when the queue is full in `ThreadPoolExecutor`?**
> It uses a **RejectedExecutionHandler**. Default is `AbortPolicy` (throws exception). Others: `CallerRunsPolicy` (caller thread runs the task), `DiscardPolicy`, `DiscardOldestPolicy`.

> **Q: `FixedThreadPool` vs `CachedThreadPool` — when to use which?**
> Fixed = predictable load, bounded resources. Cached = short-lived burst tasks, but **dangerous** — can create unlimited threads and crash with OOM.

> **Q: What if a task throws an exception inside the pool?**
> The thread dies silently (with `execute()`). Use `submit()` + `future.get()` to catch exceptions. Or set an `UncaughtExceptionHandler`.

---

## Tricky Interview Questions (Real-Life Scenarios)

> **Tip:** In interviews, don't just name the class — explain **why** that solution fits. Interviewers want to see your thought process.

### Q1: How would you design a thread-safe counter for a high-traffic web application?

**Answer:**
- `synchronized` works but is **slow under high contention** (threads keep waiting).
- `AtomicLong` is better — uses **CAS (Compare-And-Swap)**, no locks.
- For **very high contention** (millions of requests), use `LongAdder` — it splits the counter into cells and sums them on read.

```java
// Good
private AtomicLong counter = new AtomicLong(0);
counter.incrementAndGet();

// Best for HIGH contention
private LongAdder counter = new LongAdder();
counter.increment();
```

> **Follow-up: Why not just use `synchronized`?** Every thread has to wait in line. With `AtomicLong`, threads retry using CAS — no blocking. `LongAdder` is even better because each thread writes to its own cell, reducing contention to near-zero.

### Q2: Two threads are reading from a database and writing to a cache. How do you ensure the cache is updated atomically?

**Answer:**
- If you use `synchronized`, readers also get blocked — bad for performance.
- Use `ReadWriteLock` — **multiple readers can read simultaneously**, but writer gets **exclusive access**.
- This is the go-to pattern for **caching scenarios**.

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

// Reading (multiple threads can do this together)
rwLock.readLock().lock();
try { return cache.get(key); }
finally { rwLock.readLock().unlock(); }

// Writing (exclusive - blocks all readers)
rwLock.writeLock().lock();
try { cache.put(key, value); }
finally { rwLock.writeLock().unlock(); }
```

> **Follow-up: Can a reader upgrade to writer?** No! If a reader tries to acquire write lock while holding read lock → **deadlock**. You must release read lock first, then acquire write lock.

### Q3: You have a REST API that calls 3 external services. How do you call them in parallel and combine results?

**Answer:**
- Sequential calls = total time is **sum** of all three. Parallel = total time is **max** of the three.
- Use `CompletableFuture.supplyAsync()` for each call, then `allOf().join()` to wait.

```java
CompletableFuture<User> userF = CompletableFuture.supplyAsync(() -> userService.get(id));
CompletableFuture<List<Order>> ordersF = CompletableFuture.supplyAsync(() -> orderService.get(id));
CompletableFuture<Address> addressF = CompletableFuture.supplyAsync(() -> addressService.get(id));

CompletableFuture.allOf(userF, ordersF, addressF).join(); // wait for all

return new UserProfile(userF.get(), ordersF.get(), addressF.get());
```

> **Follow-up: What if one service fails?** Use `handle()` or `exceptionally()` on each future to return a default value. Don't let one failure kill the whole response.

### Q4: A singleton is being created by multiple threads simultaneously. How do you ensure only ONE instance?

**Answer:**
- Simple `synchronized` on the whole method works but is **slow** (lock acquired every time).
- **Double-checked locking** — only synchronizes on first creation.
- `volatile` is critical here — without it, a thread might see a **partially constructed** object.

```java
private static volatile Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {                  // 1st check - no lock
        synchronized (Singleton.class) {
            if (instance == null) {           // 2nd check - inside lock
                instance = new Singleton();
            }
        }
    }
    return instance;
}
```

> **Follow-up: Why do we need `volatile` here?** Without it, `instance = new Singleton()` can be reordered by JVM — another thread might see a non-null reference to an **incompletely constructed** object.

> **Follow-up: Easier alternative?** Use **enum singleton** (`enum Singleton { INSTANCE; }`) — JVM guarantees single instance, thread-safe, serialization-safe.

### Q5: You're processing 10,000 orders. If one fails, you don't want to stop others. How do you handle this?

**Answer:**
- Key insight: **isolate failures**. Each task runs independently.
- Submit all tasks, collect `Future`s, handle exceptions **per task** in a loop.
- Never let one `ExecutionException` kill your entire batch.

```java
ExecutorService executor = Executors.newFixedThreadPool(50);
List<Future<Result>> futures = new ArrayList<>();

for (Order order : orders) {
    futures.add(executor.submit(() -> process(order)));
}

for (Future<Result> f : futures) {
    try {
        Result r = f.get(); // handle success
    } catch (ExecutionException e) {
        log.error("Failed: " + e.getCause().getMessage());
        // continue — don't break!
    }
}
executor.shutdown();
```

> **Follow-up: What if you want to retry failed orders?** Collect failed orders in a separate list, then resubmit them with a delay (exponential backoff). Or use `CompletableFuture` with `.exceptionally()` that retries.

### Q6: How would you implement a rate limiter that allows max 100 requests per second?

**Answer:**
- `Semaphore` with 100 permits — each request acquires one, a scheduled task resets them every second.
- In production, prefer libraries like **Guava's RateLimiter** or **Resilience4j**.
- Key: `tryAcquire()` (non-blocking) vs `acquire()` (blocking).

```java
class RateLimiter {
    private final Semaphore semaphore = new Semaphore(100);
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    public RateLimiter() {
        scheduler.scheduleAtFixedRate(() -> {
            semaphore.release(100 - semaphore.availablePermits());
        }, 1, 1, TimeUnit.SECONDS);
    }

    public boolean allowRequest() {
        return semaphore.tryAcquire();
    }
}
```

> **Follow-up: What's the difference between rate limiting and throttling?** Rate limiting = hard cap (reject excess). Throttling = slow down (queue/delay excess). Both protect the system but behave differently for the client.

### Q7: Thread A must wait for Thread B and Thread C to finish before proceeding. How?

**Answer:**
- `CountDownLatch(2)` — each thread counts down on finish. A calls `await()` and blocks until count = 0.
- Alternative: `CompletableFuture.allOf()` if you're already using CompletableFuture.
- **Don't confuse with `CyclicBarrier`** — that's for threads waiting for *each other* (reusable).

```java
CountDownLatch latch = new CountDownLatch(2);

new Thread(() -> { doWork(); latch.countDown(); }).start(); // B
new Thread(() -> { doWork(); latch.countDown(); }).start(); // C

latch.await(); // A waits here until count = 0
```

> **Follow-up: `CountDownLatch` vs `CyclicBarrier`?**
> - `CountDownLatch` = one-shot, count goes down, can't reset.
> - `CyclicBarrier` = reusable, all threads wait for each other at a barrier point (like gaming lobbies waiting for all players).

### Q8: How do you detect and resolve a deadlock in production?

**Answer:**
1. **Detect:** `jstack <pid>` (CLI) or programmatically:
```java
long[] ids = ManagementFactory.getThreadMXBean().findDeadlockedThreads();
```
2. **Resolve:** Restart affected threads or the app.
3. **Prevent:** Lock ordering, `tryLock()` with timeout, avoid nested locks.

> **Follow-up: Can JVM resolve deadlocks automatically?** No. Unlike databases (which can detect and kill one transaction), JVM deadlocks are permanent unless you restart or the threads are interrupted externally.

### Q9: How do you make a HashMap thread-safe?

**Answer:**
- **`ConcurrentHashMap`** (best) — segment-level locking, concurrent reads + writes.
- `Collections.synchronizedMap()` — wraps with single lock, slower.
- Key interview point: `ConcurrentHashMap` **does NOT allow null keys/values** (unlike HashMap).

```java
Map<String, String> map = new ConcurrentHashMap<>();  // Best
Map<String, String> map2 = Collections.synchronizedMap(new HashMap<>());  // OK
```

> **Follow-up: Why doesn't `ConcurrentHashMap` allow null?** Because `map.get(key)` returning `null` would be ambiguous — is the key absent, or is the value actually null? In concurrent code, you can't use `containsKey()` + `get()` atomically.

> **Follow-up: Is `ConcurrentHashMap.size()` accurate?** No! It's an **estimate** in concurrent scenarios. Use it for monitoring, not for logic.

### Q10: What happens if you call `run()` instead of `start()` on a thread?

**Answer:**
- `run()` = just a **normal method call** on the current thread. No new thread created.
- `start()` = JVM creates a **new OS thread**, then calls `run()` inside it.
- This is a classic **trick question** — interviewers want to see if you know the difference.

```java
Thread t = new Thread(() -> System.out.println(Thread.currentThread().getName()));
t.run();   // prints: main (same thread!)
t.start(); // prints: Thread-0 (new thread)
```

> **Follow-up: Can you call `start()` twice on the same thread?** No! Throws `IllegalThreadStateException`. Once a thread is TERMINATED, it cannot be restarted. You must create a new `Thread` object.

> **Follow-up: What happens if you call `start()` after `run()`?** It still works — `run()` was just a regular method call, it didn't change the thread's state. But calling `start()` twice throws an exception.

---

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,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>

[image4]: <data:image/png;base64,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>