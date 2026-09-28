+++
title = "Modern C: Functions"
description = "A guide to C23 function prototypes, return rules, main and command-line arguments, recursion, preconditions, and algorithmic performance, with commented examples."
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

# [extra]
# math = false
# cover.image = "images/cover.png"
+++

In **Chapter 7: Functions** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the focus turns to unconditional control transfers, modularity, and code factorization. Functions allow developers to write logic once, establish clear interfaces, manage local variable lifecycles, and use execution stacks cleanly.

Here is a detailed breakdown of the core concepts and key takeaways from Chapter 7 across its three main sections:


### 1. Simple Functions & Prototypes (Section 7.1)
Functions implement features once so the rest of the codebase can rely on them without code duplication or copy-and-paste errors.

* **Mandatory Prototypes in C23:**
  * **Takeaway 7.1 #1:** *All functions must have prototypes*.
  * A prototype specifies both the argument parameter types and return type so the compiler can verify argument compatibility and perform automatic conversions prior to execution. Older C syntax allowing prototype-less declarations has been retired in C23.
* **Special Keyword `void`:**
  * Parameter list `(void)` indicates that a function expects no arguments.
  * Return type `void` specifies that a function yields no return value.
* **Return Rules and Function Boundaries:**
  * **Takeaway 7.1 #2:** *Functions have only one entry but can have several `return`s*.
  * **Takeaway 7.1 #3:** *A function `return` must be consistent with its type*. (An expression in a return statement is automatically converted to the function's declared return type if compatible).
  * **Takeaway 7.1 #4:** *Reaching the end of the body of a function is equivalent to a `return` statement without an expression*.
  * **Takeaway 7.1 #5:** *Reaching the end of the body of a function is only allowed for `void` functions*. (Falling off the end of a non-`void` function returns an uninitialized/garbage value and leads to an invalid execution state).

#### Example

In C23, every function must have a visible **prototype** that specifies parameter types and return type before it can be called. Functions have a single entry point but can contain multiple `return` statements. Values returned are automatically converted to the declared return type if compatible.

```c
#include <stdio.h>
#include <stdbool.h>

// 1. MANDATORY PROTOTYPES (Takeaway 7.1 #1):
// Declares function signature so the compiler can verify argument types and return type.
// 'void' in the parameter list explicitly specifies that the function takes no arguments.
void print_welcome_message(void);

// Prototype receiving an unsigned integer and returning a bool.
bool is_leap_year(unsigned year);

// Function returning 'void' (no return value yields control back to caller).
void print_welcome_message(void) {
    puts("=== C23 Function Prototypes & Execution Demo ===");
    // Takeaway 7.1 #4 & #5: Reaching the end of a void function is equivalent
    // to 'return;' without an expression. Falling off non-void functions is illegal.
}

// 2. SINGLE ENTRY, MULTIPLE RETURNS (Takeaway 7.1 #2 & #3):
bool is_leap_year(unsigned year) {
    // Precondition check with early return:
    if (year == 0) {
        return false; // First return point
    }

    // Takeaway 7.1 #3: Return expression must be consistent with return type (bool).
    // Evaluates truth of leap year rule (divisible by 4, except century years unless divisible by 400).
    return !(year % 4) && ((year % 100) || !(year % 400)); // Second return point
}

void demo_simple_functions(void) {
    print_welcome_message();

    unsigned test_year = 2024;
    // Calling function using prototype interface:
    if (is_leap_year(test_year)) {
        printf("Year %u is a leap year!\n", test_year);
    }
}
```

### 2. `main` is Special (Section 7.2)
`main` is the unique entry point called by the platform's process-startup routine and obeys specific rules established by the C standard.

* **Standard Prototypes:**
  1. `int main(void)`
  2. `int main(int argc, char* argv[])` (or `char* argv[argc+1]`)
* **Return Values and Termination:**
  * **Takeaway 7.2 #1:** *Use `EXIT_SUCCESS` and `EXIT_FAILURE` as return values for `main`* (from `<stdlib.h>`).
  * **Takeaway 7.2 #2:** *Reaching the end of `main` is equivalent to a `return` with `EXIT_SUCCESS`*. Unlike normal non-`void` functions, `main` is exempt from mandatory return statements; reaching the closing brace implicitly returns `EXIT_SUCCESS` (`0`).
  * **Takeaway 7.2 #3:** *Calling `exit(s)` is equivalent to the evaluation of `return s` in `main`*.
  * **Takeaway 7.2 #4:** *`exit` never fails and never returns to its caller*. (It is marked with C23's `[[noreturn]]` attribute).
* **Command-Line Arguments (`argc` and `argv`):**
  * **Takeaway 7.2 #5:** *All command-line arguments are transferred as strings*. Decoding numeric parameters requires conversion functions like `strtod` or `strtoul`.
  * **Takeaway 7.2 #6:** *`argv` points to the name of the program invocation*.
  * **Takeaway 7.2 #7:** *`argv[argc]` is a null pointer*.

#### Example

`main` acts as the standard entry point called by the platform's process-startup routine. Command-line arguments arrive as an array of character strings (`argv`), where `argv` holds the invocation name and `argv[argc]` is guaranteed to be `nullptr`. Program status is communicated using `EXIT_SUCCESS` or `EXIT_FAILURE`.

