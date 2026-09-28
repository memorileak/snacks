+++
title = "Modern C: Program failure"
description = "A guide to C23 program failures, from invalid operations and resource exhaustion to race conditions and livelocks, with strategies for prevention, error handling, and cleanup."
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

#### Example

Operations whose mathematical results are undefined (such as integer division or modulo by zero) or that exceed the finite bit-width representation of signed types (such as negating `INT_MIN` or shifting into the sign bit) lead to immediate program failure or hardware traps. Floating-point exceptions can be queried using `<fenv.h>` interfaces like `fetestexcept` and `feclearexcept`.

```c
#include <stdio.h>
#include <limits.h>
#include <fenv.h> // Floating-point environment testing

void demo_arithmetic_violations(void) {
  // 1. INTEGER ARITHMETIC VIOLATIONS (Takeaway 15.1.1 #1):
  // Perform only arithmetic operations that are mathematically defined within type ranges.
  int a = 10;
  int b = 0;
  if (b != 0) {
    int res = a / b; // Safe guard against integer division by zero
    (void)res;
  }

  // Negating INT_MIN overflows signed integer representation (-INT_MIN > INT_MAX)
  int min_val = INT_MIN;
  if (min_val != INT_MIN) {
    int neg = -min_val; // Avoid negating INT_MIN
    (void)neg;
  }

  // 2. FLOATING-POINT ENVIRONMENT EXCEPTION CHECKING (Takeaway 15.1.1 #2):
  // Floating-point division by zero yields INFINITY without crashing on IEC 60559 platforms.
  feclearexcept(FE_ALL_EXCEPT); // Clear current floating-point flags
  double x = 0.0;
  double div_result = 1.0 / x;

  if (fetestexcept(FE_DIVBYZERO)) {
    printf("Captured floating-point exception: FE_DIVBYZERO (Result: %g)\n", div_result);
    feclearexcept(FE_DIVBYZERO); // Reset exception state
  }
}
```

#### Example

Converting pointers across incompatible base types strips away compiler checks. Functions must always be called using their exact definition prototype and name. Access violations include null pointer dereferences, accessing freed/stale memory, modifying `const` objects, and unsequenced updates.

```c
#include <stdio.h>
#include <stdlib.h>

static void print_double(double val) {
  printf("Value: %g\n", val);
}

void demo_type_and_access_violations(void) {
  // Takeaway 15.1.4 #1: Don't convert pointers unless you must.
  // Takeaway 15.1.4 #2: Always call a function with the prototype with which it is defined.
  // Takeaway 15.1.4 #3: Call a function by its name rather than casting function pointers.
  void (*fn_ptr)(double) = print_double; // Exact prototype match
  fn_ptr(42.0);

  // ACCESS VIOLATION PREVENTION (Section 15.1.5):
  double* ptr = malloc(sizeof *ptr);
  if (ptr) {
    *ptr = 3.14159;
    free(ptr); // Release dynamic memory
    ptr = nullptr; // Takeaway 11.1.4 #2: Reset stale pointers to nullptr immediately!
  }
}
```

#### Example

An **indeterminate representation** occurs when storage contains a bit pattern with no valid interpretation for its type. Modifying representation bytes of scalar types directly (like writing non-`0`/`1` values into a `bool` variable) leads to misinterpretation.

```c
#include <stdio.h>
#include <stdbool.h>
#include <string.h>

void demo_value_misinterpretation(void) {
  // Takeaway 15.1.6 #1: Don't store values other than 0 or 1 in a bool object.
  // Takeaway 15.1.6 #2: Don't change the representation bytes of objects directly.
  bool flag = true;

  // BAD PRACTICE (Avoid!): Overwriting bool representation bytes directly via unsigned char*
  // unsigned char bad_byte = 0xFE;
  // memcpy(&flag, &bad_byte, 1); // Corrupts boolean truth-testing logic!

  if (flag) { // Clean boolean check
    puts("Bool state is valid (0 or 1).");
  }
}
```

#### Example

C23 introduces the **`unreachable()`** macro to assert that a control path will never be executed, enabling aggressive compiler optimization.

```c
#include <stdio.h>
#include <stddef.h>
#include <utility.h> // Provides C23 unreachable()

static ptrdiff_t safe_ptr_diff(unsigned char const p[static 1], unsigned char const q[static 1]) {
  // Takeaway 15.1.7 #1: Only use unreachable() where you have mathematical or logical proof.
  if (!p || !q) {
    // Takeaway 15.1.7 #2: Don't use other operations (like division by zero) to mark dead code.
    unreachable(); // Informs compiler this branch is logically impossible
  }
  return p - q;
}

void demo_unreachable(void) {
  unsigned char buf1 = {};
  unsigned char buf2 = {};
  ptrdiff_t diff = safe_ptr_diff(buf1, buf2);
  printf("Pointer offset: %td\n", diff);
}
```

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

