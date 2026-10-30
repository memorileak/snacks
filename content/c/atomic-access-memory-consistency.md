+++
title = "Modern C: Atomic access and memory consistency"
description = "A guide to C23 atomic memory consistency, covering happens-before ordering, library synchronization, sequential consistency, and explicit memory orders."
date = 2026-09-28T07:41:09Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 21. Atomic access and memory consistency

### Overview of Concurrent Access

**What it is:**
The execution of a C program is often thought of as a linear sequence of events, but modern platforms execute instructions across multiple execution pipelines, sometimes out of order, or concurrently via threads. The C standard establishes rules for how different levels of concurrent access—ranging from simple sequenced statements to long jumps, signal handlers, atomic objects, and threads—affect the program's abstract state.

**Why it matters / how it works:**
Because physical compute cores do not share a perfect global clock, C does not assume an overall fine-grained notion of time exists between different threads. Without synchronization, changes made by one thread might not be visible to another, or might be seen in an inconsistent order. The abstract state of a program's execution includes points of execution, intermediate values, stored values, and hidden state (like a mutex's lock state). Changes to these states are collectively called "effects".

**Key details:**

- Plain C code is only apparently sequential, with visibility of changes guaranteed only at sequence points.
- Long jumps and signal handlers run sequentially but can lose the effects of stores along the way.
- Thread synchronization regulates shared access to data using atomic objects or specific function calls (e.g., `thrd_join`, `mtx_lock`).

**Pitfalls:**

- Assuming a global timeline across all threads without enforcing synchronization leads to data races and undefined behavior.

> **Takeaway 21 #1** Every evaluation has an effect.

**Explanation:** Even a seemingly empty evaluation like `(void)0;` drops an intermediate value and advances the execution point to the next statement. Because advancing the execution point changes the abstract state, it is considered an effect.

**Code Demonstration:**

```c
#include <stdio.h>

int main(void) {
    int x = 5;
    // The evaluation of the assignment has the effect of storing 5 in x.
    // The evaluation of (void)0; has the effect of advancing execution.
    (void)0;
    printf("x is %d\n", x);
    return 0;
}
```

_Expected behavior/output:_ The program successfully compiles and prints "x is 5". The `(void)0;` evaluation had the effect of moving the instruction pointer forward without altering stored values.

**Recap:**

- Execution is not inherently synchronized across cores.
- The abstract state includes execution points, intermediate/stored values, and hidden state.
- Every single evaluation, even discarding a value, produces a change in the abstract state (an effect).

---

## 21.1. The “happened before” relation

### The Sequenced Before Relation

**What it is:**
The "happened before" relation is a formal rule establishing that one evaluation (and its effects) is completely resolved before another evaluation begins. Within a single thread, this is established by the "sequenced before" relation, driven primarily by the syntax of the C language.

**Why it matters / how it works:**
To reason about program correctness—especially across threads—we need partial knowledge of thread states stitched into a coherent timeline. The "happened before" relation acts as this timeline. If evaluation A happens before evaluation B, then B is guaranteed to see all the memory effects of A.

**Key details:**

- The relation is denoted as $F \rightarrow E$ (Evaluation F happened before E).
- Within a single thread, sequence points (like semicolons ending a full expression) guarantee ordering.

**Pitfalls:**

- Assuming two evaluations in a single expression are sequenced when they are actually unsequenced (e.g., `f(i++, i++)`).

> **Takeaway 21.1 #1** If F is sequenced before E, then F → E.

**Explanation:** In a single-threaded context, basic syntactical sequence points guarantee that earlier lines of code "happen before" later lines of code.

**Code Demonstration:**

```c
#include <stdio.h>

int main(void) {
    int a = 10; // Evaluation F
    int b = a + 5; // Evaluation E
    // F is sequenced before E. Therefore F -> E, so 'b' is guaranteed to see 'a' as 10.
    printf("b = %d\n", b);
    return 0;
}
```

_Expected behavior/output:_ Prints `b = 15`.
_What goes wrong if ignored:_ If C did not enforce F -> E, `b` might read uninitialized memory for `a`.

