+++
title = "Modern C: Threads"
description = "A guide to C23 threads and synchronization, covering atomics, thread-local state, mutexes, condition variables, thread lifecycles, and deadlock prevention."
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

#### Example

Concurrent threads share an address space. If one thread writes to a non-atomic object that another thread reads or writes simultaneously, the execution fails due to a race condition. Standard operations on atomic objects are indivisible and linearizable, making them the primary tool to prevent race conditions.

* **Takeaway 20.1 #1:** *If a thread $(T_0)$ writes a non-atomic object that is simultaneously read or written by another thread $(T_1)$, the execution fails*.
* **Takeaway 20.1 #2:** *In view of execution in different threads, standard operations on atomic objects are indivisible and linearizable*.
* **Takeaway 20.1 #3:** *Use the specifier syntax `_Atomic(T)` for atomic declarations*.
* **Takeaway 20.1 #4:** *There are no atomic array types* (declare arrays of atomic elements instead).
* **Takeaway 20.1 #5:** *Atomic objects are the privileged tool to force the absence of race conditions*.

```c
#include <stdio.h>
#include <stdbool.h>
#include <threads.h> // Provides thrd_t, thrd_create, thrd_join
#include <stdatomic.h> // Provides _Atomic specifier syntax

// Structure holding atomic state shared race-free across threads
typedef struct {
    // Takeaway 20.1 #3: Use specifier syntax _Atomic(T) for atomic fields
    _Atomic(size_t) counter;  // Indivisible atomic increment/read
    _Atomic(bool)   finished; // Linearizable status flag

    // Takeaway 20.1 #4: No atomic array types exist; declare an array of atomic elements
    _Atomic(double) values;
} shared_state;

// Thread start function matching prototype: int (*thrd_start_t)(void*)
static int worker_task(void* arg) {
    shared_state* state = arg;

    for (size_t i = 0; i < 1000; ++i) {
        // Atomic operations are indivisible and linearizable across threads
        state->counter++;
    }

    // Signals completion race-free
    state->finished = true;
    return 0; // Return value retrieved by thrd_join
}

void demo_simple_interthread_control(void) {
    shared_state state = { .counter = 0, .finished = false };
    thrd_t thread_id;

    // Launch worker thread
    if (thrd_create(&thread_id, worker_task, &state) == thrd_success) {
        // Wait for worker thread termination and collect exit status
        int res = 0;
        thrd_join(thread_id, &res);
        printf("Final counter = %zu (Finished flag = %s)\n",
               state.counter, state.finished ? "true" : "false");
    }
}
```

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

#### Example

Shared objects must be initialized into a valid state before any thread reads or writes them. `call_once` guarantees one-time dynamic initialization across threads.

* **Takeaway 20.2 #1:** *A properly initialized `FILE*` can be used race-free by several threads*.
* **Takeaway 20.2 #2:** *Concurrent write operations should print entire lines at once*.
* **Takeaway 20.2 #3:** *Destruction and deallocation of shared dynamic objects need a lot of care*.

```c
#include <stdio.h>
#include <threads.h>

static once_flag init_flag = ONCE_FLAG_INIT; // One-time execution flag
static FILE* shared_log_file = NULL;

static void init_logging_resource(void) {
    // One-time dynamic initialization function
    shared_log_file = stdout; // Connected to standard output stream
}

static int logging_worker(void* arg) {
    char const* thread_name = arg;

    // Guarantees init_logging_resource runs exactly once across all threads
    call_once(&init_flag, init_logging_resource);

    // Takeaway 20.2 #1 & #2: FILE* streams are thread-safe if lines are printed all at once
    if (shared_log_file) {
        fprintf(shared_log_file, "[%s] Logging full line output race-free.\n", thread_name);
    }

    return 0;
}

void demo_race_free_initialization(void) {
    thrd_t t1, t2;
    thrd_create(&t1, logging_worker, "Thread-A");
    thrd_create(&t2, logging_worker, "Thread-B");

    thrd_join(t1, NULL);
    thrd_join(t2, NULL);
    // Takeaway 20.2 #3: Ensure all threads finish before destroying shared objects
}
```

### 3. Thread-Local Data (Section 20.3)
The simplest way to eliminate race conditions is to keep data completely thread-private rather than shared.

* **Takeaway 20.3 #1:** *Pass thread-specific data through function arguments*.
* **Takeaway 20.3 #2:** *Keep thread-specific state in local variables*.
* **`thread_local` Storage Class:** Annotating a variable with `thread_local` (or `_Thread_local`) gives each thread its own independent instance.
* **Takeaway 20.3 #3:** *A `thread_local` variable has one separate instance for each thread*.
* **Takeaway 20.3 #4:** *Use `thread_local` if initialization can be determined at compile time*.
* **Thread-Specific Storage (`tss_t`):** When dynamic construction and destruction per thread are needed, use key-based thread-specific storage via `tss_create`, `tss_get`, `tss_set`, and `tss_delete` with a custom destructor callback (`tss_dtor_t`).