```c
#include <stdio.h>
#include <stdlib.h> // Provides EXIT_SUCCESS, EXIT_FAILURE, and strtod

// Standard C23 prototype for main accepting command-line arguments:
// argv is an array of argc + 1 string pointers.
int main(int argc, char* argv[argc + 1]) {

    // Takeaway 7.2 #6: argv contains the program execution invocation name.
    printf("Program Invocation Name: %s\n", argv);
    printf("Argument count (argc): %d\n", argc);

    // Takeaway 7.2 #7: argv[argc] is strictly guaranteed to be a null pointer.
    if (argv[argc] == nullptr) {
        puts("Verified: argv[argc] is nullptr.");
    }

    // Takeaway 7.2 #5: All command-line arguments are transferred as text strings.
    // Parsing requires decoding functions like strtod() from <stdlib.h>.
    if (argc > 1) {
        double val = strtod(argv, nullptr); // Convert string argument to double
        printf("Parsed numeric argument 1: %g\n", val);
    } else {
        puts("No command-line arguments supplied.");
    }

    // Takeaway 7.2 #3 & #4: calling exit(s) terminates the program immediately
    // and never returns (marked with [[noreturn]]).
    if (argc > 5) {
        puts("Too many arguments provided! Exiting early.");
        exit(EXIT_FAILURE); // Equivalent to returning EXIT_FAILURE from main
    }

    // Takeaway 7.2 #1 & #2: Reaching the end of main implicitly returns EXIT_SUCCESS.
    // Using explicit return values makes program execution success/failure explicit.
    return EXIT_SUCCESS; // Standard successful exit return code
}
```

### 3. Recursion (Section 7.3)
Recursive functions directly or indirectly call themselves. Every invocation creates a fresh set of local variables and parameters on the stack.

* **Preconditions & Termination Checks:**
  * **Takeaway 7.3 #1:** *Make all preconditions for a function explicit*. (Using runtime assertions like `assert()` from `<assert.h>`).
  * **Takeaway 7.3 #2:** *In a recursive function, first check the termination condition*. Missing a termination check causes infinite recursion, leading to stack overflow and program crash.
  * **Takeaway 7.3 #3:** *Ensure the preconditions of a recursive function in a wrapper function*. A public wrapper function validates inputs once before passing control to the recursive implementation.
* **Performance Realities:**
  * **Takeaway 7.3 #4:** *Multiple recursion may lead to exponential computation times*. (e.g., naive recursive Fibonacci calculation recomputes overlapping subproblems, yielding \\(O(\phi^n)\\) runtime).
  * **Takeaway 7.3 #5 & #6:** *A bad algorithm will never lead to a performing implementation; improving an algorithm can dramatically improve performance*. (e.g., replacing naive recursion with memoization/caching or iterative loops).

#### Example

Every recursive invocation creates a fresh set of local variables and parameters on the execution stack. A safe recursive algorithm must state preconditions explicitly using `assert()`, check its **termination condition** first to prevent stack overflow, and use **wrapper functions** to validate input before recursing.

```c
#include <stdio.h>
#include <stddef.h>
#include <assert.h> // Provides the assert() macro for runtime preconditions

// --- EUCLID'S ALGORITHM (Euclidean GCD) ---

// Internal recursive implementation:
// Assumes precondition (a <= b) holds automatically.
static size_t gcd_recursive(size_t a, size_t b) {
    // Takeaway 7.3 #1: State preconditions explicitly.
    assert(a <= b); // Asserts ordering invariant on entering recursion level

    // Takeaway 7.3 #2: Check termination condition FIRST.
    // Missing this check causes infinite recursion and stack overflow.
    if (a == 0) {
        return b; // Base case: gcd(0, b) == b
    }

    // Modulo reduces arguments while preserving precondition for next level.
    return gcd_recursive(b % a, a); // Recursive step
}

// Takeaway 7.3 #3: Public WRAPPER function validates preconditions once.
inline size_t gcd(size_t a, size_t b) {
    assert(a > 0 || b > 0); // Precondition: At least one argument must be non-zero

    // Ensure arguments satisfy (a <= b) before invoking recursive function
    if (a <= b) {
        return gcd_recursive(a, b); //
    } else {
        return gcd_recursive(b, a); //
    }
}

// --- RECURSIVE PERFORMANCE REALITIES ---

// Naive double recursion (Fibonacci sequence):
// Takeaway 7.3 #4: Multiple recursion leads to exponential time O(phi^n).
size_t fib_naive(size_t n) {
    if (n < 3) {
        return 1; // Termination check
    }
    // Recomputes identical subproblems repeatedly
    return fib_naive(n - 1) + fib_naive(n - 2);
}

// Takeaway 7.3 #5 & #6: Improving the algorithm dramatically improves performance.
// Linear recursive implementation using a constant 2-element buffer:
static void fib_linear_rec(size_t n, size_t buf[static 2]) {
    if (n > 2) {
        size_t next = buf + buf;
        buf = buf;
        buf = next;
        fib_linear_rec(n - 1, buf); // Single recursive call per step
    }
}

size_t fib_fast(size_t n) {
    if (n == 0) return 0;
    size_t buf = {1, 1}; // Buffer holds consecutive sequence values
    fib_linear_rec(n, buf);
    return buf; // Computes in O(n) linear time rather than exponential
}

void demo_recursion(void) {
    size_t x = 18, y = 30;
    printf("gcd(%zu, %zu) = %zu\n", x, y, gcd(x, y)); // Yields 6

    size_t n = 10;
    printf("Fibonacci(%zu) fast = %zu\n", n, fib_fast(n)); // Yields 55
}
```