### Synchronization via Atomic Objects

**What it is:**
Between threads, the "happened before" relation is established via synchronization. Atomic objects allow one thread to write a value and another thread to read it, creating a bridge (synchronization point) between their otherwise independent timelines.

**Why it matters / how it works:**
An atomic object forces the platform to record modifications in a globally consistent order (modification order). This prevents threads from seeing torn data or conflicting histories of a single variable.

**Key details:**

- Operations on atomics are indivisible.
- Modifications to a specific atomic object follow a strict sequence.

> **Takeaway 21.1 #2** The set of modifications of an atomic object X is performed in an order consistent with the sequenced before relation of any thread that deals with X.

**Explanation:** If Thread A writes 1 then 2 to an atomic variable, Thread B cannot observe the value 2 and later observe 1. The modifications have a linear history.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdatomic.h>
#include <threads.h>

_Atomic int shared_val = 0;

int thread_func(void* arg) {
    shared_val = 1; // sequenced before the next line
    shared_val = 2;
    return 0;
}

int main(void) {
    thrd_t t;
    thrd_create(&t, thread_func, NULL);

    // The main thread will observe the modification order (0 -> 1 -> 2).
    // It will never see 2 followed by 1.
    thrd_join(t, NULL);
    printf("Final value: %d\n", shared_val);
    return 0;
}
```

_Expected behavior/output:_ Prints `Final value: 2`.

### Acquire and Release Semantics

**What it is:**
Synchronization using atomics is directional. It has a "writer" side (release semantics) and a "reader" side (acquire semantics).

**Why it matters / how it works:**
When a thread writes to an atomic object, it "releases" all the memory modifications it made prior to that write. When another thread "acquires" that atomic object by reading the written value, all those released modifications become visible to the reader.

**Key details:**

- Write operations generally have release semantics.
- Read operations generally have acquire semantics.
- Read-modify-write operations (like `atomic_fetch_add`) have both.

> **Takeaway 21.1 #3** An acquire operation E in a thread TE synchronizes with a release operation F in another thread TF if E reads the value that F has written.

**Explanation:** Synchronization is not magic; it requires a data dependency. A data transfer must actually occur (E reading what F wrote) for the synchronization bridge to be established.

> **Takeaway 21.1 #4** If F synchronizes with E, all effects X that happened before F must be visible at all evaluations G that happen after E.

**Explanation:** This is the core of memory visibility. If Thread A does some non-atomic work, then releases an atomic, and Thread B acquires that atomic, Thread B is guaranteed to see all of Thread A's non-atomic work.

> **Takeaway 21.1 #5** We only can conclude that one evaluation happened before another if we have a sequenced chain of synchronizations that links them.

**Explanation:** The "happened before" relation is transitive. If A -> B, and B -> C, then A -> C. Without an unbroken chain of sequence points and thread synchronizations, the ordering is indeterminate.

> **Takeaway 21.1 #6** If an evaluation F happened before E, all effects that are known to have happened before F are also known to have happened before E.

**Explanation:** This reiterates the transitivity. By linking thread barriers, we can securely pass complex data structures across threads safely.

**Code Demonstration (Acquire/Release):**

```c
#include <stdio.h>
#include <stdatomic.h>
#include <threads.h>

int payload = 0; // Non-atomic data
_Atomic bool ready = false; // Atomic flag

int writer(void* arg) {
    payload = 42; // Effect X (sequenced before F)
    ready = true; // Evaluation F (Release operation)
    return 0;
}

int reader(void* arg) {
    // Wait until E reads the value F wrote
    while (!ready) {} // Evaluation E (Acquire operation)

    // Evaluation G (happens after E). payload is guaranteed to be 42.
    printf("Payload: %d\n", payload);
    return 0;
}

