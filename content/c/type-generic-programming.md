+++
title = "Modern C: Type-generic programming"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Chapter 18 of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, titled **"Type-generic programming,"** continues **Level 3: Experience** by examining how C achieves interface flexibility and type safety. 

Although C is often perceived as a strictly typed imperative language, type-generic mechanisms are omnipresent throughout the standard. Chapter 18 organizes these features into four main areas: **inherent language features**, **C11 generic selection (`_Generic`)**, **C23 type inference (`auto`, `typeof`, `typeof_unqual`)**, and **anonymous function extensions**.

Below is a detailed breakdown of the key concepts, design rules, and takeaways from Chapter 18 across its four sections.


### 1. Inherent Type-Generic Features in C (Section 18.1)
Type-generic behavior is built deeply into C's core syntax and library design:

* **Type-Generic Operators:** Operators like `==`, `!=`, `+`, `-`, `*`, and `/` work across wide integer types, real floating types, complex types, and pointer types without requiring different function names.
* **Default Promotions & Conversions:** Implicit integer promotions and usual arithmetic conversions automatically convert narrow arguments (such as `bool` or `char`) or mixed-type operands to a common wider type before evaluating expressions.
* **Macros for Generic Expressions & Statements:** Preprocessor macros combined with promotion rules allow creating type-generic expressions (such as `GRAY(R, G, B)`) or statement blocks wrapped in `do { ... } while (false)`.
* **Variadic Functions & Function Pointers:** Functions taking `...` or callbacks accepting `void*` parameters (such as `qsort` or `bsearch`) provide generic interfaces, though type safety must be maintained manually by the programmer.
* **Type-Generic C Library Headers:**
  * **`<tgmath.h>` (C99):** Dispatches mathematical calls like `sin(x)` or `fabs(x)` to the appropriate `float`, `double`, or `long double` library function based on parameter type.
  * **`<stdatomic.h>` (C11):** Operates on an unbounded set of atomic types.
  * **C23 `const`-Preserving Macro Interfaces:** C23 upgrades standard string and search functions (`memchr`, `strchr`, `strpbrk`, `strrchr`, `strstr`, `bsearch`, etc.) with type-generic macros. When called with a `const`-qualified pointer argument, they return a `const`-qualified pointer, eliminating a historical type-system flaw where non-const pointers could leak from const inputs.


### 2. Generic Selection via `_Generic` (Section 18.2)
Introduced in C11, **generic selection** (`_Generic`) provides direct language support for compile-time type-based dispatching.

* **Syntax & Evaluation:**
  \\[\text{\ttfamily \_Generic(controlling\_expression, type1: expr1, ..., default: exprN)}\\]
  The controlling expression is analyzed **strictly for its type at compile time** (it is not evaluated at runtime).
* **Key Takeaways & Constraints:**
  * **Takeaway 18.2 #1:** *The result type of a `_Generic` expression is the type of the chosen expression*.
  * **Takeaway 18.2 #2:** *Using `_Generic` with `inline` functions adds optimization opportunities*. By dispatching to specialized inline functions (e.g., `minf`, `mind`, `minl`), the compiler preserves exact floating-point precision and inlines the resulting code cleanly.
  * **Takeaway 18.2 #3:** *All choices `expression1` ... `expressionN` in a `_Generic` must be valid*. Every branch expression must be syntactically and semantically valid C code, even for branches that are not selected for a given invocation.
  * **Takeaway 18.2 #4:** *The type expressions in a `_Generic` expression should only be unqualified types, not array types or function types*. Types passed into controlling expressions undergo function parameter decay: type qualifiers (`const`, `volatile`) are stripped, arrays decay to pointers, and functions decay to function pointers.
  * **Takeaway 18.2 #5:** *The type expressions in a `_Generic` expression must refer to mutually incompatible types*.
  * **Takeaway 18.2 #6:** *The type expressions in a `_Generic` expression cannot be a pointer to a VLA*.


### 3. Type Inference: `auto`, `typeof`, and `typeof_unqual` (Section 18.3)
To prevent the combinatorial explosion of cases required by `_Generic` across C's many integer and floating types, C23 introduced explicit **type inference** keywords.

* **The `auto` Feature (Section 18.3.1):**
  * Re-purposes the `auto` keyword to infer a variable's declaration type directly from its initializer expression.
  * **Takeaway 18.3.1 #1:** *Protect local variables inside macros by a documented naming convention* (such as `swap_p1`, `swap_tmp`) to prevent collisions with caller identifiers.
  * **Takeaway 18.3.1 #2:** *Use `auto` definitions where you must ensure type consistency*. It guarantees that dependent local variables adapt automatically if an underlying expression's type is updated.
* **The `typeof` and `typeof_unqual` Operators (Section 18.3.2):**
  * **Takeaway 18.3.2 #1:** *Prefer `auto` over `typeof` for variable declarations*.
  * `typeof(expr)` inspects the exact type of an expression without evaluating it.
  * `typeof_unqual(expr)` inspects the type while dropping top-level type qualifiers (`const`, `volatile`).
  * **Compile-time Type Assertions:** Combining `typeof` with `_Generic` allows writing static assertion macros like `static_assert_compatible(A, B, REASON)` to verify that two macro parameters have compatible types before performing operations like `SWAP(X, Y)`.
  * **Narrowing Return Casts:** Casting a wide generic function result down using `(typeof_unqual(X))` allows implementing type-generic macros with simple inline functions while preserving the caller's precision.


### 4. Anonymous Functions & Compound Expressions (Section 18.4)
Statement macros wrapped in `do { ... } while (false)` cannot return values or be used inside expressions. Compiler extensions fill this gap by enabling anonymous function-like abstractions:

* **GNU Statement Expressions (`({ ... })`):** Allows multi-statement blocks to be treated as expressions that return the value of their final statement. This enables macros like `SWAP(X, Y)` or `MAX(X, Y)` to evaluate arguments exactly once into local `auto` variables before computing a result.
* **Clang/Apple Block Closures (`^`):** Constructs anonymous block closures `void ^(void) { ... } ()` that can be invoked immediately as expression blocks.


### Summary of Chapter 18 Key Takeaways

| Takeaway ID | Takeaway Rule Text |
| :--- | :--- |
| **Takeaway 18.2 #1** | *The result type of a `_Generic` expression is the type of the chosen expression*. |
| **Takeaway 18.2 #2** | *Using `_Generic` with `inline` functions adds optimization opportunities*. |
| **Takeaway 18.2 #3** | *All choices `expression1` ... `expressionN` in a `_Generic` must be valid*. |
| **Takeaway 18.2 #4** | *The type expressions in a `_Generic` expression should only be unqualified types, not array types or function types*. |
| **Takeaway 18.2 #5** | *The type expressions in a `_Generic` expression must refer to mutually incompatible types*. |
| **Takeaway 18.2 #6** | *The type expressions in a `_Generic` expression cannot be a pointer to a VLA*. |
| **Takeaway 18.3.1 #1** | *Protect local variables inside macros by a documented naming convention*. |
| **Takeaway 18.3.1 #2** | *Use `auto` definitions where you must ensure type consistency*. |
| **Takeaway 18.3.2 #1** | *Prefer `auto` over `typeof` for variable declarations*. |

