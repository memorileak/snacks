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

In **Chapter 21: Atomic access and memory consistency** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book completes **Level 3: Experience**. Having explored standard multithreading via `<threads.h>` in Chapter 20, Chapter 21 delves into the underlying memory model, formal synchronization rules, and explicit memory consistency models that govern how concurrent operations interact across hardware cores and execution pipelines.


### 1. Abstract State, Evaluations, and Effects (Intro)
Program execution alters the abstract state of the machine through jumps, value computations, side effects (stores and I/O), and updates to hidden runtime state (such as mutex locks or atomic flags).

* **Takeaway 21 #1:** *Every evaluation has an effect*.
* **Relative Time across Threads:** Multithreaded execution across physical cores lacks a single global clock. Thread memory visibility relies on explicit signals and synchronization barriers rather than absolute hardware time.


### 2. The "Happened Before" Relation (`F → E`) (Section 21.1)
The **happened before** relation (`F → E`) provides the formal partial ordering framework required to reason about memory visibility across threads.

* **Sequenced-Before Ordering:**
  * **Takeaway 21.1 #1:** *If `F` is sequenced before `E`, then `F → E`*. Within a single thread, evaluation order defined by C grammar establishes happened-before relationships.
* **Atomic Modification Order:**
  * **Takeaway 21.1 #2:** *The set of modifications of an atomic object `X` is performed in an order consistent with the sequenced before relation of any thread that deals with `X`*. All threads observe changes to a single atomic object in the same modification sequence.
* **Acquire and Release Synchronization:**
  * **Takeaway 21.1 #3:** *An acquire operation `E` in a thread $(T_E)$ synchronizes with a release operation `F` in another thread $(T_F)$ if `E` reads the value that `F` has written*.
  * **Takeaway 21.1 #4:** *If `F` synchronizes with `E`, all effects `X` that happened before `F` must be visible at all evaluations `G` that happen after `E`*.
  * **Takeaway 21.1 #5:** *We can only conclude that one evaluation happened before another if we have a sequenced chain of synchronizations that links them*.
  * **Takeaway 21.1 #6:** *If an evaluation `F` happened before `E`, all effects that are known to have happened before `F` are also known to have happened before `E`*.
* **Read-Modify-Write (RMW) Operations:** Operations like `atomic_exchange`, `atomic_compare_exchange_weak`/`strong`, `atomic_fetch_add`, and `atomic_flag_test_and_set` combine reading and writing into an indivisible step with both acquire and release semantics.

#### Example

Thread synchronization creates a **happened before** relation ($F \to E$) between evaluations. Within a single thread, evaluation order defined by C grammar establishes **sequenced-before** ordering. Across threads, an **acquire operation** $E$ in thread $T_E$ synchronizes with a **release operation** $F$ in thread $T_F$ if $E$ reads the value written by $F$. All modifications to a single atomic variable occur in a single, consistent **modification order**.

* **Takeaway 21.1 #1:** *If $F$ is sequenced before $E$, then $F \to E$*.
* **Takeaway 21.1 #2:** *The set of modifications of an atomic object $X$ is performed in an order consistent with the sequenced before relation of any thread that deals with $X$*.
* **Takeaway 21.1 #3 & #4:** *An acquire operation $E$ synchronizes with a release operation $F$ if $E$ reads what $F$ wrote, making all effects prior to $F$ visible to evaluations after $E$*.
* **Takeaway 21.1 #5 & #6:** *One evaluation knowingly happened before another only through a sequenced chain of synchronizations*.

