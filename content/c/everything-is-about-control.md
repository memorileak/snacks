+++
title = "Modern C: Everything is about control"
description = "An introduction to C23 control flow, covering conditional execution, scalar truth values, iteration, loop control, and multi-way selection with switch statements."
date = 2026-09-27

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Chapter 3 of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, titled **"Everything is about control,"** shifts focus from basic syntax and declarations to managing the execution path (control flow) of a program. While function calls provide unconditional transfers of control, C provides **five primary conditional control statements**: `if`, `for`, `do`, `while`, and `switch`.

Here is a detailed breakdown of the most important concepts, structural rules, and takeaways from this chapter:


### 1. Conditional Execution: `if` and `if-else` (Section 3.1)
The `if` construct executes a secondary block of code based on whether a controlling expression evaluates to true or false. Adding an `else` branch creates a selection statement that directs execution down one of two distinct code paths.

* **Numerical Truth Rules:**
  * **Takeaway 3.1 #1:** **The value `0` represents logical `false`**.
  * **Takeaway 3.1 #2:** **Any value different from `0` represents logical `true`**.
* **Scalar Truth Values:**
  * **Takeaway 3.1 #4:** **All scalars have a truth value**. Numerical types (`size_t`, `int`, `double`, `bool`) and pointer types are scalar types, meaning any scalar variable can be passed directly as a controlling condition.
* **Clean Boolean Testing:**
  * **Takeaway 3.1 #3:** **Don't compare to `0`, `false`, or `true`**. Writing `if (b == true)` or `if (x != 0)` is redundant and clutters code. Instead, test boolean or scalar expressions directly (e.g., `if (b)` or `if (x)` or `if (!x)`).

#### Example

In C, controlling conditions evaluate scalar expressions directly. 
* **Takeaway 3.1 #1 & #2:** `0` represents logical `false`; any non-zero value represents logical `true`.
* **Takeaway 3.1 #4:** All scalars (integers, floats, pointers, booleans) have an inherent truth value.
* **Takeaway 3.1 #3:** Don't compare directly to `0`, `false`, or `true` (e.g., write `if (b)` instead of `if (b == true)`).

```c
#include <stdio.h>
#include <stdbool.h>
#include <stddef.h>

void demo_conditional_execution(void) {
    bool flag = true;
    size_t count = 42;
    double const* ptr = nullptr; // C23 null pointer constant

    // GOOD PRACTICE (Takeaway 3.1 #3): Test boolean expressions directly
    if (flag) { 
        printf("Flag is set (clean boolean test)\n");
    }

    // BAD PRACTICE (Avoid!): if (flag == true) or if ((flag != false) == true)
    // Redundant comparisons clutter code and add visual noise.

    // SCALAR TRUTH VALUES (Takeaway 3.1 #1, #2, #4):
    // Integer count (42 != 0) evaluates to true directly
    if (count) { 
        printf("Count is non-zero (%zu) -> evaluates to true\n", count);
    }

    // Testing logical negation (!ptr evaluates to true when ptr is nullptr)
    if (!ptr) { 
        printf("Pointer is null -> !ptr evaluates to true\n");
    }

    // if-else selection statement branching
    size_t threshold = 50;
    if (count > threshold) {
        printf("Count exceeds threshold\n");
    } else {
        printf("Count (%zu) is within threshold (%zu)\n", count, threshold);
    }
}
```

### 2. Iterations: `for`, `while`, and `do-while` (Section 3.2)
C provides three construct types to repeat execution over a domain or until a condition is met:

* **Domain Iteration (`for`):**
  * Structure: `for (clause1; condition2; expression3) secondary-block`.
  * `clause1` initializes or defines the loop variable (scope is narrowly restricted to the loop).
  * `condition2` tests if iteration should continue before each pass.
  * `expression3` updates the loop variable at the end of each iteration.
  * `for` is the preferred tool for domain iterations where bounds are known.
* **Pre-Test Condition Iteration (`while`):**
  * Structure: `while (condition) secondary-block`.
  * Evaluates the condition before running the body. If the condition starts as `false`, the secondary block executes **zero times**.
* **Post-Test Condition Iteration (`do-while`):**
  * Structure: `do secondary-block while (condition);`.
  * Runs the secondary block unconditionally **at least once** before evaluating the condition. Syntactically, it requires a terminating semicolon `;` after the condition.
* **Loop Control Statements:**
  * **`break`:** Immediately terminates the loop, skipping the rest of the body and the condition check.
  * **`continue`:** Skips the remainder of the current loop body and jumps directly to the condition re-evaluation (or loop variable update in `for`).
  * **Infinite Loop Idiom:** `for (;;)` is equivalent to `while (true)`.

#### Example

C provides three primary iteration constructs:
1. **`for`**: Preferred for domain iterations where bounds or iteration counts are known.
2. **`while`**: Pre-tested loop; evaluates the condition *before* executing the body (may execute 0 times).
3. **`do-while`**: Post-tested loop; executes the body *at least once* before testing.

