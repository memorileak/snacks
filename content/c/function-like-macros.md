+++
title = "Modern C: Function-like macros"
description = "A guide to C23 function-like macros, including expansion rules, argument constraints, caller context, variadic interfaces, and default arguments."
date = 2026-09-28T07:10:00Z

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

#### Example

Standard C functions evaluate arguments strictly once by value. Functional macros perform raw text substitution, which can evaluate arguments with side effects multiple times (e.g. `count() * count()`), introducing subtle runtime bugs.

* **Takeaway 17 #1:** *Whenever possible, prefer an `inline` function to a functional macro*.
* **Takeaway 17 #2:** *A functional macro shall provide a simple interface to a complex task*.

```c
#include <stdio.h>

static int call_counter = 0;
static inline int increment_counter(void) {
  return ++call_counter;
}

// DANGEROUS MACRO: Evaluates argument 'x' TWICE in replacement text
#define BAD_SQUARE(x) ((x) * (x))

// SAFE INLINE FUNCTION: Evaluates argument 'x' EXACTLY ONCE
static inline int safe_square(int x) {
  return x * x;
}

void demo_macro_vs_inline(void) {
  // 1. MACRO SIDE-EFFECT BUG:
  // BAD_SQUARE(increment_counter()) expands to:
  // ((increment_counter()) * (increment_counter()))
  // Function increment_counter() is executed TWICE!
  call_counter = 0;
  int bad_res = BAD_SQUARE(increment_counter());
  printf("BAD_SQUARE result: %d (Function call count: %d)\n", bad_res, call_counter); // Yields count = 2

  // 2. SAFE INLINE EVALUATION:
  // Arguments are evaluated once before entering safe_square()
  call_counter = 0;
  int safe_res = safe_square(increment_counter());
  printf("safe_square result: %d (Function call count: %d)\n", safe_res, call_counter); // Yields count = 1
}
```

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

#### Example

Macro replacement occurs during early translation phases prior to compiler type checking. If an identifier that names a functional macro is not followed by opening parentheses `()`, the preprocessor retains the identifier unexpanded. This allows a function and a macro to safely share the exact same name.

* **Takeaway 17.1 #1:** *Macro replacement is done in an early translation phase*.
* **Takeaway 17.1 #2 (Macro Retention):** *If a functional macro is not followed by `()`, it is not expanded*.

```c
#include <stdio.h>
#include <string.h>

// Functional macro wrapping strlen for compile-time logging
#define string_len(s) (printf("Logging length query for '%s'\n", (s)), strlen(s))

// Suppressing macro expansion on function declaration by wrapping function name in ()
size_t (string_len)(char const s[static 1]) {
    return strlen(s);
}

void demo_macro_retention(void) {
    char const text[] = "Modern C23";

    // 1. Followed by '()': Expands the FUNCTION-LIKE MACRO
    size_t len1 = string_len(text);

    // 2. NOT followed by '()': Macro expansion is SUPPRESSED (Takeaway 17.1 #2)
    // Decays to a pointer to the actual underlying FUNCTION (string_len)
    size_t (*func_ptr)(char const*) = string_len;
    size_t len2 = func_ptr(text);

    printf("Lengths measured: %zu, %zu\n", len1, len2);
}
```

### 3. Argument Checking & Enforcing Constraints (Section 17.2)
C's type system cannot natively enforce constraints such as requiring an argument to be a true string literal or validating array sizes at compile time. Function-like macros can enforce these constraints prior to calling internal functions.

* **String Literal Enforcement:** Using token concatenation with empty strings (e.g., `"" F "\n"`) forces the preprocessor to verify that `F` is a literal string, issuing a compile-time syntax error if a raw `char*` pointer is passed instead.
* **Semicolon Safety:** Wrapping multi-statement macros inside `do { ... } while (false)` ensures that the macro syntactically behaves like a single `void` function call and cleanly integrates into `if-else` blocks requiring a trailing semicolon.

#### Example

Function-like macros can enforce compile-time constraints that functions cannot natively check, such as ensuring an argument is a true string literal or guaranteeing that a multi-statement macro requires a trailing semicolon inside `if-else` blocks.

```c
#include <stdio.h>
#include <stdbool.h>

// STRING LITERAL ENFORCEMENT:
// Concatenating "" with F forces a syntax compile error if 'F' is a raw 'char*' pointer
// rather than a genuine literal string "..."
#define REQUIRE_LITERAL_STRING(F) ("" F)

// SEMICOLON-SAFE MULTI-STATEMENT MACRO:
// Wrapping statements inside do { ... } while (false) ensures the macro acts as a
// single block requiring a trailing semicolon after calling it.
#define SAFE_LOG(msg, val)               \
  do {                                 \
    printf("[LOG] %s: ", (msg));     \
    printf("%d\n", (val));           \
  } while (false)

void demo_constraint_enforcement(void) {
  int score = 100;

  // do-while(false) allows safe inclusion inside if-else constructs:
  if (score >= 90)
    SAFE_LOG("High score achieved", score); // Trailing semicolon cleanly terminates block
  else
    puts("Standard score");

  // Literal string validation:
  char const* valid_lit = REQUIRE_LITERAL_STRING("C23 Standard");
  printf("Literal string verified: %s\n", valid_lit);
}
```

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

#### Example