#### Example

Separating data into thread-private instances eliminates race conditions without mutex overhead.

* **Takeaway 20.3 #1:** *Pass thread-specific data through function arguments*.
* **Takeaway 20.3 #2:** *Keep thread-specific state in local variables*.
* **Takeaway 20.3 #3:** *A `thread_local` variable has one separate instance for each thread*.
* **Takeaway 20.3 #4:** *Use `thread_local` if initialization can be determined at compile time*.

```c
#include <stdio.h>
#include <stdlib.h>
#include <threads.h>

// Takeaway 20.3 #3 & #4: Independent per-thread variable with compile-time initialization
static thread_local size_t thread_private_counter = 0;

static tss_t dynamic_key; // Key for dynamic thread-specific storage

static void destructor_callback(void* data_ptr) {
    // Destructor invoked automatically when a thread exits
    free(data_ptr);
}

static int thread_local_task(void* arg) {
    int id = *(int*)arg; // Takeaway 20.3 #1: Pass thread data via arguments

    // Takeaway 20.3 #2: Keep state in local variables or thread_local objects
    thread_private_counter = (size_t)id * 100;

    // DYNAMIC THREAD-SPECIFIC STORAGE (tss_t):
    int* dynamic_val = malloc(sizeof *dynamic_val);
    if (dynamic_val) {
        *dynamic_val = id * 5;
        tss_set(dynamic_key, dynamic_val); // Binds value to current thread
    }

    int* retrieved = tss_get(dynamic_key);
    printf("Thread %d: Private Counter = %zu, TSS Value = %d\n",
           id, thread_private_counter, retrieved ? *retrieved : 0);

    return 0; // Exiting thread triggers destructor_callback for dynamic_key
}

void demo_thread_local_data(void) {
    tss_create(&dynamic_key, destructor_callback); // Initialize TSS key with destructor

    thrd_t t1, t2;
    int id1 = 1, id2 = 2;
    thrd_create(&t1, thread_local_task, &id1);
    thrd_create(&t2, thread_local_task, &id2);

    thrd_join(t1, NULL);
    thrd_join(t2, NULL);

    tss_delete(dynamic_key); // Clean up TSS key
}
```

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

#### Example

Complex shared data structures must be protected inside critical sections guarded by a mutex (`mtx_t`).

* **Takeaway 20.4 #1:** *Mutex operations provide linearizability*.
* **Takeaway 20.4 #2:** *Every mutex must be initialized with `mtx_init`*.
* **Takeaway 20.4 #3:** *A thread that holds a nonrecursive mutex must not call any of the mutex lock functions for it*.
* **Takeaway 20.4 #4:** *A recursive mutex is only released after the holding thread issues as many calls to `mtx_unlock` as it has acquired locks*.
* **Takeaway 20.4 #5:** *A locked mutex must be released before the termination of the thread*.
* **Takeaway 20.4 #6:** *A thread must only call `mtx_unlock` on a mutex that it holds*.
* **Takeaway 20.4 #7:** *Each successful mutex lock corresponds to exactly one call to `mtx_unlock`*.
* **Takeaway 20.4 #8:** *A mutex must be destroyed at the end of its lifetime*.

```c
#include <stdio.h>
#include <threads.h>

typedef struct {
        mtx_t mtx;       // Mutex protecting critical data
        double data; // Critical non-atomic payload
} critical_buffer;

void demo_critical_sections(void) {
        critical_buffer buf;

        // Takeaway 20.4 #2: Always initialize a mutex before use (mtx_plain or mtx_timed)
        if (mtx_init(&buf.mtx, mtx_plain) != thrd_success) return;

        // Takeaway 20.4 #1: Mutex operations provide linearizable critical sections
        mtx_lock(&buf.mtx);
        // --- CRITICAL SECTION BEGIN ---
        buf.data = 42.0;
        // --- CRITICAL SECTION END ---
        // Takeaway 20.4 #5, #6, & #7: Each lock call must be paired with exactly one unlock by the owner
        mtx_unlock(&buf.mtx);

        // Takeaway 20.4 #8: Always destroy a mutex when its lifetime ends
        mtx_destroy(&buf.mtx);
}
```

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

#### Example

Condition variables (`cnd_t`) allow threads to sleep efficiently until signaled that a condition expression has changed.

* **Takeaway 20.5 #1:** *On return from a `cnd_t` wait, the expression must be checked again*.
* **Takeaway 20.5 #2:** *A condition variable can only be used simultaneously with one mutex*.
* **Takeaway 20.5 #3:** *A `cnd_t` must be initialized dynamically*.
* **Takeaway 20.5 #4:** *A `cnd_t` must be destroyed at the end of its lifetime*.

