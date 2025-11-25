```markdown
# CompletableFuture & Async Orchestration

## 1. Why CompletableFuture?

`Future` (Java 5) was limited because you had to block (`get()`) to retrieve the result. `CompletableFuture` (Java 8) allows you to attach callbacks and compose multiple asynchronous steps without blocking.

## 2. Critical: Specify Your Executor

By default, `CompletableFuture.supplyAsync()` uses `ForkJoinPool.commonPool()`.
**Risk:** In a high-load production environment, relying on the common pool can lead to thread starvation, affecting the entire application.

**Best Practice:** Always pass your custom `ThreadPoolExecutor`.

```java
ExecutorService customPool = new ThreadPoolExecutor(...); // Your custom pool

CompletableFuture.supplyAsync(() -> {
    return doHeavyTask();
}, customPool); // <--- ALWAYS specify this
````

## 3\. Common Patterns

### A. Chaining (Transformation)

Process the result of step 1 in step 2.

```java
CompletableFuture.supplyAsync(() -> "Hello", customPool)
    .thenApply(s -> s + " World") // Transform: "Hello World"
    .thenAccept(System.out::println); // Consume
```

### B. Combining (Composition)

Run two independent tasks in parallel and combine their results.

```java
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> getUser(id), pool);
CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(() -> getOrders(id), pool);

// Wait for BOTH to finish
userFuture.thenCombine(orderFuture, (user, orders) -> {
    return new DashboardData(user, orders);
}).thenAccept(dashboard -> render(dashboard));
```

### C. Exception Handling

Handle errors gracefully without `try-catch` blocks everywhere.

```java
CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("DB Failed");
    return 0;
}).exceptionally(ex -> {
    System.err.println("Error: " + ex.getMessage());
    return -1; // Return default value on error
});
```

## 4\. blocking vs non-blocking

  * **`join()`**: Returns the result (throws unchecked exception). **Blocks** the current thread.
  * **`get()`**: Returns the result (throws checked exception). **Blocks** the current thread.
  * **`thenAccept`/`thenApply`**: Non-blocking callbacks.

**Goal:** Keep the main thread free. Only call `join()` at the very end if absolutely necessary.
```

```
