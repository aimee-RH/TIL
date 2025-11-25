# ☕ Java TIL (Today I Learned)

A collection of core concepts, best practices, and deep dives into the Java ecosystem.

---

## 🚀 Concurrency & Multithreading
> Core mechanisms, thread pool management, and asynchronous programming.

- **[ThreadPoolExecutor Core Mechanism](concurrency/thread-pool-executor.md)**
  - The 7 core parameters explained.
  - The real task submission flow (Core -> Queue -> Max).
  - Why `Executors` static factories are dangerous in production.

- **[Virtual Threads (Java 21+)](concurrency/virtual-threads.md)**
  - Difference between Platform Threads and Virtual Threads.
  - The "One Thread Per Task" paradigm shift.
  - Why you should **stop** pooling threads when using Loom.

- **[CompletableFuture & Async Orchestration](concurrency/completable-future.md)**
  - Chaining and combining asynchronous tasks.
  - **Critical:** Using custom Thread Pools to avoid starvation.
  - Exception handling patterns.

---

## 📦 Collections & Data Structures
> *TODO: Deep dives into HashMap, ConcurrentHashMap, ArrayList, etc.*

- *(Coming Soon)*

---

## ⚙️ JVM & Performance
> *TODO: GC Algorithms, Class Loaders, JMM, etc.*

- *(Coming Soon)*

---

## 📚 Useful Resources
- [Java Platform, Standard Edition Documentation](https://docs.oracle.com/en/java/javase/)
- [JEP Dashboard (Java Enhancement Proposals)](https://openjdk.org/jeps/0)
