# Senior Java Developer — Interview Question Prep

A curated set of interview questions with senior-level answers covering Multithreading, JVM internals, Java 8, Spring Boot, Exception Handling, JPA/Hibernate, and Microservices.

---

## Table of Contents

- [Multithreading & Concurrency](#multithreading--concurrency)
- [Exception Handling (Core)](#exception-handling-core)
- [JVM, Memory & Garbage Collection](#jvm-memory--garbage-collection)
- [Java 8 Features](#java-8-features)
- [Spring Boot Internals](#spring-boot-internals)
- [Architecture, Environment & Configuration](#architecture-environment--configuration)
- [Exception Handling (REST / Web)](#exception-handling-rest--web)
- [JPA & Hibernate](#jpa--hibernate)
- [Microservices](#microservices)
- [Database Idempotency & Optimization](#database-idempotency--optimization)
- [Distributed Tracing & Design](#distributed-tracing--design)

---

## Multithreading & Concurrency

### Q2. Did you write any multi-threaded code? Have you faced any concurrency bug and how did you find and fix it?

Yes — concurrency bugs appear in almost every production backend.

**Real-world scenario:** A shared in-memory cache (a plain `HashMap`) was accessed by multiple request-handling threads. Intermittently the app spiked to 100% CPU (infinite loop) and occasionally returned corrupted data.

**How I found it:**
1. **Thread dump (`jstack`)** — multiple threads stuck in `HashMap.get()` / `HashMap.put()`. Classic symptom: concurrent modification during internal table resize corrupts the linked list → infinite loop.
2. **Code review** confirmed there was no synchronization around the shared map.

**How I fixed it:**
- Replaced `HashMap` with `ConcurrentHashMap` — solved the race without a global lock.
- Wrote tests using `CountDownLatch` / `CyclicBarrier` to force concurrent access and reproduce the bug deterministically.

**Key lessons:**
- Race conditions are **non-deterministic** — they don't fail every run, making them hard to catch in QA.
- Always use **thread-safe collections** (`ConcurrentHashMap`, `CopyOnWriteArrayList`) for shared state.
- **Thread dumps** (`jstack`, VisualVM, async-profiler) are your best diagnostic tool.
- Static analysis (SpotBugs, IntelliJ inspections) catches unsafe sharing early.

---

### Q3. You need to execute 1,000 tasks in parallel but don't want to create 1,000 threads. How would you design this?

**Core idea:** Don't create 1,000 threads — create a **bounded pool** and queue the rest. Threads pick up new tasks as they finish.

**Solution 1 — Fixed thread pool (recommended):**
```java
ExecutorService executor = Executors.newFixedThreadPool(100); // only 100 threads

for (int i = 0; i < 1000; i++) {
    final int taskId = i;
    executor.submit(() -> process(taskId));
}

executor.shutdown();
executor.awaitTermination(1, TimeUnit.HOURS);
```

**Solution 2 — `Semaphore` for fine-grained concurrency control:**
```java
Semaphore semaphore = new Semaphore(100);          // max 100 concurrent
ExecutorService executor = Executors.newCachedThreadPool();

for (int i = 0; i < 1000; i++) {
    executor.submit(() -> {
        semaphore.acquire();
        try { process(); }
        finally { semaphore.release(); }
    });
}
executor.shutdown();
```

**Solution 3 — `ThreadPoolExecutor` for full control:**
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    100, 100,                       // core & max pool size
    60L, TimeUnit.SECONDS,          // idle keep-alive
    new LinkedBlockingQueue<>(900), // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy() // backpressure
);
```

| Approach | Best When |
|----------|-----------|
| `FixedThreadPool` | Simple, most common |
| `Semaphore` | Limit concurrency in a specific code section |
| `ThreadPoolExecutor` | Need queue size + rejection policy control |

> **Follow-up — queue full in `ThreadPoolExecutor`?** A `RejectedExecutionHandler` kicks in. Default `AbortPolicy` throws; `CallerRunsPolicy` runs the task on the caller thread (natural backpressure).

> **Follow-up — Fixed vs Cached pool?** Fixed = predictable, bounded resources. Cached = bursty short tasks, but **dangerous** — can spawn unlimited threads and cause OOM.

---

### Q4. In Spring Boot, how would you configure an async task executor with `@Async`, and what happens if you do not configure a custom one?

**Default behavior (no custom executor):**
- Spring Boot falls back to `SimpleAsyncTaskExecutor` — it creates a **new thread per task** (no reuse, no queue, no pool).
- **Dangerous under load** — can spawn thousands of threads → `OutOfMemoryError`.

**Custom executor:**
```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

**Usage:**
```java
@Service
public class NotificationService {
    @Async("taskExecutor")
    public CompletableFuture<String> sendEmail(String to) {
        return CompletableFuture.completedFuture("Sent to " + to);
    }
}
```

**Key points:**
- `@EnableAsync` is required on a `@Configuration` class.
- The `@Async` method must be `public` and **called from outside the class** (proxy-based AOP — self-invocation is bypassed).
- Return `void` or `CompletableFuture<T>` so the caller can track completion/exceptions.
- Unhandled exceptions in `@Async void` are silently swallowed unless you set an `AsyncUncaughtExceptionHandler`.

> **Follow-up — `@Async` on a private method?** Won't work; AOP proxies only intercept public, externally-called methods.

---

### Q5. What's the difference between `Callable` and `Runnable`?

| Feature | `Runnable` | `Callable<T>` |
|---------|-----------|--------------|
| Return type | `void` | Generic `<T>` (returns a result) |
| Method | `run()` | `call()` |
| Checked exceptions | Cannot throw | Can throw |
| Result retrieval | None | Via `Future<T>` |
| Since | Java 1.0 | Java 5 |

```java
Runnable r = () -> System.out.println("No result");

Callable<Integer> c = () -> {
    return 10 * 2; // returns 20, can throw checked exceptions
};

ExecutorService ex = Executors.newSingleThreadExecutor();
Future<Integer> future = ex.submit(c);
System.out.println(future.get()); // 20
ex.shutdown();
```

**Use `Callable` when** you need a return value, need to throw checked exceptions, or want async computation with output.

---

### Q6. How would you run two async tasks in parallel and combine their results when both complete?

Use `CompletableFuture.supplyAsync()` per task, then `thenCombine()`:

```java
CompletableFuture<UserProfile> userF = CompletableFuture.supplyAsync(
    () -> userService.fetchProfile(userId), executor);

CompletableFuture<List<Order>> orderF = CompletableFuture.supplyAsync(
    () -> orderService.fetchOrders(userId), executor);

CompletableFuture<UserDashboard> combined =
    userF.thenCombine(orderF, (profile, orders) -> new UserDashboard(profile, orders));

UserDashboard dashboard = combined.join();
```

**For 3+ futures — use `allOf()`:**
```java
CompletableFuture.allOf(userF, orderF).join(); // both guaranteed complete
UserProfile p = userF.join();
List<Order> o = orderF.join();
```

**Key points:**
- `thenCombine()` — best for exactly 2 futures.
- `allOf().join()` — best for 3+, returns `Void`, so call `.join()` on each after.
- Always pass a **custom executor** — don't overload the shared `ForkJoinPool.commonPool()`.
- Use `.exceptionally()` / `.handle()` so one failure doesn't crash the pipeline.

> **Follow-up — total time?** `max(task1, task2)`, not the sum. That's the entire point of parallelism.

---

### Q7. What's the difference between a synchronized method and a synchronized block, and when would you choose one?

| Aspect | Synchronized Method | Synchronized Block |
|--------|--------------------|--------------------|
| Lock object | `this` (instance) / `Class.class` (static) | Any object you specify |
| Scope | Entire method | Only the critical section |
| Granularity | Coarse | Fine |
| Flexibility | Limited | High |

```java
// Method — locks entire body on 'this'
public synchronized void transfer(int amount) {
    this.balance -= amount;
}

// Block — minimal scope, custom lock
private final Object lock = new Object();
public void transfer(int amount) {
    // non-critical work runs concurrently
    synchronized (lock) {
        this.balance -= amount;
    }
}
```

**Choose:**
- **Method** — entire method is the critical section (simple cases).
- **Block** — to lock a **specific/private object**, minimize lock scope, or maximize concurrency.

> **Follow-up — why a private lock instead of `this`?** External code could `synchronized(yourObj)` and cause unexpected contention/deadlock. A private lock is encapsulated.

---

## Exception Handling (Core)

### Q8. A method throws a checked exception but the interface it implements does not declare it. How would you handle this?

**1. Wrap in an unchecked exception (most common):**
```java
public class FileProcessor implements DataProcessor {
    @Override
    public void process(String data) {
        try {
            Files.write(Path.of("output.txt"), data.getBytes()); // IOException
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to write file", e);
        }
    }
}
```

**2. Custom unchecked exception:**
```java
catch (SQLException e) {
    throw new ProcessingException("DB operation failed", e);
}
```

**3. Sneaky throws (discouraged):** Lombok's `@SneakyThrows` — breaks the contract, surprises callers.

**Best practice:**
- **Always preserve the cause** (exception chaining).
- Make the wrapping exception meaningful.
- Document runtime exceptions in Javadoc since they're not in the signature.

> **Follow-up — why not declare `throws Exception` on the interface?** It forces every caller and every implementation to handle it — lazy API design.

---

### Q9. Have you ever created a custom exception hierarchy?

Yes — essential for clean enterprise error handling.

```java
public abstract class ApplicationException extends RuntimeException {
    private final ErrorCode errorCode;
    private final HttpStatus httpStatus;

    protected ApplicationException(String msg, ErrorCode code, HttpStatus status) {
        super(msg);
        this.errorCode = code;
        this.httpStatus = status;
    }
    protected ApplicationException(String msg, Throwable cause, ErrorCode code, HttpStatus status) {
        super(msg, cause);
        this.errorCode = code;
        this.httpStatus = status;
    }
    public ErrorCode getErrorCode() { return errorCode; }
    public HttpStatus getHttpStatus() { return httpStatus; }
}

public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String resource, Long id) {
        super(resource + " not found with id: " + id,
              ErrorCode.RESOURCE_NOT_FOUND, HttpStatus.NOT_FOUND);
    }
}
```

**Handle centrally:**
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ApplicationException.class)
    public ResponseEntity<ErrorResponse> handle(ApplicationException ex) {
        return ResponseEntity.status(ex.getHttpStatus())
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
    }
}
```

**Principles:** unchecked base, carry metadata (code + HTTP status + context), hierarchical so one handler catches all, never expose internal details to clients.

---

### Q10. What is exception chaining and why is it important?

**Exception chaining** = wrapping a caught exception as the `cause` of a new one, preserving the full error trail.

```java
try {
    statement.executeQuery(sql);
} catch (SQLException e) {
    throw new DataAccessException("Failed to execute query: " + sql, e); // 'e' preserved
}
```

**Why it matters:**
1. **Root-cause traceability** — the top-level generic exception still carries the real cause (e.g., `SQLSyntaxErrorException: Unknown column`).
2. **Full stack trace** with `caused by:` chains.
3. **Layer separation** — DAO wraps `SQLException` → `DataAccessException`; service wraps that → `ServiceException`.

**Anti-patterns:**
```java
catch (IOException e) { throw new RuntimeException("Something failed"); } // cause LOST
catch (Exception e) { e.printStackTrace(); }                              // invisible in prod
catch (Exception e) { return null; }                                      // hides failure → NPE later
```

> **Follow-up — access the chain?** `Throwable.getCause()` recursively, or just log it — SLF4J/Logback print the full chain automatically.

---

## JVM, Memory & Garbage Collection

### Q11. Explain the Java Memory Model and where does an object live?

The **Java Memory Model (JMM)** defines how threads interact through memory — specifically **visibility**, **atomicity**, and **ordering** guarantees.

**Runtime memory regions:**

| Region | Stores | Shared? |
|--------|--------|---------|
| **Heap** | All objects, instance fields, arrays | Shared across threads |
| **Stack** | Method frames, local variables, references | Per-thread (private) |
| **Metaspace** | Class metadata, static variables | Shared (native memory) |
| **PC Register** | Current instruction address | Per-thread |
| **Native Method Stack** | Native (JNI) calls | Per-thread |

**Where does an object live?** Always on the **heap**. The stack only holds the **reference** (pointer) to it.

```java
void m() {
    Order order = new Order(); // 'order' reference → stack; Order object → heap
}
```

**JMM visibility concepts:**
- **happens-before** — the core ordering guarantee. A write in one thread is visible to another only if a happens-before relationship exists (e.g., `volatile` write/read, lock release/acquire, thread start/join).
- **`volatile`** — guarantees visibility and prevents reordering, but not atomicity for compound ops.
- **CPU caches** — without proper synchronization, one thread may not see another's write because values sit in CPU caches, not main memory.

> **Follow-up — String literals?** In the **String Pool** (part of the heap since Java 7). `new String("x")` always creates a separate heap object.

---

### Q12. What is the difference between heap memory and stack memory?

| Aspect | Stack | Heap |
|--------|-------|------|
| Stores | Method frames, locals, references | Objects, instance variables |
| Scope | Thread-private | Shared across threads |
| Lifetime | Freed on method return (LIFO) | Freed by Garbage Collector |
| Size | Small (~512KB–1MB/thread) | Large (`-Xmx`) |
| Speed | Very fast | Slower (GC + allocation) |
| Error | `StackOverflowError` | `OutOfMemoryError` |
| Thread safety | Inherently safe (private) | Needs synchronization |

```java
public void process(Order order) {     // 'order' reference → stack
    int qty = order.getQuantity();      // primitive → stack
    List<Item> items = new ArrayList<>(); // reference → stack, ArrayList → heap
}
// On return: stack frame popped; heap objects eligible for GC if unreferenced
```

- **Primitives** as locals live on the stack; **objects** always on the heap.
- **Static variables** live in **Metaspace**.

> **Follow-up — control stack size?** `-Xss` (e.g., `-Xss512k`). Bigger stacks help deep recursion but reduce the max number of threads.

---

### Q13. Have you ever faced `OutOfMemoryError` in production?

Yes — one of the most common production issues.

**Scenario:** A Spring Boot service threw `OutOfMemoryError: Java heap space` after days of uptime (a slow memory leak).

**Diagnosis:**
1. Enabled auto heap dump: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/logs/heap.hprof`
2. Analyzed with **Eclipse MAT** — found a `HashMap` holding 2M session objects that were never evicted.
3. **Root cause:** a custom cache with no TTL and no size limit grew unbounded.

**Fix:**
- Replaced raw map with **Caffeine** cache: `maximumSize(10_000)` + `expireAfterAccess(30, MINUTES)`.
- Exposed cache size via Micrometer → Grafana dashboards + alerts.
- Set `-Xmx2g -Xms2g` (min = max to avoid resize pauses).

**Common causes:**

| Cause | Fix |
|-------|-----|
| Unbounded cache/collection | Bounded caches with eviction |
| Resource/connection leaks | try-with-resources |
| Large query results in memory | Pagination / streaming |
| Too many threads | Thread pools |
| Classloader leaks (redeploys) | Fix static refs, clean restart |

> **Follow-up — heap OOM vs Metaspace OOM?** Heap = too many objects; Metaspace = too many loaded classes (heavy reflection, dynamic proxies, classloader leaks).

---

### Q14. How did you debug a memory leak and what tools did you use?

**Step-by-step approach:**
1. **Confirm the leak** — monitor heap over time (JConsole, VisualVM, Grafana). A leak shows a **sawtooth that trends upward** — memory not reclaimed after GC.
2. **Capture a heap dump** — `jmap -dump:live,format=b,file=heap.hprof <pid>` or auto on OOM.
3. **Analyze with Eclipse MAT** — use the **Leak Suspects report** and **Dominator Tree** to find which objects retain the most memory.
4. **Find GC roots** — "Path to GC Roots" shows *why* an object isn't collected (what's still referencing it).
5. **Fix and verify** — remove the offending reference, re-run load test, confirm flat memory.

**Tools:**
- **Eclipse MAT** — best for heap dump analysis (leak suspects, dominator tree).
- **VisualVM / JConsole** — live monitoring, sampling.
- **async-profiler / Java Flight Recorder (JFR)** — low-overhead allocation profiling in production.
- **`jmap`, `jstat`, `jcmd`** — CLI heap dumps and GC stats.

**Common leak sources:** static collections that only grow, unclosed resources, listeners/callbacks never deregistered, `ThreadLocal` not cleaned in pooled threads, caches without eviction.

> **Tip:** `ThreadLocal` leaks are subtle — in a thread pool, threads live forever, so a forgotten `ThreadLocal.remove()` keeps objects alive indefinitely.

---

### Q15. We have a garbage collector — why do we still see memory leaks?

Because **GC only reclaims unreachable objects**. A memory leak in Java = objects that are **still reachable** (referenced) but **logically no longer needed**.

```java
class Cache {
    // grows forever, nothing ever removed → reachable but useless = leak
    private static final Map<String, Object> CACHE = new HashMap<>();
    public void add(String k, Object v) { CACHE.put(k, v); }
}
```

**Classic causes of reachable-but-unneeded objects:**
- **Static collections** that only grow.
- **Unclosed resources** (streams, connections) held by references.
- **Listeners/observers** never deregistered.
- **`ThreadLocal`** not cleared in pooled threads.
- **Inner classes / lambdas** capturing an outer reference longer than expected.

**The fix is always:** remove the reference when the object is no longer needed — eviction policies, `remove()`, try-with-resources, weak references (`WeakHashMap`).

> **Key insight:** GC solves *dangling pointers* and *manual free* errors, not *logical* leaks. The developer still owns object lifecycle decisions.

---

### Q16. Difference between Minor GC and Major (Full) GC?

The heap is split into **Young Generation** (Eden + two Survivor spaces) and **Old/Tenured Generation**.

| Aspect | Minor GC | Major / Full GC |
|--------|----------|-----------------|
| Region | Young Generation | Old Generation (+ often whole heap) |
| Trigger | Eden fills up | Old gen fills / promotion failure |
| Frequency | Frequent | Infrequent |
| Speed | Fast (short pause) | Slow (long stop-the-world) |
| What it collects | Short-lived objects | Long-lived objects |

**Object lifecycle:**
1. New objects → **Eden**.
2. Minor GC: survivors copied to a **Survivor** space; Eden cleared.
3. Objects surviving enough minor GCs are **promoted** to the **Old Generation**.
4. **Major/Full GC** cleans the Old Gen — much more expensive, larger pauses.

> **Follow-up — why care?** Frequent Full GCs cause long stop-the-world pauses (latency spikes). Modern collectors (**G1**, **ZGC**, **Shenandoah**) minimize these. G1 is default since Java 9; ZGC/Shenandoah offer sub-millisecond pauses for large heaps.

---

## Java 8 Features

### Q17. Why were lambda expressions introduced in Java 8 and how do they differ from anonymous classes?

**Why introduced:**
- Enable **functional programming** and concise behavior-passing.
- Power the **Streams API** and make APIs more readable.
- Reduce boilerplate of anonymous inner classes.

```java
// Anonymous class — verbose
Runnable r1 = new Runnable() {
    @Override public void run() { System.out.println("Hello"); }
};

// Lambda — concise
Runnable r2 = () -> System.out.println("Hello");
```

**Differences:**

| Aspect | Anonymous Class | Lambda |
|--------|-----------------|--------|
| `this` reference | Refers to the **anonymous class** instance | Refers to the **enclosing** instance |
| Compilation | Generates a separate `.class` file | Uses `invokedynamic` (no extra class) |
| State | Can have fields | No fields, just behavior |
| Target | Any class/interface | **Functional interface** only (1 abstract method) |
| `new` keyword | Creates a new object each time | May reuse instances (JVM-optimized) |

> **Follow-up — why does `this` differ?** In an anonymous class, `this` is the inner instance. In a lambda, there's no new scope, so `this` is the **outer** object — important when accessing enclosing fields.

---

### Q18. In which scenario would you avoid streams and prefer a plain `for` loop?

Streams are great for readability and pipelines, but prefer a **plain `for` loop** when:

1. **Performance-critical hot paths** — streams have overhead (object allocation, lambda invocation). A simple `for` over a primitive array is faster.
2. **Working with primitives** to avoid boxing — `IntStream` helps, but a raw loop avoids it entirely.
3. **Need to break early on complex conditions / index manipulation** — `for` with `break`/`continue` is clearer than `filter`/`findFirst` gymnastics.
4. **Checked exceptions inside the body** — lambdas can't throw checked exceptions cleanly; a loop with try/catch is simpler.
5. **Modifying multiple external variables** — streams discourage mutating shared state; a loop is more natural.
6. **Index-based access or multiple collections in lockstep** — e.g., iterating two lists by index.

```java
// Awkward as a stream — clearer as a loop
for (int i = 0; i < items.size(); i++) {
    if (items.get(i).isInvalid()) {
        log.warn("Bad item at index {}", i);
        break;
    }
}
```

> **Rule of thumb:** Use streams for **declarative transformations**; use loops for **imperative control flow, performance, and side effects**.

---

## Spring Boot Internals

### Q19. What exactly happens when a Spring Boot application starts?

`SpringApplication.run()` triggers this sequence:

1. **Create `SpringApplication`** — deduces the application type (servlet/reactive/none) from the classpath.
2. **Fire listeners & prepare `Environment`** — load `application.properties`/`.yml`, profiles, env vars, command-line args.
3. **Print banner**.
4. **Create the `ApplicationContext`** — e.g., `AnnotationConfigServletWebServerApplicationContext`.
5. **Auto-configuration** — `@EnableAutoConfiguration` (via `spring.factories` / `AutoConfiguration.imports`) conditionally configures beans based on classpath (`@ConditionalOnClass`, `@ConditionalOnMissingBean`).
6. **Component scanning** — `@ComponentScan` discovers `@Component/@Service/@Repository/@Controller`.
7. **Instantiate & wire beans** — dependency injection, run `BeanPostProcessor`s.
8. **Start the embedded server** (Tomcat/Jetty/Undertow) for web apps.
9. **Fire `ApplicationRunner` / `CommandLineRunner`** callbacks.
10. **Application ready** — publishes `ApplicationReadyEvent`.

> **Follow-up — what makes auto-config work?** `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. Auto-config classes are conditionally applied based on what's on the classpath.

---

### Q20. Explain component scanning. What happens if a `@Component` class is outside the base package?

**Component scanning** is how Spring discovers and registers beans automatically. By default, `@SpringBootApplication` scans the package of the main class **and all sub-packages**.

```
com.example.app          <- @SpringBootApplication here
 ├─ service              <- scanned
 ├─ controller           <- scanned
 └─ repository           <- scanned
com.other.util           <- NOT scanned (outside base package)
```

**If a `@Component` is outside the base package:** it **won't be detected** — you'll get `NoSuchBeanDefinitionException` when something tries to inject it.

**Fixes:**
1. Move the class under the main package (preferred).
2. Add the package explicitly:
   ```java
   @SpringBootApplication(scanBasePackages = {"com.example.app", "com.other.util"})
   ```
3. Use `@ComponentScan(basePackages = "com.other.util")`.
4. Import directly with `@Import(SomeConfig.class)`.

> **Follow-up — best practice?** Keep the main class in a **root package** above all your code so the default scan covers everything.

---

### Q21. Explain the Spring Bean lifecycle. Have you used `@PostConstruct` / `@PreDestroy`?

**Lifecycle phases:**
1. **Instantiation** — Spring creates the bean instance.
2. **Populate properties** — dependency injection.
3. **`Aware` callbacks** — `BeanNameAware`, `ApplicationContextAware`, etc.
4. **`BeanPostProcessor.postProcessBeforeInitialization()`**.
5. **`@PostConstruct`** — your custom init logic.
6. **`InitializingBean.afterPropertiesSet()`** / custom `init-method`.
7. **`BeanPostProcessor.postProcessAfterInitialization()`** — AOP proxies created here.
8. **Bean is ready** for use.
9. **`@PreDestroy`** — cleanup before destruction.
10. **`DisposableBean.destroy()`** / custom `destroy-method`.

```java
@Component
public class ConnectionManager {

    @PostConstruct
    public void init() {
        // open connection pool, warm cache, validate config
    }

    @PreDestroy
    public void cleanup() {
        // close connections, flush buffers, release resources
    }
}
```

**Yes**, I use them often:
- **`@PostConstruct`** — initialize resources after DI completes (e.g., warm a cache, open a pool, validate properties).
- **`@PreDestroy`** — graceful cleanup on shutdown (close connections, stop schedulers).

> **Note:** `@PreDestroy` is **not called for `prototype`-scoped beans** — Spring doesn't manage their full lifecycle after creation.

---

### Q22. Among Spring scopes, when would you use `prototype` scope?

**Singleton (default)** — one shared instance per container. **Prototype** — a **new instance on every injection / `getBean()` call**.

**Use `prototype` when:**
- The bean holds **mutable, request-specific or conversational state** that must not be shared between callers.
- You need a **fresh, stateful object** each time (e.g., a builder, a per-operation accumulator, a non-thread-safe helper).
- Thread safety would otherwise require locking on a shared singleton.

```java
@Component
@Scope("prototype")
public class ReportBuilder {
    private final List<Section> sections = new ArrayList<>(); // per-instance state
}
```

**Caveats:**
- Spring creates a prototype but **does not manage its full lifecycle** — `@PreDestroy` won't run; the caller owns cleanup.
- Injecting a prototype into a singleton gives you **one fixed instance** (created once). To get a fresh one each time, use `ObjectProvider<T>`, `@Lookup`, or a `Provider<T>`.

```java
@Autowired
private ObjectProvider<ReportBuilder> builderProvider;

public Report build() {
    ReportBuilder builder = builderProvider.getObject(); // new instance each call
    ...
}
```

---

## Architecture, Environment & Configuration

### Q23. How do you externalize secrets like database passwords in Spring Boot?

**Never hardcode secrets** in `application.properties` or source control. Options (in order of maturity):

1. **Environment variables** — `SPRING_DATASOURCE_PASSWORD`. Spring maps relaxed-binding automatically.
   ```properties
   spring.datasource.password=${DB_PASSWORD}
   ```
2. **Externalized config files** outside the JAR / outside the repo.
3. **Secret managers** (preferred in production):
   - **HashiCorp Vault** (Spring Cloud Vault)
   - **AWS Secrets Manager / Parameter Store**
   - **Azure Key Vault**, **GCP Secret Manager**
   - **Kubernetes Secrets** (mounted as env vars or files)
4. **Encrypted config** — Jasypt for property encryption, or Spring Cloud Config with encryption.

```yaml
# Spring Cloud Vault example
spring:
  cloud:
    vault:
      uri: https://vault.internal:8200
      authentication: KUBERNETES
```

**Best practices:**
- Secrets injected at **runtime**, never baked into images.
- Rotate secrets regularly; use short-lived dynamic credentials (Vault can generate per-request DB creds).
- Restrict access via IAM/RBAC; audit secret access.

---

### Q24. What's the difference between `@ConfigurationProperties` and `@Value`?

| Aspect | `@Value` | `@ConfigurationProperties` |
|--------|----------|----------------------------|
| Binding | Single property via SpEL | **Group** of properties to a POJO |
| Type safety | Weak (string parsing) | Strong (typed fields) |
| Relaxed binding | No | Yes (`my-prop` ↔ `myProp` ↔ `MY_PROP`) |
| Validation | No | Yes (`@Validated` + JSR-303) |
| Nested objects | Awkward | First-class support |
| Best for | One or two values | Structured configuration |

```java
// @Value — single value
@Value("${app.timeout:30}")
private int timeout;

// @ConfigurationProperties — grouped, type-safe, validated
@Component
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {
    @NotBlank private String host;
    private int port = 25;
    private List<String> recipients;
    // getters/setters
}
```
```yaml
app:
  mail:
    host: smtp.example.com
    port: 587
    recipients: [a@x.com, b@x.com]
```

> **Rule of thumb:** Use `@ConfigurationProperties` for related groups of settings; reserve `@Value` for one-off values.

---

### Q25. How does Spring Boot resolve property precedence?

Spring Boot merges many property sources; **higher priority overrides lower**. Simplified order (highest → lowest):

1. **Command-line arguments** (`--server.port=9090`)
2. **`SPRING_APPLICATION_JSON`** (inline JSON)
3. **OS environment variables**
4. **Java system properties** (`-Dkey=value`)
5. **Profile-specific external files** (`application-{profile}.properties` outside the JAR)
6. **External `application.properties`** (outside the JAR)
7. **Profile-specific packaged files** (`application-{profile}.properties` inside the JAR)
8. **Packaged `application.properties`** (inside the JAR)
9. **`@PropertySource`** annotations
10. **Default properties** (`SpringApplication.setDefaultProperties`)

**Key rules:**
- **External beats internal**, **profile-specific beats generic**, **command-line beats almost everything**.
- This lets you ship sensible defaults in the JAR and override per environment without rebuilding.

> **Follow-up — `.properties` vs `.yml`?** Same precedence rules; YAML just adds structure. If both exist, the last one loaded for a given source wins (order is well-defined but avoid mixing for the same keys).

---

## Exception Handling (REST / Web)

### Q26. How would you design a global exception handling strategy?

Use a centralized **`@RestControllerAdvice`** with typed handlers and a consistent error response contract.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return build(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String msg = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return build(HttpStatus.BAD_REQUEST, msg);
    }

    @ExceptionHandler(Exception.class) // catch-all — last resort
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex);          // log internally
        return build(HttpStatus.INTERNAL_SERVER_ERROR, "Something went wrong"); // sanitized
    }

    private ResponseEntity<ErrorResponse> build(HttpStatus status, String message) {
        return ResponseEntity.status(status)
            .body(new ErrorResponse(status.value(), message, Instant.now()));
    }
}
```

**Principles:**
- **Consistent error contract** (code, message, timestamp, traceId) across all endpoints.
- **Specific handlers first**, generic catch-all last.
- **Log full details internally**, return **sanitized** messages to clients.
- Include a **correlation/trace ID** so support can match a client error to server logs.
- Map domain exceptions → proper HTTP status codes.

---

### Q27. Status codes for: resource not found, validation failure, unauthorized, internal server error?

| Scenario | Status Code |
|----------|-------------|
| Resource not found | **404 Not Found** |
| Validation failure (bad input) | **400 Bad Request** (or **422 Unprocessable Entity** for semantic validation) |
| Unauthorized (not authenticated) | **401 Unauthorized** |
| Forbidden (authenticated but no permission) | **403 Forbidden** |
| Internal server error | **500 Internal Server Error** |

> **Note:** **401** = "who are you?" (authentication missing/invalid). **403** = "I know you, but you can't do this" (authorization). They're commonly confused.

---

### Q28. What status code for successfully deleting data?

- **204 No Content** — success with **no response body** (most common for DELETE).
- **200 OK** — success **with** a response body (e.g., returning the deleted entity or a confirmation).
- **202 Accepted** — if deletion is **asynchronous** (queued for later processing).

> Idempotency: DELETE is idempotent. Deleting an already-deleted resource can return **204** (treat as success) or **404** — pick one and be consistent.

---

### Q29. How would you avoid leaking stack traces?

**Never** send raw stack traces or internal error details to clients — they leak implementation details (class names, SQL, file paths) useful to attackers.

**How:**
1. **Global handler returns sanitized messages:**
   ```java
   @ExceptionHandler(Exception.class)
   public ResponseEntity<ErrorResponse> handle(Exception ex) {
       log.error("Internal error", ex);                 // full detail in logs
       return ResponseEntity.status(500)
           .body(new ErrorResponse(500, "Internal server error")); // generic to client
   }
   ```
2. **Disable error details in production:**
   ```properties
   server.error.include-stacktrace=never
   server.error.include-message=never
   server.error.include-exception=false
   ```
3. **Disable Whitelabel error page** or customize it.
4. **Return a correlation/trace ID** so support can find the real error in logs without exposing it.
5. Ensure framework defaults (Spring Security, container error pages) don't echo exceptions.

---

## JPA & Hibernate

### Q30. Difference between `save`, `persist`, `merge`, and `saveOrUpdate`?

| Method | API | Behavior | Returns |
|--------|-----|----------|---------|
| **`persist()`** | JPA | Makes a **new** transient entity managed. No ID returned by the call (void). Throws if entity already has an ID/exists. | `void` |
| **`save()`** | Hibernate | Inserts a transient entity, **returns the generated ID** immediately. | `Serializable` (id) |
| **`merge()`** | JPA | Copies the state of a **detached** entity into a managed copy and **returns the managed instance**. The passed object stays detached. | managed entity |
| **`saveOrUpdate()`** | Hibernate | Inserts if transient, updates if detached — reattaches to the session. | `void` |

```java
// JPA persist — new entity
entityManager.persist(newOrder);

// Hibernate save — returns id
Long id = (Long) session.save(newOrder);

// JPA merge — detached entity back into context
Order managed = entityManager.merge(detachedOrder); // use 'managed', not 'detachedOrder'
```

**Spring Data JPA `repository.save()`:**
- Internally calls **`persist()` for new entities** and **`merge()` for existing ones** (decided by whether the ID is null / `@Version`).

> **Common bug:** After `merge()`, keep working with the **returned** instance — the argument you passed is still detached and changes to it won't be tracked.

---

### Q31. In a `@Transactional` method, if you modify an entity without calling `save()`, will it persist?

**Yes** — due to **dirty checking** (automatic dirty checking / "flush on commit").

```java
@Transactional
public void updatePrice(Long id, BigDecimal newPrice) {
    Product product = productRepository.findById(id).orElseThrow();
    product.setPrice(newPrice); // managed entity — NO save() needed
    // On transaction commit, Hibernate flushes and issues UPDATE automatically
}
```

**Why:** Inside a transaction, the entity returned by `findById` is **managed** (attached to the persistence context). At flush/commit, Hibernate compares the entity's current state with its loaded snapshot and auto-generates an `UPDATE` for changed fields.

**It will NOT persist if:**
- The entity is **detached** (loaded in a different transaction / context already closed).
- You're outside any transaction (no persistence context to track changes).
- You replaced the reference with a `new` object instead of mutating the managed one.

> **Follow-up — why does this surprise people?** Developers expect an explicit `save()`. Dirty checking is implicit — powerful but can cause **accidental updates** if you mutate a managed entity unintentionally.

---

## Microservices

### Q32. What's the difference between retry and circuit breaker?

| Aspect | Retry | Circuit Breaker |
|--------|-------|-----------------|
| Purpose | Handle **transient** failures | Prevent **cascading** failures |
| Mechanism | Re-attempt the call N times | Stop calling a failing service entirely |
| When it helps | Brief network blips, momentary unavailability | Sustained downstream outage |
| Risk if misused | Amplifies load on a struggling service | — |

**Retry** — re-attempt a failed call (ideally with exponential backoff + jitter):
```java
@Retry(name = "inventoryService", fallbackMethod = "fallback")
public Inventory check(String sku) { ... }
```

**Circuit breaker** — tracks failure rate; when it crosses a threshold it **opens** and short-circuits calls (fail fast), then periodically tries **half-open** to test recovery:
```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallback")
public Inventory check(String sku) { ... }
```

**States:** **Closed** (normal) → **Open** (failing, reject fast) → **Half-Open** (trial calls) → back to Closed or Open.

> **Best practice — combine them:** Retry transient errors, but wrap with a circuit breaker so repeated failures don't overload an already-failing service. Use **Resilience4j** (or legacy Hystrix).

---

### Q33. Have you worked with the Bulkhead pattern? Explain it.

**Bulkhead** isolates resources so that a failure in one part doesn't sink the whole application — named after watertight compartments in a ship's hull.

**Idea:** Partition thread pools / connections per dependency. If one downstream service is slow and saturates **its** pool, other services keep working because they use **separate** pools.

```java
@Bulkhead(name = "paymentService", type = Bulkhead.Type.THREADPOOL)
public PaymentResult pay(Order order) { ... }
```

**Two types in Resilience4j:**
- **Semaphore bulkhead** — limits concurrent calls via permits.
- **Thread-pool bulkhead** — each dependency gets its own bounded thread pool + queue.

**Without bulkheads:** one slow dependency exhausts the shared thread pool → **every** request blocks → total outage.

**With bulkheads:** the slow dependency's pool fills up and fails fast; unrelated endpoints stay healthy.

> **Real example:** Separate pools for `payment`, `inventory`, and `recommendation` services so a recommendation outage can never block checkout.

---

### Q34. Explain the Saga pattern and how it differs from Two-Phase Commit (2PC).

A **Saga** manages a distributed transaction as a **sequence of local transactions**, each in its own service. If a step fails, **compensating transactions** undo the previous steps (semantic rollback).

**Two implementation styles:**
- **Choreography** — services react to each other's events (no central coordinator). Simple but harder to trace.
- **Orchestration** — a central **orchestrator** tells each service what to do and triggers compensations on failure. Easier to monitor.

```
Order Saga (orchestration):
  1. Create Order        --(fail)--> (nothing to undo)
  2. Reserve Inventory   --(fail)--> Cancel Order
  3. Charge Payment      --(fail)--> Release Inventory + Cancel Order
  4. Confirm Order
```

**Saga vs Two-Phase Commit:**

| Aspect | Saga | Two-Phase Commit (2PC) |
|--------|------|------------------------|
| Consistency | **Eventual** | **Strong** (ACID across services) |
| Locking | No long-held locks | Holds locks until commit (blocking) |
| Coordinator failure | Resilient | Coordinator is a single point of failure |
| Scalability | High (loosely coupled) | Poor (synchronous, blocking) |
| Rollback | Compensating transactions | Automatic atomic rollback |
| Best for | Microservices, long-running flows | Tightly coupled, short transactions |

> **Why Saga in microservices?** 2PC doesn't scale — it blocks resources and couples services. Sagas trade strong consistency for availability and scalability (BASE over ACID), which fits distributed systems.

---

## Database Idempotency & Optimization

### Q35. How do you ensure an event is not processed twice in the database?

**Goal: idempotent consumption.** Common strategies:

1. **Idempotency key + unique constraint (most reliable):**
   - Each event carries a unique `event_id`.
   - Store processed IDs in a table with a **UNIQUE** constraint; insert before processing.
   ```sql
   INSERT INTO processed_events (event_id, processed_at) VALUES (?, now());
   -- Duplicate insert violates UNIQUE → skip processing
   ```
2. **Upsert / idempotent writes** — design the operation so re-applying it has no extra effect (`INSERT ... ON CONFLICT DO NOTHING`, or `UPDATE ... WHERE status = 'PENDING'`).
3. **Optimistic locking (`@Version`)** — prevents double updates on the same row version.
4. **Dedup at the messaging layer** — exactly-once semantics where available (e.g., Kafka transactional producers + consumer offset management), though end-to-end idempotency in the DB is still the safety net.

```java
@Transactional
public void handle(PaymentEvent event) {
    if (processedEventRepo.existsById(event.getId())) {
        return; // already processed
    }
    processedEventRepo.save(new ProcessedEvent(event.getId())); // unique constraint guards races
    applyPayment(event);
}
```

> **Key insight:** Messaging usually guarantees **at-least-once** delivery, so the **consumer must be idempotent**. The database UNIQUE constraint is the ultimate guard against races between concurrent consumers.

---

### Q36. A table with 10 million rows and a query is timing out. What's your approach?

**Systematic diagnosis → fix:**

1. **Get the execution plan** — `EXPLAIN ANALYZE`. Look for **full table scans**, expensive sorts, nested loops on large sets.
2. **Add the right indexes:**
   - Index columns used in `WHERE`, `JOIN`, `ORDER BY`.
   - Use **composite indexes** matching the query's column order (leftmost-prefix rule).
   - Consider **covering indexes** so the query is served from the index alone.
3. **Rewrite the query:**
   - Avoid `SELECT *` — fetch only needed columns.
   - Avoid functions on indexed columns in `WHERE` (kills index usage), e.g., `WHERE YEAR(created) = 2024`.
   - Replace `OFFSET`-based pagination with **keyset pagination** (`WHERE id > ? ORDER BY id LIMIT ?`).
4. **Reduce the working set** — filter earlier, paginate, or pre-aggregate.
5. **Partition the table** — range/hash partitioning so queries hit fewer rows.
6. **Caching** — cache hot results (Redis) if data is read-heavy.
7. **Archiving** — move old/cold rows to an archive table.
8. **Read replicas** — offload heavy reads.

```sql
-- Bad: index on created_at unused
WHERE DATE(created_at) = '2024-01-01'
-- Good: range condition preserves index
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'
```

> **Order of attack:** Always start with `EXPLAIN` — never guess. Most timeouts are a missing/unused index or `OFFSET` pagination on deep pages.

---

## Distributed Tracing & Design

### Q37. With five microservices, how would you trace a request across them?

Use **distributed tracing** with a propagated **correlation/trace ID**.

**How it works:**
1. The first service (or gateway) generates a unique **trace ID** (and per-hop **span IDs**).
2. The trace context is **propagated** in HTTP headers (`traceparent` / W3C Trace Context, or B3 headers) on every downstream call.
3. Each service logs with the trace ID and reports spans to a tracing backend.
4. The backend stitches spans into one end-to-end **trace timeline**.

**Tooling:**
- **Instrumentation:** **Micrometer Tracing** (Spring Boot 3) or **Spring Cloud Sleuth** (Boot 2), **OpenTelemetry**.
- **Backends:** **Zipkin**, **Jaeger**, **Grafana Tempo**, or APM tools (Datadog, New Relic).
- **Log correlation:** include trace/span IDs in every log line (MDC) so logs across services can be joined.

```
Gateway --traceId=abc123--> OrderService --abc123--> PaymentService
                                       \--abc123--> InventoryService --abc123--> NotificationService
```

**Benefits:** pinpoint **which service** is slow or failing, see latency per hop, and correlate logs across services for one request.

> **Best practice:** Add the trace ID to error responses so support can jump straight to the relevant trace.

---

### Q38. A colleague wrote a singleton that isn't thread-safe. How would you fix it?

**Problem — lazy init without synchronization (race condition):**
```java
public class Config {
    private static Config instance;
    private Config() {}
    public static Config getInstance() {
        if (instance == null) {           // two threads can both pass this check
            instance = new Config();      // → multiple instances
        }
        return instance;
    }
}
```

**Fixes (best first):**

**1. Enum singleton (simplest, recommended):** thread-safe, serialization-safe, reflection-safe.
```java
public enum Config {
    INSTANCE;
    public void doWork() { ... }
}
```

**2. Eager initialization** — if creation is cheap and always needed:
```java
public class Config {
    private static final Config INSTANCE = new Config(); // JVM guarantees thread-safe class init
    private Config() {}
    public static Config getInstance() { return INSTANCE; }
}
```

**3. Bill Pugh / holder idiom** — lazy + thread-safe without synchronization cost:
```java
public class Config {
    private Config() {}
    private static class Holder {
        private static final Config INSTANCE = new Config();
    }
    public static Config getInstance() { return Holder.INSTANCE; }
}
```

**4. Double-checked locking with `volatile`** — when lazy init is required:
```java
public class Config {
    private static volatile Config instance; // volatile is critical!
    private Config() {}
    public static Config getInstance() {
        if (instance == null) {
            synchronized (Config.class) {
                if (instance == null) {
                    instance = new Config();
                }
            }
        }
        return instance;
    }
}
```

> **Why `volatile` in DCL?** Without it, `instance = new Config()` can be reordered, letting another thread see a **non-null but partially constructed** object.

> **Recommendation:** Prefer the **enum** or **holder idiom** — they're the cleanest, avoid locking overhead, and are robust against serialization/reflection attacks.

---

*End of prep document.*
