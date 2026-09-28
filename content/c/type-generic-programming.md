+++
title = "Modern C: Type-generic programming"
description = "A guide to C23 type-generic programming, covering inherent generic features, _Generic dispatch, type inference with auto and typeof, and anonymous function extensions."
date = 2026-09-28T07:16:58Z

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

#### Example

C has always possessed inherent type-generic mechanisms: arithmetic operators (like `+`, `==`, `*`), implicit integer promotions, usual arithmetic conversions, function pointers, `void*` pointers, and library headers like `<tgmath.h>`. C23 enhances this by making standard search functions (`strstr`, `strchr`, `memchr`, etc.) **`const`-preserving type-generic macros**.

```c
#include <stdio.h>
#include <string.h>

// 1. MACROS USING ARITHMETIC CONVERSIONS (Section 18.1.3):
// Evaluates RGB channels type-generically; integer promotion or float conversion determines result type.
#define GRAY(R, G, B) (((R) + (G) + (B)) / 3)

void demo_inherent_type_genericity(void) {
        // 2. C23 CONST-PRESERVING TYPE-GENERIC MACROS (Section 18.1.7):
        char const unmut_str[] = "haystack_const";
        char mut_str[] = "haystack_mutable";
        char const needle[] = "stack";

        // C23 strstr macro preserves 'const' qualification:
        char const* p_const = strstr(unmut_str, needle); // OK: Returns 'char const*' for 'char const*' input
        char* p_mut = strstr(mut_str, needle);          // OK: Returns 'char*' for 'char*' input

        // Prevents historical type-system hole where const pointers could leak into non-const pointers:
        // char* bad_ptr = strstr(unmut_str, needle); // COMPILER ERROR in C23!

        printf("Const search: %s, Mutable search: %s\n", p_const, p_mut);

        // Using macro GRAY with unsigned char results in promoted int; with float results in float
        unsigned char r = 100, g = 150, b = 200;
        printf("Gray value: %d\n", GRAY(r, g, b));
}
```

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

#### Example

Introduced in C11, `_Generic(controlling_expr, type1: expr1, ..., default: exprN)` selects an expression based **strictly on the compile-time type** of the controlling expression.

* **Takeaway 18.2 #1:** *The result type of a `_Generic` expression is the type of the chosen expression*.
* **Takeaway 18.2 #2:** *Using `_Generic` with `inline` functions adds optimization opportunities*.
* **Takeaway 18.2 #3:** *All choices `expression1` ... `expressionN` in a `_Generic` must be valid*.
* **Takeaway 18.2 #4:** *Type expressions in `_Generic` should be unqualified types* (qualifiers are stripped automatically from the controlling expression).

```c
#include <stdio.h>
#include <limits.h>

// INLINE DISPATCH FUNCTIONS (Takeaway 18.2 #2):
static inline float minf(float a, float b) { return a < b ? a : b; }
static inline double mind(double a, double b) { return a < b ? a : b; }
static inline long double minl(long double a, long double b) { return a < b ? a : b; }

// TYPE-GENERIC MINIMUM MACRO:
// Sum (A)+(B) acts as controlling expression; usual arithmetic conversions pick the wider type.
#define min(A, B) \
    _Generic((A) + (B), \
        float: minf, \
        long double: minl, \
        default: mind)((A), (B))

// COMPILE-TIME INTEGER CONSTANT EXPRESSIONS VIA _Generic:
// Evaluates to a compile-time max value constant without invoking any function calls.
#define MAXVAL(X) \
    _Generic((X), \
        bool: (bool)+1, \
        signed char: SCHAR_MAX, \
        unsigned char: UCHAR_MAX, \
        signed int: INT_MAX, \
        unsigned int: UINT_MAX, \
        float: FLT_MAX, \
        double: DBL_MAX, \
        default: LDBL_MAX)

void demo_generic_selection(void) {
    float f1 = 3.14f, f2 = 2.71f;
    double d1 = 100.0, d2 = 50.0;

    // Dispatches to minf (preserving float precision without promoting to double)
    float res_f = min(f1, f2);
    // Dispatches to mind
    double res_d = min(d1, d2);

    printf("minf: %f, mind: %g\n", res_f, res_d);
    printf("Max value for unsigned int: %u\n", MAXVAL(0U));
}
```

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

