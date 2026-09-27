+++
title = "Modern C: Everything is about control"
description = ""
date = 2026-09-27

[taxonomies]
tags = ["modernc"]

# [extra]
# math = false
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


### Summary Table of Control Constructs in C

| Construct | Type | Evaluation Timing | Best Used For |
| :--- | :--- | :--- | :--- |
| **`if` / `if-else`** | Selection | Evaluated before entering branch | Binary decision paths based on scalar conditions |
| **`for`** | Iteration | Pre-tested before each loop pass | Counting/bounded domain iterations with defined loop variables |
| **`while`** | Iteration | Pre-tested before each loop pass | Loops running 0 or more times based on a state condition |
| **`do-while`** | Iteration | Post-tested after each loop pass | Loops that must execute at least once (e.g., user input/convergence) |
| **`switch`** | Selection | Evaluated once at entry | Multi-way branching based on fixed integer/character constant values |
