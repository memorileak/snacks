+++
title = "Modern C: Program failure"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Chapter 15 of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, titled **"Program failure,"** concludes **Level 2: Cognition**. Rather than focusing on what happens *after* an error occurs (often misleadingly labeled under the broad jargon "undefined behavior" or UB), Gustedt systematically analyzes **why** programs fail and **how** developers can prevent, detect, and handle failures gracefully.

Below is a detailed breakdown of the fundamental concepts, categories of program failure, design rules, and key takeaways from Chapter 15 across its six core sections:


### 1. Wrongdoings (Section 15.1)
Wrongdoings are specific actions, events, or omissions during execution that directly cause a program failure.

* **Arithmetic Violations (Section 15.1.1):**
  * Exceptional mathematical conditions with no defined result—such as **integer division by zero** or **modulo by zero**—cause immediate program failure or traps.
  * Finite bit-representation violations include negating `INT_MIN` or shifting bits by negative/out-of-range counts or shifting into a signed bit.
  * **Takeaway 15.1.1 #1:** *The program execution should only perform arithmetic operations that are mathematically defined within the range of the underlying type*.
  * **Takeaway 15.1.1 #2:** *The floating-point environment of the platforms determines the floating-point operations that result in program failure* (e.g., querying exceptions via `fetestexcept` and `<fenv.h>`).
* **Type & Function Prototype Violations (Section 15.1.4):**
  * Converting object or function pointers to incompatible types removes compiler safeguards and causes execution derailment.
  * **Takeaway 15.1.4 #1:** *Don’t convert pointers unless you must*.
  * **Takeaway 15.1.4 #2:** *Always call a function with the prototype with which it is defined*.
  * **Takeaway 15.1.4 #3:** *Call a function by its name* (avoiding unnecessary function pointer casts).
* **Access Violations (Section 15.1.5):**
  * Common direct pointer/memory wrongdoings include: null pointer dereferences, accessing freed/stale memory, unsequenced modifications of the same object, out-of-bounds array access, modifying `const`-qualified objects, `restrict`/`volatile` rule violations, and double `free()` calls.
* **Value Misinterpretation / Indeterminate Representations (Section 15.1.6):**
  * Bit patterns that have no valid interpretation for a given type are called **indeterminate representations** in C23 (formerly *trap representations*).
  * **Takeaway 15.1.6 #1:** *Don’t store values other than `0` or `1` in a `bool` object*.
  * **Takeaway 15.1.6 #2:** *Don’t change the representation bytes of objects directly*.
* **Explicit Invalidation with C23 `unreachable()` (Section 15.1.7):**
  * C23 introduces `unreachable()` to assert that a specific control path will never be taken, allowing aggressive compiler optimization.
  * **Takeaway 15.1.7 #1:** *Only use `unreachable()` where you have proof*.
  * **Takeaway 15.1.7 #2:** *Don’t use other operations than `unreachable()` to mark a control path that will never be taken* (e.g., do not use division by zero to mark dead code).


### 2. Program State Degradation (Section 15.2)
Unlike single wrongdoings, state degradation occurs when no single line of code is individually at fault, but execution resources slowly deteriorate until failure becomes inevitable.

* **Unbounded Recursion & Stack Overflow (Section 15.2.1 & 15.2.2):**
  * Lacking progress or an explicit termination condition in recursive calls depletes call-context memory on the **stack**, resulting in a stack overflow crash.
* **Heap Storage Exhaustion (Section 15.2.2):**
  * Exhausting runtime **heap** allocation capacity causes `malloc`, `calloc`, `realloc`, `strdup`, or `strndup` to return `nullptr`. Testing returns prevents catastrophic failure.
* **Other Scarce Execution Resources (Section 15.2.3):**
  * Runtime environments contain bounded scarce resources that must be released symmetrically:
    * **File Streams:** Limited by `FOPEN_MAX` (`fopen` / `fclose`).
    * **Temporary Files:** Limited by `TMP_MAX` (`tmpfile` / `remove`).
    * **Concurrency Primitives:** Thread contexts (`thrd_create` / `thrd_join`), mutexes (`mtx_init` / `mtx_destroy`), condition variables (`cnd_init` / `cnd_destroy`), and thread-specific storage (`tss_create` / `tss_delete`).


### 3. Unfortunate Incidents (Section 15.3)
Unfortunate incidents occur when individually valid operations fail due to unpredictable, distant interactions across different parts of the program.

