+++
title = "Modern C: Function-like macros"
description = ""
date = 2026-09-28

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 17: Function-like macros** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book moves into **Level 3: Experience**. While macros can easily obfuscate source code or introduce subtle side-effect bugs if used carelessly, C23 leverages function-like macros to build type-safe, expressive, and flexible interfaces that functions alone cannot easily achieve.

Below is a detailed breakdown of the core concepts, expansion mechanics, variadic utilities, and key takeaways from Chapter 17 across its five main sections:


### 1. General Principles: Macros vs. Inline Functions (Section 17)
The primary rule of modern C macro design is to use function-like macros only when standard C functions are insufficient.

* **Prefer `inline` Functions:**
  * **Takeaway 17 #1:** *Whenever possible, prefer an `inline` function to a functional macro*.
  * Functions evaluate arguments exactly once, whereas a macro can evaluate an argument expression with side effects multiple times (e.g., `SQUARE(count())` expands to `count() * count()`, triggering two invocations). An `inline` function eliminates call overhead while maintaining strict call-by-value evaluation.
* **Macro Purpose:**
  * **Takeaway 17 #2:** *A functional macro shall provide a simple interface to a complex task*.
  * Function-like macros excel at compile-time argument checking, context inspection (file/line logging), variadic parameter lists, type-generic selection, and default argument handling.


### 2. How Function-Like Macros Work (Section 17.1)
Macros operate purely via **textual replacement** during preprocessing, before the compiler assigns grammatical or type semantics to tokens.

* **Early Phase Token Replacement:**
  * **Takeaway 17.1 #1:** *Macro replacement is done in an early translation phase, before any other interpretation is given to the tokens that compose the program*.
* **Expansion Rules:**
  1. The macro definition is temporarily disabled during its own replacement to prevent infinite recursion.
  2. Arguments inside the outer parentheses `()` are split by unnested commas.
  3. Arguments are recursively expanded for other macros.
  4. Parameter occurrences in the replacement text are substituted with the expanded argument fragments.
  5. The resulting replacement text is rescanned for further macro expansions.
* **Macro Retention:**
  * **Takeaway 17.1 #2 (Macro Retention):** *If a functional macro is not followed by `()`, it is not expanded*.
  * This allows a function and a macro to share the exact same name. Enclosing a function declaration name in parentheses—e.g., `char const* (string_literal)(char const str[static 1])`—suppresses macro expansion during function declaration. Passing a function name without `()` decays it to a function pointer without expanding the macro.


### 3. Argument Checking & Enforcing Constraints (Section 17.2)
C's type system cannot natively enforce constraints such as requiring an argument to be a true string literal or validating array sizes at compile time. Function-like macros can enforce these constraints prior to calling internal functions.

* **String Literal Enforcement:** Using token concatenation with empty strings (e.g., `"" F "\n"`) forces the preprocessor to verify that `F` is a literal string, issuing a compile-time syntax error if a raw `char*` pointer is passed instead.
* **Semicolon Safety:** Wrapping multi-statement macros inside `do { ... } while (false)` ensures that the macro syntactically behaves like a single `void` function call and cleanly integrates into `if-else` blocks requiring a trailing semicolon.


### 4. Accessing the Context of Invocation (Section 17.3)
Macros can inspect the caller's execution environment using predefined macros: `__FILE__` (source filename), `__LINE__` (line number), and `__func__` (enclosing function name).

* **Line Number Mismatches:**
  * **Takeaway 17.3 #1:** *The line number in `__LINE__` may not fit into an `int`*.
  * **Takeaway 17.3 #2:** *Using `__LINE__` is inherently dangerous*.
* **Stringification Operator (`#`):**
  * Placing `#` before a macro parameter converts the raw textual argument directly into a string literal.
  * **Takeaway 17.3 #3:** *Stringification with the operator `#` does not expand macros in its argument*. Writing `# __LINE__` produces the string `"__LINE__"` rather than `"25"`.
  * **Takeaway 17.3 #4:** *Nested macro definitions may expand macro arguments several times*. To stringify `__LINE__` as a numeric text literal, code must use a two-level nested macro:
    ```c
    #define STRINGIFY(X) #X
    #define STRGY(X) STRINGIFY(X) // Expands __LINE__ to '25' before stringifying to "25"
    ```
* **Token Concatenation (`##`):** Glues adjacent preprocessing tokens together to generate new identifier names dynamically.


### 5. Variable-Length Argument Lists & C23 `__VA_OPT__` (Section 17.4)

#### Variadic Macros (Section 17.4.1)
Variadic macros accept arbitrary parameter counts using `...` and expand them via `__VA_ARGS__`.

* **C23 `__VA_OPT__` Feature:** In older C standards, omitting trailing variadic arguments caused syntax errors due to left-over trailing commas in functions like `fprintf(stderr, format, __VA_ARGS__)`.
* C23 introduces **`__VA_OPT__(tokens)`**, which expands its enclosed `tokens` (such as a comma `,`) **only if `__VA_ARGS__` is non-empty**:
  ```c
  #define TRACE_PRINT(F, ...) \
      fprintf(stderr, F __VA_OPT__(,) __VA_ARGS__)
  ```

#### Variadic Functions & Why to Avoid Them (Section 17.4.2)
Variadic functions (declared with `...` and accessed via `<stdarg.h>`) are prone to runtime bugs due to a lack of type safety.

* **Default Argument Promotions:**
  * **Takeaway 17.4.2 #1:** *When passed to a variadic parameter, all arithmetic types are converted as for arithmetic operations, with the exception of `float` arguments, which are converted to `double`*. Narrow integer types (`char`, `short`) promote to `int`.
* **Type Safety & Portability:**
  * **Takeaway 17.4.2 #2 & #3:** *A variadic function has to receive valid information about the type of each argument in the variadic list*, and *using variadic functions is not portable unless each argument is forced to a specific type*.
  * **Takeaway 17.4.2 #4:** *Avoid variadic functions for new interfaces*. Instead, prefer variadic macros that pack arguments into compound literal array initializers (e.g., `(long double const[NARGS]){ __VA_ARGS__ }`) where conversions are verified at compile time.
* **`va_list` Limitations:**
  * **Takeaway 17.4.2 #5 & #6:** *The `va_arg` mechanism doesn’t give access to the length of the `va_list`*, and *a variadic function needs a specific convention for the length of the list*.
  * **C23 `<stdarg.h>` Update:** In C23, `va_start(ap)` no longer requires the second parameter specifying the last named argument.


### 6. Providing Default Arguments (Section 17.5)
C functions do not natively support default parameter values. However, macro wrappers combined with `__VA_OPT__` can supply fallback default values when optional parameters are omitted.

* **Overloading C Library Calls:** By wrapping library functions like `strtoul(...)` with helper macros (`CALL3` / `DEFAULT3`), calls omitting the `endptr` and `base` arguments automatically supply `"0"`, `nullptr`, and `0` as defaults:
  ```c
  // Calling strtoul(argv) cleanly defaults endptr to nullptr and base to 0
  #define strtoul(...) CALL3(strtoul, "0", nullptr, 0, __VA_ARGS__)
  ```