```c
#include <stdio.h>
#include <stddef.h>
#include <stdbool.h>

void demo_iterations(void) {
    // 1. DOMAIN ITERATION (for loop)
    // Loop variable 'i' is declared inside the initial clause, restricting its scope to the loop.
    // 'i' counts down from 5 to 1. The condition 'i' implicitly checks (i != 0).
    printf("1. Countdown using 'for': ");
    for (size_t i = 5; i; --i) {
        printf("%zu ", i);
    }
    printf("\n");

    // 2. PRE-TEST CONDITION ITERATION (while loop)
    // Checks condition before each iteration. Runs 0 or more times.
    size_t energy = 3;
    printf("2. Energy drainage using 'while': ");
    while (energy) { // Evaluates energy != 0
        printf("[%zu remaining] ", energy);
        --energy;
    }
    printf("\n");

    // 3. POST-TEST CONDITION ITERATION (do-while loop)
    // Runs secondary block AT LEAST ONCE before evaluating condition.
    // Syntactically requires a terminating semicolon ';' after while(cond);.
    size_t attempts = 0;
    printf("3. Process loop using 'do-while': ");
    do {
        ++attempts;
        printf("(Attempt %zu executed) ", attempts);
    } while (attempts < 1); // Condition checked after body execution
    printf("\n");

    // 4. BREAK AND CONTINUE CONTROL
    printf("4. Loop with break and continue: ");
    for (size_t k = 1; k <= 10; ++k) {
        if (k % 2 == 0) {
            continue; // Skip even numbers, jumping straight to ++k update
        }
        if (k > 7) {
            break; // Terminate loop completely when k exceeds 7
        }
        printf("%zu ", k); // Prints 1 3 5 7
    }
    printf("\n");
}
```

### 3. Multiple Selection: `switch` (Section 3.3)
The `switch` statement selects one of several code execution paths based on an integer value, replacing tedious cascades of `if-else` blocks.

* **Jump Targets and Fall-Through:**
  * `case` labels and `default` act as jump targets inside the `switch` block.
  * When a matching `case` is found, control jumps there. Execution continues sequentially down into subsequent `case` blocks (**fall-through**) unless explicitly stopped by a **`break`** statement.
  * The `default` label executes if no `case` matches the controlling expression.
* **Strict Constraints on `case` Labels:**
  * **Takeaway 3.3 #1:** **`case` values must be integer constant expressions**. Values must be fixed at compile time (e.g., literal numbers like `4` or character constants like `'m'`); variables cannot be used as `case` values.
  * **Takeaway 3.3 #2:** **`case` values must be unique** within a single `switch` statement.
  * **Takeaway 3.3 #3:** **`case` labels must not jump beyond a variable definition**.

#### Example

The `switch` statement handles multi-way branching based on integer expressions. `case` labels act as **jump targets**; execution falls through into subsequent `case` blocks unless stopped by a `break` statement.

* **Takeaway 3.3 #1:** `case` values must be integer constant expressions (ICE) known at compile time.
* **Takeaway 3.3 #2:** `case` values must be unique within a single `switch` statement.
* **Takeaway 3.3 #3:** `case` labels must not jump beyond a variable definition.

```c
#include <stdio.h>

void demo_multiple_selection(char corvid_code) {
    // Controlling expression 'corvid_code' is evaluated once at entry
    switch (corvid_code) {
        // Takeaway 3.3 #1: 'm', 'r', 'j', 'c' are character constants (integer values)
        // Takeaway 3.3 #2: Each case value is unique
        case 'm':
            puts("Selected: Magpie");
            break; // Prevents fall-through, exiting the switch

        case 'r':
            puts("Selected: Raven");
            break;

        case 'j':
        case 'J': // Intentional grouping: both 'j' and 'J' fall through to the same action
            puts("Selected: Jay");
            break;

        case 'c': {
            // Takeaway 3.3 #3: Enclose block in braces {} when defining local variables 
            // to ensure case labels do not jump over variable initializations!
            int chough_id = 42; 
            printf("Selected: Chough (ID: %d)\n", chough_id);
            break;
        }

        default:
            puts("Selected: Unknown corvid bird");
            break;
    }
}
```

### Summary Table of Control Constructs in C

| Construct | Type | Evaluation Timing | Best Used For |
| :--- | :--- | :--- | :--- |
| **`if` / `if-else`** | Selection | Evaluated before entering branch | Binary decision paths based on scalar conditions |
| **`for`** | Iteration | Pre-tested before each loop pass | Counting/bounded domain iterations with defined loop variables |
| **`while`** | Iteration | Pre-tested before each loop pass | Loops running 0 or more times based on a state condition |
| **`do-while`** | Iteration | Post-tested after each loop pass | Loops that must execute at least once (e.g., user input/convergence) |
| **`switch`** | Selection | Evaluated once at entry | Multi-way branching based on fixed integer/character constant values |
