+++
title = "Modern C: Organization and documentation"
description = "A guide to structuring C23 projects, documenting interfaces, clarifying implementations, writing safe macros, and using pure functions and standard attributes."
date = 2026-09-28T03:55:22Z

[taxonomies]
tags = ["modernc", "documentation"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 10. Organization and documentation

**What it is:**
Programming is not just a technical endeavor; it is a collaborative, societal, and economic activity. Proper organization and documentation are essential for long-term project success. The C language requires a disciplined approach to code structure to ensure that code remains readable and maintainable over time, even by the original author.

**Why it matters / how it works:**
When reading or maintaining code, developers must understand the _what_, _what for_, _how_, and _in which manner_ of the implementation. Beginners often underestimate the effort required for organization, leading to unmaintainable codebases. By separating these concerns and documenting them appropriately, you reduce cognitive load for users and maintainers.

**Key details:**

- **What:** The interface specification (what is being done).
- **What for:** The purpose of the interface.
- **How:** The structural organization of the code.
- **In which manner:** The specific implementation details (algorithms, edge cases).

> **Takeaway 10 #1:** _Function interfaces describe_ what _is done._

This means that a function's prototype (its name, return type, and parameters) should clearly indicate its core operation. The signature itself acts as a contract between the caller and the function.

> **Takeaway 10 #2:** _Interface comments document the purpose of a function._

Comments attached to the interface (usually in header files) should explain _why_ this function exists and what role it plays in the broader application or library.

> **Takeaway 10 #3:** _Function code shows_ how _the function is organized._

The actual C code inside the function definition should be written clearly enough to reveal its logical flow (the "how"). Good naming conventions and structured statements make this self-evident.

> **Takeaway 10 #4:** _Code comments explain the manner in which function details are implemented._

In-line comments within the function body (in the `.c` file) should be reserved for explaining non-obvious implementation details, tricky algorithms, or the specific "manner" in which a problem was solved.

> **Takeaway 10 #5:** _Separate interface and implementation._

This is a fundamental rule in C programming. The interface (what the user sees) must be physically and logically separated from the implementation (how it works). This is achieved by using header files (`.h`) for interfaces and translation units (`.c`) for implementations.

> **Takeaway 10 #6:** _Document the interface; explain the implementation._

Comments in `.h` files should focus on documenting usage (for consumers of the API), while comments in `.c` files should explain complex internal logic (for maintainers of the API).

**Pitfalls:**

- Mixing interface declarations and implementation details in a single file makes the code harder to digest for users who only want to _use_ the functions.
- Over-commenting obvious code (e.g., `i++; // increment i`) clutters the "how" and distracts from the "in which manner."

**Code Demonstration:**

```c
// math_utils.h - INTERFACE (The "What" and "What for")
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

/**
 * @brief Computes the square of an integer.
 *
 * This function takes a signed integer and returns its mathematical square.
 * It does not check for overflow.
 *
 * @param x The integer to square.
 * @return The squared value.
 */
int math_square(int x); // Describes *what* is done.

#endif // MATH_UTILS_H

// math_utils.c - IMPLEMENTATION (The "How" and "In which manner")
#include "math_utils.h"

int math_square(int x) {
    // The code shows *how* it is done.
    // The comment explains the *manner* if it were a complex algorithm
    // (though simple here, imagine a fast inverse square root).
    return x * x;
}
```

_Expected behavior:_ The interface acts as a clean contract for users. If a user includes `math_utils.h`, they don't need to look at `math_utils.c` to know how to use the function.

**Recap:**

- Organize code by the _what_, _what for_, _how_, and _in which manner_ principles.
- Separate interfaces (`.h`) from implementations (`.c`).
- Direct your documentation to the right audience: interface documentation for users, implementation comments for maintainers.

---

## 10.1. Interface documentation

**What it is:**
C does not have a built-in documentation standard like Java (Javadoc) or Rust (rustdoc). However, the community heavily relies on external tools, with **Doxygen** being the most prominent. Interface documentation involves writing structured comments in header files that a tool can parse to generate HTML, PDF, or man-page documentation.

**Why it matters / how it works:**
Users of your library should only need to read the generated documentation (or the header files), not your source code. By using structured tags (like `@brief`, `@param`, `@see`), you create a predictable, searchable, and professional manual for your code.

**Key details:**

- Doxygen comments usually start with `/**` and end with `*/`.
- Common tags include `@brief` (short description), `@param` (parameter explanation), and `@see` (references to related functions/macros).

> **Takeaway 10.1 #1:** _Document interfaces thoroughly._

Every function, struct, enum, and macro exposed in a header file must have accompanying documentation explaining its usage, constraints, and return values.

> **Takeaway 10.1 #2:** _Structure your code in units that have strong semantic connections._

Group related functions, types, and macros into a single header file. If you have a `struct brian`, all functions manipulating a `brian` should live in `brian.h`. Do not scatter related concepts across multiple files.

**Pitfalls:**

- Forgetting include guards (`#ifndef HEADER_H ...`) in header files, leading to multiple definition errors during compilation.
- Documenting implementation details in the header file, which clutters the API reference.

**Code Demonstration:**

```c
// heron_k.h
#ifndef HERON_K_H
#define HERON_K_H

/**
 * @file heron_k.h
 * @brief Interfaces for Heron's approximation process.
 */

/**
 * @brief Use the Heron process to approximate a to the power of 1/k.
 *
 * Computes the k-th root of a. As a special feature, if k is -1,
 * it computes the multiplicative inverse of a.
 *
 * @param a Must be greater than 0.0.
 * @param k Should not be 0.
 *
 * @return The approximated root or inverse.
 */
double heron(double a, signed k) [[__unsequenced__]];

#endif
```

_Expected behavior:_ A tool like Doxygen will parse this file and create a clean web page showing the function signature, the brief description, and a formatted table of parameters.

**Recap:**

- Use structured comments (like Doxygen) in your header files.
- Document thoroughly: parameters, constraints, and return values.
- Group semantically related elements into the same header file and protect them with include guards.

---

## 10.2. Implementation

**What it is:**
The implementation (the `.c` files) is where the actual logic of your program resides. Writing a good implementation means writing code that is self-explanatory, literal, and obvious in its control flow.

**Why it matters / how it works:**
Code is read far more often than it is written. The primary goal of an implementation is not just to execute correctly on a machine, but to be readable by humans. Good naming conventions, logical grouping (`{}`), and clear control structures drastically reduce the need for comments.

**Key details:**

> **Takeaway 10.2 #1:** _Implement literally._

Do exactly what the code says it does. Avoid overly clever tricks or obscured logic. If you are calculating a sum, make it look like a sum.

> **Takeaway 10.2 #2:** _Control flow must be obvious._

Use comprehensive control statements (`if`, `for`, `while`) and compound statements (`{}`) clearly. A reader should be able to trace the execution path visually without heavy mental lifting. Avoid convoluted nested loops or excessive `goto` statements where simpler structures suffice.

**Pitfalls:**

- Writing "clever" one-liners that compress multiple operations into a single unreadable expression.
- Hiding complex reasoning; if a non-obvious algorithm is required, it must be explicitly explained via a comment.

**Recap:**

- Write code that reads like a descriptive text of the operations being performed.
- Keep control flow simple, visual, and obvious.
- Reserve comments for the "why" behind non-obvious implementation decisions.

### 10.2.1. Macros

**What it is:**
The C preprocessor allows the creation of macros—textual replacements that happen before actual compilation. Function-like macros look like function calls but are expanded in-place by the preprocessor.

**Why it matters / how it works:**
Macros can be powerful tools (e.g., type-generic programming via `<tgmath.h>`, or compiler magic like `errno`), but they are also notorious for obfuscating control flow and causing subtle, hard-to-track bugs. Modern C prefers `inline` functions and proper language constructs over macros wherever possible.

**Key details:**

- _(Note from standard C takeaways)_: Macros should not change control flow in a surprising way.

> **Takeaway 10.2.1 #2:** _Function-like macros should syntactically behave like function calls._

If you must write a macro that acts like a function, it must not break when placed in contexts where a standard function call would succeed (e.g., inside an `if` statement without braces).

**Pitfalls:**
The book outlines several severe pitfalls when using macros:

- **`if` without `else`**: A macro expanding to multiple statements can break an `if` statement if not properly enclosed.
- **Trailing semicolons**: A macro that includes a trailing semicolon in its definition can prematurely terminate an external control structure.
- **Comma operators**: The comma is ambiguous (list separator vs. control operator). Avoid it in macros.
- **Continuable expressions**: Expressions in macro arguments can bind incorrectly to surrounding operators if not heavily parenthesized. Always put parentheses around parameters and the entire replacement text.
- **Multiple evaluation**: Because macros are text replacements, passing an expression like `i++` into a macro that uses its parameter twice will result in `i` being incremented twice.

**Code Demonstration:**

```c
#include <stdio.h>

// BAD MACRO: Multiple evaluation and lack of safe scoping
#define BAD_SQUARE(x) x * x

// GOOD MACRO: Behaving syntactically like a function (if it were returning a value,
// it would be enclosed in parens, but here we show a statement macro)
// We use do { ... } while(0) to ensure it consumes a trailing semicolon safely.
#define SAFE_PRINT_SQUARE(x) do { \
    int _val = (x); /* Evaluate exactly once */ \
    printf("Square is %d\n", _val * _val); \
} while(0)

int main(void) {
    int i = 2;
    // Pitfall: Multiple evaluation
    // Expands to: i++ * i++ (Undefined behavior due to multiple modifications!)
    // int result = BAD_SQUARE(i++);

    int j = 2;
    // Expected behavior: Evaluates j++ once, correctly prints 4.
    SAFE_PRINT_SQUARE(j++);

    return 0;
}
```

_Expected behavior:_ The safe macro evaluates the argument once by assigning it to a local variable, preventing the multiple evaluation pitfall. The `do { ... } while(0)` construct ensures it requires a semicolon at the end, just like a function call.

**Recap:**

- Function-like macros are dangerous because they are naive text replacements.
- Always wrap macro parameters in parentheses.
- Prevent multiple evaluations of arguments.
- Use `do { ... } while(false/0)` for macros containing multiple statements to ensure they behave syntactically like function calls.

### 10.2.2. Pure functions

**What it is:**
A **pure function** is a function that only interacts with the rest of the program through its interface (its parameters and return value). It does not read or modify global state, nor does it perform I/O. Its output depends _solely_ on its input.

**Why it matters / how it works:**
Pure functions are incredibly easy for humans to reason about because there are no hidden side-effects. Furthermore, modern compilers can aggressively optimize pure functions (e.g., memoization, eliminating redundant calls) because they are guaranteed not to alter the program's hidden state.

**Key details:**

> **Takeaway 10.2.2 #3:** _Express small tasks as pure functions whenever possible._

When building data structures, prefer small functions that take values, compute a new value, and return it. This approaches a functional or object-oriented style, accepting a slight performance cost (copying data) in exchange for massive gains in correctness and readability.

**Pitfalls:**

- Relying on global variables (`extern` or file-scope `static`) inside small helper functions.
- Mutating passed pointers when returning a new value would have sufficed for small data types.

**Code Demonstration:**

```c
#include <stdbool.h>
#include <stddef.h>

// A simple structure for rational numbers
struct rat {
    bool sign;
    size_t num;
    size_t denom;
};
typedef struct rat rat;

// Helper pure function: Greatest Common Divisor
size_t gcd(size_t a, size_t b) [[__unsequenced__]] {
    while (b != 0) {
        size_t temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}

// PURE FUNCTION: Normalizes a rational number.
// It takes a value (not a pointer), modifies the local copy, and returns it.
// [[__unsequenced__]] explicitly tells the compiler it is pure.
rat rat_get_normal(rat x) [[__unsequenced__]] {
    size_t c = gcd(x.num, x.denom);
    if (c > 0) {
        x.num /= c;
        x.denom /= c;
    }
    return x;
}

int main(void) {
    rat my_rat = { .sign = true, .num = 10, .denom = 20 };
    // We pass by value. my_rat is unchanged, we get a new normalized rat.
    rat normalized = rat_get_normal(my_rat);

    return 0;
}
```

_Expected behavior:_ `rat_get_normal` operates purely on the `x` value passed to it. It has no side effects and returns a brand new `rat` struct, making it a pure function.

**Recap:**

- Pure functions only depend on inputs and only affect their return values.
- Use pure functions for small, manageable tasks to improve readability and compiler optimization.
- Passing small `struct`s by value is a highly effective way to enforce pure behavior.

### 10.2.3. Attributes

**What it is:**
Attributes (introduced heavily in C23) provide a standardized way to annotate C code with extra information that the compiler can use for diagnostics, safety checks, and performance optimizations.

**Why it matters / how it works:**
Attributes allow you to tell the compiler specific properties about your functions. For instance, annotating a pure function with `[[__unsequenced__]]` guarantees to the compiler that the function's return value only depends on its arguments, allowing the compiler to heavily optimize consecutive calls.

**Key details:**

- Attributes use the `[[ ... ]]` syntax.
- The C standard provides standard attributes, but often provides a macro-safe double-underscore variant (e.g., `[[__unsequenced__]]` instead of `[[unsequenced]]`).

> **Takeaway 10.2.3 #2:** _Use the double underscore forms of attributes in header files._

When defining interfaces in header files that might be included in a variety of contexts (or mixed with legacy code or C++), using the `__attribute__` double-underscore format prevents naming collisions with user-defined macros that might coincidentally share the attribute's name (like `#define unsequenced 1`).

**Pitfalls:**

- Using plain attribute names (like `[[nodiscard]]` or `[[unsequenced]]`) in public header files. If a user has a macro with the same name, it will break the compilation of the header file.

**Code Demonstration:**

```c
#ifndef RATIONALS_H
#define RATIONALS_H

#include <stddef.h>
#include <stdbool.h>

struct rat {
    bool sign;
    size_t num;
    size_t denom;
};
typedef struct rat rat;

// SAFE ATTRIBUTE USAGE:
// Using the double-underscore version [[__unsequenced__]] prevents
// macro collisions when users include this header.
rat rat_get_normal(rat x) [[__unsequenced__]];

// Using [[__nodiscard__]] ensures the compiler warns the user if they
// call this function but ignore the returned normalized value.
rat rat_get(signed sign, size_t num, size_t denom) [[__nodiscard__]];

#endif
```

_Expected behavior:_ The compiler understands the optimization properties and constraints of these functions without risking macro pollution in the user's codebase. If a user calls `rat_get(1, 1, 2);` without assigning the result, the compiler will issue a warning.

**Recap:**

- Attributes provide crucial metadata to the compiler for optimization and safety.
- Always use the double-underscore version of standard attributes (e.g., `[[__unsequenced__]]`) in header files to avoid macro expansion collisions.

---

## Summary

Chapter 10 emphasizes that writing C code is as much about human communication as it is about instructing a computer. Excellent software design rigorously separates interface (the _what_ and _what for_, placed in `.h` files) from implementation (the _how_ and _in which manner_, placed in `.c` files). The chapter advocates for exhaustive interface documentation using tools like Doxygen, while stressing that the actual implementation should be written literally, with an obvious control flow that minimizes the need for internal comments. Furthermore, we must be wary of function-like macros, preferring pure functions passed by value. Finally, C23 attributes provide a modern, robust way to annotate functions for compiler optimization, provided we use the safe, double-underscore variants in our public headers.

---

## Self-Check Questions

1.  **What is the purpose of an interface according to the chapter's takeaways?**
    Function interfaces describe _what_ is done.

2.  **Where should you document the purpose of a function, and where should you explain its complex implementation?**
    Document the purpose in the interface (header file). Explain the complex implementation within the code comments of the implementation file (`.c` file).

3.  **Why should we separate interfaces and implementations into different files?**
    To reduce cognitive load for users who only need to know how to use the functions, and to logically organize the project.

4.  **What tool does the book recommend for generating documentation from header file comments?**
    Doxygen.

5.  **How should code be structured among different header files?**
    Code should be structured into units (files) that have strong semantic connections, typically grouping all functions operating on a specific data type together.

6.  **What does "Implement literally" mean in the context of C programming?**
    It means the code should read exactly as what it is doing, avoiding convoluted logic or "clever" obfuscation. The control flow must be visually obvious.

7.  **Why are function-like macros considered dangerous?**
    They are simple textual replacements. They can break control flow (like `if` statements without `else`), suffer from trailing semicolon issues, and cause multiple evaluation of their arguments.

8.  **How should a function-like macro syntactically behave?**
    It should behave exactly like a function call (e.g., safely absorbing a trailing semicolon and safely scoping variables).

9.  **What is a pure function?**
    A function whose return value depends solely on its input arguments, with no side-effects or reliance on global state. The book recommends expressing small tasks as pure functions whenever possible.

10. **Why should you use the double underscore form of attributes (like `[[__unsequenced__]]`) in header files?**
    To prevent naming collisions with macros that a user might have defined in their own codebase prior to including your header file.