```c
#include <stdio.h>
#include <stdbool.h>
#include <threads.h>
#include <stdatomic.h>

// Shared payload data (non-atomic) and atomic synchronization flag
static double shared_payload = 0.0;
static _Atomic(bool) ready_flag = false;

static int producer_thread(void* arg) {
  (void)arg;

  // 1. SEQUENCED-BEFORE (Takeaway 21.1 #1):
  // Writing shared_payload is sequenced before the release store to ready_flag.
  shared_payload = 42.0;

  // 2. RELEASE OPERATION (Takeaway 21.1 #3):
  // Writing true to atomic ready_flag acts as a release operation.
  atomic_store(&ready_flag, true); // Release write

  return 0;
}

static int consumer_thread(void* arg) {
  (void)arg;

  // 3. ACQUIRE OPERATION (Takeaway 21.1 #3 & #4):
  // Reading ready_flag inside a loop acts as an acquire operation.
  // Once it reads 'true' (the value written by producer_thread), synchronization is established.
  while (!atomic_load(&ready_flag)) {
    // Busy-wait or yield until release store is observed
    thrd_yield();
  }

  // 4. HAPPENED-BEFORE VISIBILITY (Takeaway 21.1 #4 & #6):
  // Because ready_flag release synchronized with ready_flag acquire,
  // the write to 'shared_payload' in producer_thread is GUARANTEED to be visible here!
  printf("Consumer read payload safely: %g\n", shared_payload);

  return 0;
}

void demo_happened_before_relation(void) {
  thrd_t t1, t2;
  thrd_create(&t1, producer_thread, NULL);
  thrd_create(&t2, consumer_thread, NULL);

  thrd_join(t1, NULL);
  thrd_join(t2, NULL);
}
```

### 3. C Library Calls that Provide Synchronization (Section 21.2)
Standard C library functions form paired release-acquire synchronization points across threads:

| Release Operation | Acquire Operation |
| :--- | :--- |
| `thrd_create(..., f, x)` | Entry to start function `f(x)` |
| `thrd_exit` / `return` from `f` | Start of `tss_t` destructors or `thrd_join(id)` / `atexit` / `at_quick_exit` |
| `call_once(&obj, g)` (first call) | `call_once(&obj, h)` (all subsequent calls) |
| Mutex release (`mtx_unlock`, `cnd_wait` entry) | Mutex acquisition (`mtx_lock`, `cnd_wait` return) |

* **Mutex Critical Sections:**
  * **Takeaway 21.2 #1:** *Critical sections protected by the same mutex occur sequentially*.
  * **Takeaway 21.2 #2:** *In a critical section protected by the mutex `mut`, all effects of previous critical sections protected by `mut` are visible*.
* **Condition Variables:**
  * **Takeaway 21.2 #3:** *`cnd_wait` and `cnd_timedwait` have release-acquire semantics for the mutex*. (They release the mutex when suspending and acquire it when waking up).
  * **Takeaway 21.2 #4:** *Calls to `cnd_signal` and `cnd_broadcast` synchronize via the mutex*.
  * **Takeaway 21.2 #5:** *Calls to `cnd_signal` and `cnd_broadcast` should occur inside a critical section protected by the same mutex as the waiters*.

#### Example

Standard C library functions form paired release-acquire synchronization points across threads:
* **`thrd_create` / `thrd_join`**: `thrd_create` synchronizes with thread entry; thread exit (`return` or `thrd_exit`) synchronizes with `thrd_join`.
* **`call_once`**: The first call executing the callback acts as a release operation for all subsequent acquire calls.
* **Mutexes (`mtx_lock` / `mtx_unlock`)**: Critical sections protected by the same mutex occur sequentially.
* **Condition Variables (`cnd_wait` / `cnd_signal`)**: `cnd_wait` has release-acquire semantics for the mutex. Calls to `cnd_signal` should occur inside a critical section protected by the same mutex as the waiters.

* **Takeaway 21.2 #1 & #2:** *Critical sections protected by the same mutex occur sequentially; all effects of previous critical sections are visible*.
* **Takeaway 21.2 #3:** *`cnd_wait` and `cnd_timedwait` have release-acquire semantics for the mutex*.
* **Takeaway 21.2 #4 & #5:** *Calls to `cnd_signal` and `cnd_broadcast` synchronize via the mutex and should occur inside a critical section*.

