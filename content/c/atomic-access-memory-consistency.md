+++
title = "Modern C: Atomic access and memory consistency"
description = ""
date = 2026-09-28

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
  * **Takeaway 21.1 #3:** *An acquire operation `E` in a thread \\(T_E\\) synchronizes with a release operation `F` in another thread \\(T_F\\) if `E` reads the value that `F` has written*.
  * **Takeaway 21.1 #4:** *If `F` synchronizes with `E`, all effects `X` that happened before `F` must be visible at all evaluations `G` that happen after `E`*.
  * **Takeaway 21.1 #5:** *We can only conclude that one evaluation happened before another if we have a sequenced chain of synchronizations that links them*.
  * **Takeaway 21.1 #6:** *If an evaluation `F` happened before `E`, all effects that are known to have happened before `F` are also known to have happened before `E`*.
* **Read-Modify-Write (RMW) Operations:** Operations like `atomic_exchange`, `atomic_compare_exchange_weak`/`strong`, `atomic_fetch_add`, and `atomic_flag_test_and_set` combine reading and writing into an indivisible step with both acquire and release semantics.


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


### 4. Sequential Consistency (Section 21.3)
**Sequential consistency** is the strongest and default memory consistency model in C.

* **Global Linearization:**
  * **Takeaway 21.3 #1:** *All atomic operations with sequential consistency occur in one global modification order, regardless of the atomic object they are applied to*.
  * **Takeaway 21.3 #2:** *All operators and functional interfaces on atomics that don’t specify otherwise have sequential consistency*.
* **Default Atomic Functions:** Standard functions like `atomic_store`, `atomic_load`, `atomic_exchange`, `atomic_fetch_add`, and `atomic_compare_exchange_strong` enforce sequential consistency unless explicit memory orders are passed.


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

