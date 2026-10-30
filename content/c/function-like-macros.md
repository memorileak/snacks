+++
title = "Modern C: Function-like macros"
description = "A guide to C23 function-like macros, including expansion rules, argument constraints, caller context, variadic interfaces, and default arguments."
date = 2026-09-28T07:10:00Z

[taxonomies]
tags = ["modernc", "macros"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 17. Function-like macros

### Understanding Function-like Macros vs. Inline Functions

**What it is:** Function-like macros are a preprocessor feature in C that allows you to define text replacement rules that syntactically look like function calls. Unlike normal constants (object-like macros), function-like macros take arguments.

**Why it matters / how it works:** Macros operate purely through textual substitution during the preprocessor phase, long before the compiler understands C syntax or types. Because of their complexity, lack of type safety, and potential to obfuscate code, they are often frowned upon. However, they are essential for providing easy-to-use, type-generic interfaces (like `<tgmath.h>`), enforcing compile-time argument checks, or working with variable-length argument lists.

**Key details:**

- They are a legacy of early C but remain irreplaceable for specific advanced tasks.
- If a task can be done with a standard `inline` function (fixed number of arguments, known types), you must use an `inline` function instead to preserve type safety.
- Function-like macros can evaluate their arguments multiple times (a severe pitfall) if not written carefully.

**Pitfalls:**

- **Double Evaluation:** If a macro like `#define square_macro(X) (X*X)` is called with an expression that has side effects (e.g., `square_macro(count())`), the expression is evaluated twice (`count() * count()`), leading to unexpected behavior and degraded performance.

**Connections:** Relates heavily to Chapter 11 (Functions) and Chapter 16 (Performance), where `inline` functions are introduced as the modern, safe alternative to simple macros.

> **Takeaway 17 #1:** _Whenever possible, prefer an `inline` function to a functional macro._

> **Takeaway 17 #2:** _A functional macro shall provide a simple interface to a complex task._

**Code Demonstration:**

```c
#include <stdio.h>

// A function with a side effect
unsigned count(void) {
    static unsigned counter = 0;
    ++counter;
    return counter;
}

// BAD: Functional macro that evaluates X twice
#define square_macro(X) ((X)*(X))

// GOOD: Inline function evaluating x exactly once
inline unsigned square_unsigned(unsigned x) {
    return x * x;
}

int main(void) {
    // What goes wrong if we use the macro:
    // square_macro(count()) expands to ((count())*(count()))
    // count() is called twice!
    unsigned bad_result = square_macro(count());
    printf("Macro result: %u\n", bad_result); // Outputs: 2 (1 * 2)

    // Expected behavior using inline function:
    unsigned good_result = square_unsigned(count());
    printf("Inline result: %u\n", good_result); // Outputs: 9 (3 * 3)

    return 0;
}
```

**Recap:**

- Function-like macros perform textual substitution.
- Always prefer `inline` functions for type safety and to avoid double-evaluation side effects.
- Reserve function-like macros for complex tasks like type-generic programming or variadic interfaces.

---

## 17.1. How function-like macros work

### Preprocessor Mechanics and Expansion Rules

**What it is:** The strict, multi-step process the C preprocessor uses to expand function-like macros into intermediate C code before actual compilation begins.

**Why it matters / how it works:** Understanding these steps is crucial to predicting how nested macros expand and how to avoid syntax errors. The preprocessor temporarily disables the macro's own definition during its expansion to prevent infinite recursion.

**Key details:**

- A macro definition requires an identifier, a parameter list `(X, Y)`, and replacement text.
- The expansion follows 8 strict rules:
  1.  The macro's definition is temporarily disabled.
  2.  The argument list is scanned and separated by commas (ignoring commas inside nested parentheses). The argument count must match the parameter count.
  3.  Arguments are recursively expanded.
  4.  Resulting fragments are assigned to parameters.
  5.  A copy of the replacement text is made; parameters are replaced by the fragments.
  6.  The new text is scanned _again_ for further macro replacements.
  7.  The final text is inserted into the source.
  8.  The macro definition is re-enabled.

**Pitfalls:**

- Macro parameters must be valid identifiers.
- The preprocessing phase knows absolutely nothing about C keywords, types, or variables. It is blind text manipulation.

> **Takeaway 17.1 #1:** _Macro replacement is done in an early translation phase, before any other interpretation is given to the tokens that compose the program._

### Macro Retention and Function Decay

**What it is:** A specific rule stating that a function-like macro is _only_ expanded if its identifier is immediately followed by an opening parenthesis `(`.

**Why it matters / how it works:** This allows a macro and a function to share the exact same identifier safely. If you wrap the function name in parentheses (e.g., `(my_func)(arg)`), the preprocessor sees a parenthesis before the macro identifier, not after, so it skips expansion.

**Key details:**

- If `func_name` is defined as a macro, `func_name(x)` expands the macro.
- `(func_name)(x)` bypasses the macro and calls the function of the same name.
- `func_ptr = func_name;` bypasses the macro (no trailing parentheses) and decays the function to a pointer.

> **Takeaway 17.1 #2 (macro retention):** _If a functional macro is not followed by (), it is not expanded._

**Code Demonstration:**

```c
#include <stdio.h>

// 1. Function definition
inline char const* string_literal_func(char const str[static 1]) {
    return str;
}

// 2. Macro definition sharing the same name
#define string_literal_func(S) string_literal_func("" S "")

int main(void) {
    // Macro expansion occurs here because of trailing ()
    // Expands to: string_literal_func("" "Hello" "")
    printf("%s\n", string_literal_func("Hello"));

    // Macro retention: No trailing () immediately after the identifier.
    // The preprocessor leaves (string_literal_func) alone.
    // It calls the actual function.
    printf("%s\n", (string_literal_func)("World"));

    // Function decay: no (), resolves to function pointer
    char const* (*func_ptr)(char const*) = string_literal_func;
    printf("%s\n", func_ptr("Pointer"));

    return 0;
}
```

_Expected behavior:_ All three prints succeed. The first utilizes the macro to ensure string literal concatenation, while the second and third bypass the macro to reference the underlying function.

**Recap:**

- Macros are expanded in 8 distinct steps, heavily relying on recursive evaluation.
- The preprocessor is ignorant of C types and semantics.
- Function-like macros require trailing parentheses to expand; otherwise, they are retained as raw identifiers.

---

## 17.2. Argument checking

### Forcing Types at Compile Time

**What it is:** The practice of using macro replacement text to enforce that the arguments passed to the macro match expected types, mimicking the type checking of real functions.

**Why it matters / how it works:** Because macros lack inherent type signatures, passing a pointer when an integer is expected (or vice versa) can cause silent logical errors or crashes at runtime (especially with variadic functions like `printf`). We can force compiler errors by manipulating the replacement text.

**Key details:**

- **String literals:** By forcing concatenation (`"" F ""`), you ensure the argument `F` _must_ be a string literal. If the user passes a `char*` variable, compilation fails because variables cannot be concatenated with string literals by the preprocessor.
- **Scalar enforcement:** Adding `+0` to an argument (e.g., `(X) + 0`) forces the compiler to treat it as a scalar (integer, float, or pointer).
- **Floating-point enforcement:** Adding `+0.0F` promotes arithmetic types to floating-point.
- **Pointer cast checking:** Casting to `(void*)` or using compound literals can force pointer context.

**Pitfalls:**

- Type-checking tricks can make macro definitions exceptionally hard to read and debug.
- You must be careful not to introduce side effects or alter the intended value (e.g., adding `0` is safe, but multiplying by `1` might evaluate side effects depending on placement).

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdbool.h>

#define NDEBUG 0 // Set to 1 to disable traces

#if NDEBUG
# define TRACE_ON 0
#else
# define TRACE_ON 1
#endif

// Forces F to be a string literal.
// A char* variable will cause a compile error here.
#define TRACE_PRINT1(F, X) \
    do { if (TRACE_ON) fprintf(stderr, "" F "\n", X); } while (false)

// Forces X to be evaluated as a floating type (or promotable arithmetic)
#define TRACE_VALUE0(HEAD, X) \
    TRACE_PRINT1(HEAD " %Lg", (X) + 0.0L)

int main(void) {
    int sum = 42;
    // Expected output: "my_favorite_variable: 42"
    TRACE_VALUE0("my_favorite_variable:", sum);

    // What goes wrong if:
    // char* dyn_str = "dynamic";
    // TRACE_PRINT1(dyn_str, sum);
    // ^ This fails to compile because "" dyn_str "\n" is invalid syntax.

    return 0;
}
```

**Recap:**

- Macros inherently lack type safety.
- You can trick the compiler into checking types by using operations only valid for specific types (e.g., literal concatenation, arithmetic addition).
- Encapsulate macro logic in `do { ... } while(false)` to ensure it behaves like a single statement safely.

---

## 17.3. Accessing the context of invocation

### Contextual Macros and Stringification

**What it is:** Using special, predefined preprocessor macros (`__LINE__`, `__FILE__`, `__func__`, etc.) and the stringification operator (`#`) to inject information about _where_ the macro is being called into the code itself.

**Why it matters / how it works:** This breaks the separation between caller and callee context, which is highly desirable for debugging and tracing. The `#` operator converts a macro argument directly into a string literal without evaluating it.

**Key details:**

- `__LINE__`: Current line number (integer constant).
- `__FILE__`: Current file name (string literal).
- `__DATE__` / `__TIME__`: Compilation timestamps.
- `__func__`: The name of the current function (this is a local `static const char[]`, _not_ a preprocessor string literal, so it cannot be concatenated with `""`).
- **The `#` operator:** `#X` turns the argument `X` into `"X"`.
- **Two-level stringification:** Because `#` _prevents_ macro expansion of its argument, if you want to stringify the _result_ of a macro (like `__LINE__`), you must use a two-level macro trick: `STRGY(X)` -> `STRINGIFY(X)` -> `#X`.

**Pitfalls:**

- Relying on `__LINE__` can be dangerous if line numbers exceed standard integer limits (though rare).
- Since `#` prevents expansion, `STRINGIFY(__LINE__)` results in `"__LINE__"`. You must use the intermediate `STRGY` macro.
- Nested macro definitions can cause arguments to be evaluated multiple times if you aren't careful.

> **Takeaway 17.3 #1:** _The line number in `__LINE__` may not fit into an `int`._

> **Takeaway 17.3 #2:** _Using `__LINE__` is inherently dangerous._

> **Takeaway 17.3 #3:** _Stringification with the operator `#` does not expand macros in its argument._

> **Takeaway 17.3 #4:** _Nested macro definitions may expand macro arguments several times._

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdbool.h>

// The two-level stringification trick
#define STRINGIFY(X) #X
#define STRGY(X) STRINGIFY(X)

// Macro that captures file, line, function, and the expression itself
#define TRACE_EXPR(X) \
    do { \
        fprintf(stderr, "%s:%s:%s: (%s) = %g\n", \
                __FILE__, STRGY(__LINE__), __func__, #X, (double)(X)); \
    } while (false)

int main(void) {
    double sum = 889.0;
    int argc = 2;

    // Expected output: file.c:20:main: (sum*argc) = 1778
    TRACE_EXPR(sum * argc);

    // Counter-example: Using single-level stringification
    // #define BAD_STR(X) #X
    // BAD_STR(__LINE__) results in the string "__LINE__", not "23".

    return 0;
}
```

**Recap:**

- Predefined macros like `__FILE__` and `__func__` provide valuable invocation context.
- The stringification operator (`#`) turns macro arguments into strings but blocks their internal macro expansion.
- A two-level macro is required to stringify the _value_ of another macro (like `__LINE__`).

---

## 17.4. Variable-length argument lists

### 17.4.1. Variadic macros

**What it is:** Macros that can accept an arbitrary number of arguments, denoted by `...` in the parameter list and accessed via `__VA_ARGS__` in the replacement text.

**Why it matters / how it works:** Variadic macros replace the need for highly unsafe variadic functions (like `printf`) in many contexts, allowing you to build wrappers around them safely. C23 introduced `__VA_OPT__`, which drastically improves handling of empty argument lists.

**Key details:**

- `...` must be the last parameter.
- `__VA_ARGS__` expands to the comma-separated list of provided arguments.
- **The empty list problem:** Prior to C23, passing no variable arguments to `MACRO(F, ...)` resulted in a dangling comma (e.g., `printf(F, )`), causing compile errors.
- **The C23 Solution (`__VA_OPT__`):** `__VA_OPT__(,)` expands to a comma _only if_ `__VA_ARGS__` is not empty.
- You can count arguments using complex macro lists (like the `ALEN` trick in the book) that map the length of `__VA_ARGS__` to a hardcoded decrementing list.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdbool.h>

// C23 variadic macro using __VA_OPT__
#define TRACE_PRINT8(F, ...) \
    do { \
        fprintf(stderr, "%s: " F "\n", \
                __func__ __VA_OPT__(,) __VA_ARGS__); \
    } while (false)

int main(void) {
    int sum = 889;
    int argc = 2;

    // Expected: uses __VA_OPT__(,) to insert a comma before sum
    TRACE_PRINT8("a collection: %d, %d", sum, argc);

    // Expected: __VA_ARGS__ is empty, __VA_OPT__(,) does NOT insert a comma.
    // Resulting code is valid: fprintf(stderr, "%s: " "another string" "\n", __func__);
    TRACE_PRINT8("another string");

    return 0;
}
```

### 17.4.2. A detour: Variadic functions

**What it is:** C functions declared with `...` (e.g., `int printf(char const*, ...);`) that process variable arguments at runtime using the `<stdarg.h>` header (`va_list`, `va_start`, `va_arg`, `va_end`).

**Why it matters / how it works:** They are mentioned mostly as a historical note and a warning. They lack type safety because the compiler doesn't know what types to convert the arguments to. It relies on "default argument promotions" (e.g., `float` becomes `double`, narrow integers become `int`).

**Key details:**

- `va_start(va)` initializes the list. (Since C23, the second argument to `va_start` is no longer required).
- `va_arg(va, type)` retrieves the next argument. You _must_ pass the correct type.
- `va_end(va)` cleans up.
- The function itself has no idea how long the list is; it must rely on a convention (like a format string or a length parameter).

**Pitfalls:**

- If `va_arg` requests a `double` but the user passed an `int`, the program reads memory incorrectly (Undefined Behavior).
- You cannot query the length of a `va_list`.

> **Takeaway 17.4.2 #1:** _When passed to a variadic parameter, all arithmetic types are converted as for arithmetic operations, with the exception of `float` arguments, which are converted to `double`._

> **Takeaway 17.4.2 #2:** _A variadic function has to receive valid information about the type of each argument in the variadic list._

> **Takeaway 17.4.2 #3:** _Using variadic functions is not portable unless each argument is forced to a specific type._

> **Takeaway 17.4.2 #5:** _The `va_arg` mechanism doesn’t give access to the length of the `va_list`._

> **Takeaway 17.4.2 #6:** _A variadic function needs a specific convention for the length of the list._

**Recap:**

- Variadic macros (`...` and `__VA_ARGS__`) are powerful and safer when combined with C23's `__VA_OPT__`.
- Variadic functions are legacy, type-unsafe, and require manual type decoding and length tracking. Avoid writing new variadic functions.

---

## 17.5. Default arguments

### Emulating Default Function Arguments

**What it is:** Using variadic macros and `__VA_OPT__` dispatching techniques to allow users to call standard library functions (like `strtoul`) with fewer arguments, automatically filling in defaults (like base `0` and `nullptr`).

**Why it matters / how it works:** C does not support default arguments in function signatures (unlike C++). However, macros can count the arguments provided and route the call to different expansions, appending missing defaults seamlessly.

**Key details:**

- The implementation relies on creating a cascade of macros: `DEFAULT1`, `DEFAULT2`, `DEFAULT3`.
- It splits the `__VA_ARGS__` list. If an argument is missing, the default is substituted.
- `ID_I`, `ID_II`, `ID_III` macros are used to evaluate and output the exact number of comma-separated arguments needed.
- This removes boilerplate (e.g., writing `nullptr, 0` every time you call `strtoul`).

**Pitfalls:**

- This is highly advanced macro magic. Creating these dispatch chains is prone to subtle syntactical errors.
- If the user provides _too many_ arguments, the macro cascade can fail cryptically.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

// A highly simplified version of default argument dispatch
// Let's create a macro for a custom function that expects 3 args,
// but we want to default the last two to NULL and 0.

void my_func_impl(const char* str, char** endptr, int base) {
    printf("Called with: %s, %p, %d\n", str, (void*)endptr, base);
}

// If 1 arg is passed: my_func(X) -> my_func_impl(X, NULL, 0)
// If 3 args are passed: my_func(X, Y, Z) -> my_func_impl(X, Y, Z)
// (Note: This is a simplistic implementation for illustration;
// the book's DEFAULT3 is much more robust for handling 1, 2, or 3 args).

#define my_func(...) my_func_dispatch(__VA_ARGS__, NULL, 0)
// This naive approach forces 3 arguments if only 1 is given,
// but it breaks if the user actually gives 3 (resulting in 5 args).
// The book uses __VA_OPT__ to count and branch correctly.

// Proper C23 book-style branching (simplified logic):
#define CALL_WITH_DEFAULTS(STR, ...) \
    my_func_impl(STR, __VA_OPT__(__VA_ARGS__) /* fallback logic omitted for brevity */ )

int main(void) {
    // Calling the implementation directly
    my_func_impl("123", NULL, 0);

    // The book's macro allows you to write:
    // strtoul(argv[1]);
    // Which expands to: strtoul(argv[1], nullptr, 0);
    return 0;
}
```

**Recap:**

- C lacks native default function arguments.
- Variadic macros and `__VA_OPT__` can be chained to emulate default arguments.
- This simplifies library usage at the cost of complex, under-the-hood macro infrastructure.

---

## Summary

Function-like macros in modern C are a double-edged sword. While they should be avoided in favor of `inline` functions whenever a strict, type-safe interface is possible, they remain vital for tasks that functions simply cannot perform. They operate in a preliminary textual translation phase, meaning they evaluate without any knowledge of C types. By leveraging operators like stringification (`#`), variadic arguments (`...` and `__VA_ARGS__`), and C23's `__VA_OPT__`, developers can build sophisticated debugging traces, type-generic programming interfaces, compile-time argument checks, and default-argument wrappers that vastly improve the developer experience and safety of standard library calls. Variadic functions, on the other hand, are largely obsolete due to their lack of type safety and should be avoided.

---

## Self-Check Questions

1. **Why should you generally prefer an `inline` function over a function-like macro for simple mathematical operations?**
   `inline` functions are type-safe and evaluate their arguments exactly once, preventing double-evaluation side effects.

2. **During which compilation phase are macros expanded?**
   In the preprocessor phase, before any semantic analysis or type checking occurs.

3. **What happens to the macro definition while its replacement text is being expanded?**
   It is temporarily disabled to prevent infinite recursion.

4. **How can you force the compiler to call an actual function named `strtoul` instead of expanding a macro of the same name?**
   Surround the function name in parentheses: `(strtoul)(...)`. This prevents the macro from expanding due to the macro retention rule.

5. **How do you force a macro argument `X` to be evaluated as a scalar type (integer, float, or pointer) to catch compile-time errors?**
   By adding `0` to it, e.g., `(X) + 0`.

6. **What is the purpose of the `#` operator in a macro?**
   It is the stringification operator; it converts the macro argument directly into a string literal.

7. **Why does `STRINGIFY(__LINE__)` result in the string `"__LINE__"` rather than the actual line number?**
   Because the `#` operator prevents the expansion of macros in its argument. A two-level macro (e.g., `STRGY(X)`) is needed to expand the argument first.

8. **What does `__VA_ARGS__` represent inside a variadic macro?**
   It represents the comma-separated list of all arguments passed to the `...` portion of the macro parameter list.

9. **What specific problem does the C23 `__VA_OPT__(,)` feature solve?**
   It solves the trailing comma problem when a variadic macro is called with an empty argument list by only inserting a comma if `__VA_ARGS__` contains arguments.

10. **What happens to a `float` argument when it is passed to a variadic function (not a macro)?**
    It undergoes default argument promotion and is automatically converted to a `double`.