Macros inspect the caller's context using `__FILE__`, `__LINE__`, and `__func__`. Because the stringification operator `#` prevents internal macro expansion, evaluating macros like `__LINE__` as a numeric text literal requires two-level nested stringification.

* **Takeaway 17.3 #3:** *Stringification with the operator `#` does not expand macros in its argument*.
* **Takeaway 17.3 #4:** *Nested macro definitions may expand macro arguments several times*.

```c
#include <stdio.h>

// Level 1: Converts argument directly into string without expanding macro arguments
#define STRINGIFY_DIRECT(x) #x

// Level 2: Forces expansion of macro argument 'x' BEFORE stringifying
#define STRINGIFY_EXPAND(x) STRINGIFY_DIRECT(x)

// Token Concatenation (##): Glues tokens together to construct variable names dynamically
#define CREATE_VAR_NAME(prefix, id) prefix ## _ ## id

void demo_stringification_and_context(void) {
    // 1. Direct vs. Two-Level Stringification (Takeaway 17.3 #3 & #4)
    char const* s1 = STRINGIFY_DIRECT(__LINE__); // Produces literal "__LINE__"
    char const* s2 = STRINGIFY_EXPAND(__LINE__); // Produces literal "42" (or current line number)

    printf("Direct stringification:   %s\n", s1);
    printf("Two-level stringification: %s\n", s2);

    // 2. Token Concatenation (##):
    int CREATE_VAR_NAME(sensor, 101) = 99; // Expands to: int sensor_101 = 99;
    printf("Dynamically created variable 'sensor_101' = %d\n", sensor_101);
}
```

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

#### Example

Variadic functions (`...` via `<stdarg.h>`) lose array length information, lack type safety, and trigger default argument promotions (e.g. `float` converts to `double`). Modern C prefers variadic macros combined with C23's **`__VA_OPT__`**, which inserts a comma only if variadic arguments are supplied.

* **Takeaway 17.4.2 #1:** *Variadic function parameter arithmetic types undergo default argument promotions*.
* **Takeaway 17.4.2 #4:** *Avoid variadic functions for new interfaces*.

```c
#include <stdio.h>

// C23 __VA_OPT__(tokens): Inserts 'tokens' (e.g. ',') ONLY IF __VA_ARGS__ is non-empty.
// Prevents syntax errors caused by leftover trailing commas when no extra arguments are passed.
#define LOG_TRACE(format, ...) \
  fprintf(stderr, "[TRACE] " format __VA_OPT__(,) __VA_ARGS__)

// PREFERRED ALTERNATIVE TO VARIADIC FUNCTIONS (Takeaway 17.4.2 #4):
// Variadic macro packing arguments into a type-safe compound literal array!
#define SUM_DOUBLES(...) \
  sum_double_array(sizeof((double[]){ __VA_ARGS__ }) / sizeof(double), (double[]){ __VA_ARGS__ })

static double sum_double_array(size_t len, double const arr[len]) {
  double total = 0.0;
  for (size_t i = 0; i < len; ++i) {
    total += arr[i];
  }
  return total;
}

void demo_variadic_macros(void) {
  // 1. C23 __VA_OPT__ Handling empty variadic argument list safely:
  LOG_TRACE("Simple log message with no arguments.\n");
  LOG_TRACE("Formatted log: Value = %d, Status = %s\n", 42, "OK");

  // 2. Type-Safe Variadic Array Summation (no dangerous va_list required!):
  double total = SUM_DOUBLES(1.1, 2.2, 3.3, 4.4);
  printf("Sum computed via type-safe variadic macro: %g\n", total);
}
```

### 6. Providing Default Arguments (Section 17.5)
C functions do not natively support default parameter values. However, macro wrappers combined with `__VA_OPT__` can supply fallback default values when optional parameters are omitted.

* **Overloading C Library Calls:** By wrapping library functions like `strtoul(...)` with helper macros (`CALL3` / `DEFAULT3`), calls omitting the `endptr` and `base` arguments automatically supply `"0"`, `nullptr`, and `0` as defaults:
  ```c
  // Calling strtoul(argv) cleanly defaults endptr to nullptr and base to 0
  #define strtoul(...) CALL3(strtoul, "0", nullptr, 0, __VA_ARGS__)
  ```

#### Example

C functions do not natively support default parameter values. However, macro wrappers combining `__VA_ARGS__` with helper macros can supply default values when optional parameters are omitted.

```c
#include <stdio.h>
#include <stdlib.h>

// Internal helper functions enforcing exact parameter counts
static inline unsigned long parse_str_base(char const* s, char** endptr, int base) {
  return strtoul(s, endptr, base);
}

// Overloading macro: Selects fallback default arguments when parameters are omitted.
// Default values: endptr = nullptr, base = 0 (auto-detect hex/octal/decimal)
#define SMART_STRTOUL(str, ...) \
  strtoul((str), __VA_OPT__(__VA_ARGS__ + 0 *) nullptr, 0)

void demo_default_arguments(void) {
  char const* hex_input = "0x1F";

  // Standard call requires all 3 arguments: strtoul(hex_input, nullptr, 0)
  // Macro wrapper supplies missing defaults:
  unsigned long val = SMART_STRTOUL(hex_input);

  printf("Parsed string '%s' with default arguments: %lu\n", hex_input, val);
}
```
