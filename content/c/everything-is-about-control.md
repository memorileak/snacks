+++
title = "Modern C: Everything is about control"
description = "An introduction to C23 control flow, covering conditional execution, scalar truth values, iteration, loop control, and multi-way selection with switch statements."
date = 2026-09-27T16:34:00Z

[taxonomies]
tags = ["modernc", "basic"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 3. Everything is about control

### The Concept of Control Flow

**What it is:** Control flow refers to the order in which individual statements, instructions, or function calls of an imperative program are executed or evaluated. In C, control is transferred either unconditionally (via function calls and returns) or conditionally (via specialized control statements).

**Why it matters / how it works:** Without control flow constructs, a program would execute strictly from top to bottom, one statement at a time, making it impossible to perform tasks that require decision-making or repetition. The C language provides five primary conditional control statements: `if`, `for`, `do`, `while`, and `switch`. These evaluate a condition (often a Boolean expression or integer value) to dictate which path the abstract state machine takes.

**Key details:**

- Functions are a form of _unconditional_ control transfer (`call` transfers to the function, `return` transfers back).
- C has other conditional constructs discussed later: the ternary operator (`? :`), preprocessor conditionals (`#if`, `#ifdef`), and type-generic selection (`_Generic`).

**Pitfalls:** Assuming statements always execute sequentially can lead to bugs, especially when loops or complex conditions cause jumps that skip necessary variable initializations or resource cleanups.

**Connections:** Links to earlier concepts of the _abstract state machine_ (Chapter 5) and future chapters on variations in control flow (Chapter 19) involving `goto`, `longjmp`, and signal handlers.

---

## 3.1. Conditional execution

### The `if` Statement and Boolean Context

**What it is:** The `if` statement evaluates a controlling expression to decide whether to execute a subsequent block of code (the secondary block). It allows for branching logic where code is executed only if a certain condition holds true, optionally providing an `else` path if the condition is false.

**Why it matters / how it works:** Conditional execution allows software to react to data and states. In C, the controlling expression is placed in parentheses, and the selection evaluates whether this expression is "true" or "false". C does not restrict this evaluation strictly to the `bool` type; any numeric value can act as a condition.

**Key details:**

- General syntax: `if (condition) secondary-block0 else secondary-block1`.
- The secondary block is usually a compound statement enclosed in curly braces `{ ... }`.
- Comparisons can use operators like `==` (equality) and `!=` (inequality).
- Any numeric evaluation can serve directly as a conditional without needing explicit logical operators.

> Takeaway 3.1 #1: _The value 0 represents logical false._

**Explanation:** In C's boolean context, any integer, floating-point, or pointer value that evaluates to exactly zero is treated as `false`. This is baked into the language's core design.

> Takeaway 3.1 #2: _Any value different from 0 represents logical true._

**Explanation:** Conversely, any non-zero value, whether it is `1`, `-42`, `3.14`, or a valid memory address, is evaluated as logical `true`.

> Takeaway 3.1 #3: _Don't compare to 0, false, or true._

**Explanation:** Because of C's built-in interpretation of numeric values as truth values, explicitly comparing a variable to `0`, `true`, or `false` (e.g., `if (i != 0)`) is redundant and considered poor coding style by experienced C programmers. Instead of `if (b != false == true)`, one should simply write `if (b)`. The negation is handled directly with the `!` operator (e.g., `if (!b)`).

> Takeaway 3.1 #4: _All scalars have a truth value._

**Explanation:** In C, "scalar types" include all numerical types (such as `size_t`, `int`, `double`, `bool`) and pointer types. Because they are scalars, they can be directly evaluated in an `if`, `while`, or `for` condition to determine their truth value based on whether they are non-zero.

**Pitfalls:**

- Accidentally using the assignment operator `=` instead of the equality operator `==` in an `if` condition (e.g., `if (a = 0)` assigns 0 to `a` and evaluates to false, which is rarely what the programmer intends).
- Relying on floating-point exactness for true/false evaluation can be dangerous due to precision issues (e.g., `if (float_val)` might evaluate to true if it is `0.0000001` due to rounding).

**Connections:** Connects to the scalar types overview in Table 3.1 (Chapter 5 basics), including `size_t`, `double`, and pointer types.

#### Code Demonstration: Conditional Execution

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

int main(void) {
    size_t count = 5;
    bool is_ready = true;
    double offset = 0.0;

    // Demonstrating Takeaway 3.1 #1 and #2: 0 is false, non-zero is true.
    // Demonstrating Takeaway 3.1 #3: Using the value directly without explicit comparison.
    if (count) { // Equivalent to (count != 0). Evaluates to true.
        printf("Count is non-zero (true).\n");
    }

    if (!offset) { // offset is 0.0, which evaluates to false. ! false is true.
        printf("Offset is zero.\n");
    }

    // Demonstrating Takeaway 3.1 #4: Scalars (bool here) have a truth value.
    if (is_ready) {
        printf("System is ready.\n");
    }

    // What goes wrong if we violate Takeaway 3.1 #3 (bad style):
    if (is_ready == true) { // Redundant and amateurish
        printf("Still ready, but poorly written.\n");
    }

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ The program will print "Count is non-zero (true).", "Offset is zero.", "System is ready.", and "Still ready, but poorly written." to the console.
_What goes wrong if..._ If you write `if (count = 0)` instead of `if (count == 0)`, `count` is mutated to 0, the expression evaluates to `0` (false), and the block is skipped unexpectedly.

**Recap: Conditional execution**

- `if` statements redirect control flow based on a condition.
- `0` acts as `false`, and all other scalar values act as `true`.
- Directly evaluate variables in control expressions instead of redundantly comparing them to `0` or `true`.

---

## 3.2. Iterations

### Domain Iteration with `for`

**What it is:** Iteration is the mechanism of doing something repeatedly for a specified number of times or until a condition is met. The `for` statement is C's preferred mechanism for looping over a specific domain (a known sequence or range of values).

**Why it matters / how it works:** Iterations drastically reduce code repetition. A `for` loop organizes iteration into a clean header with three clauses: initialization (`clause1`), loop condition (`condition2`), and iteration update (`expression3`). The condition is evaluated before every pass, and the update occurs at the end of the secondary block.

**Key details:**

- Syntax: `for (clause1; condition2; expression3) secondary-block`
- `clause1` should ideally be a variable definition, scoping the loop variable tightly to the loop (e.g., `for (size_t i = 0; ...)`).
- Multiple variables can be declared in `clause1` if they share the same type (e.g., `for (size_t i = 0, stop = 10; i < stop; ++i)`).
- The condition is evaluated just like an `if` condition; if it evaluates to 0, the loop ends.
- Omitting the middle condition, like `for (;;) `, creates an infinite loop (equivalent to `while(true)`).

### `while` and `do-while` loops

**What it is:** C provides two other looping constructs: `while` evaluates a condition before executing its block, and `do-while` evaluates the condition _after_ executing its block.

**Why it matters / how it works:** These are used when the number of iterations isn't tied to a specific sequence or domain, but purely to a state condition. `do-while` guarantees that the secondary block is executed at least once because the truth check happens at the bottom of the loop.

**Key details:**

- Syntax: `while (condition) secondary-block`
- Syntax: `do secondary-block while (condition);` (Note the necessary semicolon at the end).

### `break` and `continue`

**What it is:** These are jump statements specifically used inside loop blocks to alter standard iterative control flow.

**Why it matters / how it works:** Sometimes a loop needs to end prematurely or skip the current cycle based on an internal check.

**Key details:**

- `break`: Immediately terminates the innermost enclosing loop statement. No further expressions in the block are executed, and the loop condition is not re-evaluated.
- `continue`: Immediately stops executing the current iteration's block, jumps to the update expression (in a `for` loop), and re-evaluates the loop condition to decide whether to start the next iteration.

**Pitfalls:**

- **Unsigned Underflow:** In a loop like `for (size_t i = 9; i <= 9; --i)`, because `size_t` is unsigned, when `i` becomes `0` and is decremented, it wraps around to `SIZE_MAX`. Because `SIZE_MAX` is definitely not `<= 9`, the loop stops, but relying on underflow mechanics can be confusing and lead to infinite loops if the condition isn't carefully crafted.
- Forgetting the trailing semicolon in a `do-while` loop results in a syntax error.

**Connections:** Connects to C23's emphasis on block scope (declaring variables inside `for` headers) and array indexing (which relies heavily on `for` loops utilizing `size_t`).

#### Code Demonstration: Iterations

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

int main(void) {
    // 1. Standard domain iteration
    printf("Domain Iteration:\n");
    for (size_t i = 0, stop = 3; i < stop; ++i) {
        printf("Iteration %zu\n", i);
    }

    // 2. while loop with continue and break
    printf("\nWhile loop with jumps:\n");
    size_t count = 5;
    while (count) { // Truth evaluation (count != 0)
        --count;
        if (count == 3) {
            printf("Skipping 3...\n");
            continue; // Skip the rest of this iteration
        }
        if (count == 0) {
            printf("Breaking at 0 before print...\n");
            break; // Terminate loop early
        }
        printf("Count is %zu\n", count);
    }

    // 3. do-while loop
    printf("\nDo-while loop:\n");
    bool execute_once = false;
    do {
        printf("This executes at least once, even if condition is false.\n");
    } while (execute_once);

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_
The first loop prints Iterations 0, 1, 2. The while loop counts down, skips printing "3", prints "2", "1", and breaks exactly when count reaches 0 before the final print. The do-while loop prints its message exactly once.

_What goes wrong if..._ If you place a `continue` inside a `while` loop before the condition variable is updated, it creates an infinite loop because the update step is skipped forever.

**Recap: Iterations**

- `for` loops are best for domain iterations and allow block-scoped loop variables.
- `while` and `do-while` are used for condition-based looping.
- `break` exits a loop entirely, while `continue` jumps to the next iteration cycle.

---

## 3.3. Multiple selection

### The `switch` Statement

**What it is:** The `switch` statement is a control flow construct that allows a program to test an integer or character expression against a list of constant values (`case` labels), providing a cleaner alternative to deep, complex `if-else if` cascades.

**Why it matters / how it works:** When a variable can take on many specific values, `switch` makes the code significantly more readable. It works by evaluating the controlling expression once, then jumping execution directly to the matching `case` label. If no match is found, it jumps to the `default` label (if provided).

**Key details:**

- Syntax: `switch (expression) { case constant: ... break; default: ... }`.
- The `case` and `default` keywords act merely as _jump targets_.
- Execution _falls through_ sequentially into the next case unless explicitly stopped via a `break` statement. This can be used intentionally to group cases.

> Takeaway 3.3 #1: _case values must be integer constant expressions._

**Explanation:** You cannot use variables, arrays, or floating-point numbers in a `case` label. The compiler needs to know the exact integer value at compile time to build an efficient jump table. Characters like `'m'` or `'r'` are valid because they evaluate to integer constant expressions (ASCII values) in C.

> Takeaway 3.3 #2: _case values must be unique for each switch statement._

**Explanation:** A `switch` statement cannot have duplicate `case` labels. If it did, the compiler would not know which jump target to choose when the evaluated expression matches that value.

> Takeaway 3.3 #3: _case labels must not jump beyond a variable definition._

**Explanation:** Because a `switch` is fundamentally a jump, jumping past a variable declaration that involves initialization (e.g., `case 1: int x = 5; case 2: printf("%d", x);`) means the variable enters scope, but its initialization is bypassed. This leads to undefined behavior or compile-time errors. In C23, avoiding this requires enclosing `case` specific variables in their own compound blocks `{}`.

**Pitfalls:**

- **Fall-through:** Forgetting a `break;` statement causes the program to continue executing the code in the subsequent `case` blocks, regardless of whether their labels match. Sometimes this is intended, but often it is a severe bug.
- **Missing semicolons prior to C23:** Prior to C23, labels couldn't exist without a trailing statement. If a `switch` case ended with an empty label (e.g., `case 0: }`), a dummy semicolon was required.

**Connections:** The concept of `switch` labels acting as jump targets prepares the reader for Chapter 19's detailed look at the `goto` statement, as both fundamentally rely on labeling points in the codebase.

#### Code Demonstration: Multiple selection

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    char arg = 'j';
    int count = 3;

    // Demonstrating standard switch usage
    printf("Evaluating Corvid:\n");
    switch (arg) {
        case 'm': // Takeaway 3.3 #1: Character literals are integer constants
            puts("this is a magpie");
            break;
        case 'r':
            puts("this is a raven");
            break;
        case 'j':
            puts("this is a jay");
            break; // Jumps out of switch here
        case 'c':
            puts("this is a chough");
            break;
        default:
            puts("this is an unknown corvid");
            // No break needed at the end, but good practice
    }

    // Demonstrating intentional fall-through and Takeaway 3.3 #3 (scoping constraint)
    printf("\nTriangle Drawing:\n");
    switch (count) {
        default: puts("++++"); // Fall-through intentional
        case 3:  puts("+++");
        case 2:  puts("++");
        case 1:  puts("+");
        case 0:
            // In C23, this is fine. Prior to C23, this would need a `;` to not be an empty label.
    }

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ The first `switch` outputs "this is a jay". The second `switch`, due to intentional fall-through (no `break` statements), jumps to `case 3:` and outputs `+++`, `++`, `+`.

_What goes wrong if..._ If you write `case var:` where `var` is a variable, the code will fail to compile (violating Takeaway 3.3 #1). If you forget the `break` under `case 'j'`, it would print "this is a jay" _and_ "this is a chough".

**Recap: Multiple selection**

- `switch` statements evaluate an integer expression and jump to a matching `case` label.
- Labels must be unique integer constants.
- `break` is necessary to prevent execution from "falling through" into subsequent cases.

---

## Summary

Chapter 3 lays out the foundational constructs for directing program execution in C. Through conditional statements (`if`), iterative structures (`for`, `while`, `do-while`), and multiple selection jumps (`switch`), the programmer can implement complex abstract state machines. Crucially, C leans on mathematical simplicity: `0` is strictly logical false, and any non-zero scalar equates to logical true. Understanding that control constructs like `switch` and `for` rely on simple internal jumps and truth evaluations helps avoid common pitfalls, such as infinite loops, accidental fall-throughs, or redundant Boolean comparisons, resulting in cleaner, idiomatic C code.

---

## Self-Check Questions

1.  **What numeric value represents "false" in a C condition?**
    The value `0`.

2.  **How are non-zero scalar values interpreted in an `if` condition?**
    Any value different from `0` is treated as logical `true`.

3.  **Why is it considered bad style to write `if (pointer != 0)` or `if (flag == true)`?**
    Because all scalars have an inherent truth value; writing `if (pointer)` or `if (flag)` is more concise, readable, and avoids redundancy.

4.  **What are the three parts of a `for` loop header?**
    1. Variable declaration/initialization.
    2. Loop condition check.
    3. Update expression.

5.  **What is the difference between a `while` loop and a `do-while` loop?**
    `while` checks its condition _before_ the first execution, while `do-while` executes its block once _before_ checking the condition.

6.  **What happens when a `break` statement is encountered inside a loop?**
    Execution immediately exits the loop, bypassing any remaining statements in the block and ignoring the loop condition.

7.  **What is the role of a `continue` statement inside a `for` loop?**
    It skips the rest of the current iteration's block and jumps straight to the loop's update expression, then re-evaluates the condition.

8.  **Can you use a string literal like `"test"` as a `case` value in a `switch` statement?**
    No, `case` values must be integer constant expressions (like `1` or `'a'`).

9.  **What is "fall-through" in a `switch` statement?**
    When a `case` block does not end with a `break;`, execution continues straight into the code of the next `case` label.

10. **Why must you be careful defining variables inside `switch` cases?**
    Because a `case` is a jump target; if you jump past the definition and initialization of a local variable to another case, that variable will be uninitialized, leading to undefined behavior.
