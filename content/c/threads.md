+++
title = "Modern C: Threads"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 20: Threads** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book explores concurrent execution in C. Standard C multithreading (introduced in C11 via `<threads.h>` and refined in C23) allows programs to split execution into concurrent **threads**, enabling asynchronous tasks, parallel processing, and responsive user interfaces.

Below is a detailed breakdown of the core concepts, synchronization primitives, resource management rules, and key takeaways from Chapter 20 across its seven main sections.


### 1. Simple Interthread Control (Section 20.1)
Threads run concurrently as independent execution flows sharing a single address space. The basic thread control API is defined in `<threads.h>`:
* **Thread Creation (`thrd_create`):** Spawns a new thread executing a start function matching `int (*thrd_start_t)(void*)`.
* **Thread Joining (`thrd_join`):** Blocks the calling thread until the specified thread finishes and retrieves its integer return value.
* **Race Conditions & Atomics:** Simultaneously reading and writing non-atomic shared variables across threads without synchronization causes a **race condition**, which leads to execution failure.
* **Takeaway 20.1 #1:** *If a thread \\(T_0\\) writes a non-atomic object that is simultaneously read or written by another thread \\(T_1\\), the execution fails*.
* **Takeaway 20.1 #2:** *In view of execution in different threads, standard operations on atomic objects are indivisible and linearizable*.
* **Takeaway 20.1 #3:** *Use the specifier syntax `_Atomic(T)` for atomic declarations*. Prefer specifier syntax over qualifier syntax for clarity.
* **Takeaway 20.1 #4:** *There are no atomic array types*. To make array elements atomic, declare an array of atomic base elements (e.g., `_Atomic(double) A`).
* **Takeaway 20.1 #5:** *Atomic objects are the privileged tool to force the absence of race conditions*.


### 2. Race-Free Initialization and Destruction (Section 20.2)
Shared objects must be initialized into a controlled state before any concurrent thread reads or writes them, and they must never be accessed after destruction.

* **Initialization Strategies:**
  1. **Static Duration:** Objects with static storage duration are initialized before thread creation.
  2. **Parent Pre-Initialization:** The thread creating a shared object initializes it *before* launching concurrent threads that access it.
  3. **One-Time Dynamic Initialization (`call_once`):** For static objects requiring dynamic setup after startup, `call_once(&flag, func)` guarantees that `func` is executed exactly once across all threads, holding back other threads until initialization completes.
* **Standard I/O Streams:** Most C library stream functions are required to be thread-safe.
* **Takeaway 20.2 #1:** *A properly initialized `FILE*` can be used race-free by several threads*.
* **Takeaway 20.2 #2:** *Concurrent write operations should print entire lines at once*.
* **Takeaway 20.2 #3:** *Destruction and deallocation of shared dynamic objects need a lot of care*. Ensure all threads have finished accessing shared memory before invoking `free()` or closing file streams.


### 3. Thread-Local Data (Section 20.3)
The simplest way to eliminate race conditions is to keep data completely thread-private rather than shared.

* **Takeaway 20.3 #1:** *Pass thread-specific data through function arguments*.
* **Takeaway 20.3 #2:** *Keep thread-specific state in local variables*.
* **`thread_local` Storage Class:** Annotating a variable with `thread_local` (or `_Thread_local`) gives each thread its own independent instance.
* **Takeaway 20.3 #3:** *A `thread_local` variable has one separate instance for each thread*.
* **Takeaway 20.3 #4:** *Use `thread_local` if initialization can be determined at compile time*.
* **Thread-Specific Storage (`tss_t`):** When dynamic construction and destruction per thread are needed, use key-based thread-specific storage via `tss_create`, `tss_get`, `tss_set`, and `tss_delete` with a custom destructor callback (`tss_dtor_t`).


### 4. Critical Data and Critical Sections (`mtx_t`) (Section 20.4)
For complex data structures (like arrays, matrices, or structs) that cannot be declared atomic, thread access must be serialized using a **mutex** (`mtx_t`) to guard the **critical section**.

* **Mutex Lifetime & Operations:**
  * **Takeaway 20.4 #1:** *Mutex operations provide linearizability*. Locking a mutex ensures all prior modifications made before releasing that mutex are visible to the acquiring thread.
  * **Takeaway 20.4 #2:** *Every mutex must be initialized with `mtx_init`*. Standard C defines no static initializer for `mtx_t`. Mutex types include `mtx_plain`, `mtx_timed`, and recursive variants (`mtx_recursive`).
  * **Takeaway 20.4 #3:** *A thread that holds a nonrecursive mutex must not call any of the mutex lock functions for it*. Doing so causes immediate deadlock.
  * **Takeaway 20.4 #4:** *A recursive mutex is only released after the holding thread issues as many calls to `mtx_unlock` as it has acquired locks*.
  * **Takeaway 20.4 #5:** *A locked mutex must be released before the termination of the thread*.
  * **Takeaway 20.4 #6:** *A thread must only call `mtx_unlock` on a mutex that it holds*.
  * **Takeaway 20.4 #7:** *Each successful mutex lock corresponds to exactly one call to `mtx_unlock`*.
  * **Takeaway 20.4 #8:** *A mutex must be destroyed at the end of its lifetime*. Call `mtx_destroy` before a mutex goes out of scope or is freed.


### 5. Communicating Through Condition Variables (`cnd_t`) (Section 20.5)
Condition variables (`cnd_t`) allow threads to sleep efficiently until signaled that a specific predicate or state change has occurred, avoiding wasteful busy-polling.

