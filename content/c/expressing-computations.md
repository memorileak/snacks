+++
title = "Modern C: Expressing computations"
description = "An overview of C23 operators, arithmetic, assignments, boolean logic, conditional expressions, side effects, and evaluation order for writing clear and predictable computations."
date = 2026-09-27T17:14:46Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Here is a detailed breakdown of the key takeaways and fundamental concepts from **Chapter 4: Expressing computations** in Jens Gustedt's *Modern C: A Guide to the C23 Standard*.


### 1. Operands and Operators (Section 4.1)
* **Value vs. Object distinction:** In C computations, **operators** (e.g., `+`, `!=`) act upon **operands** (values or objects). A value is a mathematical quantity, while an object is a named memory location holding a value.
* **Types of Operators:** The C standard distinguishes three operator categories: **value operators** (compute new values), **object operators** (modify or inspect stored objects), and **type operators** (`sizeof`, `alignof`, `offsetof`).
* **Bounded Non-Negative Integers:** The fundamental unsigned type `size_t` represents non-negative quantities up to a platform-defined upper bound `SIZE_MAX`.

#### Example

In C, **operators** (e.g., `+`, `=`) act upon **operands** (values or objects). **Value operators** compute new mathematical values without altering their inputs, while **object operators** modify or inspect named objects in memory. Unsigned sizes are represented by `size_t`, which is bounded in the range `[0, SIZE_MAX]`.

```c
#include <stdio.h>
#include <stddef.h>
#include <stdint.h> // Provides SIZE_MAX

void demo_operands_and_operators(void) {
  // 'a' and 'b' are addressable objects (lvalues) holding values
  size_t a = 42; 
  size_t b = 10;
    
  // '+' is a value operator applied to operands 'a' and 'b' (yields value 52)
  // '=' is an object operator storing that value into object 'c'
  size_t c = a + b; 
    
  // Printing size_t using the standard '%zu' modifier
  printf("a + b = %zu\n", c);
    
  // size_t values are strictly non-negative and bounded by SIZE_MAX
  printf("Platform SIZE_MAX bound = %zu\n", SIZE_MAX);
}
```


### 2. Arithmetic and Modular Rules (Section 4.2)
* **Unsigned Arithmetic is Always Well-Defined:** Unsigned integer operations (`+`, `-`, `*`) never generate undefined behavior on their own. As long as the mathematical result fits in `[0, SIZE_MAX]`, the result is exact.
* **Modular Wrap-Around:** Arithmetic on `size_t` implicitly operates **modulo $\text{SIZE\\_MAX} + 1$**. When an operation overflows, it safely wraps around (e.g., `SIZE_MAX + 1` wraps to `0`, and `0 - 1` evaluates to `SIZE_MAX`).
* **Integer Division and Remainder:**
  * For unsigned integers, division `/` (quotient) and remainder `%` satisfy: $\text{a} == (\text{a} / \text{b}) * \text{b} + (\text{a} \\% \text{b})$.
  * Both `/` and `%` can **never overflow**, and their outputs are always smaller than or equal to the inputs.
  * **Division by Zero is strictly forbidden** and results in runtime failure.

#### Example

Unsigned arithmetic on `size_t` is strictly well-defined and never causes undefined overflow. When an unsigned computation exceeds `SIZE_MAX`, it wraps around implicitly **modulo $\text{SIZE\\_MAX} + 1$**. Division (`/`) and remainder (`%`) satisfy the exact mathematical identity $a == (a / b) \\cdot b + (a \\% b)$, provided $b \\neq 0$.

```c
#include <stdio.h>
#include <stddef.h>
#include <stdint.h>

void demo_arithmetic_and_modulo(void) {
    size_t a = 25;
    size_t b = 7;

    // Division & Remainder Identity: a == (a / b) * b + (a % b)
    size_t quotient  = a / b;  // 3 (integer division)
    size_t remainder = a % b;  // 4 (remainder)
    size_t restored  = quotient * b + remainder; // 25

    printf("%zu == (%zu / %zu) * %zu + (%zu %% %zu) -> %zu\n", 
           a, a, b, b, a, b, restored);

    // Unsigned Wrap-Around (Modulo Arithmetic):
    size_t max_val = SIZE_MAX;
    size_t overflow_val  = max_val + 1; // Wraps around safely to 0
    size_t underflow_val = 0 - 1;       // Wraps around safely to SIZE_MAX

    printf("SIZE_MAX + 1 wraps to: %zu\n", overflow_val);
    printf("0 - 1        wraps to: %zu\n", underflow_val);
    
    // NOTE: Unsigned / and % can NEVER overflow, but division by 0 is forbidden!
}
```