#### Example

State degradation occurs when no single line of code is at fault, but runtime execution resources slowly deteriorate until system failure becomes inevitable.

* **Stack Overflow:** Caused by unbounded recursion lacking progress or explicit termination checks.
* **Heap Storage Exhaustion:** Occurs when `malloc`, `calloc`, `realloc`, `strdup`, or `strndup` return `nullptr` due to memory exhaustion.
* **Scarce Execution Resources:** Resources like file streams (`FOPEN_MAX`), temporary files (`TMP_MAX`), thread contexts, and mutexes must be released symmetrically.

```c
#include <stdio.h>
#include <stdlib.h>

void demo_state_degradation(void) {
  // Checking Heap Resource Exhaustion (Takeaway 15.5 #2)
  size_t huge_size = (size_t)-1 / 2; // Intentionally excessive request
  double* memory = malloc(huge_size);

  if (!memory) {
    // Graceful handling of heap exhaustion without crashing
    fputs("State Degradation Guard: malloc returned nullptr (Out of Memory).\n", stderr);
  } else {
    free(memory);
  }
}
```

### 3. Unfortunate Incidents (Section 15.3)
Unfortunate incidents occur when individually valid operations fail due to unpredictable, distant interactions across different parts of the program.

* **Escalating State Degradation:** Ignoring resource exhaustion (e.g., continuing execution after a failed allocation) causes erratic, systemic crashes.
* **Collisions & Race Conditions (Section 15.3.2):**
  * **Takeaway 15.3.2 #1:** *Don’t read and modify the same object within the same arithmetic expression* (e.g., `x++ + x` evaluates with unsequenced side effects).
  * Concurrent modifications across signal handlers or multiple threads create unsequenced race conditions unless synchronized using atomic types or mutexes.
* **Inappropriate Context Calls:** Calling non-reentrant or non-thread-safe library functions (such as `signal` inside multithreaded code, or misplacing `setjmp`) jeopardizes runtime stability.
* **Deadlocks:** Concurrent threads locking multiple mutexes out of order create permanent execution blockages.

#### Example

Unfortunate incidents occur when individually valid operations fail due to unpredictable, unsequenced, or distant interactions across the program.

```c
#include <stdio.h>

void demo_unfortunate_incidents(void) {
  int x = 5;

  // Takeaway 15.3.2 #1: Don't read and modify the same object within the same arithmetic expression.
  // BAD PRACTICE (Avoid!): printf("%d\n", x++ + x); // Unsequenced modification and read!

  // GOOD PRACTICE: Separate side effects into distinct, sequenced statements
  int val1 = x;
  x += 1;
  int sum = val1 + x;
  printf("Sequenced calculation result: %d (x = %d)\n", sum, x);
}
```

### 4. Series of Unfortunate Events & Livelocks (Section 15.4)
Nasty failures can occur when code runs endlessly over a finite set of states without making visible progress.

* **Infinite Loops Without Progress:**
  * **Takeaway 15.4 #1:** *A program execution that loops over a finite set of states with no observable side effects has failed*.
  * Compilers may assume loops without observable side effects (I/O, global state changes, or explicit exits) terminate or trigger `unreachable()`, optimizing them away unexpectedly.
* **Livelocks:** In multithreaded systems, threads repeatedly change their state in response to each other without accomplishing real work (resembling a cyclic detour trap).

#### Example

A program that enters an endless loop over a finite set of states with no observable side effects (such as I/O, global state changes, or explicit exits) has failed.

```c
#include <stdio.h>
#include <stdbool.h>

void demo_series_of_unfortunate_events(void) {
  size_t iterations = 0;

  // Takeaway 15.4 #1: A loop over a finite set of states MUST produce observable progress.
  for (size_t i = 0; i < 5; ++i) {
    printf("Observable progress step: %zu\n", i); // Output side effect ensures progress
    ++iterations;
  }
}
```

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

#### Example

To prevent failures, programs must enforce **precondition checks** before executing risky operations and inspect return values of functions that manage resources.