int main(void) {
    thrd_t t1, t2;
    thrd_create(&t1, writer, NULL);
    thrd_create(&t2, reader, NULL);
    thrd_join(t1, NULL);
    thrd_join(t2, NULL);
    return 0;
}
```

_Expected behavior/output:_ Prints `Payload: 42`. The `ready` atomic safely transports the visibility of `payload` from the writer thread to the reader thread.

**Recap:**

- "Sequenced before" establishes ordering within one thread.
- Atomic variables establish ordering between threads via modification orders.
- Writing an atomic (release) and reading that atomic (acquire) creates a synchronization bridge, ensuring memory visibility.

---

## 21.2. C library calls that provide synchronization

### Library Functions as Synchronization Pairs

**What it is:**
Beyond raw atomic variables, the C standard library provides functions that implicitly carry acquire and release semantics, forming synchronization pairs.

**Why it matters / how it works:**
Functions like `thrd_create`, `thrd_join`, `mtx_lock`, and `call_once` encapsulate memory barriers. This allows programmers to write safe concurrent code without directly managing `_Atomic` variables for every interaction.

**Key details:**

- `thrd_create` synchronizes with the start of the thread function.
- `thrd_exit` or returning from a thread synchronizes with `thrd_join`.
- `call_once`'s first execution synchronizes with all subsequent calls using the same flag.

**Pitfalls:**

- Modifying shared data before calling `thrd_create` is safe, but modifying it concurrently without a mutex or atomic causes data races.

### Mutex Synchronization

**What it is:**
A mutex (`mtx_t`) creates a critical section where only one thread can execute at a time. The act of unlocking a mutex acts as a release, and locking it acts as an acquire.

> **Takeaway 21.2 #1** Critical sections protected by the same mutex occur sequentially.

**Explanation:** Because unlocking synchronizes with the next lock, multiple threads competing for the same mutex will be forced into a strict, non-overlapping sequence.

> **Takeaway 21.2 #2** In a critical section protected by the mutex mut, all effects of previous critical sections protected by mut are visible.

**Explanation:** The release/acquire nature of `mtx_unlock` and `mtx_lock` guarantees that memory mutated in Thread A's critical section is fully visible to Thread B when Thread B enters its critical section.

**Code Demonstration:**

```c
#include <stdio.h>
#include <threads.h>

int counter = 0;
mtx_t mtx;

int incrementer(void* arg) {
    mtx_lock(&mtx); // Acquire
    counter++;      // Safe access
    mtx_unlock(&mtx); // Release
    return 0;
}

int main(void) {
    mtx_init(&mtx, mtx_plain);
    thrd_t t1, t2;
    thrd_create(&t1, incrementer, NULL);
    thrd_create(&t2, incrementer, NULL);
    thrd_join(t1, NULL);
    thrd_join(t2, NULL);
    printf("Counter: %d\n", counter);
    mtx_destroy(&mtx);
    return 0;
}
```

_Expected behavior/output:_ Prints `Counter: 2`. The critical sections are strictly ordered.

### Condition Variables

**What it is:**
Condition variables (`cnd_t`) allow threads to sleep until a specific condition is met, avoiding busy-waiting.

> **Takeaway 21.2 #3** cnd_wait and cnd_timedwait have release-acquire semantics for the mutex.

**Explanation:** Before putting the thread to sleep, `cnd_wait` performs a _release_ operation by unlocking the mutex. When the thread wakes up, it performs an _acquire_ operation by re-locking the mutex before returning.

> **Takeaway 21.2 #4** Calls to cnd_signal and cnd_broadcast synchronize via the mutex.

**Explanation:** A signal wakes up a sleeping thread, but the actual memory visibility synchronization happens because the awakened thread must reacquire the mutex that the signaling thread recently released.

> **Takeaway 21.2 #5** Calls to cnd_signal and cnd_broadcast should occur inside a critical section protected by the same mutex as the waiters.

**Explanation:** Modifying the condition data and signaling should be done while holding the mutex. This ensures the sleeping thread doesn't miss the signal due to a race condition.

**Code Demonstration:**

```c
#include <stdio.h>
#include <threads.h>
#include <stdbool.h>

mtx_t mtx;
cnd_t cv;
bool ready = false;

int waiter(void* arg) {
    mtx_lock(&mtx);
    while (!ready) {
        cnd_wait(&cv, &mtx); // Unlocks mtx, waits, then re-locks mtx
    }
    printf("Awake!\n");
    mtx_unlock(&mtx);
    return 0;
}

