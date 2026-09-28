+++
title = "Modern C: Performance"
description = "A guide to safe C23 optimization, covering compiler assistance, inline linkage, restrict-qualified pointers, optimization attributes, and statistical performance measurement."
date = 2026-09-28T07:02:58Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 16: Performance** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book opens **Level 3: Experience**. While C is widely chosen for its high-performance capabilities, Gustedt stresses that performance optimizations must never compromise safety, correctness, or maintainability.

Below is a detailed breakdown of the fundamental concepts, compiler optimization mechanisms, design constraints, and statistical measurement rules covered in Chapter 16 across its four core sections:


### 1. Prime Directives of Optimization & General Rules (Section 16)
Before applying specific compiler keywords or attributes, developers must follow fundamental rules to prevent trading software reliability for marginal execution gains.

* **Premature Optimization:**
  * **Takeaway 16 #1:** ***Premature optimization is the root of all evil*** (quoting Donald Knuth).
* **Safety vs. Performance:**
  * **Takeaway 16 #2:** ***Do not trade safety for performance***. Sacrificing safety checks introduces severe bugs—such as out-of-bounds array access, uninitialized memory reads, stale pointer dereferences, or signed integer overflow—which lead to data loss, security vulnerabilities, or crashes.
* **Compiler Assistance & Aliasing:**
  * **Takeaway 16 #3:** ***Optimizers are clever enough to eliminate unused initializations***.
  * **Takeaway 16 #4:** ***The different notations of pointer arguments to functions result in the same binary code***. Parameter annotations like `double a[static 1]`, `double a[static N]`, or VLA parameter notation pass identical information to the optimizer without altering machine code output.
  * **Takeaway 16 #5:** ***Not taking addresses of local variables helps the optimizer because it inhibits aliasing***. When a local variable's address is never retrieved with `&`, the compiler guarantees it cannot alias with other pointers, enabling it to cache the value directly inside CPU registers.

#### Example

Before applying specific compiler keywords, optimization must respect software safety. Compiler optimizers are sophisticated enough to eliminate unused initializations, and avoid taking addresses of local variables (`&`) so the compiler can store them directly in CPU registers without aliasing concerns.

```c
#include <stdio.h>
#include <stddef.h>

// Takeaway 16 #4: Annotating pointer parameters using array/static/VLA syntax
// communicates accessibility guarantees to the compiler without altering binary code.

// 1. Single non-null object guarantee (static 1):
void process_single(double v[static 1]) { //
  *v *= 2.0;
}

// 2. Fixed collection with minimum element count guarantee (static N):
void process_fixed(double v[static 8]) { //
  for (size_t i = 0; i < 8; ++i) {
    v[i] += 1.0;
  }
}

// 3. Dynamic VLA parameter with guaranteed accessible bound:
void process_vla(size_t n, double v[static n]) { //
  for (size_t i = 0; i < n; ++i) {
    v[i] *= 0.5;
  }
}

void demo_general_rules(void) {
  // Takeaway 16 #3: Optimizers eliminate unused initializations automatically.
  double value = 0.0; // Safe initialization; compiler eliminates dead store if overwritten
  value = 42.0;

  // Takeaway 16 #5: Avoid taking the address (&) of local variables when unneeded.
  // By NOT calling &value, the compiler guarantees 'value' cannot be aliased by pointers,
  // allowing it to cache 'value' inside a hardware CPU register.
  process_single(&value);

  double dataset = {1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0};
  process_fixed(dataset);
  process_vla(8, dataset);
}
```

### 2. Inline Functions (Section 16.1)
Modular code relies on functions, but function calls incur runtime overhead (stack frame allocation, register saving, context jumps, and struct copying). Inlining replaces the function call with its body directly at the call site.

* **Optimization Opportunities:**
  * **Takeaway 16.1 #1:** ***Inlining can open up a lot of optimization opportunities***. Exposing the function body to the caller's context allows the compiler to perform constant folding, dead-branch elimination, and register-level optimizations across function boundaries.
* **C99/C23 `inline` Rules & Linkage:**
  * **Takeaway 16.1 #3 & #4:** ***An `inline` function definition goes in a header file***, and ***is visible in all translation units (TUs)***.
  * **Takeaway 16.1 #2 & #5:** ***An additional compatible declaration without `inline` goes in exactly one TU***. This guarantees that a non-inlined external function symbol is emitted in that single translation unit if the compiler chooses not to inline a call.
  * **Takeaway 16.1 #6:** ***Only expose functions as `inline` if you consider them stable***. Modifying an inline header definition forces a full recompilation of all dependent source files.
  * **Takeaway 16.1 #7:** ***All identifiers local to an `inline` function should be protected by a convenient naming convention*** (such as a unique project prefix) to avoid accidental expansion by user macros.
* **Restrictions on `inline` Functions:**
  * **Takeaway 16.1 #8:** ***`inline` functions can’t access `static` functions by name***.
  * **Takeaway 16.1 #9:** ***`inline` functions can’t access modifiable `static` objects by name***.
  * **Takeaway 16.1 #A:** ***`inline` functions can’t define modifiable `static` objects***. (Because an `inline` definition is included across multiple TUs, binding static names would create ambiguous static instances).

