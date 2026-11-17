+++
title = "C Macro: From Simple to Complex"
description = "A simple to complex example of C macro usage."
date = 2026-11-17T10:18:39+00:00

[taxonomies]
tags = ["macro"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

A C macro is a preprocessor directive (`#define`) that performs text substitution on your source code before compilation. Mastering macros involves progressing from basic constant replacements to advanced code-generation techniques, while carefully avoiding common syntax pitfalls.

## Level 1: Object-Like Macros (Constants & Flags)

At the simplest level, macros act as symbolic constants or empty flags.

**Symbolic Constants**. Replace magic numbers with readable names. Do not end `#define` lines with a semicolon.

```c
#define MAX_BUFFER 1024
#define PI 3.14159
```

**Conditional Flags**. Define an empty macro to toggle features or debug logs on and off during compilation.

```c
#define DEBUG_MODE

#ifdef DEBUG_MODE
    printf("Debug info enabled\n");
#endif
```

**Predefined Macros**. The compiler automatically provides built-in macros like `__FILE__`, `__LINE__`, `__DATE__`, and `__TIME__` for diagnostics.

## Level 2: Function-Like Macros & Parentheses Hygiene

Macros can accept parameters to act like inline functions without function-call overhead. Because macros do blind text substitution, operator precedence bugs are common unless you wrap every parameter and the entire expression in parentheses.

```c
// BAD: SQUARE(2 + 3) expands to 2 + 3 * 2 + 3 = 11
#define BAD_SQUARE(x) x * x

// GOOD: ((2 + 3) * (2 + 3)) = 25
#define SQUARE(x) ((x) * (x))
#define MAX(a, b) ((a) > (b) ? (a) : (b))
```

**Watch Out for Side Effects**: Never pass expressions with increment/decrement operators (like `MAX(i++, j++)`) into a macro. The argument gets evaluated twice.

## Level 3: Multi-Line Macros & The `do { ... } while (0)` Idiom

You can write multi-line macros using the backslash (`\`) line-continuation character. When wrapping multiple statements, use `do { ... } while (0)` so the macro behaves like a single C statement inside if/else blocks.

```c
// Safe multi-line macro
#define SWAP(a, b, type) do { \
    type temp = (a);          \
    (a) = (b);                \
    (b) = temp;               \
} while (0)

// Usage
if (x > y)
    SWAP(x, y, int); // Works cleanly without breaking an 'else' branch
else
    printf("No swap needed\n");
```

## Level 4: Stringification (`#`) and Token Pasting (`##`)

According to the GCC Preprocessor Documentation, two special operators allow macros to manipulate code tokens directly:

**Stringification (`#`)**. Converts a macro argument into a string literal.

```c
#define PRINT_VAR_INT(var) printf(#var " = %d\n", (var))

int score = 95;
PRINT_VAR_INT(score); // Expands to: printf("score" " = %d\n", (score));
```

**Token Pasting (`##`)**. Concatenates two tokens into a single identifier. This is heavily used in embedded systems and code generation.

```c
#define MAKE_COMMAND(name) cmd_##name

void cmd_start(void) { /* ... */ }
void cmd_stop(void)  { /* ... */ }

// Calls cmd_start()
MAKE_COMMAND(start)();
```

## Level 5: Variadic Macros (`__VA_ARGS__`)

Variadic macros accept a variable number of arguments using an ellipsis (`...`), similar to printf. The Microsoft C/C++ Preprocessor Reference notes that `__VA_ARGS__` expands to all extra arguments passed.

```c
// Custom logging macro with file and line metadata
#define LOG_ERR(fmt, ...) \
    fprintf(stderr, "[ERROR] %s:%d: " fmt "\n", __FILE__, __LINE__, ##__VA_ARGS__)

// Usage
LOG_ERR("Failed to open file");
LOG_ERR("Connection failed with code %d", 404);
```

**Note**: Using `##__VA_ARGS__` in GCC/Clang or `__VA_OPT__(,)` in C23 cleanly removes the trailing comma when zero extra arguments are passed.

## Level 6: X-Macros & Type-Generic Macros (`_Generic`)

At the highest level of complexity, macros generate repetitive data structures or simulate function overloading.

**X-Macros**. Define a list of items once, then redefine an `X` macro multiple times to generate enums, string arrays, and switch cases in sync.

```c
// 1. Define the data table
#define COLOR_LIST    \
    X(RED,   "Red")   \
    X(GREEN, "Green") \
    X(BLUE,  "Blue")

// 2. Generate Enum
typedef enum {
    #define X(id, name) COLOR_##id,
    COLOR_LIST
    #undef X
} Color;

// 3. Generate String Lookup Table
const char* color_names[] = {
    #define X(id, name) name,
    COLOR_LIST
    #undef X
};
```

**C11 `_Generic` Dispatch**. Combine macros with C11's `_Generic` keyword to create type-safe, overloaded macros.

```c
#define PRINT_ANY(x) _Generic((x), \
    int:    printf("%d\n", (x)),   \
    double: printf("%.2f\n", (x)), \
    char*:  printf("%s\n", (x))    \
)
```

## Summary of Evolution

| Level            | Feature               | Primary Use Case                             | Key Risk / Rule                               |
| ---------------- | --------------------- | -------------------------------------------- | --------------------------------------------- |
| 1. Object-Like   | `#define PI 3.14`     | Constants and feature toggles                | Never add a trailing semicolon                |
| 2. Function-Like | `#define MAX(a,b)`    | Inline math/logic shortcuts                  | Wrap every variable and expression in `()`    |
| 3. Multi-Line    | `do { ... } while(0)` | Multi-statement helper blocks                | Avoid double-evaluating side effects (`i++`)  |
| 4. Token Ops     | `#` and `##`          | Debugging logs and dynamic symbol names      | Arguments are not expanded before `#` or `##` |
| 5. Variadic      | `...` / `__VA_ARGS__` | Custom printf-style logging wrappers         | Handle empty variadic arguments carefully     |
| 6. Meta-Macros   | X-Macros & `_Generic` | Synchronized enums/tables & type overloading | Harder to debug; inspect with `gcc -E`        |