int signaler(void* arg) {
    mtx_lock(&mtx);
    ready = true;
    cnd_signal(&cv); // Should occur inside the critical section
    mtx_unlock(&mtx);
    return 0;
}

int main(void) {
    mtx_init(&mtx, mtx_plain);
    cnd_init(&cv);
    thrd_t t1, t2;
    thrd_create(&t1, waiter, NULL);
    thrd_create(&t2, signaler, NULL);
    thrd_join(t1, NULL);
    thrd_join(t2, NULL);
    cnd_destroy(&cv);
    mtx_destroy(&mtx);
    return 0;
}
```

_Expected behavior/output:_ The `waiter` thread sleeps until `signaler` sets `ready` and signals. Prints `Awake!`.

**Recap:**

- Thread creation/joining acts as memory barriers.
- Mutexes link critical sections sequentially using release/acquire semantics.
- Condition variables temporarily release mutexes and require re-acquiring them, keeping synchronization intact.

---

## 21.3. Sequential consistency

### The Default Consistency Model

**What it is:**
Sequential consistency is the strongest memory consistency model. It guarantees that all atomic operations appear to execute in a single, global, interleaved sequence across all threads.

**Why it matters / how it works:**
Without sequential consistency, operations on _different_ atomic variables might be observed in different orders by different threads. Sequential consistency forces the hardware to resolve all atomic operations globally.

**Key details:**

- It combines acquire/release causal ordering with a strict total order.
- It is the default for standard C atomic operations.

> **Takeaway 21.3 #1** All atomic operations with sequential consistency occur in one global modification order, regardless of the atomic object they are applied to.

**Explanation:** If Thread A modifies atomic `X` and Thread B modifies atomic `Y`, all threads in the system will agree whether `X` was modified before `Y` or vice versa.

> **Takeaway 21.3 #2** All operators and functional interfaces on atomics that don’t specify otherwise have sequential consistency.

**Explanation:** Using `+=' on an `_Atomic`type, or calling`atomic_store`, guarantees sequential consistency automatically.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdatomic.h>
#include <threads.h>

_Atomic int X = 0;
_Atomic int Y = 0;
int read_X = 0, read_Y = 0;

int thread1(void* arg) {
    X = 1; // Sequentially consistent store
    read_Y = Y; // Sequentially consistent load
    return 0;
}

int thread2(void* arg) {
    Y = 1; // Sequentially consistent store
    read_X = X; // Sequentially consistent load
    return 0;
}

int main(void) {
    thrd_t t1, t2;
    thrd_create(&t1, thread1, NULL);
    thrd_create(&t2, thread2, NULL);
    thrd_join(t1, NULL);
    thrd_join(t2, NULL);
    // Because of sequential consistency, it is impossible for both read_X and read_Y to be 0.
    printf("read_X: %d, read_Y: %d\n", read_X, read_Y);
    return 0;
}
```

_Expected behavior/output:_ Either `read_X` is 1, `read_Y` is 1, or both are 1. They will never both be 0.
_What goes wrong if ignored:_ On weaker memory models (if we bypassed sequential consistency), CPUs might reorder the store and load, resulting in both reading 0.

**Recap:**

- Sequential consistency provides a single, global timeline for all atomic operations.
- It is the default for simple assignment/operators on `_Atomic` variables.
- It incurs a performance penalty by preventing the CPU from reordering certain instructions.

---

## 21.4. Other consistency models

### Explicit Memory Orders

**What it is:**
For performance-critical code, forcing sequential consistency is often too expensive. C allows programmers to explicitly weaken the memory model using functions suffixed with `_explicit`.

**Why it matters / how it works:**
By specifying exact memory orders (like `memory_order_relaxed` or `memory_order_acquire`), the compiler and CPU are granted permission to reorder instructions for maximum efficiency, as long as they respect the requested constraints.

**Key details:**

- `memory_order_seq_cst`: Default, total global order.
- `memory_order_acq_rel`: Acquire-release causality.
- `memory_order_release` / `memory_order_acquire`: One-way synchronization.
- `memory_order_consume`: Weaker data-dependency tracking.
- `memory_order_relaxed`: No synchronization, only guarantees atomicity of the operation itself.

> **Takeaway 21.4 #1** Synchronizing functional interfaces for atomic objects with _explicit appended allow us to specify their consistency model.

**Explanation:** Instead of `atomic_store(&x, 1)`, you use `atomic_store_explicit(&x, 1, memory_order_release)` to specify exactly what kind of memory barrier is required.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdatomic.h>
#include <threads.h>

_Atomic int counter = 0;

int worker(void* arg) {
    for (int i = 0; i < 1000; i++) {
        // We only care about the final count, not synchronizing other memory.
        // Relaxed ordering is much faster.
        atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
    }
    return 0;
}

int main(void) {
    thrd_t t1, t2;
    thrd_create(&t1, worker, NULL);
    thrd_create(&t2, worker, NULL);
    thrd_join(t1, NULL);
    thrd_join(t2, NULL);

    // We can read normally here because thrd_join provides a synchronization barrier.
    printf("Final count: %d\n", counter);
    return 0;
}
```

