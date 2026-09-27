+++
title = "Modern C: Expressing computations"
description = "An overview of C23 operators, arithmetic, assignments, boolean logic, conditional expressions, side effects, and evaluation order for writing clear and predictable computations."
date = 2026-09-27

[taxonomies]
tags = ["modernc"]

# [extra]
# math = false
# cover.image = "images/cover.png"
+++

Here is a detailed breakdown of the key takeaways and fundamental concepts from **Chapter 4: Expressing computations** in Jens Gustedt's *Modern C: A Guide to the C23 Standard*.


### 1. Operands and Operators (Section 4.1)
* **Value vs. Object distinction:** In C computations, **operators** (e.g., `+`, `!=`) act upon **operands** (values or objects). A value is a mathematical quantity, while an object is a named memory location holding a value.
* **Types of Operators:** The C standard distinguishes three operator categories: **value operators** (compute new values), **object operators** (modify or inspect stored objects), and **type operators** (`sizeof`, `alignof`, `offsetof`).
* **Bounded Non-Negative Integers:** The fundamental unsigned type `size_t` represents non-negative quantities up to a platform-defined upper bound `SIZE_MAX`.


### 2. Arithmetic and Modular Rules (Section 4.2)
* **Unsigned Arithmetic is Always Well-Defined:** Unsigned integer operations (`+`, `-`, `*`) never generate undefined behavior on their own. As long as the mathematical result fits in `[0, SIZE_MAX]`, the result is exact.
* **Modular Wrap-Around:** Arithmetic on `size_t` implicitly operates **modulo \\(\text{SIZE\_MAX} + 1\\)**. When an operation overflows, it safely wraps around (e.g., `SIZE_MAX + 1` wraps to `0`, and `0 - 1` evaluates to `SIZE_MAX`).
* **Integer Division and Remainder:**
  * For unsigned integers, division `/` (quotient) and remainder `%` satisfy: \\(\text{a} == (\text{a} / \text{b}) * \text{b} + (\text{a} \% \text{b})\\).
  * Both `/` and `%` can **never overflow**, and their outputs are always smaller than or equal to the inputs.
  * **Division by Zero is strictly forbidden** and results in runtime failure.


### 3. Operators That Modify Objects & Side Effects (Section 4.3)
* **Assignment Operators:** In assignments (`a = 42` or `+=`, `-=`, `*=`, `/=`, `%=`), the left side must be an addressable object (lvalue), and the right side is a value (rvalue).
* **Prefix vs. Postfix Modifications:**
  * **Prefix** (`++a`, `--a`) modifies the object first and yields the **new** value to the surrounding expression.
  * **Postfix** (`a++`, `a--`) modifies the object but yields the **original** value prior to modification.
* **Golden Rules for Clean Code:**
  * **"Side effects in value expressions are evil."** Avoid embedding variable modifications inside complex arithmetic.
  * **"Never modify more than one object in a single statement."** Combining multiple object modifications in one statement obscures control flow and causes bugs.


### 4. Boolean Context: Comparisons & Logic (Section 4.4)
* **Truth Values as Arithmetic Integers:**
  * Comparison (`==`, `!=`, `<`, `>`, `<=`, `>=`) and logic (`!`, `&&`, `||`) operators strictly return `false` (`0`) or `true` (`1`).
  * Because `false` and `true` are numeric (`0` and `1`), these boolean results can be directly used as arithmetic terms or array indices.
* **Short-Circuit Evaluation:**
  * Logical AND (`&&`) and logical OR (`||`) evaluate left-to-right and **skip evaluation of the second operand** if the first operand fully determines the result.
  * For instance, `if (b != 0 && (a / b > 1))` safely avoids division by zero because `(a / b)` is evaluated only when `b != 0` is true.


### 5. The Ternary Operator (Section 4.5)
* **Conditional Expressions:** The ternary operator `cond ? A : B` evaluates `cond` first, then evaluates **only one** of the two branches (`A` or `B`) based on whether `cond` is true or false.
* Unlike an `if` statement, the ternary operator is an expression that yields a value, making it suitable for clean return statements or inline initializations (e.g., `return (a < b) ? a : b;`).


### 6. Evaluation Order and Sequencing (Section 4.6)
* **Sequenced vs. Unsequenced Operators:**
  * Only four operators strictly sequence their operands from left to right: `&&`, `||`, `?:`, and the comma operator `,`.
  * **Most operators do not sequence their operands.** In `f(a) + g(b)`, the compiler may evaluate `f(a)` or `g(b)` in any arbitrary order.
* **Function Arguments are Unsequenced:** Function argument lists do not guarantee an evaluation order (e.g., in `printf("%g %g", f(a), f(b))`, either function may run first).
* **Banning Side Effects in Function Expressions:** Because operand evaluation order is non-deterministic across compilers, function calls placed within expressions **must never rely on side effects**.
* **The Comma Operator Trap:** Avoid using the comma operator `,` in value expressions. For example, `A[i, j]` is not a 2D matrix index in C; it evaluates `i`, discards it, and indexes `A[j]`.