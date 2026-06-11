# Exception Handling in Java

A complete reference covering exception basics, the `try-catch-finally` mechanics, custom exceptions, and centralized exception handling in Spring Boot — with examples and common interview questions.

---

## Table of Contents

- [1. Exception Hierarchy Basics](#1-exception-hierarchy-basics)
- [2. Checked vs Unchecked Exceptions](#2-checked-vs-unchecked-exceptions)
- [3. try-catch-finally Mechanics](#3-try-catch-finally-mechanics)
- [4. Tricky try-catch-finally Questions](#4-tricky-try-catch-finally-questions)
- [5. Multi-catch and Exception Chaining](#5-multi-catch-and-exception-chaining)
- [6. Creating Custom Exceptions](#6-creating-custom-exceptions)
- [7. Centralized Exception Handling in Spring Boot](#7-centralized-exception-handling-in-spring-boot)
- [8. Common Interview Questions](#8-common-interview-questions)

---

## 1. Exception Hierarchy Basics

Everything in Java's error model descends from `Throwable`:

```
Throwable
├── Error                     (serious problems — don't catch: OutOfMemoryError, StackOverflowError)
└── Exception
    ├── RuntimeException       (unchecked — NullPointerException, IllegalArgumentException, ...)
    └── (other) Exception      (checked — IOException, SQLException, ...)
```

| Term | Description |
|------|-------------|
| **`Throwable`** | Root of all errors and exceptions. Only `Throwable` (or subclasses) can be thrown/caught. |
| **`Error`** | Serious JVM-level problems an app should **not** try to recover from. |
| **`Exception`** | Conditions a program **can** catch and handle. |
| **`RuntimeException`** | Unchecked — programming bugs, not forced to handle. |

---

## 2. Checked vs Unchecked Exceptions

| Aspect | Checked Exception | Unchecked Exception |
|--------|-------------------|---------------------|
| Extends | `Exception` (not `RuntimeException`) | `RuntimeException` |
| Compile-time check | **Yes** — must `throws` or `try-catch` | No |
| Represents | Recoverable conditions (file missing, network) | Programming errors (null, bad index) |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |

```java
// Checked — compiler forces handling
public void readFile(String path) throws IOException {
    Files.readAllLines(Path.of(path)); // may throw IOException
}

// Unchecked — no declaration required
public int divide(int a, int b) {
    return a / b; // may throw ArithmeticException at runtime
}
```

> **Rule of thumb:** Use **checked** exceptions when the caller can reasonably recover; use **unchecked** for programming errors that should be fixed in code.

---

## 3. try-catch-finally Mechanics

```java
try {
    // code that may throw
    int result = 10 / 0;
} catch (ArithmeticException e) {
    // handle the exception
    System.out.println("Cannot divide by zero: " + e.getMessage());
} finally {
    // ALWAYS runs (except a few edge cases — see below)
    System.out.println("Cleanup done");
}
```

### When does the `finally` block NOT execute?

- If the JVM exits abruptly via `System.exit()`.
- If the thread executing the block is killed or the JVM crashes.
- If there is an infinite loop (or it blocks forever) in the `try` block so `finally` is never reached.
- A hardware/power failure or fatal `Error` that halts the JVM.

### try-with-resources (preferred for closeable resources)

Auto-closes resources (no manual `finally`), even if an exception is thrown:

```java
try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
    return reader.readLine();
} catch (IOException e) {
    log.error("Read failed", e);
    throw new UncheckedIOException(e);
}
// reader.close() is called automatically
```

> Any resource implementing `AutoCloseable` works. Multiple resources are closed in **reverse** order of declaration.

---

## 4. Tricky try-catch-finally Questions

### Q1. What is the output? (return inside try with a finally)

![][image1]

```java
public static int test() {
    try {
        return 1;
    } finally {
        return 2; // finally's return OVERRIDES try's return
    }
}
// Output: 2
```

**Why:** A `return` (or value change) in `finally` overrides whatever `try`/`catch` returned. Avoid returning from `finally` — it silently swallows exceptions and results.

---

### Q2. What is the output?

![][image2]

**Output:** `test 2`

The `finally` block executes after the `try`/`catch` completes, producing the final printed result.

---

### Q3. Can you have a `try` block without a `catch` or `finally` block?

**No.** A `try` must be followed by **at least one** `catch` **or** a `finally` (or both). These are all valid:

```java
try { ... } catch (Exception e) { ... }            // try-catch
try { ... } finally { ... }                         // try-finally
try { ... } catch (Exception e) { ... } finally {}  // try-catch-finally
try (Resource r = ...) { ... }                      // try-with-resources (implicit finally)
```

---

### Q4. What happens if you put `System.exit(0)` in the `try` or `catch` block? Will `finally` execute?

`System.exit(int)` can throw a `SecurityException` (if a `SecurityManager` forbids it). 
- If that `SecurityException` is thrown, JVM does **not** shut down → `finally` **executes**.
- Otherwise, `System.exit()` terminates the JVM immediately → `finally` does **NOT** execute.

```java
try {
    System.exit(0);
} finally {
    System.out.println("This usually never prints");
}
```

---

### Q5. Is it necessary that each `try` block be followed by a `catch` block?

**No.** A `try` can be paired with `finally` only. `finally` runs whether or not an exception is thrown — ideal for cleanup.

---

## 5. Multi-catch and Exception Chaining

### Multi-catch

A single `catch` can handle multiple exception types using `|`:

```java
try {
    process();
} catch (IOException | SQLException e) {  // multi-catch
    log.error("I/O or DB failure", e);
}
```

**Restriction:** You cannot combine a class **and its ancestor** in the same multi-catch — it's redundant and won't compile:

```java
// INVALID — Exception is an ancestor of NumberFormatException
catch (NumberFormatException | Exception ex) { ... }   // compile error

// VALID — unrelated types
catch (NumberFormatException | IllegalStateException ex) { ... }
```

### Exception chaining (preserve the root cause)

Always pass the original exception as the `cause` so the full trail is preserved:

```java
try {
    statement.executeQuery(sql);
} catch (SQLException e) {
    // 'e' is preserved as the cause — never lose the root cause
    throw new DataAccessException("Failed to run query: " + sql, e);
}
```

```
DataAccessException: Failed to run query: SELECT ...
  Caused by: SQLException: Unknown column 'xyz'
```

> **Anti-pattern:** `throw new RuntimeException(e.getMessage());` loses the stack trace and cause. Always pass `e`, not `e.getMessage()`.

---

## 6. Creating Custom Exceptions

### 6.1 Why create custom exceptions?

- **Domain clarity** — `InsufficientBalanceException` is clearer than a generic `RuntimeException`.
- **Targeted handling** — catch/handle specific business errors differently.
- **Carry metadata** — error codes, HTTP status, context fields.

### 6.2 Simple custom unchecked exception

```java
public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String message) {
        super(message);
    }

    public ResourceNotFoundException(String message, Throwable cause) {
        super(message, cause); // supports exception chaining
    }
}
```

```java
public Order getOrder(Long id) {
    return orderRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + id));
}
```

### 6.3 Custom checked exception

```java
public class PaymentDeclinedException extends Exception {  // extends Exception => checked
    public PaymentDeclinedException(String message) {
        super(message);
    }
}

public void charge(Card card) throws PaymentDeclinedException {
    if (!card.isValid()) {
        throw new PaymentDeclinedException("Card declined: " + card.getLast4());
    }
}
```

### 6.4 Custom exception hierarchy (enterprise pattern)

Build a base exception that carries metadata, then derive specific ones:

```java
// 1. Error codes (centralized)
public enum ErrorCode {
    RESOURCE_NOT_FOUND, VALIDATION_FAILED, BUSINESS_RULE_VIOLATION, EXTERNAL_SERVICE_FAILURE
}

// 2. Base application exception
public abstract class ApplicationException extends RuntimeException {
    private final ErrorCode errorCode;
    private final HttpStatus httpStatus;

    protected ApplicationException(String message, ErrorCode code, HttpStatus status) {
        super(message);
        this.errorCode = code;
        this.httpStatus = status;
    }

    protected ApplicationException(String message, Throwable cause, ErrorCode code, HttpStatus status) {
        super(message, cause);
        this.errorCode = code;
        this.httpStatus = status;
    }

    public ErrorCode getErrorCode() { return errorCode; }
    public HttpStatus getHttpStatus() { return httpStatus; }
}

// 3. Specific exceptions
public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String resource, Long id) {
        super(resource + " not found with id: " + id,
              ErrorCode.RESOURCE_NOT_FOUND, HttpStatus.NOT_FOUND);
    }
}

public class BusinessRuleViolationException extends ApplicationException {
    public BusinessRuleViolationException(String rule) {
        super("Business rule violated: " + rule,
              ErrorCode.BUSINESS_RULE_VIOLATION, HttpStatus.UNPROCESSABLE_ENTITY);
    }
}
```

**Best practices for custom exceptions:**
- Prefer **unchecked** (extends `RuntimeException`) to avoid polluting every method signature.
- Always provide a `(String, Throwable)` constructor for **chaining**.
- Carry useful **metadata** (error code, HTTP status, context).
- Name them meaningfully (end with `Exception`).
- Don't create one per trivial case — group logically.

---

## 7. Centralized Exception Handling in Spring Boot

Spring Boot handles exceptions centrally using **`@ControllerAdvice` / `@RestControllerAdvice`** with **`@ExceptionHandler`** — keeping controllers clean.

### 7.1 Global handler with `@RestControllerAdvice`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handle a specific custom exception
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
        ErrorResponse body = new ErrorResponse(HttpStatus.NOT_FOUND.value(),
                                               ex.getMessage(), Instant.now());
        return new ResponseEntity<>(body, HttpStatus.NOT_FOUND);
    }

    // Handle validation failures
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(err -> err.getField() + ": " + err.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return new ResponseEntity<>(
            new ErrorResponse(HttpStatus.BAD_REQUEST.value(), message, Instant.now()),
            HttpStatus.BAD_REQUEST);
    }

    // Catch-all (must be LAST resort)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(Exception ex) {
        log.error("Unexpected error", ex); // log full detail internally
        ErrorResponse body = new ErrorResponse(HttpStatus.INTERNAL_SERVER_ERROR.value(),
                                               "Internal Server Error", Instant.now());
        return new ResponseEntity<>(body, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

### 7.2 Consistent error response DTO

```java
public class ErrorResponse {
    private int status;
    private String message;
    private Instant timestamp;
    // constructor, getters
}
```

### 7.3 Local handling with `@ExceptionHandler` inside a controller

If you want to handle exceptions for **one controller only**, declare `@ExceptionHandler` inside that controller:

![][image3]

```java
@RestController
public class OrderController {

    @GetMapping("/orders/{id}")
    public Order get(@PathVariable Long id) {
        return service.findById(id); // may throw ResourceNotFoundException
    }

    // Applies ONLY to this controller
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }
}
```

> **Precedence:** A controller-local `@ExceptionHandler` takes priority over a global `@ControllerAdvice` one for the same exception type.

### 7.4 Avoid leaking stack traces

Never return raw stack traces to clients — they leak internal details. Log internally, return sanitized messages:

```properties
server.error.include-stacktrace=never
server.error.include-message=never
server.error.include-exception=false
```

---

## 8. Common Interview Questions

### Q. How do you create a custom exception in Java?
Extend `RuntimeException` (unchecked) or `Exception` (checked), provide constructors that accept a message and a `Throwable` cause (for chaining), and optionally add metadata fields like an error code. See [section 6](#6-creating-custom-exceptions).

### Q. Checked vs unchecked — which should a custom exception extend?
Prefer **unchecked** (`RuntimeException`) in most modern apps to avoid forcing `throws` everywhere. Use **checked** only when the caller can meaningfully recover and you want to force handling.

### Q. What is exception chaining and why does it matter?
Wrapping the original exception as the `cause` of a new one (`new MyException(msg, e)`). It preserves the **root cause** and full stack trace across layers — critical for debugging.

### Q. Difference between `throw` and `throws`?
- `throw` — actually throws an exception instance: `throw new IllegalArgumentException("bad");`
- `throws` — declares in a method signature that it may throw checked exceptions: `void m() throws IOException`.

### Q. Difference between `final`, `finally`, and `finalize`?
- `final` — keyword for constants / non-overridable methods / non-extendable classes.
- `finally` — block that always runs after `try`/`catch`.
- `finalize()` — deprecated `Object` method called by GC before reclaiming an object (don't rely on it).

### Q. Can a `finally` block override a returned value?
Yes — a `return` in `finally` overrides the `try`/`catch` return and can swallow exceptions. **Avoid returning from `finally`.**

### Q. What is the difference between `@ControllerAdvice` and `@RestControllerAdvice`?
`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`. Use `@RestControllerAdvice` for REST APIs so handler return values are serialized directly to the response body.

### Q. How do you handle validation errors globally in Spring Boot?
Catch `MethodArgumentNotValidException` (for `@Valid` request bodies) or `ConstraintViolationException` (for `@Validated` params) in a `@RestControllerAdvice` and map field errors to a structured response. See [section 7.1](#71-global-handler-with-restcontrolleradvice).

### Q. How do you avoid leaking sensitive details in error responses?
Log full details server-side, return a generic sanitized message + correlation/trace ID to the client, and disable `server.error.include-stacktrace`/`include-message`. See [section 7.4](#74-avoid-leaking-stack-traces).

---

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,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>