### 3. Operators That Modify Objects & Side Effects (Section 4.3)
* **Assignment Operators:** In assignments (`a = 42` or `+=`, `-=`, `*=`, `/=`, `%=`), the left side must be an addressable object (lvalue), and the right side is a value (rvalue).
* **Prefix vs. Postfix Modifications:**
  * **Prefix** (`++a`, `--a`) modifies the object first and yields the **new** value to the surrounding expression.
  * **Postfix** (`a++`, `a--`) modifies the object but yields the **original** value prior to modification.
* **Golden Rules for Clean Code:**
  * **"Side effects in value expressions are evil."** Avoid embedding variable modifications inside complex arithmetic.
  * **"Never modify more than one object in a single statement."** Combining multiple object modifications in one statement obscures control flow and causes bugs.

#### Example

Assignments and modifications create **side effects** (changes stored in objects). **Prefix** operations (`++a`) increment the object first and return the *new* value; **postfix** operations (`a++`) return the *original* value before incrementing. Gustedt lays out two primary rules: avoid side effects in value expressions and never modify more than one object per statement.

```c
#include <stdio.h>

void demo_modifying_objects(void) {
    int x = 10;

    // Prefix increment (++x): modifies object first, yields NEW value (11)
    int pre_res = ++x; 
    printf("Prefix ++x: pre_res = %d, x = %d\n", pre_res, x); // 11, 11

    // Postfix increment (x++): modifies object, but yields ORIGINAL value (11)
    int post_res = x++; 
    printf("Postfix x++: post_res = %d, x = %d\n", post_res, x); // 11, 12

    // GOOD PRACTICE: Keep object modifications isolated.
    // BAD PRACTICE (Avoid!): a = b = c += ++d; 
    // Chaining multiple object modifications in one statement introduces subtle bugs.
}
```


### 4. Boolean Context: Comparisons & Logic (Section 4.4)
* **Truth Values as Arithmetic Integers:**
  * Comparison (`==`, `!=`, `<`, `>`, `<=`, `>=`) and logic (`!`, `&&`, `||`) operators strictly return `false` (`0`) or `true` (`1`).
  * Because `false` and `true` are numeric (`0` and `1`), these boolean results can be directly used as arithmetic terms or array indices.
* **Short-Circuit Evaluation:**
  * Logical AND (`&&`) and logical OR (`||`) evaluate left-to-right and **skip evaluation of the second operand** if the first operand fully determines the result.
  * For instance, `if (b != 0 && (a / b > 1))` safely avoids division by zero because `(a / b)` is evaluated only when `b != 0` is true.

#### Example

Comparison (`==`, `!=`, `<`, `>`) and logic (`!`, `&&`, `||`) operators strictly return `0` (`false`) or `1` (`true`). Because boolean results are integer numbers, they can be directly used in arithmetic calculations or as array indices. Logical AND (`&&`) and OR (`||`) utilize **short-circuit evaluation**—they evaluate operands left-to-right and skip the second operand if the outcome is already known.

```c
#include <stdio.h>
#include <stdbool.h>

void demo_boolean_and_short_circuit(void) {
    size_t a = 15;
    size_t b = 10;

    // Comparison operators return 1 (true) or 0 (false)
    bool is_greater = (a > b); 
    printf("Is %zu > %zu? %d\n", a, b, is_greater);

    // Using boolean truth values as array indices directly:
    // sign_counts stores numbers >= 1.0; sign_counts stores numbers < 1.0
    double sample = 0.5;
    size_t sign_counts = {0, 0};
    
    sign_counts[(sample < 1.0)] += 1; // (sample < 1.0) yields 1 (true), incrementing index 1
    printf("Count of numbers < 1.0: %zu\n", sign_counts);

    // Short-Circuit Evaluation:
    // If 'divisor == 0', the left side evaluates to false, so the right side 
    // '(dividend / divisor > 1)' IS NEVER EVALUATED, avoiding division by zero!
    size_t divisor = 0;
    size_t dividend = 100;

    if (divisor != 0 && ((dividend / divisor) > 1)) {
        printf("Division result is greater than 1\n");
    } else {
        printf("Safely skipped division by zero via short-circuit evaluation!\n");
    }
}
```


