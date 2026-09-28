+++
title = "Modern C: Performance"
description = ""
date = 2026-09-28

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


### 3. Exclusive Pointer Access via `restrict` (Section 16.2)
The `restrict` type qualifier informs the compiler that a pointer is the **sole initial reference** to its target object within its scope.

* **Exclusive Access:**
  * **Takeaway 16.2 #1:** ***A `restrict`-qualified pointer has to provide exclusive access***. It promises that no other pointer within the same execution block will be used to access or modify the underlying memory region.
* **Caller Constraints:**
  * **Takeaway 16.2 #2:** ***A `restrict`-qualification constrains the caller of a function***. The burden of proof rests on the caller to guarantee that memory buffers passed to `restrict` parameters (e.g., `memcpy`) do not overlap.


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

## Examples