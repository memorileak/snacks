+++
title = "Modern C: Performance"
description = "A guide to safe C23 optimization, covering compiler assistance, inline linkage, restrict-qualified pointers, optimization attributes, and statistical performance measurement."
date = 2026-09-28T07:02:58Z

[taxonomies]
tags = ["modernc", "performance"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## Chapter 16. Performance

### The Philosophy of C Optimization

**What it is:** In C, optimizing code for performance often relies on the programmer communicating specific properties of the code to the compiler. This is achieved through language features like `register`, `inline`, `restrict`, `alignas`, `[[unsequenced]]`, and `[[reproducible]]`.

**Why it matters / how it works:** These features are unique because they constrain the _programmer_, not the compiler. By applying these constraints (e.g., promising that a pointer has exclusive access to memory), the compiler can safely assume certain properties and generate faster, more aggressive machine code. The defining characteristic of these features is that removing them from a valid program will _not_ change its semantics; they purely serve as optimization hints.

**Key details:**

- `register` (C89): Helps avoid aliasing among local variables.
- `inline` (C99), `restrict` (C99): Optimize function calls and pointer aliasing.
- `alignas` (C11): Improves cache line memory access.
- `[[unsequenced]]`, `[[reproducible]]` (C23): Enable aggressive optimization for pure functions.

**Recap:**

- Optimization keywords in C constrain the programmer to guarantee certain behaviors.
- These features do not alter the semantic correctness of the code, only its performance.

---

## 16.1. Inline functions

### Modular Functions and The Overhead Problem

**What it is:** C programs use functions to modularize code, separating interface from implementation and encapsulating local state. However, small modular functions introduce execution overhead (e.g., passing arguments, saving registers, jumping execution).

**Why it matters / how it works:** While small functions improve readability, the compiler might struggle to optimize them if they are defined in separate Translation Units (TUs). The `inline` keyword acts as a hint that a function is a good candidate for **inlining**—replacing the function call directly with the function's body at the call site.

> Takeaway 16.1 #1 _Inlining can open up a lot of optimization opportunities._

When a function is inlined, the compiler can perform inter-procedural optimizations, treating the function's code as if it were written directly inside the caller.

### The Mechanics of the `inline` Keyword

**What it is:** The `inline` keyword allows the same function _definition_ to be present in multiple TUs (usually via a header file) without violating the One Definition Rule (ODR) or causing multiple-symbol-definition errors during linking.

**Why it matters / how it works:** `inline` does _not_ force the compiler to inline the function; it only makes it legally possible by making the definition visible everywhere.

> Takeaway 16.1 #2 _Adding a compatible declaration without the inline keyword ensures the emission of the function symbol in the current TU._

> Takeaway 16.1 #3 _An inline function definition is visible in all TUs._

> Takeaway 16.1 #4 _An inline definition goes in a header file._

> Takeaway 16.1 #5 _An additional declaration without inline goes in exactly one TU._

**Key details & Rules:**

- If a function is marked `inline` in a header file, the compiler might inline it. But if it decides _not_ to inline it, it needs a real symbol to call.
- To provide this real symbol, you must declare the function _without_ the `inline` keyword in exactly _one_ `.c` file.
- Pointers to the same `inline` function will always compare equal, even if the object code resides in different places, because the linker unifies the address to the emitted symbol.

### Code Demonstration: `inline` Mechanics

```c
// --- toto.h ---
#ifndef TOTO_H
#define TOTO_H

typedef struct { int value; } toto;

// Takeaway 16.1 #4: Inline definition goes in a header.
// Visible to all TUs that include toto.h (Takeaway 16.1 #3)
inline toto* toto_init(toto* toto_x) {
    if (toto_x) {
        *toto_x = (toto){ 0 }; // Initialize to 0
    }
    return toto_x;
}
#endif

// --- toto.c ---
#include "toto.h"

// Takeaway 16.1 #2 and #5: Emitting the symbol.
// Instantiate in exactly one TU by providing a declaration without 'inline'.
toto* toto_init(toto*);

// --- main.c ---
#include "toto.h"
#include <stdio.h>

int main(void) {
    toto t;
    toto_init(&t); // Compiler may inline this directly.
    printf("Value: %d\n", t.value);
    return 0;
}

```

_Expected Behavior:_ The code compiles and prints `Value: 0`. If `toto_init` is called via a function pointer, the linker resolves it to the symbol emitted in `toto.c`.

_What goes wrong:_ If you omit the declaration in `toto.c` and the compiler decides _not_ to inline the call in `main.c`, the linker will fail with an "undefined reference" error.

### Pitfalls and Constraints of `inline`

**What it is:** Because `inline` functions exist in a weird state (defined in headers, visible across TUs), they face strict rules regarding their internal variables and identifiers.

> Takeaway 16.1 #6 _Only expose functions as inline if you consider them stable._

> Takeaway 16.1 #7 _All identifiers local to an inline function should be protected by a convenient naming convention._

> Takeaway 16.1 #8 _inline functions can’t access static functions by name._

> Takeaway 16.1 #9 _inline functions can’t access modifiable static objects by name._

> Takeaway 16.1 #A _inline functions can’t define modifiable static objects._

**Why it matters:**

- **Stability:** Once in a header, changing an `inline` function requires recompiling every TU that includes it.
- **Static Access:** An `inline` function doesn't belong to a single TU. If it were allowed to access a `static` variable by name, which `static` variable would it mean? The one in `foo.c` or `bar.c`? To prevent this ambiguity, the standard forbids it. (However, passing a pointer to a `static` object into the function is perfectly fine).

**Recap:**

- `inline` provides definitions in headers to allow compiler optimizations.
- Must be instantiated in exactly one TU to guarantee a linker symbol.
- Cannot interact with `static` (TU-local) functions or modifiable `static` objects by name.

---

## 16.2. Using restrict qualifiers

### Exclusive Pointer Access

**What it is:** `restrict` is a type qualifier for pointers (introduced in C99). It asserts to the compiler that for the lifetime of the pointer, the object it points to is accessed _exclusively_ through that pointer (or pointers derived from it).

**Why it matters / how it works:** One of the biggest roadblocks to compiler optimization is **aliasing** (two pointers pointing to the same memory). If a compiler sees a write to pointer A and a read from pointer B, it must assume they might point to the same location, forcing it to reload data from memory instead of keeping it in registers. `restrict` breaks this assumption, granting the compiler freedom to optimize.

> Takeaway 16.2 #1 _A restrict-qualified pointer has to provide exclusive access._

> Takeaway 16.2 #2 _A restrict-qualification constrains the caller of a function._

**Key details:**

- It places the burden of correctness entirely on the programmer (the caller).
- Standard library functions frequently use it (e.g., `memcpy` vs. `memmove`). `memcpy` uses `restrict` and expects no overlap. `memmove` omits `restrict` and safely handles overlap (at the cost of performance).
- Functions like `printf` use `restrict` on their format strings and stream pointers to guarantee they don't alias other arguments.

### Code Demonstration: `restrict` Aliasing

```c
#include <stdio.h>
#include <string.h>

// Both pointers are restricted. The caller promises arrays don't overlap.
void add_arrays(size_t n, int restrict out[n], const int restrict in[n]) {
    for (size_t i = 0; i < n; ++i) {
        // Compiler can optimize this loop heavily (e.g., using SIMD instructions)
        // because it knows writing to 'out' won't secretly change 'in'.
        out[i] += in[i];
    }
}

int main(void) {
    int A[5] = {1, 2, 3, 4, 5};
    int B[5] = {10, 20, 30, 40, 50};

    // Valid use: A and B do not overlap.
    add_arrays(5, A, B);

    // INVALID USE (Restrict Violation):
    // add_arrays(4, A+1, A); // The ranges overlap! Undefined behavior.

    // Standard Library violation example (conceptual):
    char const* format = "First two bytes in stdin object: %.2s\n";
    // fprintf(stdin, format, (char*)stdin); // Restrict violation! Stream and arg alias.

    return 0;
}

```

_Expected Behavior:_ Arrays are added efficiently.

_What goes wrong:_ Passing overlapping arrays to a `restrict` qualified function leads to Undefined Behavior. The compiler might generate SIMD instructions that fetch and write in chunks, corrupting the data if they overlap.

**Recap:**

- `restrict` promises the compiler that a pointer is the sole access path to memory.
- It enables aggressive loop and memory optimizations.
- Violating this promise (aliasing `restrict` pointers) causes undefined behavior.

---

## 16.3. Unsequenced and reproducible attributes

### Pure Functions (`[[unsequenced]]`)

**What it is:** C23 introduced the standard attributes `[[unsequenced]]` and `[[reproducible]]`. `[[unsequenced]]` is used to denote **pure functions**—functions whose return value depends _only_ on their parameters and have absolutely no side effects.

**Why it matters / how it works:** If a function is pure, the compiler can cache its result, reorder its execution, or eliminate the call entirely if the result isn't used.

> Takeaway 16.3 #1 _All pure functions should have the attribute [[unsequenced]]._

> Takeaway 16.3 #2 _A function with the attribute [[unsequenced]] shall not read nonconstant global variables or system state._

> Takeaway 16.3 #4 _A function with the attribute [[unsequenced]] shall not apply visible modifications to global variables or system states._

> Takeaway 16.3 #A _A function with the attribute [[unsequenced]] shall not modify the local static state, even through other function calls._

**Key details & Pitfalls:**

- You cannot read `errno`. (Takeaway 16.3 #5: _A function that returns possible errors through errno is not pure and shall not have the attribute [[unsequenced]]._)
- Floating-point math interacts with the global FPU state (rounding modes, exceptions). Therefore, general floating-point math ruins purity. (Takeaway 16.3 #3: _In general, a function that uses floating point arithmetic is not pure and shall not have the attribute [[unsequenced]]._)
- _Pragmas:_ You can locally alter FPU state. (Takeaway 16.3 #6: _Pragmas that change the floating point state act locally within the current scope._ / Takeaway 16.3 #7: _Type attributes accumulate within the current scope._)

### Caching and State (`[[reproducible]]`)

**What it is:** `[[reproducible]]` is a slightly weaker attribute than `[[unsequenced]]`. It means the function always produces the same output for the same input and has no _observable_ side effects from the outside, but it _might_ have internal side effects (like modifying a static cache/hash table).

> Takeaway 16.3 #8 _A function with the attribute [[reproducible]] may temporarily modify the global state as long as it restores it to its original value._

> Takeaway 16.3 #B _A function with the attribute [[reproducible]] shall only modify the local static state if that state is not observable from outside the function._

**Interaction with Pointers:**
Functions with pointer parameters can still be pure/reproducible if they only read/write to the memory strictly provided via the pointers, acting like a mapping operation.

> Takeaway 16.3 #9 _For functions with [[unsequenced]] and [[reproducible]] attribute annotate pointer parameters with restrict._

### Code Demonstration: Attributes

```c
#include <stddef.h>

// Pure function: Output only depends on input, no global state touched.
// Pointer is marked restrict as required by Takeaway 16.3 #9.
int compute_hash(char const restrict str[static 1]) [[unsequenced]] {
    int hash = 5381;
    while (*str) {
        hash = ((hash << 5) + hash) + *str++;
    }
    return hash;
}

// Reproducible function: Has hidden internal static state (caching),
// but from the caller's perspective, it has no side effects and always
// returns the same result for the same input.
size_t expensive_hash(char const restrict str[static 1]) [[reproducible]] {
    static char const* cached_str = NULL;
    static size_t cached_hash = 0;

    if (str == cached_str) {
        return cached_hash; // Internal state read
    }

    cached_str = str; // Internal state write (not observable outside)
    cached_hash = compute_hash(str);
    return cached_hash;
}

```

_Expected Behavior:_ The compiler is free to eliminate redundant calls to `compute_hash` or `expensive_hash` if called multiple times with the same pointer in a loop.

**Recap:**

- `[[unsequenced]]` guarantees absolute mathematical purity (no side effects, no global state reads).
- `[[reproducible]]` allows hidden internal state changes (like caching or temporarily changing floating-point modes) as long as it isn't externally observable.
- Pointers passed to these functions must be `restrict` qualified.

---

## 16.4. Measurement and inspection

### The Science of Profiling

**What it is:** Performance assessment is an empirical science. Humans are bad at guessing where code spends its time, so optimization must be driven by rigorous measurement.

**Why it matters / how it works:** Without measurement, programmers fall into the trap of "premature optimization," making code complex without actual speed gains.

> Takeaway 16.4 #1 _Don’t speculate about the performance of code; verify it rigorously._

> Takeaway 16.4 #2 _Complexity assessment of algorithms requires proofs._

> Takeaway 16.4 #3 _Performance assessment of code requires measurement._

### The Observer Effect in Code

**What it is:** The act of measuring code alters the code's behavior. Functions like `timespec_get` take time to execute, require system calls, force the compiler to save registers, and can invalidate CPU caches.

> Takeaway 16.4 #4 _All measurements introduce bias._

> Takeaway 16.4 #5 _Instrumentation changes compile-time and run-time properties._

**Key details:**

- **Optimizer Interference:** If you write an empty `for` loop to establish a baseline measurement, an optimizing compiler will delete the loop entirely because it has no side effects.
- **The Volatile Trick:** To force the compiler to execute a loop for benchmarking without doing real work, you must use `volatile` variables. `volatile` tells the compiler that the variable may change outside the program's control, forcing it to emit real load/store instructions.

### Hardening with Statistics

**What it is:** Because OS scheduling, interrupts, and caches cause run-time variability, a single measurement is meaningless. You must repeat experiments and use statistics (mean, variance, standard deviation).

> Takeaway 16.4 #6 _The relative standard deviation of run times must be in a low percentage range._

> Takeaway 16.4 #7 _Collecting higher-order moments of measurements to compute variance and skew is simple and cheap._

> Takeaway 16.4 #8 _Run-time measurements must be hardened with statistics._

### Code Demonstration: Benchmarking Bias and Loops

```c
#include <time.h>
#include <stdint.h>
#include <stdio.h>

int main(void) {
    struct timespec t_start, t_end;
    uint64_t iterations = 100000000;

    // Measurement 1: Empty loop (Compiler might delete this entirely!)
    timespec_get(&t_start, TIME_UTC);
    for (uint64_t i = 0; i < iterations; ++i) {
        /* do nothing */
    }
    timespec_get(&t_end, TIME_UTC);
    printf("Empty loop time (likely 0 due to optimization)\n");

    // Measurement 2: Volatile loop (Compiler forced to emit instructions)
    timespec_get(&t_start, TIME_UTC);
    for (uint64_t volatile i = 0; i < iterations; ++i) {
        /* volatile forces loop execution */
    }
    timespec_get(&t_end, TIME_UTC);
    // Real time taken by loop logic overhead is recorded here.
    printf("Volatile loop measured successfully.\n");

    return 0;
}

```

_Expected Behavior:_ The first loop will likely register as taking 0 nanoseconds on high optimization levels (`-O3`), while the second loop will record actual CPU cycles taken to increment and compare a variable in memory.

**Recap:**

- Never guess performance; prove algorithmic complexity and measure real times.
- Beware of compiler optimizations (like loop deletion) ruining benchmarks.
- Acknowledge instrumentation bias and use statistics (mean, standard deviation) over multiple runs to get accurate data.

---

## Summary

Chapter 16 focuses on the tools C provides to bridge the gap between high-level code structure and low-level machine performance. By providing strict guarantees to the compiler using features like `inline` (for function expansion), `restrict` (for exclusive pointer access), and `[[unsequenced]]` / `[[reproducible]]` (for mathematical purity and caching), programmers can unlock aggressive optimizations like vectorization and dead-code elimination. However, these tools are double-edged swords; misusing them results in Undefined Behavior rather than compiler errors. Finally, any application of these optimizations must be verified using rigorous, statistically sound profiling methods, keeping in mind that the act of measuring code inherently alters its execution profile and limits the compiler's ability to optimize around the instrumentation.

---

## Self-Check Questions

1. **Does placing the `inline` keyword on a function guarantee that the compiler will inline it?**
   _Answer:_ No, it is only a hint and a mechanism to allow multiple definitions across TUs without linker errors.

2. **Why must an `inline` function also have a declaration without the `inline` keyword in exactly one `.c` file?**
   _Answer:_ To ensure that a concrete, callable symbol is emitted into the compiled binary in case the compiler chooses not to inline the function at a specific call site.

3. **What happens if an `inline` function tries to modify a `static` variable defined outside of it?**
   _Answer:_ It is invalid in C. Because the `inline` function is visible across multiple TUs, it is ambiguous which TU's local `static` state it would be modifying.

4. **What is the core promise a programmer makes when using the `restrict` keyword?**
   _Answer:_ They promise that the memory accessed through that pointer is accessed _exclusively_ through that pointer (or derived pointers) in that scope, meaning it does not alias with other pointers.

5. **Why doesn't the standard library `memmove` function use the `restrict` keyword?**
   _Answer:_ Because `memmove` is explicitly designed to handle overlapping memory regions safely, which fundamentally violates the exclusive access rule of `restrict`.

6. **Can a function that prints to the console (`printf`) be marked with `[[unsequenced]]`?**
   _Answer:_ No. Printing modifies global system state (I/O buffers, the terminal display), and `[[unsequenced]]` strictly forbids observable side effects.

7. **What is the main difference between `[[unsequenced]]` and `[[reproducible]]`?**
   _Answer:_ `[[unsequenced]]` forbids _all_ state modifications. `[[reproducible]]` allows internal, unobservable state modifications (like updating a `static` cache variable) as long as the function's external behavior remains deterministic.

8. **Why must pointer parameters to `[[unsequenced]]` functions be marked as `restrict`?**
   _Answer:_ To ensure the purity of the function is maintained, it must be guaranteed that the memory it reads from/writes to doesn't alias with other data causing unpredictable side effects.

9. **What problem arises if you try to benchmark an empty `for` loop to establish a baseline time?**
   _Answer:_ An optimizing compiler will realize the loop has no observable effect on the program and will completely delete the loop, making your baseline measurement 0.

10. **How can you prevent the compiler from optimizing away a benchmarking loop?**
    _Answer:_ By declaring the loop control variable, or an accumulator variable inside the loop, as `volatile`. This forces the compiler to execute the memory operations.