```c
#include <stdio.h>
#include <stdlib.h>

// Takeaway 15.5 #1: Ensure all preconditions for an operation that could fail.
static double safe_divide(double numerator, double denominator, bool* success_flag) {
  if (denominator == 0.0) {
    if (success_flag) *success_flag = false;
    return 0.0; // Return fallback value on precondition failure
  }
  if (success_flag) *success_flag = true;
  return numerator / denominator;
}

void demo_dealing_with_failures(void) {
  bool ok = false;
  double result = safe_divide(10.0, 0.0, &ok);
  if (!ok) {
    fputs("Precondition check failed: Division by zero avoided.\n", stderr);
  } else {
    printf("Result: %g\n", result);
  }

  // SYSTEM TERMINATION MECHANISMS (Section 8.8 & 15.5):
  // 1. return from main / exit(): Runs normal library & atexit cleanups.
  // 2. quick_exit(): Runs at_quick_exit handlers, bypasses standard exit handlers.
  // 3. _Exit(): Immediate OS termination bypassing application handlers.
  // 4. abort(): Abnormal termination raising SIGABRT.
}
```

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

#### Example

When writing robust library functions that perform multiple resource allocations or system calls, failure handling often dominates the code.

* **Preserving `errno`:** Functions performing internal cleanup should save `errno` upon entry and restore it via an `error_cleanup` helper.
* **Centralized `goto CLEANUP`:** Jumping to a single cleanup block prevents code duplication and avoids deeply nested `if-else` trees.
* **Rules for `goto` Labels:**
  * **Takeaway 15.6 #1:** *Labels for `goto` are visible in the entire function containing them*.
  * **Takeaway 15.6 #2:** *`goto` can only jump to a label inside the same function*.
  * **Takeaway 15.6 #3:** *`goto` should not jump over variable initializations*.

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>

// Macro fallbacks for platform error codes from <errno.h>
#ifndef EFAULT
# define EFAULT EDOM
#endif
#ifndef ENOMEM
# define ENOMEM ERANGE
#endif

// Helper function to restore original errno state while returning a negative error code
static inline int error_cleanup(int err, int prev_errno) {
  errno = prev_errno; // Preserves external errno state
  return -err;        // Returns negative error status
}

// Function allocating resources and demonstrating centralized goto CLEANUP
int process_data_buffer(size_t len, double const input[restrict len]) {
  // 1. PRECONDITION CHECKS:
  if (!input || len == 0) {
    return -EFAULT; // Return negative error code for invalid parameters
  }

  int saved_errno = errno;
  double* temp_buf = nullptr;
  double* work_buf = nullptr;

  // 2. FIRST RESOURCE ALLOCATION:
  temp_buf = malloc(len * sizeof *temp_buf);
  if (!temp_buf) {
    return error_cleanup(ENOMEM, saved_errno); // Clean failure without changing errno
  }

  // 3. SECOND RESOURCE ALLOCATION:
  work_buf = malloc(len * sizeof *work_buf);
  if (!work_buf) {
    // Jump to central cleanup label to free temp_buf without repeating code
    goto CLEANUP;
  }

  // Perform actual work on buffers...
  for (size_t i = 0; i < len; ++i) {
    work_buf[i] = input[i] * 2.0;
  }

  printf("Buffer successfully processed %zu elements.\n", len);

  // CENTRAL CLEANUP BLOCK (Takeaways 15.6 #1, #2, #3):
  CLEANUP:
    free(work_buf); // Safe to call free() on allocated memory or nullptr
    free(temp_buf);

    if (!work_buf && temp_buf == nullptr) {
      return error_cleanup(ENOMEM, saved_errno);
    }
    return 0; // Success return code
}

void demo_error_checking_and_cleanup(void) {
  double data = {1.0, 2.0, 3.0};
  int status = process_data_buffer(3, data);
  printf("Function completed with status code: %d\n", status);
}
```

### Summary of Failure Types & Mitigation Strategies

| Failure Category | Primary Cause | Typical Manifestation | Recommended Mitigation |
| :--- | :--- | :--- | :--- |
| **Wrongdoings** | Arithmetic errors, illegal casts, access violations | Crashes, traps, memory corruption | Validate preconditions; avoid pointer casts; use `unreachable()` only with proof. |
| **State Degradation** | Stack/heap/stream resource exhaustion | Allocation returns `nullptr`, stack overflow | Limit recursion depth; check `malloc`/`fopen` returns. |
| **Unfortunate Incidents** | Unsequenced side effects, race conditions, deadlocks | Erratic results, silent data corruption, hangs | Avoid unsequenced expressions; lock mutexes in fixed order; synchronize shared state. |
| **Series of Unfortunate Events** | Infinite loops without side effects, livelocks | Endless execution without progress | Ensure loops perform observable progress/I/O. |
