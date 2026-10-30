+++
title = "Modern C: Threads"
description = "A guide to C23 threads and synchronization, covering atomics, thread-local state, mutexes, condition variables, thread lifecycles, and deadlock prevention."
date = 2026-09-28T07:31:17Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 20. Threads

### Overview

**What it is:** Threads are a variation of control flow that allow programs to pursue several _tasks_ concurrently. A task is a part of the job to be executed by a program such that different tasks can be done with little or no interaction between each other. Threads in C are managed via the `<threads.h>` header, utilizing types like `thrd_t` (an opaque thread identifier) and `thrd_start_t` (a function pointer for the thread's starting point).

**Why it matters / how it works:** Concurrency enables programs to utilize modern multi-core processors efficiently, performing actions like drawing, computing, and handling I/O simultaneously (as demonstrated by the book's B9 Game of Life example). The `thrd_create` function launches a new thread passing a `void*` argument, and `thrd_join` blocks the calling thread until the target thread terminates.

**Key details:**

- `thrd_create` signature: `int thrd_create(thrd_t*, thrd_start_t, void*);`
- Thread start function signature: `int func(void*);`
- `thrd_join` captures the thread's return value

**Pitfalls:** Failing to manage simultaneous access to shared objects causes race conditions.

**Connections:** Builds upon standard control flow and introduces concurrency, requiring an understanding of atomic access (Section 21).

**Recap:**

- Threads run tasks concurrently.
- Managed via `<threads.h>`.
- Created with `thrd_create` and synchronized at the end with `thrd_join`.

```c
#include <stdio.h>
#include <threads.h>

// A simple thread start function compatible with thrd_start_t
int my_thread_func(void* arg) {
    int* val = (int*)arg;
    printf("Hello from thread! Value: %d\\n", *val);
    return 0; // Return value collected by thrd_join
}

int main(void) {
    thrd_t thread_id;
    int data = 42;

    // Create the thread, pass the address of data
    if (thrd_create(&thread_id, my_thread_func, &data) != thrd_success) {
        return 1;
    }

    // Wait for the thread to finish
    thrd_join(thread_id, nullptr);
    printf("Thread finished.\\n");
    return 0;
}
/* Expected behavior/output:
Hello from thread! Value: 42
Thread finished.
What goes wrong if... we forget thrd_join? The main thread might exit before the spawned thread finishes its printf, terminating the program prematurely.
*/

```

---

## 20.1 Simple interthread control

### Race Conditions and Atomics

**What it is:** A race condition occurs when there is potentially simultaneous, unprotected access to shared variables by multiple threads, where at least one access is a write . C solves this fundamentally via atomic objects.

> Takeaway 20.1 #1 _If a thread T0 writes a non-atomic object that is simultaneously read or written by another thread T1, the execution fails._

**Explanation:** Non-atomic objects do not guarantee indivisible read/write cycles. Interleaved instructions can leave the variable in an undefined, corrupted state .

> Takeaway 20.1 #2 _In view of execution in different threads, standard operations on atomic objects are indivisible and linearizable._

**Explanation:** "Indivisible" means the operation completes fully without interruption. "Linearizable" means operations across threads appear in a strict, globally consistent sequential order .

> Takeaway 20.1 #3 _Use the specifier syntax `_Atomic(T)` for atomic declarations._

**Explanation:** This qualifies a type `T` to be handled atomically by the compiler and hardware.

> Takeaway 20.1 #4 _There are no atomic array types._

**Explanation:** You cannot declare an array itself as atomic; atomic operations apply to scalar objects . (You can have an array of atomic objects, though it's often inefficient).

> Takeaway 20.1 #5 _Atomic objects are the privileged tool to force the absence of race conditions._

**Explanation:** The compiler explicitly manages atomic objects to ensure safe concurrent access, removing the undefined behavior of data races .

**Recap:**

- Data races cause undefined behavior.
- Atomics (`_Atomic(T)`) provide indivisible, safe access.
- Arrays cannot be atomic .

```c
#include <stdio.h>
#include <threads.h>

// _Atomic specifier prevents race conditions on this shared boolean
_Atomic(bool) finished = false;

int signal_thread(void* arg) {
    // Indivisible write operation
    finished = true;
    return 0;
}

int main(void) {
    thrd_t tid;
    thrd_create(&tid, signal_thread, nullptr);

    // Indivisible read operations
    while (!finished) {
        // block/wait
    }
    printf("Finished flag detected as true.\\n");
    thrd_join(tid, nullptr);
    return 0;
}
/* Expected behavior/output:
Finished flag detected as true.
What goes wrong if... `finished` was a plain `bool`?
A data race occurs (Takeaway 20.1 #1), which could lead the compiler to optimize out the while loop entirely, causing an infinite loop.
*/

```

---

## 20.2 Race-free initialization and destruction

### Thread-safe Setup via call_once

**What it is:** Guaranteeing that shared data is set up before any thread accesses it and safely destroyed when all threads are done . `call_once` ensures a function is executed exactly once across all threads .

> Takeaway 20.2 #1 _A properly initialized `FILE*` can be used race-free by several threads._

**Explanation:** Most C standard stream functions (except `fopen`/`fclose`) are required to be race-free, making thread-safe logging possible .

> Takeaway 20.2 #2 _Concurrent write operations should print entire lines at once._

**Explanation:** While `FILE*` accesses are race-free (no corruption), character-by-character printing from multiple threads will interleave output confusingly. Printing a full line ensures logical coherence .

> Takeaway 20.2 #3 _Destruction and deallocation of shared dynamic objects need a lot of care._

**Explanation:** Knowing when the last thread has finished using an object is difficult . Premature freeing causes use-after-free bugs. Memory leaks occur if never freed .

**Recap:**

- Use static initialization where possible.
- Use `call_once` with `once_flag` for dynamic one-time initialization .
- Clean up securely (e.g., in `main` after all `thrd_join` calls) .

```c
#include <stdio.h>
#include <threads.h>

once_flag flag = ONCE_FLAG_INIT;

void do_init(void) {
    printf("Initialization occurred exactly once.\\n");
}

int worker(void* arg) {
    // Guarantees do_init is called only once, even with multiple workers
    call_once(&flag, do_init);
    return 0;
}

int main(void) {
    thrd_t t1, t2;
    thrd_create(&t1, worker, nullptr);
    thrd_create(&t2, worker, nullptr);

    thrd_join(t1, nullptr);
    thrd_join(t2, nullptr);
    return 0;
}
/* Expected behavior/output:
Initialization occurred exactly once.
What goes wrong if... we used a simple boolean instead of call_once?
Two threads checking `if(!initialized)` simultaneously might both run the initialization code.
*/

```

---

## 20.3 Thread-local data

### Data Isolation

**What it is:** Separating data by thread so synchronization is completely unnecessary . C provides the `thread_local` storage class and `tss_t` (thread-specific storage) .

> Takeaway 20.3 #1 _Pass thread-specific data through function arguments._

**Explanation:** The easiest way to isolate state is by passing distinct heap/stack pointers to the `thrd_create` void pointer argument .

> Takeaway 20.3 #2 _Keep thread-specific state in local variables._

**Explanation:** Function-local automatic variables are unique to the thread executing the function .

> Takeaway 20.3 #3 _A `thread_local` variable has one separate instance for each thread._

**Explanation:** Similar to static storage, but instantiated per-thread .

> Takeaway 20.3 #4 _Use `thread_local` if initialization can be determined at compile time._

**Explanation:** Because it works like static storage duration, dynamic initialization (like `malloc`) isn't directly compatible at declaration. For dynamic needs, use `tss_t` with `tss_create` .

**Recap:**

- Prefer function arguments/local variables for safety .
- `thread_local` provides distinct file/static-scope variables per thread .
- `tss_t` handles dynamic thread-specific storage .

```c
#include <stdio.h>
#include <threads.h>

// Each thread gets its own copy of this variable
thread_local int thread_id = 0;

int worker(void* arg) {
    thread_id = *(int*)arg; // Safely writes without locking
    printf("Thread %d sees thread_id: %d\\n", *(int*)arg, thread_id);
    return 0;
}

int main(void) {
    thrd_t t1, t2;
    int id1 = 1, id2 = 2;
    thrd_create(&t1, worker, &id1);
    thrd_create(&t2, worker, &id2);
    thrd_join(t1, nullptr);
    thrd_join(t2, nullptr);
    return 0;
}
/* Expected output:
Thread 1 sees thread_id: 1
Thread 2 sees thread_id: 2
What goes wrong if... `thread_local` was omitted?
A race condition on a global `thread_id` would result in undefined behavior.
*/

```

---

## 20.4 Critical data and critical sections

### Mutexes (`mtx_t`)

**What it is:** When data is too large/complex for `_Atomic` (like large structs or arrays), we protect the code block accessing it (the _critical section_) using a mutex (`mtx_t` for mutual exclusion) .

> Takeaway 20.4 #1 _Mutex operations provide linearizability._

**Explanation:** A thread acquiring a mutex can rely on seeing all changes made by the previous threads that held and released it .

> Takeaway 20.4 #2 _Every mutex must be initialized with `mtx_init`._

**Explanation:** Mutexes have no static initializer in C; `mtx_init(&mtx, mtx_plain)` is mandatory before use .

> Takeaway 20.4 #3 _A thread that holds a nonrecursive mutex must not call any of the locking functions for the same mutex again._

**Explanation:** Attempting to lock an already-held non-recursive mutex causes a deadlock .

> Takeaway 20.4 #8 _A mutex must be destroyed at the end of its lifetime._

**Explanation:** Mutexes tie up system resources. You must call `mtx_destroy` before freeing the memory of a dynamically allocated mutex, or before its scope ends .

**Recap:**

- Use `mtx_t` for large data protection .
- Must explicitly initialize (`mtx_init`) and destroy (`mtx_destroy`) .
- Provides global ordering (linearizability) .

```c
#include <stdio.h>
#include <threads.h>

mtx_t mtx;
int shared_counter = 0;

int worker(void* arg) {
    // Enter critical section
    mtx_lock(&mtx);
    shared_counter++; // Safe access to non-atomic shared data
    mtx_unlock(&mtx);
    // Exit critical section
    return 0;
}

int main(void) {
    mtx_init(&mtx, mtx_plain); // Mandatory initialization

    thrd_t t1, t2;
    thrd_create(&t1, worker, nullptr);
    thrd_create(&t2, worker, nullptr);

    thrd_join(t1, nullptr);
    thrd_join(t2, nullptr);

    printf("Final counter: %d\\n", shared_counter);
    mtx_destroy(&mtx); // Mandatory destruction
    return 0;
}
/* Expected output:
Final counter: 2
What goes wrong if... mtx_destroy is omitted?
System resources associated with the mutex might leak.
*/

```

---

## 20.5 Communicating through condition variables

### Event Waiting (`cnd_t`)

**What it is:** A condition variable (`cnd_t`) suspends a thread efficiently until a specific condition becomes true, avoiding busy-waiting .

> Takeaway 20.5 #1 _On return from a `cnd_t` wait, the expression must be checked again._

**Explanation:** Wait functions (`cnd_wait`, `cnd_timedwait`) can suffer from "spurious wakeups", or another thread might have changed the condition before the waking thread re-acquired the mutex. Always wrap waits in a `while` loop checking the condition .

> Takeaway 20.5 #2 _A condition variable can only be used simultaneously with one mutex._

**Explanation:** To guarantee consistency during the unlock-wait-relock cycle, `cnd_wait` strictly binds to a specific mutex during the wait .

> Takeaway 20.5 #3 _A `cnd_t` must be initialized dynamically._

**Explanation:** `cnd_init` is required before use .

> Takeaway 20.5 #4 _A `cnd_t` must be destroyed at the end of its lifetime._

**Explanation:** Like mutexes, `cnd_destroy` must be called to free OS resources .

**Recap:**

- `cnd_wait` temporarily releases the mutex while waiting, and re-acquires it before returning .
- Always wait inside a `while (condition_is_false)` loop .
- Wake waiters using `cnd_signal` (one) or `cnd_broadcast` (all) .

```c
#include <stdio.h>
#include <threads.h>

mtx_t mtx;
cnd_t cv;
bool ready = false;

int waiter(void* arg) {
    mtx_lock(&mtx);
    // Takeaway 20.5 #1: Always check expression in a loop
    while (!ready) {
        cnd_wait(&cv, &mtx);
    }
    printf("Waiter proceeding!\\n");
    mtx_unlock(&mtx);
    return 0;
}

int main(void) {
    mtx_init(&mtx, mtx_plain);
    cnd_init(&cv);

    thrd_t t;
    thrd_create(&t, waiter, nullptr);

    // Simulate work, then wake the thread
    mtx_lock(&mtx);
    ready = true;
    cnd_signal(&cv); // Wake up the waiter
    mtx_unlock(&mtx);

    thrd_join(t, nullptr);

    cnd_destroy(&cv);
    mtx_destroy(&mtx);
    return 0;
}
/* Expected output:
Waiter proceeding!
*/

```

---

## 20.6 More sophisticated thread management

### Detached Threads and System Lifetimes

**What it is:** Threads don't have a hierarchical parent-child structure; any thread can be detached (`thrd_detach`), allowing it to run independently without being `join`ed .

> Takeaway 20.6 #1 _Returning from main or calling exit terminates all threads._

**Explanation:** Regardless of their state, all threads are aggressively shut down if `main` returns or `exit()` is called . To allow detached threads to finish smoothly, use `thrd_exit(0)` inside `main` .

> Takeaway 20.6 #2 _While blocking on `mtx_t` or `cnd_t`, a thread frees processing resources._

**Explanation:** Blocking synchronization puts the thread to sleep, yielding CPU time slices to other threads, unlike busy-wait loops (`while(!flag) {}`) which burn CPU cycles .

**Recap:**

- Detach threads when their return value isn't needed.
- Blocking primitives save system CPU time .

```c
#include <stdio.h>
#include <threads.h>

int background_task(void* arg) {
    printf("Background task running...\\n");
    return 0;
}

int main(void) {
    thrd_t t;
    thrd_create(&t, background_task, nullptr);

    // Detach the thread. OS will reclaim resources when it finishes.
    thrd_detach(t);

    // Using thrd_exit in main prevents early termination of detached threads
    // thrd_exit(0);
    return 0;
}

```

---

## 20.7 Ensure liveness

### Preventing Deadlocks

**What it is:** A deadlock occurs when two or more threads block indefinitely, waiting for each other to release mutexes.

> Takeaway 20.7 #1 _Critical sections that need several mutexes to be locked should always lock these mutexes in the same order._

**Explanation:** If Thread A locks Mutex 1 then Mutex 2, and Thread B locks Mutex 2 then Mutex 1, a race can cause A to hold 1 and wait for 2, while B holds 2 and waits for 1. Total gridlock .

> Takeaway 20.7 #2 _Prefer `cnd_timedwait` over `cnd_wait` to avoid deadlocks._

**Explanation:** A timed wait allows the thread to break out of an infinite wait if a signal is missed due to a logic bug or unhandled edge case .

**Recap:**

- Always lock multiple mutexes in a globally agreed-upon order .
- Use timeouts on condition variables to guarantee forward progress .

---

## Summary

Threads introduce powerful concurrency to C via the `<threads.h>` API, allowing tasks to operate concurrently . Safe interaction requires strict synchronization. `_Atomic` variables handle simple data sharing indivisibly . For larger structures, Mutexes (`mtx_t`) protect critical sections , and condition variables (`cnd_t`) coordinate task execution efficiently without busy-waiting . Managing thread-local data (`thread_local`), one-time initializations (`call_once`), and observing lock hierarchies are critical techniques to avoid race conditions, memory leaks, and deadlocks .

## Self-Check Questions

1. **Q: How do you initiate a new thread in C?**
   A: By calling `thrd_create`, providing a `thrd_t` pointer, a start function, and a `void*` argument .

2. **Q: What happens if two threads write to a standard `int` simultaneously?**
   A: A race condition occurs, resulting in undefined behavior (execution fails) .

3. **Q: How do you prevent race conditions on small scalar variables?**
   A: Declare them with the `_Atomic` type specifier .

4. **Q: What is the primary purpose of a mutex?**
   A: To protect critical sections, ensuring only one thread can access complex shared data at a time .

5. **Q: Why must a `cnd_wait` call be placed inside a `while` loop?**
   A: To re-evaluate the condition expression upon waking up, protecting against spurious wakeups or state changes .

6. **Q: How are `mtx_t` and `cnd_t` objects properly cleaned up?**
   A: Using `mtx_destroy` and `cnd_destroy`, respectively, at the end of their lifetimes .

7. **Q: What happens to a spawned thread if `main` returns?**
   A: The thread is abruptly terminated regardless of its execution state .

8. **Q: How can you prevent deadlocks when multiple mutexes are needed?**
   A: Always acquire the mutexes in the exact same order across all threads .