#### Example

C23 introduces type inference keywords to avoid the combinatorial case explosion of `_Generic`.

* **`auto`**: Infers a local variable's declaration type directly from its initializer expression.
* **`typeof(expr)`**: Inspects the exact type of an expression without evaluating it at runtime.
* **`typeof_unqual(expr)`**: Inspects the type while dropping top-level qualifiers (`const`, `volatile`).

* **Takeaway 18.3.1 #1:** *Protect local variables inside macros by a documented naming convention* (e.g., `swap_p1`, `swap_tmp`).
* **Takeaway 18.3.1 #2:** *Use `auto` definitions where you must ensure type consistency*.
* **Takeaway 18.3.2 #1:** *Prefer `auto` over `typeof` for variable declarations*.

```c
#include <stdio.h>
#include <stdbool.h>
#include <assert.h>

// COMPILE-TIME TYPE COMPATIBILITY ASSERTION USING typeof:
#define static_assert_compatible(A, B, REASON) \
    static_assert(_Generic((typeof(A)*)nullptr, \
                           typeof(B)*: true, \
                           default: false), \
                  "Expected compatible types: " REASON)

// TYPE-GENERIC SWAP MACRO USING C23 auto & typeof:
#define SWAP(X, Y) \
    do { \
        /* Evaluates X and Y addresses once into auto pointer variables */ \
        auto const swap_p1 = &(X); \
        auto const swap_p2 = &(Y); \
        static_assert_compatible(*swap_p1, *swap_p2, #X " and " #Y " must have compatible types"); \
        /* Infers temporary storage type matching the underlying object */ \
        auto swap_tmp = *swap_p1; \
        *swap_p1 = *swap_p2; \
        *swap_p2 = swap_tmp; \
    } while (false)

// NARROWING CAST VIA typeof_unqual:
// Uses a wide inline function and casts the result back down to the caller's precision.
static inline long double absolute_impl(long double x) {
    return x < 0.0L ? -x : x;
}
#define absolute(X) ((typeof_unqual(X))absolute_impl(X))

void demo_type_inference(void) {
    int a = 10, b = 20;
    SWAP(a, b); // Swaps int variables cleanly
    printf("Swapped ints: a = %d, b = %d\n", a, b);

    float f = -5.5f;
    float abs_f = absolute(f); // Returns 'float' (typeof_unqual narrows long double back to float)
    printf("Absolute float: %f\n", abs_f);
}
```

### 4. Anonymous Functions & Compound Expressions (Section 18.4)
Statement macros wrapped in `do { ... } while (false)` cannot return values or be used inside expressions. Compiler extensions fill this gap by enabling anonymous function-like abstractions:

* **GNU Statement Expressions (`({ ... })`):** Allows multi-statement blocks to be treated as expressions that return the value of their final statement. This enables macros like `SWAP(X, Y)` or `MAX(X, Y)` to evaluate arguments exactly once into local `auto` variables before computing a result.
* **Clang/Apple Block Closures (`^`):** Constructs anonymous block closures `void ^(void) { ... } ()` that can be invoked immediately as expression blocks.

#### Example

Statement macros wrapped in `do { ... } while (false)` cannot return values or be used inside expression contexts (such as conditional operators or loop conditions). Compiler extensions provide statement expressions and block closures to create functional abstractions.

```c
#include <stdio.h>

// GCC/CLANG STATEMENT EXPRESSION ({ ... }):
// Evaluates multi-statement blocks as an expression, returning the value of the final statement.
#define SAFE_MAX(X, Y) \
    ({ \
        /* Evaluates arguments once into local auto variables to prevent side-effect bugs */ \
        auto const max_x = (X); \
        auto const max_y = (Y); \
        max_x > max_y ? max_x : max_y; /* Final statement value is returned */ \
    })

void demo_anonymous_functions(void) {
    int x = 5, y = 9;

    // SAFE_MAX can be used inside expressions and evaluates x++ and y++ exactly once:
    int m = SAFE_MAX(x++, y++);

    printf("SAFE_MAX result: %d (x = %d, y = %d)\n", m, x, y);
}
```

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