```c
#include <stdio.h>
#include <stdbool.h>
#include <threads.h>

typedef struct {
  mtx_t mtx;
  cnd_t cnd;
  bool  data_ready; // Guarded non-atomic payload predicate
  int   payload_val;
} sync_container;

static int worker_producer(void* arg) {
  sync_container* sc = arg;

  // Takeaway 21.2 #1 & #2: Acquire mutex lock to begin critical section
  mtx_lock(&sc->mtx);

  sc->payload_val = 100;
  sc->data_ready = true;

  // Takeaway 21.2 #5: cnd_signal should occur inside the critical section
  // protected by the same mutex as the waiters to guarantee visibility.
  cnd_signal(&sc->cnd); // Takeaway 21.2 #4: Synchronizes via the mutex

  mtx_unlock(&sc->mtx); // Mutex release operation
  return 0;
}

static int worker_consumer(void* arg) {
  sync_container* sc = arg;

  mtx_lock(&sc->mtx); // Acquire lock

  // Takeaway 21.2 #3: cnd_wait releases the mutex while sleeping and
  // re-acquires it with acquire semantics upon waking up.
  while (!sc->data_ready) {
    cnd_wait(&sc->cnd, &sc->mtx);
  }

  printf("Consumer retrieved critical payload: %d\n", sc->payload_val);
  mtx_unlock(&sc->mtx);
  return 0;
}

void demo_library_synchronization(void) {
  sync_container sc = { .data_ready = false, .payload_val = 0 };
  mtx_init(&sc.mtx, mtx_plain);
  cnd_init(&sc.cnd);

  thrd_t p, c;
  thrd_create(&p, worker_producer, &sc); // thrd_create release-synchronizes with thread start
  thrd_create(&c, worker_consumer, &sc);

  thrd_join(p, NULL); // Thread exit release-synchronizes with thrd_join acquire
  thrd_join(c, NULL);

  cnd_destroy(&sc.cnd);
  mtx_destroy(&sc.mtx);
}
```

### 4. Sequential Consistency (Section 21.3)
**Sequential consistency** is the strongest and default memory consistency model in C.

* **Global Linearization:**
  * **Takeaway 21.3 #1:** *All atomic operations with sequential consistency occur in one global modification order, regardless of the atomic object they are applied to*.
  * **Takeaway 21.3 #2:** *All operators and functional interfaces on atomics that don’t specify otherwise have sequential consistency*.
* **Default Atomic Functions:** Standard functions like `atomic_store`, `atomic_load`, `atomic_exchange`, `atomic_fetch_add`, and `atomic_compare_exchange_strong` enforce sequential consistency unless explicit memory orders are passed.

#### Example

**Sequential consistency** (`memory_order_seq_cst`) is the default and strongest memory consistency model in C. It enforces that all atomic operations across all threads occur in **one global modification order**.

* **Takeaway 21.3 #1:** *All atomic operations with sequential consistency occur in one global modification order, regardless of the atomic object they are applied to*.
* **Takeaway 21.3 #2:** *All operators and functional interfaces on atomics that don't specify otherwise have sequential consistency*.

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdatomic.h>