```c
#include <stdio.h>
#include <stdbool.h>
#include <threads.h>
#include <time.h>

typedef struct {
    mtx_t mtx;
    cnd_t cnd;
    bool  work_ready; // Predicate expression
} job_queue;

static int consumer_thread(void* arg) {
    job_queue* q = arg;

    mtx_lock(&q->mtx);

    // Takeaway 20.5 #1: Always re-check the condition expression inside a while loop!
    while (!q->work_ready) {
        struct timespec ts;
        timespec_get(&ts, TIME_UTC);
        ts.tv_sec += 1; // 1-second timeout limit

        // Takeaway 20.5 #2: Use cnd_timedwait with the associated mutex
        cnd_timedwait(&q->cnd, &q->mtx, &ts);
    }

    printf("Consumer: Work processed successfully!\n");
    mtx_unlock(&q->mtx);
    return 0;
}

void demo_condition_variables(void) {
    job_queue q = { .work_ready = false };

    mtx_init(&q.mtx, mtx_plain);
    // Takeaway 20.5 #3: Initialize condition variable dynamically using cnd_init
    cnd_init(&q.cnd);

    thrd_t consumer;
    thrd_create(&consumer, consumer_thread, &q);

    // Producer updates predicate state inside the locked critical section
    mtx_lock(&q.mtx);
    q.work_ready = true;
    cnd_signal(&q.cnd); // Wake up waiting consumer thread
    mtx_unlock(&q.mtx);

    thrd_join(consumer, NULL);

    // Takeaway 20.5 #4: Destroy condition variable and mutex at end of lifetime
    cnd_destroy(&q.cnd);
    mtx_destroy(&q.mtx);
}
```

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

#### Example

Threads are peer tasks rather than strict hierarchies.

* **Takeaway 20.6 #1:** *Returning from `main` or calling `exit` terminates all threads*.
* **Takeaway 20.6 #2:** *While blocking on `mtx_t` or `cnd_t`, a thread frees processing resources*.

```c
#include <stdio.h>
#include <threads.h>

static int detached_worker(void* arg) {
        // Detaches current thread so its system resources are reclaimed automatically upon exit
        thrd_detach(thrd_current());

        printf("Detached worker executing concurrently...\n");

        // Voluntarily yield current time slice to other active threads
        thrd_yield();

        // Takeaway 20.6 #2: thrd_sleep releases processing resources while waiting
        struct timespec delay = { .tv_nsec = 50000000 }; // 50 ms delay
        thrd_sleep(&delay, NULL);

        // Terminate ONLY this thread without killing sibling threads
        thrd_exit(0);
}

void demo_advanced_thread_management(void) {
        thrd_t t;
        thrd_create(&t, detached_worker, NULL);

        // Give detached thread time to finish before main thread exits
        struct timespec wait_time = { .tv_nsec = 100000000 }; // 100 ms
        thrd_sleep(&wait_time, NULL);

        // Takeaway 20.6 #1: Returning from main or calling exit terminates ALL running threads
}
```

### 7. Ensuring Liveness & Preventing Deadlocks (Section 20.7)
A **deadlock** occurs when threads freeze permanently while waiting for resources locked by each other.

* **Lock Hierarchy Rule:**
  * **Takeaway 20.7 #1:** *Critical sections that need several mutexes to be locked should always lock these mutexes in the same order*. Enforcing a global locking order eliminates circular wait deadlocks.
* **Timeout Guards:**
  * **Takeaway 20.7 #2:** *Prefer `cnd_timedwait` over `cnd_wait` to avoid deadlocks*. Bounding wait times guarantees that threads eventually wake up to re-evaluate state or exit gracefully.

#### Example

A deadlock occurs when threads freeze permanently waiting for resources locked by each other.

* **Takeaway 20.7 #1:** *Critical sections that need several mutexes to be locked should always lock these mutexes in the same order*.
* **Takeaway 20.7 #2:** *Prefer `cnd_timedwait` over `cnd_wait` to avoid deadlocks*.

```c
#include <stdio.h>
#include <threads.h>
#include <time.h>

static mtx_t mutex_A;
static mtx_t mutex_B;

void safe_multi_lock_operation(void) {
        // Takeaway 20.7 #1: Always acquire multiple mutexes in a fixed global order (A then B)
        mtx_lock(&mutex_A);
        mtx_lock(&mutex_B);

        // --- Safe Critical Section ---

        mtx_unlock(&mutex_B);
        mtx_unlock(&mutex_A);
}

void safe_bounded_condition_wait(cnd_t* cnd, mtx_t* mtx, bool* predicate) {
        mtx_lock(mtx);

        while (!*predicate) {
                struct timespec timeout;
                timespec_get(&timeout, TIME_UTC);
                timeout.tv_sec += 2; // 2-second upper limit

                // Takeaway 20.7 #2: Prefer cnd_timedwait over cnd_wait to prevent infinite deadlock blocking
                int res = cnd_timedwait(cnd, mtx, &timeout);
                if (res == thrd_timedout) {
                        printf("Wait timed out! Re-evaluating liveness...\n");
                        break; // Break out to avoid permanent blockage
                }
        }

        mtx_unlock(mtx);
}
```

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
