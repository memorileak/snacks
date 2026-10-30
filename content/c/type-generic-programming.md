+++
title = "Modern C: Type-generic programming"
description = "A guide to C23 type-generic programming, covering inherent generic features, _Generic dispatch, type inference with auto and typeof, and anonymous function extensions."
date = 2026-09-28T07:16:58Z

[taxonomies]
tags = ["modernc", "generic"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 18.1. Inherent type-generic features in C

### 18.1.1. Operators

**What it is:** Operators in C, such as equality (`==`, `!=`), arithmetic (`+`, `*`, `/`), and relational operators (`<`, `>=`), are inherently type-generic. They can accept operands of various types, ranging from wide integers (signed/unsigned, long, long long) to real and complex floating-point types, and even pointers.

**Why it matters / how it works:** This built-in genericity means programmers do not need to call type-specific functions (like `add_int` or `add_float`) for basic operations. The compiler automatically determines the appropriate machine instructions based on the types of the operands provided to the operator.

**Key details:**

- Most binary operators (like `==`, `+`, `*`) require their operands to be of the same type.
- If operands differ, C applies implicit conversions to harmonize them before performing the operation.
- Shift operators, pointer addition, and array subscripting are exceptions; they can handle differing operand types without requiring conversion to a common type.

**Pitfalls:** Mixing signed and unsigned integers in operators can lead to unexpected conversions and logical errors due to the rules of arithmetic conversions.

**Connections:** This relates closely to "Default promotions and conversions" (18.1.2) where the rules for harmonizing these types are defined.

```c
#include <stdio.h>
#include <stdbool.h>

int main(void) {
    int a = 5;
    double b = 2.5;
    // The + operator is inherently type-generic.
    // 'a' is implicitly converted to double before addition.
    double result = a + b;

    printf("Result: %f\n", result);
    return 0;
}
/*
Expected behavior/output:
Result: 7.500000
*/
```

### 18.1.2. Default promotions and conversions

**What it is:** This is the complex set of rules C uses to convert types when operands of an operator don't match, or when passing arguments to functions, assignments, and initializations. Narrow integer types are promoted, and types are converted "upward" in a hierarchy.

**Why it matters / how it works:** These rules prevent the need to write multiple versions of identical functions or expressions. When a narrow type (like `short` or `bool`) is used in arithmetic, it is first promoted (usually to `signed int`). If a binary operator has mixed types (e.g., `int` and `float`), the "narrower" type is converted to the "wider" type (e.g., `float`) to preserve values during the operation.

**Key details:**

- **Promotions:** Narrow types (`char`, `short`, `bool`) are promoted to `signed int` (or `unsigned int` if they cannot fit in `signed int`) before arithmetic.
- **Conversions:** Work upward (e.g., `int` -> `double`).
- Function arguments are automatically converted if a prototype is visible.

**Pitfalls:** Unintentional loss of precision or unexpected sign flips when interacting with `unsigned` types, as a negative `signed` integer converted to an `unsigned` type results in a large positive value.

**Connections:** Heavily tied to expressions and operators (18.1.1) and understanding binary representations (Chapter 5).

### 18.1.3. Macros

**What it is:** C's preprocessor (`#define`) performs textual replacement before compilation. This allows for type-generic expressions and declarations because macros do not have types themselves; they simply inject code that takes on the type of whatever arguments are passed in.

**Why it matters / how it works:** Preprocessor macros are evaluated purely syntactically. Combined with default arithmetic promotions, a macro like `#define SQUARE(X) ((X)*(X))` works identically whether `X` is an `int` or a `double`.

**Key details:**

- Can create type-generic expressions.
- Can create type-generic declarations (e.g., declaring arrays of varying types).

**Pitfalls:** Macros are oblivious to type safety and scope. Passing an expression with side effects (like `x++`) to a macro that evaluates its argument multiple times leads to undefined or unintended behavior.

**Connections:** Function-like macros (Chapter 17) use this textual replacement to emulate generic functions.

### 18.1.4. Variadic functions

**What it is:** Functions that accept a variable number of arguments of variable types, denoted by the ellipsis `...` in their signature (e.g., `printf`).

**Why it matters / how it works:** They provide a legacy way to write interfaces that can process arbitrary data. Under the hood, arguments matching the `...` undergo default argument promotions (`float` to `double`, narrow integers to `int`). The function must use `<stdarg.h>` macros to extract these values blindly based on some convention (like a format string).

**Key details:**

- Requires a known parameter before the `...` to anchor the argument list.
- Types must be manually deduced at runtime via developer-defined conventions.

**Pitfalls:** Complete lack of type safety. If you tell `printf` to expect an `int` (`%d`) but pass a `double`, the program exhibits undefined behavior.

**Connections:** Discussed extensively in Chapter 17 as a legacy mechanism that is generally inferior to modern type-safe macros.

### 18.1.5. Function pointers

**What it is:** Variables that store the memory address of executable code (functions) rather than data.

**Why it matters / how it works:** They enable algorithms to be written generically by deferring type-specific logic to a callback function provided by the user. For instance, `qsort` sorts an array of any type because it uses a function pointer to figure out how to compare two elements.

**Key details:**

- The pointer must perfectly match the signature of the function it points to.

**Pitfalls:** Syntax can be extremely dense and hard to read. Dereferencing a null function pointer causes a crash.

**Connections:** Callbacks and pointers (Chapter 11).

### 18.1.6. void pointers

**What it is:** Pointers of type `void*` (or `void const*`). `void` is an incomplete type that represents "no specific type".

**Why it matters / how it works:** Any pointer to an object can be implicitly converted to a `void*` and back again. This allows library functions like `qsort` or `bsearch` to accept arrays of any data type. The callback function (18.1.5) usually takes `void*` parameters, casts them back to the correct underlying type, and operates on them.

**Key details:**

- Cannot be dereferenced directly (since the compiler doesn't know the size or representation of what it points to).
- Must be explicitly converted back to the original pointer type before use.

**Pitfalls:** Completely bypasses the type system. If you cast a `void*` to a `double*` but it actually points to an `int`, you will access a non-value representation and trigger undefined behavior.

**Connections:** Often paired with function pointers (18.1.5) to build generic data structures or algorithms.

### 18.1.7. Type-generic C library functions

**What it is:** Interfaces in the standard library (like `<tgmath.h>`) that provide a single macro name (e.g., `cos`) that automatically maps to the correct underlying function (e.g., `cosf`, `cos`, `cosl`) based on the argument's type.

**Why it matters / how it works:** It simplifies coding by abstracting away the explicit suffixes (`f` for float, `l` for long double). In C23, this was expanded to preserve `const` correctness in functions like `strchr` and `bsearch` using generic macros.

**Key details:**

- Historically implemented via compiler magic relying on the `sizeof` the arguments.
- C23 standardized `const`-contract preservation (if you pass a `const char*` to `strchr`, it returns a `const char*`).

**Pitfalls:** Since these often wrap standard functions, shadowing standard names or attempting to take the address of a type-generic macro (e.g., `&cos`) can lead to compilation errors or unexpected behavior.

**Connections:** The concept directly motivates the introduction of `_Generic` (18.2).

**Section 18.1 Recap:**

- C has inherently generic operators that rely on complex conversion rules.
- Macros and `void*` pointers provide legacy, unsafe genericity.
- The C library increasingly relies on generic macros (like in `<tgmath.h>`) to provide safe, simplified interfaces.

---

## 18.2. Generic selection

### 18.2.1. The _Generic keyword

**What it is:** A keyword introduced in C11 (`_Generic`) that allows compile-time selection of an expression based on the type of a controlling expression.

**Why it matters / how it works:** It is the standard mechanism to create type-generic macros safely. The compiler evaluates the type of the controlling expression, matches it against a list of type associations, and replaces the `_Generic` block entirely with the corresponding expression.

> **Takeaway 18.2 #1** _The result type of a `_Generic` expression is the type of the chosen expression._

**Explanation:** Because `_Generic` is evaluated at compile time, the final expression injected into the code completely adopts the type of the matched branch. It does not try to unify the types of all possible branches (unlike the ternary operator `?:`).

```c
#include <stdio.h>

#define PRINT_TYPE_NAME(X) _Generic((X), \
    int: "integer", \
    double: "double", \
    default: "other")

int main(void) {
    int a = 5;
    double b = 3.14;
    // The macro evaluates the type of 'a' and 'b' at compile time.
    // The result type of the expression is 'char*' (string literal).
    printf("a is %s\n", PRINT_TYPE_NAME(a));
    printf("b is %s\n", PRINT_TYPE_NAME(b));
    return 0;
}
/*
Expected behavior/output:
a is integer
b is double
*/
```

> **Takeaway 18.2 #2** _Using `_Generic` with `inline` functions adds optimization opportunities._

**Explanation:** When `_Generic` maps to specific inline functions, the compiler knows the exact types being handled. This precise type information allows the compiler to inline the function body directly, resulting in highly optimized machine code without function call overhead.

```c
#include <stdio.h>

static inline double mind(double a, double b) { return a < b ? a : b; }
static inline float minf(float a, float b) { return a < b ? a : b; }

// Generic macro selecting an inline function based on the promoted type of A+B
#define min(A, B) _Generic((A)+(B), \
    float: minf, \
    default: mind)((A), (B))

int main(void) {
    float x = 1.5f, y = 2.0f;
    // Compiles directly to the minf implementation, highly optimized.
    printf("Min: %f\n", min(x, y));
    return 0;
}
/*
Expected behavior/output:
Min: 1.500000
*/
```

> **Takeaway 18.2 #3** _All choices expression1 ... expressionN in a `_Generic` must be valid._

**Explanation:** Even though only one branch is selected, the compiler parses all branches. Every expression in the `_Generic` selection must be syntactically valid code. You cannot have a branch that would cause a syntax error, even if that branch is never selected.

> **Takeaway 18.2 #4** _The type expressions in a `_Generic` expression should only be unqualified types, not array types or function types._

**Explanation:** The controlling expression of `_Generic` undergoes a process similar to function argument decay: qualifiers (like `const` or `volatile`) are stripped, arrays decay to pointers, and functions decay to function pointers. Therefore, checking for `const int` in the association list will never match; you must check for `int`.

> **Takeaway 18.2 #5** _The type expressions in a `_Generic` expression must refer to mutually incompatible types._

**Explanation:** You cannot have overlapping or identical types in the association list. For example, checking for `signed int` and `int` in the same `_Generic` block will cause a compiler error because they are the exact same type. The compiler needs absolute, unambiguous resolution.

> **Takeaway 18.2 #6** _The type expressions in a `_Generic` expression cannot be a pointer to a VLA._

**Explanation:** Variable-Length Arrays (VLAs) have types that depend on runtime values. Since `_Generic` is evaluated entirely at compile time, it cannot resolve types that require runtime evaluation.

**Key details:**

- Syntax: `_Generic(controlling_expression, type1: exp1, type2: exp2, default: exp_def)`
- `default:` handles any type not explicitly matched.

**Pitfalls:** Combinatorial explosion. If writing a generic macro for 2 parameters (like `min`), testing every combination of types quickly results in dozens of branches.

**Connections:** This standardizes the implementation of `<tgmath.h>` (18.1.7).

**Section 18.2 Recap:**

- `_Generic` resolves expressions based on type at compile time.
- All branches must be syntactically valid, and types must be distinct, unqualified, and non-VLA.
- Pairing `_Generic` with `inline` functions produces safe, highly optimized type-generic interfaces.

---

## 18.3. Type inferrence

### 18.3.1. The auto feature

**What it is:** C23 repurposed the `auto` keyword to deduce the type of a variable directly from its initializer, removing the need to specify the type explicitly.

**Why it matters / how it works:** It prevents repetition and guarantees that a variable holding a result perfectly matches the type of the expression computing that result. This is particularly vital inside macros to avoid unwanted implicit conversions or precision loss.

> **Takeaway 18.3.1 #1** _Protect local variables inside macros by a documented naming convention._

**Explanation:** When defining local variables inside a macro block, name them uniquely (e.g., prefixing with `_macroname_`) to prevent them from shadowing variables in the outer scope where the macro is called.

> **Takeaway 18.3.1 #2** _Use `auto` definitions where you must ensure type consistency._

**Explanation:** Hardcoding a type like `float y = 3 * x;` hides bugs if `x` is later changed to a `double`. Using `auto const y = 3 * x;` ensures `y` is automatically upgraded to `double`, preventing silent precision loss.

```c
#include <stdio.h>

// Safe swap macro using 'auto' for type consistency
#define SWAP(X, Y) do { \
    auto _swap_temp = (X); \
    (X) = (Y); \
    (Y) = _swap_temp; \
} while(0)

int main(void) {
    double a = 1.123456789, b = 9.876543210;
    // _swap_temp perfectly deduces to 'double'. No precision lost.
    SWAP(a, b);
    printf("a: %f, b: %f\n", a, b);
    return 0;
}
/*
Expected behavior/output:
a: 9.876543, b: 1.123457
*/
```

### 18.3.2. The typeof feature

**What it is:** C23 introduced `typeof(expr)` and `typeof_unqual(expr)` to query the type of an expression at compile time without evaluating the expression itself.

**Why it matters / how it works:** It acts as a type name in declarations or casts. While `auto` requires an initializer, `typeof` can be used to declare a variable without initializing it immediately, or to cast a value to match another variable's type. `typeof_unqual` strips qualifiers like `const` and `volatile`.

> **Takeaway 18.3.2 #1** _Prefer `auto` over `typeof` for variable declarations._

**Explanation:** `auto` is cleaner, easier to read, and less prone to edge-case bugs (like double-evaluation risks in complex macros). `typeof` should be reserved for scenarios where `auto` isn't feasible, such as testing type compatibility or casting return types.

**Key details:**

- `typeof` retains qualifiers (`const int` remains `const int`).
- `typeof_unqual` strips qualifiers, which is useful when you need to declare a mutable copy of a `const` variable.

**Pitfalls:** Overusing `typeof` creates messy, hard-to-maintain code when `auto` would suffice.

**Connections:** `typeof` can be fed directly into `_Generic` expressions to perform compile-time type assertions.

**Section 18.3 Recap:**

- `auto` infers variable types from initializers, preventing precision loss.
- `typeof` and `typeof_unqual` allow type querying without evaluation.
- Use `auto` for declarations; reserve `typeof` for complex casts and type traits.

---

## 18.4. Anonymous functions

**What it is:** Non-standard compiler extensions (like GCC's statement expressions `({ ... })` or block closures `void^(void){...}()`) that allow creating blocks of code that return a value and can be embedded directly within expressions.

**Why it matters / how it works:** Traditional macros enclosed in `do { ... } while(0)` are statements, not expressions, meaning they cannot return a value (e.g., you can't write `int z = SWAP(x, y);`). Statement expressions solve this by executing a block and resolving to the value of the last statement inside the block.

**Key details:**

- They allow the declaration of local variables (using `auto`) to cache macro arguments, preventing the "double evaluation" bug (e.g., `MAX(a++, b)`).
- The final expression inside the `({ ... })` dictates the return type and value.

**Pitfalls:** This is an extension, not standard C (as of C23). Relying on it makes code unportable to compilers that do not support GCC extensions.

**Connections:** These extensions fix the inherent flaws of function-like macros (18.1.3) by providing local scope and value return, effectively acting like lambdas in other languages.

```c
#include <stdio.h>

// MAX macro using GCC statement expression extension to prevent double evaluation.
#define MAX(X, Y) ({ \
    auto const _max_x = (X); \
    auto const _max_y = (Y); \
    (_max_x > _max_y) ? _max_x : _max_y; \
})

int main(void) {
    int a = 5;
    int b = 10;
    // If a standard macro MAX(a++, b) was used, 'b' might be evaluated twice.
    // Here, _max_x and _max_y cache the values safely.
    int max_val = MAX(a++, b);

    printf("Max: %d, a: %d\n", max_val, a);
    return 0;
}
/*
Expected behavior/output:
Max: 10, a: 6
*/
```

**Section 18.4 Recap:**

- Anonymous functions/statement expressions are compiler extensions.
- They solve the double-evaluation problem in macros by allowing local variable caching.
- They allow block constructs to return values and act as expressions.

---

## Summary

Chapter 18 explores the mechanisms C provides for writing type-generic code. It begins with the inherent genericity of C's operators and the historical, type-unsafe methods using macros, variadic functions, and `void*` pointers. It then introduces modern, type-safe solutions: the `_Generic` keyword from C11 allows compile-time code selection based on types, while C23's `auto` and `typeof` features enable seamless type inference to prevent precision loss and repetition. Finally, it notes compiler extensions like anonymous functions (statement expressions) that bridge the gap between macros and safe, expression-oriented block execution, demonstrating a clear evolution toward robust, type-safe, generic programming in modern C.

## Self-Check Questions

1.  **Why do binary operators in C like `+` generally not require explicit type-generic programming?**
    Because operators are inherently generic; C automatically applies default arithmetic conversions and promotions to harmonize the types of the operands before execution.

2.  **What is a major safety drawback of using variadic functions (`...`) for generic programming?**
    They completely bypass type checking at compile time, leading to undefined behavior if the extracted type does not match the passed type.

3.  **When is a `void*` pointer necessary in type-generic programming?**
    It is used as a generic placeholder for memory addresses, particularly in generic algorithms (like `qsort`) that rely on function pointers to operate on arbitrary data types.

4.  **What dictates the return type of a `_Generic` selection?**
    The type of the final expression is exactly the type of the expression in the specific branch chosen at compile time.

5.  **Can you use `const int` as a type matcher in a `_Generic` expression?**
    No, the controlling expression has its qualifiers dropped (similar to function decay), so type associations must be unqualified types (like `int`).

6.  **Why must all branches of a `_Generic` expression be syntactically valid?**
    Because the compiler parses the entire `_Generic` construct, even the unselected branches, meaning any syntax error will halt compilation.

7.  **What does the C23 `auto` keyword do?**
    It infers the type of a variable directly from its initializer expression.

8.  **How does `auto` improve safety inside macros?**
    It prevents the hardcoding of types, ensuring that temporary variables inside macros dynamically match the types of the arguments passed, preventing silent precision loss.

9.  **What is the difference between `typeof` and `typeof_unqual`?**
    `typeof` retains the qualifiers (like `const`) of the expression it evaluates, while `typeof_unqual` explicitly strips them.

10. **What problem with standard C macros do "statement expressions" (anonymous functions) solve?**
    They allow a macro to contain a block of code (with local variable declarations to prevent double evaluation) that successfully returns a value as an expression.