void demo_sequential_consistency(void) {
  _Atomic(int)  counter = 0;
  _Atomic(bool) flag = false;

  // Takeaway 21.3 #2: Standard C functional interfaces on atomics default to memory_order_seq_cst
  atomic_store(&flag, true); // Globally ordered store

  // Read-Modify-Write (RMW) atomic operations returning prior value:
  int prev_val = atomic_fetch_add(&counter, 5); // Indivisible RMW with total ordering

  // atomic_exchange replaces value and returns previous value
  int exchanged = atomic_exchange(&counter, 42);

  // atomic_compare_exchange_strong compares counter with expected (10); updates to desired (100) if equal
  int expected = 42;
  bool success = atomic_compare_exchange_strong(&counter, &expected, 100);

  printf("Prev: %d, Exchanged: %d, CAS Success? %s (Final: %d)\n",
       prev_val, exchanged, success ? "true" : "false", atomic_load(&counter));

  // Note: atomic_init sets an initial value without synchronization barriers
  _Atomic(int) un-synced;
  atomic_init(&un-synced, 0); // Cheap non-synchronizing initialization
}
```

### 5. Explicit Consistency Models & `_explicit` Functions (Section 21.4)
When hardware performance demands weaker synchronization barriers, explicit functional interfaces can be appended with `_explicit` to specify precise `memory_order` flags.

* **Takeaway 21.4 #1:** *Synchronizing functional interfaces for atomic objects with `_explicit` appended allow us to specify their consistency model*.
* **The 6 `memory_order` Enums:**
  1. **`memory_order_seq_cst`**: Full sequential consistency (total global order).
  2. **`memory_order_acq_rel`**: Acquire-release semantics for Read-Modify-Write operations.
  3. **`memory_order_release`**: Release semantics for store operations (`atomic_store`, `atomic_flag_clear`).
  4. **`memory_order_acquire`**: Acquire semantics for load operations (`atomic_load`).
  5. **`memory_order_consume`**: Data-dependency-ordered acquire semantics.
  6. **`memory_order_relaxed`**: Guarantees atomicity/indivisibility without enforcing synchronization or memory ordering barriers (ideal for performance counters).
* **Operation Constraints:** Store-only operations cannot specify acquire semantics, load-only operations cannot specify release semantics, and compare-exchange functions take two memory orders (one for success, one for failure).

#### Example

When maximum hardware performance is required, functions appended with **`_explicit`** allow selecting fine-grained `memory_order` flags.

* **Takeaway 21.4 #1:** *Synchronizing functional interfaces for atomic objects with `_explicit` appended allow us to specify their consistency model*.
* **The `memory_order` Enums**:
  * **`memory_order_seq_cst`**: Total global ordering (default).
  * **`memory_order_acq_rel`**: Combined acquire-release semantics for Read-Modify-Write operations.
  * **`memory_order_release`**: Release semantics for store operations.
  * **`memory_order_acquire`**: Acquire semantics for load operations.
  * **`memory_order_relaxed`**: Guarantees atomicity/indivisibility without enforcing synchronization or memory ordering barriers (ideal for performance statistics/counters).

```c
#include <stdio.h>
#include <stdbool.h>
#include <threads.h>
#include <stdatomic.h>

static _Atomic(size_t) global_metrics_counter = 0;
static _Atomic(bool)   data_published = false;
static int             data_buffer = 0;

static int explicit_writer(void* arg) {
  (void)arg;

  // 1. RELAXED MEMORY ORDER (memory_order_relaxed):
  // Indivisible atomic increment with NO synchronization barriers (fast performance counter)
  atomic_fetch_add_explicit(&global_metrics_counter, 1, memory_order_relaxed);

  data_buffer = 999;

  // 2. RELEASE MEMORY ORDER (memory_order_release):
  // Ensures data_buffer store happens before data_published release store
  atomic_store_explicit(&data_published, true, memory_order_release);

  return 0;
}

static int explicit_reader(void* arg) {
  (void)arg;

  // 3. ACQUIRE MEMORY ORDER (memory_order_acquire):
  // Synchronizes with memory_order_release store in writer
  while (!atomic_load_explicit(&data_published, memory_order_acquire)) {
    thrd_yield();
  }

  // data_buffer is guaranteed visible here due to acquire-release pair
  printf("Explicit Acquire Read Buffer: %d\n", data_buffer);

  // 4. READ-MODIFY-WRITE WITH ACQ_REL (memory_order_acq_rel):
  size_t prev_count = atomic_fetch_add_explicit(&global_metrics_counter, 1, memory_order_acq_rel);
  printf("Previous relaxed metric count: %zu\n", prev_count);

  return 0;
}

void demo_explicit_consistency_models(void) {
  thrd_t w, r;
  thrd_create(&w, explicit_writer, NULL);
  thrd_create(&r, explicit_reader, NULL);

  thrd_join(w, NULL);
  thrd_join(r, NULL);
}
```