#### Example

Inlining replaces function calls directly at the call site, eliminating call overhead and unlocking optimizations like dead-branch elimination and constant folding. C99/C23 specifies exact header and translation unit (TU) linkage rules for `inline` functions.

**Header File (`myproj_math.h`)**

```c
#ifndef MYPROJ_MATH_H
#define MYPROJ_MATH_H

#include <stddef.h>

// Takeaway 16.1 #3 & #4: An inline definition goes in a header file and is visible to all TUs.
// Takeaway 16.1 #7: Protect all local parameters and variables with a project prefix (myproj_)
// to avoid accidental expansion by user preprocessor macros.
inline size_t myproj_square(size_t myproj_x) {
    // Takeaway 16.1 #8, #9, & #A: Inline functions CANNOT access static functions or modifiable
    // static objects by name, nor can they define modifiable static objects.
    return myproj_x * myproj_x;
}

#endif // MYPROJ_MATH_H
```

**Translation Unit (`myproj_math.c`)**

```c
#include "myproj_math.h"

// Takeaway 16.1 #2 & #5: Adding a compatible declaration WITHOUT the 'inline' keyword
// in exactly ONE translation unit forces the compiler to emit a non-inlined external
// function symbol (e.g. if the caller takes a pointer to the function or inlining is disabled).
size_t myproj_square(size_t); // Emits global symbol for myproj_square
```

### 3. Exclusive Pointer Access via `restrict` (Section 16.2)
The `restrict` type qualifier informs the compiler that a pointer is the **sole initial reference** to its target object within its scope.

* **Exclusive Access:**
  * **Takeaway 16.2 #1:** ***A `restrict`-qualified pointer has to provide exclusive access***. It promises that no other pointer within the same execution block will be used to access or modify the underlying memory region.
* **Caller Constraints:**
  * **Takeaway 16.2 #2:** ***A `restrict`-qualification constrains the caller of a function***. The burden of proof rests on the caller to guarantee that memory buffers passed to `restrict` parameters (e.g., `memcpy`) do not overlap.

#### Example

The `restrict` qualifier asserts that a pointer provides exclusive access to its target memory block during the execution of its scope. This informs the optimizer that no other pointer will read or modify the same underlying memory, placing the burden of non-overlapping buffer guarantees on the caller.

```c
#include <stdio.h>
#include <stddef.h>

// Takeaway 16.2 #1 & #2: 'restrict' promises that 'out' and 'in' point to non-overlapping
// memory regions, allowing vectorization and register caching.
void vector_add(size_t len,
                double* restrict out,
                double const* restrict in1,
                double const* restrict in2) {
    for (size_t i = 0; i < len; ++i) {
        // Because of 'restrict', the compiler knows writes to out[i]
        // can never modify in1[j] or in2[j].
        out[i] = in1[i] + in2[i];
    }
}

void demo_restrict(void) {
    double a = {1.0, 2.0, 3.0, 4.0};
    double b = {10.0, 20.0, 30.0, 40.0};
    double result = {};

    // Caller guarantee: 'result', 'a', and 'b' refer to distinct, non-overlapping memory blocks.
    vector_add(4, result, a, b);
}
```

### 4. C23 Optimization Attributes (`[[unsequenced]]` & `[[reproducible]]`) (Section 16.3)
C23 standardizes function attributes that explicitly inform the compiler about a function's purity, side effects, and state dependencies.

* **`[[unsequenced]]` (Pure Functions):**
  * **Takeaway 16.3 #1:** ***All pure functions should have the attribute `[[unsequenced]]`***.
  * **Takeaway 16.3 #2 & #4:** ***A function with the attribute `[[unsequenced]]` shall not read nonconstant global variables or system state***, and ***shall not apply visible modifications to global variables or system states***.
  * **Takeaway 16.3 #3 & #5:** ***Functions using floating-point arithmetic or returning errors through `errno` are generally not pure and shall not have `[[unsequenced]]`*** (because floating-point control modes, rounding flags, and `errno` are mutable global/thread states).
  * **Takeaway 16.3 #A:** ***A function with `[[unsequenced]]` shall not modify local `static` state, even through other function calls***.
* **Local Pragmas & FP State Guarantees:**
  * **Takeaway 16.3 #6 & #7:** ***Pragmas changing floating-point state act locally within current scope***, and ***type attributes accumulate within current scope***. Scoping pragmas like `#pragma FP_CONTRACT OFF` and `#pragma FENV_ROUND FE_TONEAREST` locally isolates FP state, allowing functions like `sqrt` to be safely attributed as `[[unsequenced]]` in local scopes.
* **`[[reproducible]]` (Weaker Purity):**
  * **Takeaway 16.3 #8 & #B:** ***A function with `[[reproducible]]` may temporarily modify global state as long as it restores it***, and ***may modify local `static` state if that state is not observable from outside***.
  * **Takeaway 16.3 #9:** ***For functions with `[[unsequenced]]` and `[[reproducible]]` attributes, annotate pointer parameters with `restrict`***.

#### Example

