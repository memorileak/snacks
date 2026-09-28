+++
title = "Modern C: Functions"
description = ""
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


### 3. Recursion (Section 7.3)
Recursive functions directly or indirectly call themselves. Every invocation creates a fresh set of local variables and parameters on the stack.

* **Preconditions & Termination Checks:**
  * **Takeaway 7.3 #1:** *Make all preconditions for a function explicit*. (Using runtime assertions like `assert()` from `<assert.h>`).
  * **Takeaway 7.3 #2:** *In a recursive function, first check the termination condition*. Missing a termination check causes infinite recursion, leading to stack overflow and program crash.
  * **Takeaway 7.3 #3:** *Ensure the preconditions of a recursive function in a wrapper function*. A public wrapper function validates inputs once before passing control to the recursive implementation.
* **Performance Realities:**
  * **Takeaway 7.3 #4:** *Multiple recursion may lead to exponential computation times*. (e.g., naive recursive Fibonacci calculation recomputes overlapping subproblems, yielding \\(O(\phi^n)\\) runtime).
  * **Takeaway 7.3 #5 & #6:** *A bad algorithm will never lead to a performing implementation; improving an algorithm can dramatically improve performance*. (e.g., replacing naive recursion with memoization/caching or iterative loops).