* **Waiting & Signaling Mechanics:**
  * Threads suspend execution using `cnd_wait(&cnd, &mtx)` or `cnd_timedwait(&cnd, &mtx, &ts)`. The associated mutex is temporarily released during the wait and reacquired before the function returns.
  * Waking sleeping threads is triggered via `cnd_signal(&cnd)` (wakes one thread) or `cnd_broadcast(&cnd)` (wakes all waiting threads).
* **Condition Predicate Loop:**
  * **Takeaway 20.5 #1:** *On return from a `cnd_t` wait, the expression must be checked again*. Spurious wakeups or state changes by other threads mean `cnd_wait` calls must always be wrapped inside a `while (!predicate)` loop.
* **Mutex Coupling & Lifetime:**
  * **Takeaway 20.5 #2:** *A condition variable can only be used simultaneously with one mutex*.
  * **Takeaway 20.5 #3:** *A `cnd_t` must be initialized dynamically*. Use `cnd_init(&cnd)`.
  * **Takeaway 20.5 #4:** *A `cnd_t` must be destroyed at the end of its lifetime*. Use `cnd_destroy(&cnd)`.


### 6. More Sophisticated Thread Management (Section 20.6)
Threads are peer tasks rather than strict hierarchies. Any thread can manage others if it holds their `thrd_t` handle.

* **Process Termination:**
  * **Takeaway 20.6 #1:** *Returning from `main` or calling `exit` terminates all threads*.
  * To exit `main` without terminating background threads, invoke `thrd_exit(result)` instead of `return`.
* **Resource Release & Detaching:**
  * Threads that will not be joined via `thrd_join` must be detached using `thrd_detach(thrd_current())` to free system resources automatically upon exit.
* **Resource Yielding:**
  * **Takeaway 20.6 #2:** *While blocking on `mtx_t` or `cnd_t`, a thread frees processing resources*.
  * Threads can voluntarily pause or yield time slices using `thrd_sleep(&duration, &remaining)` or `thrd_yield()`.


### 7. Ensuring Liveness & Preventing Deadlocks (Section 20.7)
A **deadlock** occurs when threads freeze permanently while waiting for resources locked by each other.

* **Lock Hierarchy Rule:**
  * **Takeaway 20.7 #1:** *Critical sections that need several mutexes to be locked should always lock these mutexes in the same order*. Enforcing a global locking order eliminates circular wait deadlocks.
* **Timeout Guards:**
  * **Takeaway 20.7 #2:** *Prefer `cnd_timedwait` over `cnd_wait` to avoid deadlocks*. Bounding wait times guarantees that threads eventually wake up to re-evaluate state or exit gracefully.


### Summary Table of Chapter 20 Key Takeaways

| Takeaway ID | Takeaway Rule Text |
| :--- | :--- |
| **20.1 #1** | *If a thread \\(T_0\\) writes a non-atomic object that is simultaneously read or written by another thread \\(T_1\\), the execution fails*. |
| **20.1 #2** | *In view of execution in different threads, standard operations on atomic objects are indivisible and linearizable*. |
| **20.1 #3** | *Use the specifier syntax `_Atomic(T)` for atomic declarations*. |
| **20.1 #4** | *There are no atomic array types*. |
| **20.1 #5** | *Atomic objects are the privileged tool to force the absence of race conditions*. |
| **20.2 #1** | *A properly initialized `FILE*` can be used race-free by several threads*. |
| **20.2 #2** | *Concurrent write operations should print entire lines at once*. |
| **20.2 #3** | *Destruction and deallocation of shared dynamic objects need a lot of care*. |
| **20.3 #1** | *Pass thread-specific data through function arguments*. |
| **20.3 #2** | *Keep thread-specific state in local variables*. |
| **20.3 #3** | *A `thread_local` variable has one separate instance for each thread*. |
| **20.3 #4** | *Use `thread_local` if initialization can be determined at compile time*. |
| **20.4 #1** | *Mutex operations provide linearizability*. |
| **20.4 #2** | *Every mutex must be initialized with `mtx_init`*. |
| **20.4 #3** | *A thread that holds a nonrecursive mutex must not call any of the mutex lock functions for it*. |
| **20.4 #4** | *A recursive mutex is only released after the holding thread issues as many calls to `mtx_unlock` as it has acquired locks*. |
| **20.4 #5** | *A locked mutex must be released before the termination of the thread*. |
| **20.4 #6** | *A thread must only call `mtx_unlock` on a mutex that it holds*. |
| **20.4 #7** | *Each successful mutex lock corresponds to exactly one call to `mtx_unlock`*. |
| **20.4 #8** | *A mutex must be destroyed at the end of its lifetime*. |
| **20.5 #1** | *On return from a `cnd_t` wait, the expression must be checked again*. |
| **20.5 #2** | *A condition variable can only be used simultaneously with one mutex*. |
| **20.5 #3** | *A `cnd_t` must be initialized dynamically*. |
| **20.5 #4** | *A `cnd_t` must be destroyed at the end of its lifetime*. |
| **20.6 #1** | *Returning from `main` or calling `exit` terminates all threads*. |
| **20.6 #2** | *While blocking on `mtx_t` or `cnd_t`, a thread frees processing resources*. |
| **20.7 #1** | *Critical sections that need several mutexes to be locked should always lock these mutexes in the same order*. |
| **20.7 #2** | *Prefer `cnd_timedwait` over `cnd_wait` to avoid deadlocks*. |