C23 introduces standard attributes to annotate function purity and state dependency.

* **`[[unsequenced]]`**: Annotates pure functions that do not read or modify global/system state (including `errno`) and depend strictly on input value parameters.
* **`[[reproducible]]`**: Weaker purity attribute allowing temporary modification of global state (if restored before returning) or unobservable cached/memoized state.

```c
#include <stdio.h>
#include <math.h>
#include <fenv.h>

// Takeaway 16.3 #1: Pure value functions should be annotated with [[unsequenced]].
[[unsequenced]] inline double cube_value(double x) { //
  return x * x * x;
}

// Takeaway 16.3 #3, #5, #6, & #7: Floating-point arithmetic and errno checks generally prevent
// global unsequenced attribution. However, local pragmas (#pragma FP_CONTRACT / FENV_ROUND)
// and local scope attributes allow functions to be locally optimized.
[[reproducible]] inline double compute_distance(double const x[restrict static 2]) { //
  #pragma FP_CONTRACT OFF
  #pragma FENV_ROUND FE_TONEAREST //

  // Locally assert that sqrt will not encounter negative numbers or set errno
  extern double sqrt(double) [[unsequenced]]; // Local attribute scoping
  return sqrt(x * x + x * x); //
}

// Takeaway 16.3 #8 & #B: [[reproducible]] allows unobservable internal static state (such as a lookup table).
[[reproducible]] size_t cached_lookup(size_t key) { //
  static size_t cache = {}; // Internal static cache not observable from caller
  if (cache[key % 256] != 0) {
    return cache[key % 256];
  }
  size_t computed = key * 31U + 7U;
  cache[key % 256] = computed;
  return cache[key % 256];
}
```

### 5. Rigorous Performance Measurement & Statistics (Section 16.4)
Human intuition is notoriously unreliable at predicting code performance under modern compiler optimizations and CPU hardware architectures.

* **Verification over Speculation:**
  * **Takeaway 16.4 #1:** ***Don’t speculate about the performance of code; verify it rigorously***.
  * **Takeaway 16.4 #2 & #3:** ***Complexity assessment of algorithms requires proofs***, whereas ***performance assessment of code requires measurement***.
* **Measurement Bias & Instrumentation:**
  * **Takeaway 16.4 #4 & #5:** ***All measurements introduce bias***, and ***instrumentation changes compile-time and run-time properties***. Inserting timing calls (such as `timespec_get`) forces compilers to save registers, emit system calls, and invalidate cache assumptions.
* **Statistical Hardening:**
  * **Takeaway 16.4 #6:** ***The relative standard deviation (\\(\sigma / \mu\\)) of run times must be in a low percentage range*** to be considered statistically valid.
  * **Takeaway 16.4 #7:** ***Collecting higher-order moments of measurements to compute variance and skew is simple and cheap*** (using incremental statistical algorithms like Welford's or Pébay's).
  * **Takeaway 16.4 #8:** ***Run-time measurements must be hardened with statistics***.

#### Example

Performance assessment requires actual runtime measurement rather than speculation. However, inserting instrumentation (like `timespec_get`) introduces bias and alters compiler register allocations. Measurements must be hardened using statistical moments (mean and low relative standard deviation).

```c
#include <stdio.h>
#include <time.h>
#include <stdint.h>

// Structure to accumulate raw statistical moments (0th to 2nd moment)
typedef struct {
  double count;
  double mean;
  double M2; // Running sum of squared differences (Welford's algorithm)
} benchmark_stats;

// Takeaway 16.4 #7: Collecting statistical moments to compute variance is simple and cheap.
[[reproducible]] inline void stats_accumulate(benchmark_stats* s, double val) { //
  s->count += 1.0;
  double delta = val - s->mean;
  s->mean += delta / s->count;
  double delta2 = val - s->mean;
  s->M2 += delta * delta2;
}

void demo_measurement_statistics(void) {
  struct timespec start = {}, end = {};
  benchmark_stats timing_stats = {};
  uint64_t iterations = 10000;

  // Takeaway 16.4 #4 & #5: Time measurements introduce bias and alter compile-time properties.
  // Repeated execution inside loops minimizes single-call bias.
  for (size_t run = 0; run < 10; ++run) {
    timespec_get(&start, TIME_UTC); //

    // Volatile store ensures loop body is executed without being optimized away
    for (uint64_t volatile i = 0; i < iterations; ++i) { //
      /* Code under test */
    }

    timespec_get(&end, TIME_UTC); //

    double elapsed_sec = (double)(end.tv_sec - start.tv_sec) +
              (double)(end.tv_nsec - start.tv_nsec) * 1e-9;

    stats_accumulate(&timing_stats, elapsed_sec / (double)iterations);
  }

  // Takeaway 16.4 #6 & #8: Run-time measurements must be hardened with statistics.
  double variance = (timing_stats.count > 1.0) ? (timing_stats.M2 / (timing_stats.count - 1.0)) : 0.0;
  printf("Mean time per iteration: %5.2e sec (Samples: %.0f, Variance: %5.2e)\n",
       timing_stats.mean, timing_stats.count, variance);
}
```