### 5. The Ternary Operator (Section 4.5)
* **Conditional Expressions:** The ternary operator `cond ? A : B` evaluates `cond` first, then evaluates **only one** of the two branches (`A` or `B`) based on whether `cond` is true or false.
* Unlike an `if` statement, the ternary operator is an expression that yields a value, making it suitable for clean return statements or inline initializations (e.g., `return (a < b) ? a : b;`).

#### Example

The **ternary operator** (`cond ? A : B`) evaluates a condition first, then evaluates **only one** of its two branches based on the truth value. Unlike an `if` block, it is an expression that returns a concrete value, making it ideal for pure functions or concise inline choices.

```c
#include <stdio.h>
#include <stddef.h>

// Pure function returning the smaller of two size_t values
size_t size_min(size_t a, size_t b) {
  return (a < b) ? a : b; // Evaluates 'a' if a < b is true; else 'b'
}

void demo_ternary_operator(void) {
  size_t x = 45;
  size_t y = 30;

  size_t min_val = size_min(x, y);
  printf("The minimum of %zu and %zu is %zu\n", x, y, min_val);

  // Branching protection: Only the required branch is executed!
  int count = 0;
  bool flag = true;
    
  // Because 'flag' is true, ++count runs, and --count is completely ignored.
  int output = flag ? ++count : --count;
  printf("Output: %d, Count: %d\n", output, count); // Output = 1, Count = 1
}
```


### 6. Evaluation Order and Sequencing (Section 4.6)
* **Sequenced vs. Unsequenced Operators:**
  * Only four operators strictly sequence their operands from left to right: `&&`, `||`, `?:`, and the comma operator `,`.
  * **Most operators do not sequence their operands.** In `f(a) + g(b)`, the compiler may evaluate `f(a)` or `g(b)` in any arbitrary order.
* **Function Arguments are Unsequenced:** Function argument lists do not guarantee an evaluation order (e.g., in `printf("%g %g", f(a), f(b))`, either function may run first).
* **Banning Side Effects in Function Expressions:** Because operand evaluation order is non-deterministic across compilers, function calls placed within expressions **must never rely on side effects**.
* **The Comma Operator Trap:** Avoid using the comma operator `,` in value expressions. For example, `A[i, j]` is not a 2D matrix index in C; it evaluates `i`, discards it, and indexes `A[j]`.

#### Example

Most C operators **do not sequence their operands**. In expressions like `f(a) + g(b)` or `printf("%d %d", f(a), f(b))`, the compiler may run `f(a)` or `f(b)` in any order. Consequently, function calls embedded within value expressions **must never have side effects**. Additionally, using the comma operator inside expressions (like array indexing) is a common beginner trap.

```c
#include <stdio.h>

// Pure function without side effects (safe for expressions)
int pure_add(int x, int y) {
    return x + y;
}

void demo_evaluation_order(void) {
    int a = 5;
    int b = 10;
    
    // Pure function calls in value expressions are completely safe from ordering bugs
    int sum = pure_add(a, b);
    printf("Sum: %d\n", sum);

    // THE COMMA OPERATOR TRAP:
    // The comma operator evaluates left-to-right and returns the rightmost operand.
    // Writing arr[i, j] in C does NOT access a 2D matrix; it evaluates 'i', 
    // discards it, and accesses arr[j]!
    int arr = {10, 20, 30, 40, 50};
    int value = arr[(1, 3)]; // Evaluates 1, discards it, yields 3 -> evaluates arr (40)
    
    printf("arr[(1, 3)] yields arr = %d (Never use the comma operator for indexing!)\n", value);
}
```