_Expected behavior/output:_ Prints `Final count: 2000`. The count is accurate because the operation is still indivisible (atomic), but it executed faster by omitting unnecessary cross-thread memory flushes.

**Recap:**

- Use `_explicit` interfaces to request weaker memory consistency models.
- `memory_order_relaxed` is ideal for simple counters where you don't need to pass other data between threads.
- Only lower the memory order when profiling shows a bottleneck, as it makes reasoning about the code significantly harder.

---

## Summary

Chapter 21 explores the complex reality of memory state and execution timelines in C. It introduces the "happened before" relation, establishing that program correctness in a concurrent environment relies strictly on explicitly linking thread timelines via synchronization. This synchronization is provided by atomic variables (via modification orders and acquire-release semantics) and specific C library functions (like mutexes, condition variables, and thread management functions). The default behavior for atomic operations is sequential consistency, which creates a total global ordering of operations at the cost of performance. To optimize execution, C provides explicit atomic operations allowing developers to select weaker consistency models (such as relaxed or acquire-release) when total sequential consistency is unnecessary.

---

## Self-Check Questions

1. **What is an "effect" in the context of C's abstract state machine?**
   _Answer:_ Any change to the abstract state, including modifying variables, doing I/O, or merely advancing the point of execution.
2. **What does the "sequenced before" relation guarantee?**
   _Answer:_ It guarantees that within a single thread, one evaluation and all its effects are completed before the next evaluation begins.
3. **How do atomic objects synchronize two threads?**
   _Answer:_ Through acquire and release semantics: when Thread A writes (releases) an atomic, and Thread B reads (acquires) that value, Thread B can see all effects Thread A made prior to the write.
4. **Is the "happened before" relation transitive?**
   _Answer:_ Yes. If A happened before B, and B happened before C, then A happened before C.
5. **How does unlocking a mutex interact with the "happened before" relation?**
   _Answer:_ Unlocking a mutex performs a release operation, which synchronizes with the acquire operation performed by the next thread that locks the mutex.
6. **Why do `cnd_wait` and `cnd_timedwait` require a mutex as an argument?**
   _Answer:_ They temporarily release the mutex to sleep and re-acquire it upon waking, ensuring the thread re-enters the critical section safely to evaluate the condition.
7. **Where should a call to `cnd_signal` ideally be placed?**
   _Answer:_ Inside a critical section protected by the same mutex that the waiting threads are using.
8. **What is sequential consistency?**
   _Answer:_ The default memory model for atomics that guarantees all atomic operations across all threads appear to execute in a single, globally agreed-upon order.
9. **Why might a programmer choose not to use sequential consistency?**
   _Answer:_ For performance reasons; sequential consistency prevents hardware optimizations and instruction reordering.
10. **What does `memory_order_relaxed` guarantee?**
    _Answer:_ It guarantees only that the specific atomic operation is indivisible (no data tearing), but it provides no synchronization or ordering guarantees with respect to other memory operations.
