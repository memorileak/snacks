+++
title = "Modern C: Expressing computations"
description = "An overview of C23 operators, arithmetic, assignments, boolean logic, conditional expressions, side effects, and evaluation order for writing clear and predictable computations."
date = 2026-09-27T17:14:46Z

[taxonomies]
tags = ["modernc", "basic"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

# Chapter 4: Expressing Computations

## 4.1. Operands and operators

### Expressions, Operands, and Operators

**What it is:** Expressions are code snippets that compute a value based on other values. Computations are carried out using operators (symbols or combinations of symbols like `+` or `!=`) applied to operands (the values or objects being operated on). C categorizes operators based on what they act upon: values, objects, or types.

**Why it matters / how it works:** Understanding the distinction between operands and operators is fundamental to mastering C syntax. The language provides a vast array of operators to manipulate data. Notably, C programs frequently use the `size_t` type (an unsigned integer type representing non-negative integers up to `SIZE_MAX`) to represent sizes, cardinalities, or domain dimensions. The compiler uses the types of the operands to determine the precise mathematical or bitwise instruction to execute on the CPU.

**Key details:**

- **Unsigned integer types:** Types representing non-negative numbers starting at $0$.
- **Value operators:** Operations on pure values (e.g., arithmetic `+`, comparison `==`).
- **Object operators:** Operations that modify an object's state (e.g., assignment `=`, increment `++`).
- **Type operators:** Operations returning constants based on a type (e.g., `sizeof`, `alignof`).

**Pitfalls:**

- Confusing the context of an operator (e.g., using an object operator where a value is expected, leading to unintended side effects).
- Assuming mathematical sets (like $\mathbb{N}$ for natural numbers) map perfectly to C types without limits. C's `size_t` is bounded by `SIZE_MAX`.

**Connections:** This builds upon the variable declarations introduced in Chapter 2. Later chapters will heavily rely on understanding how types are promoted and converted when operators combine different operands (Chapter 5).

**Recap:**

- Expressions compute values using operands and operators.
- Operators are divided into those that operate on values, objects, or types.
- `size_t` is the standard unsigned integer type for representing sizes.

```c
#include <stdio.h>
#include <stddef.h>
#include <stdlib.h>

int main(void) {
    // 'a' and 'b' are operands. '=' and '+' are operators.
    size_t a = 5;
    size_t b = 10;

    // The expression (a + b) combines operands 'a' and 'b' using the value operator '+'
    // The object operator '=' assigns the resulting value to 'c'
    size_t c = a + b;

    printf("Result: %zu\n", c);
    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ Prints "Result: 15".
_What goes wrong if:_ You use an uninitialized variable as an operand; the expression will evaluate using garbage data, resulting in undefined behavior.

---

## 4.2. Arithmetic

### 4.2.1. +, -, and *

**What it is:** The arithmetic operators `+` (addition), `-` (subtraction/negation), and `*` (multiplication) perform standard mathematical computations. C provides both binary forms (e.g., `a - b`) and unary forms (e.g., `-b`, `+a`).

**Why it matters / how it works:** These operators are the workhorses of algorithmic computation. In C, arithmetic on unsigned types is guaranteed to be well-behaved and predictable. The standard dictates that operations on `size_t` will always yield a valid mathematical result if that result falls within the bounds of the type.

**Key details:**

- Subexpressions can be grouped using parentheses `()` to enforce evaluation order.
- Unary `-b` provides the value that, when added to `b`, results in $0$.

> **Takeaway 4.2.1 #1** _Unsigned arithmetic is always well defined._

**Explanation:** Unlike signed arithmetic (which can trigger undefined behavior on overflow), unsigned arithmetic operations are strictly defined by the C standard to operate within their modulo range. You will never encounter a hard crash or undefined behavior simply by adding or multiplying unsigned integers.

> **Takeaway 4.2.1 #2** _The operations +, -, and * on `size_t` provide the mathematically correct result if it is representable as a `size_t`._

**Explanation:** As long as the true mathematical result of your calculation lies within the closed interval $[0, \text{SIZE\_MAX}]$, the C operator will yield that exact correct value.

**Pitfalls:**

- **Arithmetic overflow:** Occurs when the mathematical result exceeds `SIZE_MAX`. C handles this gracefully for unsigned types by wrapping around, but this might represent a logical bug in your application if you did not anticipate numbers that large.

**Connections:** This connects directly to the concept of modulo arithmetic discussed in the next subsection, and to the binary representation of integers in Chapter 5.

**Recap:**

- Binary and unary `+`, `-`, and `*` work largely as expected.
- Unsigned arithmetic is completely safe from undefined behavior.
- Results are mathematically exact if they fit within the bounds of the type.

```c
#include <stdio.h>
#include <stddef.h>
#include <stdlib.h>

int main(void) {
    size_t a = 45;
    size_t b = 7;

    // Grouping with parentheses enforces order
    size_t c = (a - b) * 2; // (45 - 7) * 2 = 76

    // Standard precedence rules apply: multiplication before subtraction
    size_t d = a - b * 2;   // 45 - (7 * 2) = 31

    // Unary operators
    size_t e = (+a + -b) * 2; // Wraps around temporarily, but math remains correct

    printf("c = %zu, d = %zu\n", c, d);
    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ Prints "c = 76, d = 31".

### 4.2.2. Division and remainder

**What it is:** The `/` operator performs integer division (evaluating how many times the right operand fits into the left), and the `%` operator calculates the remainder of that division.

**Why it matters / how it works:** Division and remainder are fundamentally paired operations. They are crucial for tasks ranging from algorithmic partitioning to implementing cyclic data structures (like a 24-hour clock). The wrap-around behavior of unsigned types is defined exactly in terms of the modulo operator `%`.

**Key details:**

- `/` and `%` come in pairs.
- Division by zero is strictly forbidden.
- Values outside the unsigned type's range "wrap around" using modulo arithmetic.

> **Takeaway 4.2.2 #1** _For unsigned values, `a == (a/b)*b + (a%b)`._

**Explanation:** This identity confirms that the quotient and the remainder, when recombined with the divisor, exactly reconstruct the original dividend.

> **Takeaway 4.2.2 #2** _Unsigned `/` and `%` are only well defined if the second operand is not 0._

**Explanation:** Division by zero is mathematically undefined. In a C program, attempting to evaluate `a / 0` or `a % 0` triggers an exceptional condition, usually resulting in a fatal program crash (e.g., a hardware exception or floating-point exception, even for integers).

> **Takeaway 4.2.2 #3** _Arithmetic on `size_t` implicitly computes modulo `SIZE_MAX` + 1._

**Explanation:** The bounds of `size_t` are $0$ to $\text{SIZE\_MAX}$. If a calculation produces a value outside this range, the abstract state machine applies `% (\text{SIZE\_MAX} + 1)` to the mathematical result to map it back into the valid range.

> **Takeaway 4.2.2 #4** _In the case of overflow, unsigned arithmetic wraps around._

**Explanation:** This is the practical consequence of Takeaway 4.2.2 #3. For instance, `0 - 1` mathematically yields $-1$. Modulo `SIZE_MAX` + 1, this wraps around to `SIZE_MAX`. Thus, adding `-1` to an unsigned value `a` effectively computes `a + SIZE_MAX`, which wraps to `a - 1`.

> **Takeaway 4.2.2 #5** _The result of unsigned `/` and `%` is always smaller than the operands._

**Explanation:** For division `a / b` (where $b > 1$), the quotient is smaller than $a$. For remainder `a % b`, the result is strictly bounded in the range $[0, b-1]$, meaning it is always smaller than the divisor $b$.

> **Takeaway 4.2.2 #6** _Unsigned `/` and `%` can’t overflow._

**Explanation:** Because the results of division and remainder are mathematically guaranteed to be smaller than the original operands, they can never produce a value exceeding `SIZE_MAX`. Hence, overflow is impossible for these two operations.

**Pitfalls:**

- Accidental division by zero due to unvalidated user input or unexpected state.
- Misunderstanding that `a % 60` will yield values from $0$ to $59$.

**Connections:** This section explains _why_ `-1` cast to `size_t` evaluates to `SIZE_MAX`, a concept vital for understanding implicit conversions in Chapter 5.

**Recap:**

- `/` is integer division, `%` is the remainder.
- Never divide by 0.
- Unsigned arithmetic inherently uses modulo math, resulting in safe wrap-around on overflow.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t a = 14;
    size_t b = 3;

    // Takeaway 4.2.2 #1 check
    size_t quotient = a / b; // 4
    size_t remainder = a % b; // 2

    if (a == (quotient * b) + remainder) {
        printf("Identity holds: %zu == (%zu * %zu) + %zu\n", a, quotient, b, remainder);
    }

    // Simulating a 24-hour clock (Time is 20:00, what is it 8 hours later?)
    size_t current_time = 20;
    size_t hours_later = 8;
    size_t new_time = (current_time + hours_later) % 24;

    printf("Time after 8 hours: %zu:00\n", new_time);

    // Overflow wrap-around (0 - 1)
    size_t zero = 0;
    size_t wrap = zero - 1;
    printf("Wrapped zero minus one: %zu (SIZE_MAX)\n", wrap);

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ Prints the identity check, "Time after 8 hours: 4:00", and the massive number corresponding to `SIZE_MAX` on the system.

---

## 4.3. Operators that modify objects

### Assignment and Side Effects

**What it is:** Operators that modify an object's state in memory. The most basic is the assignment operator `=`. C also provides compound assignments (`+=`, `-=`, `*=`, `/=`, `%=`) and increment/decrement operators (`++`, `--`).

**Why it matters / how it works:** Assignment writes a computed value (often called an _rvalue_) into an object in memory (often called an _lvalue_). The compound and increment operators are shortcuts that both read an object's current value, apply an arithmetic operation, and write the new value back to the object.

**Key details:**

- `an_object @= some_expression;` is generally equivalent to `an_object = (an_object @ (some_expression));`.
- Prefix increment (`++i`) updates the object and returns the _new_ value to the surrounding expression.
- Postfix increment (`i++`) updates the object but returns the _old_ value to the surrounding expression.

> **Takeaway 4.3 #1** _Operators must have all their characters directly attached to each other._

**Explanation:** Compound operators are treated as single lexical tokens by the compiler. Writing `+ =` instead of `+=` will result in a syntax error because the compiler sees an arithmetic `+` followed unexpectedly by an assignment `=`.

> **Takeaway 4.3 #2** _Side effects in value expressions are evil._

**Explanation:** An expression evaluates to a value. A _side effect_ occurs when evaluating that expression also alters the state of the system (e.g., modifying a variable). Mixing complex evaluation with state mutation makes code incredibly difficult to trace and reason about.

> **Takeaway 4.3 #3** _Never modify more than one object in a statement._

**Explanation:** C allows constructs like `a = b = c += ++d;`. This modifies `d`, `c`, `b`, and `a` in a single line. This is frowned upon because it obfuscates the program's logic and increases the likelihood of undefined behavior due to sequenced evaluation rules. Keep object modifications on separate lines.

**Pitfalls:**

- Spacing out characters in compound operators (`+ =`).
- Using prefix and postfix operators inside complex math expressions, causing confusion about when the variable is actually updated.
- Modifying the same variable twice in one expression (e.g., `i = i++`), which is undefined behavior.

**Connections:** The concept of side effects and sequence points will be deeply explored in Chapter 19 when discussing normal and abnormal control flow.

**Recap:**

- Assignment `=` writes data to an object.
- Compound operators (`+=`, `*=`) combine arithmetic and assignment.
- Never modify multiple objects in a single expression to avoid confusing side effects.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t i = 0;

    // Correct usage of compound assignment
    for (i = 0; i < 25; i += 7) {
        printf("%zu ", i);
    }
    printf("\n");

    // Prefix vs Postfix behavior
    size_t a = 5;
    size_t b = ++a; // 'a' becomes 6, 'b' gets 6

    size_t c = 5;
    size_t d = c++; // 'c' becomes 6, 'd' gets 5

    printf("a: %zu, b: %zu\n", a, b);
    printf("c: %zu, d: %zu\n", c, d);

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ Prints loop sequence "0 7 14 21", then "a: 6, b: 6", then "c: 6, d: 5".
_What goes wrong if:_ You write `a + = 5;` - the compiler throws a syntax error.

---

## 4.4. Boolean context

### 4.4.1. Comparison

**What it is:** Operators that compare two scalar values: `==` (equal), `!=` (not equal), `<` (less than), `>` (greater than), `<=` (less than or equal to), and `>=` (greater than or equal to).

**Why it matters / how it works:** Comparisons evaluate a condition and return a definitive truth value. These truth values dictate the control flow in statements like `if`, `for`, and `while`. C maps truth directly to integers: $0$ is false, and $1$ is true.

**Key details:**

- Strict comparisons (`<`, `>`) vs. inclusive comparisons (`<=`, `>=`).
- The header `<iso646.h>` provides a macro `not_eq` as a text replacement for `!=`.

> **Takeaway 4.4.1 #1** _Comparison operators return the value `false` or `true`._

**Explanation:** In C, `false` and `true` are fundamentally the integer values `0` and `1`. Because comparison operators evaluate strictly to `0` or `1`, you can safely use the result mathematically or as an array index.

**Pitfalls:**

- Confusing the assignment operator `=` with the equality operator `==`. Writing `if (a = 5)` assigns $5$ to `a` and evaluates to true, rather than checking if `a` is $5$.

**Connections:** This connects to Control Flow (Chapter 3) where these operators form the controlling expressions for `if` and `while` statements.

**Recap:**

- Comparison operators check relationships between values.
- They evaluate strictly to integer values `0` (`false`) or `1` (`true`).
- These results can be used directly in arithmetic or array indexing.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t a = 10;
    size_t b = 20;

    // Because comparisons return 0 or 1, they can be used in arithmetic
    // Here c will be 1 because exactly one condition (a < b) is true.
    size_t c = (a < b) + (a == b) + (a > b);

    printf("Result of sum of comparisons: %zu\n", c);

    // Using comparison result as an array index
    size_t sign[2] = {0, 0};
    double largeA[] = {0.5, 1.2, 3.4, 0.1};

    for (size_t i = 0; i < 4; ++i) {
        // (largeA[i] < 1.0) is either 0 (false) or 1 (true)
        sign[(largeA[i] < 1.0)] += 1;
    }

    printf("Elements >= 1.0: %zu\n", sign[0]);
    printf("Elements < 1.0: %zu\n", sign[1]);

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ Prints "Result...: 1", then "Elements >= 1.0: 2", then "Elements < 1.0: 2".

### 4.4.2. Logic

**What it is:** Logic operators act on existing boolean conditions: `!` (logical NOT), `&&` (logical AND), and `||` (logical OR).

**Why it matters / how it works:** These allow the composition of complex conditions from simpler ones. If the operands are not explicitly boolean, they are implicitly evaluated against $0$ (any non-zero value is true, $0$ is false).

**Key details:**

- `!` negates the condition: `!true` is `false`, `!false` is `true`.
- `&&` yields true only if both operands evaluate to true.
- `||` yields true if at least one operand evaluates to true.
- Applying `!!` to any scalar value squashes it to strictly `0` or `1`.

> **Takeaway 4.4.2 #1** _Logic operators return the value `false` or `true`._

**Explanation:** Just like comparison operators, logical operators yield strictly `0` or `1`.

**Short-circuit evaluation:** This is a critical property of `&&` and `||`. The compiler evaluates the left operand first. If the left operand definitively determines the outcome, the right operand is _completely ignored and not executed_. For `&&`, if the left is false, the expression is false. For `||`, if the left is true, the expression is true.

**Pitfalls:**

- Placing operations with side effects (like `i++`) on the right side of a `&&` or `||`. Because of short-circuiting, the side effect might not occur, leading to highly unpredictable program states.

**Connections:** The concept of truth value evaluation (Takeaway 3.1 #4: "All scalars have a truth value") applies directly before the logic operators take effect.

**Recap:**

- `!`, `&&`, and `||` combine truth values.
- They evaluate strictly to `0` or `1`.
- `&&` and `||` utilize short-circuit evaluation, skipping the right operand if the result is already decided.

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t a = 10;
    size_t b = 0;
    size_t x = 0;

    // Short-circuit evaluation guarantees we never divide by zero here.
    // Because 'b != 0' is false, the right operand '((a/b) > 1)' is skipped entirely.
    if (b != 0 && ((a/b) > 1)) {
        ++x;
    }

    printf("Executed safely. x = %zu\n", x);

    // Using !! to squash a value to 0 or 1
    size_t val = 42;
    size_t truth_value = !!val;
    printf("Double negation of %zu is %zu\n", val, truth_value);

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ Prints "Executed safely. x = 0", then "Double negation of 42 is 1".

---

## 4.5. The ternary or conditional operator

**What it is:** The ternary operator `?:` is an expression-level analog to the `if-else` statement. It takes three operands: a condition, an expression if true, and an expression if false. Format: `condition ? expression_if_true : expression_if_false`.

**Why it matters / how it works:** It allows for concise inline decisions. Unlike an `if` statement, which controls execution flow of blocks, the ternary operator evaluates to a specific value and can be embedded directly within larger expressions.

**Key details:**

- Like logical AND/OR, the ternary operator utilizes short-circuit evaluation. The condition is evaluated first. Only the chosen expression (true branch or false branch) is evaluated; the other is skipped.

**Pitfalls:**

- Nesting ternary operators (`a ? b : c ? d : e`) makes code notoriously unreadable.
- Differences in types between the true and false branch expressions can lead to unexpected implicit conversions.

**Connections:** This provides a functional alternative to `if-else` control flow (Chapter 3.1).

**Recap:**

- `?:` evaluates a condition to select one of two expressions.
- It returns the value of the selected branch.
- It short-circuits, ensuring only the selected branch is executed.

```c
#include <stdio.h>
#include <stdlib.h>

// A pure function using the ternary operator
size_t size_min(size_t a, size_t b) {
    return (a < b) ? a : b;
}

int main(void) {
    size_t x = 100;
    size_t y = 50;

    size_t smaller = size_min(x, y);
    printf("The minimum is: %zu\n", smaller);

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ Prints "The minimum is: 50".

---

## 4.6. Evaluation order

**What it is:** The rules dictating the sequence in which the compiler evaluates the sub-components of an expression.

**Why it matters / how it works:** When you write a complex expression `f(a) + f(b)`, you might assume `f(a)` executes before `f(b)`. C makes no such guarantee for most operators. Relying on an unstated order can lead to bugs that only appear on specific compilers or optimization levels.

**Key details:**

- Only four operators guarantee left-to-right evaluation sequencing.
- The comma `,` operator evaluates its left operand, discards the result, evaluates the right operand, and returns the right operand's value.
- Commas used to separate function arguments or list initializations are _syntactic separators_, not the comma operator, and do not enforce order.

> **Takeaway 4.6 #1** _`&&`, `||`, `?:`, and `,` evaluate their first operand first._

**Explanation:** Because `&&`, `||`, and `?:` rely on conditions to determine if subsequent code should run (short-circuiting), the condition (first operand) must absolutely be evaluated first. The comma operator specifically exists to enforce sequencing.

> **Takeaway 4.6 #2** _Don’t use the `,` operator._

**Explanation:** The comma operator leads to confusing code. If you need sequential evaluation, use separate statements separated by semicolons `;`.

> **Takeaway 4.6 #3** _Most operators don’t sequence their operands._

**Explanation:** For operators like `+`, `*`, `=`, `==`, the compiler is free to evaluate the left side first, the right side first, or even interleave their evaluations if it deems it optimal.

> **Takeaway 4.6 #4** _Function calls don’t sequence their argument expressions._

**Explanation:** In `printf("%g and %g\n", f(a), f(b));`, the compiler might execute `f(b)` before `f(a)`.

> **Takeaway 4.6 #5** _Functions calls within expressions should not have side effects._

**Explanation:** Because the evaluation order of most operators and function arguments is unspecified, if `f(a)` modifies a global variable that `f(b)` reads, the result of your program becomes entirely unpredictable and compiler-dependent.

**Pitfalls:**

- Assuming left-to-right evaluation for addition, multiplication, or function arguments.
- Using the `,` operator to cram multiple operations into a single line.

**Connections:** This is the foundation for avoiding side-effect bugs discussed in Chapter 4.3 and deepens into Sequence Points later in the book (Chapter 19).

**Recap:**

- Only `&&`, `||`, `?:`, and the comma `,` operator strictly evaluate left-to-right.
- Never rely on the evaluation order of other operators or function arguments.
- Avoid side-effects inside expressions to prevent unpredictable behavior.

```c
#include <stdio.h>
#include <stdlib.h>

// A function with a nasty side effect
size_t global_state = 0;
size_t mutate_and_return(size_t val) {
    global_state += val;
    return global_state;
}

int main(void) {
    // Takeaway 4.6 #4: We do not know if mutate_and_return(1) or mutate_and_return(2)
    // runs first.
    // If (1) runs first, it prints: 1 and 3
    // If (2) runs first, it prints: 3 and 2
    // NEVER DO THIS.
    printf("%zu and %zu\n", mutate_and_return(1), mutate_and_return(2));

    // Safe usage leveraging Takeaway 4.6 #1
    // The left side of && executes first guaranteed.
    size_t x = 0;
    if (x == 0 || mutate_and_return(5)) {
         printf("Short circuited! global_state is still %zu\n", global_state);
    }

    return EXIT_SUCCESS;
}
```

_Expected behavior/output:_ The `printf` output is compiler-dependent. The `if` statement safely short-circuits, so `mutate_and_return(5)` never runs.

---

## Summary

In C, computations are expressed through the interaction of operands (data) and operators (instructions). Arithmetic operations on unsigned integer types like `size_t` form a safe, mathematically exact foundation that uses modulo arithmetic to guarantee defined wrap-around behavior upon overflow. State mutation is handled via assignment and increment operators, but combining these with side effects inside complex expressions leads to dangerous code. Conditions in C are inherently numerical, with $0$ representing `false` and any non-zero value representing `true`. Both comparison and logical operators evaluate strictly to $0$ or $1$, with logical operators `&&` and `||`, as well as the ternary `?:` operator, employing short-circuit evaluation. Critically, C leaves the evaluation order of operands for most operators and function arguments completely unspecified, making side effects within complex expressions a major source of unpredictable behavior.

---

## Self-Check Questions

1.  **What happens if you subtract 5 from 3 when using `size_t` variables?**
    The result wraps around using modulo `SIZE_MAX + 1`, resulting in `SIZE_MAX - 1`.

2.  **Why is unsigned arithmetic considered safer than signed arithmetic in C?**
    Because unsigned arithmetic is explicitly defined to wrap around upon overflow, avoiding undefined behavior.

3.  **What values do comparison operators (like `==` or `<`) return?**
    They strictly return the integer `1` for true and `0` for false.

4.  **Explain "short-circuit evaluation" and list the operators that use it.**
    It means stopping evaluation as soon as the result is definitively known from the left operand. The operators are `&&`, `||`, and `?:`.

5.  **What is wrong with the statement `a = b = c += ++d;`?**
    It modifies multiple objects in a single statement, introducing confusing side effects and risking undefined behavior.

6.  **Why should you avoid placing functions with side effects as arguments to another function?**
    Because C does not specify the order in which function arguments are evaluated.

7.  **What mathematical operation does C use to manage unsigned values that exceed their maximum limit?**
    Modulo arithmetic (`% (SIZE_MAX + 1)`).

8.  **Is the comma in `printf("%d, %d", a, b)` the comma operator? Explain.**
    No, it is a syntactic separator for function arguments. It does not guarantee left-to-right evaluation like the true comma operator does.