* **Escalating State Degradation:** Ignoring resource exhaustion (e.g., continuing execution after a failed allocation) causes erratic, systemic crashes.
* **Collisions & Race Conditions (Section 15.3.2):**
  * **Takeaway 15.3.2 #1:** *Don’t read and modify the same object within the same arithmetic expression* (e.g., `x++ + x` evaluates with unsequenced side effects).
  * Concurrent modifications across signal handlers or multiple threads create unsequenced race conditions unless synchronized using atomic types or mutexes.
* **Inappropriate Context Calls:** Calling non-reentrant or non-thread-safe library functions (such as `signal` inside multithreaded code, or misplacing `setjmp`) jeopardizes runtime stability.
* **Deadlocks:** Concurrent threads locking multiple mutexes out of order create permanent execution blockages.


### 4. Series of Unfortunate Events & Livelocks (Section 15.4)
Nasty failures can occur when code runs endlessly over a finite set of states without making visible progress.

* **Infinite Loops Without Progress:**
  * **Takeaway 15.4 #1:** *A program execution that loops over a finite set of states with no observable side effects has failed*.
  * Compilers may assume loops without observable side effects (I/O, global state changes, or explicit exits) terminate or trigger `unreachable()`, optimizing them away unexpectedly.
* **Livelocks:** In multithreaded systems, threads repeatedly change their state in response to each other without accomplishing real work (resembling a cyclic detour trap).


### 5. Dealing with Failures (Section 15.5)
The most effective way to deal with failure is anticipatory prevention and checking error indicators.

* **Takeaway 15.5 #1:** *Ensure all preconditions for an operation that could fail* (defensive programming before executing risky code).
* **Takeaway 15.5 #2:** *The return of operations that might exhaust resources should be checked for errors* (verifying library returns and `errno`).
* **Takeaway 15.5 #3:** *Unfortunate events can only be avoided with a careful algorithm design*.
* **System Termination Mechanisms:**
  * Controlled user cleanup: `return` from `main` or `exit()`.
  * Alternate/fast cleanup: `quick_exit()` (invokes `at_quick_exit` handlers).
  * OS-level termination: `_Exit()` (bypasses application handlers).
  * Abnormal termination: `abort()` (raises `SIGABRT`, bypassing system cleanup).


### 6. Error Checking, Preserving `errno`, & `goto` Cleanup (Section 15.6)

Robust functions validate input preconditions, preserve external error states, and use centralized cleanup blocks.

* **Standard Error Macros (`<errno.h>`):**
  * Custom functions indicate failure by returning negative platform error codes like `-EFAULT` (invalid pointer), `-EOVERFLOW` (result too large), or `-ENOMEM` (out of memory).
* **Preserving `errno` State:**
  * Functions performing internal cleanup or status checks should save `errno` on entry and restore it via a helper (e.g., `error_cleanup`) so callers receive unchanged diagnostic context.
* **Centralized Resource Cleanup with `goto`:**
  * When a multi-step function fails halfway through (e.g., after allocating memory or opening a stream), jumping to a single `CLEANUP:` label ensures all allocated resources are freed without duplicating code or creating deeply nested `if-else` trees.
* **Rules for `goto` Labels:**
  * **Takeaway 15.6 #1:** *Labels for `goto` are visible in the entire function that contains them*.
  * **Takeaway 15.6 #2:** *`goto` can only jump to a label inside the same function*.
  * **Takeaway 15.6 #3:** *`goto` should not jump over variable initializations*.


### Summary of Failure Types & Mitigation Strategies

| Failure Category | Primary Cause | Typical Manifestation | Recommended Mitigation |
| :--- | :--- | :--- | :--- |
| **Wrongdoings** | Arithmetic errors, illegal casts, access violations | Crashes, traps, memory corruption | Validate preconditions; avoid pointer casts; use `unreachable()` only with proof. |
| **State Degradation** | Stack/heap/stream resource exhaustion | Allocation returns `nullptr`, stack overflow | Limit recursion depth; check `malloc`/`fopen` returns. |
| **Unfortunate Incidents** | Unsequenced side effects, race conditions, deadlocks | Erratic results, silent data corruption, hangs | Avoid unsequenced expressions; lock mutexes in fixed order; synchronize shared state. |
| **Series of Unfortunate Events** | Infinite loops without side effects, livelocks | Endless execution without progress | Ensure loops perform observable progress/I/O. |
