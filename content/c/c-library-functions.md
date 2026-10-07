+++
title = "Modern C: C library functions"
description = "An overview of the C23 standard library, including error handling, checked arithmetic, bit operations, math, I/O, strings, time, environment access, and program termination, with commented examples."
date = 2026-09-28T03:24:25Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

The C standard is divided into two major components: the core C language and the C library. The library provides fundamental tools and features essential for everyday programming, ensuring standardized interfaces and portability across different platforms. By offering a clear application programming interface (API), it allows the separation of the compiler implementation (such as gcc or clang) from the library implementation (such as glibc, dietlibc, or musl).

## 8.1. General properties of the C library and its functions

### 8.1.1. Headers

**What it is:**
The C library encompasses a vast array of functions whose interface descriptions are bundled into files known as **headers**. These header files organize related features and functions so they can be easily included in a program.

**Why it matters / how it works:**
By standardizing these interfaces, headers act as a platform abstraction layer. They abstract away platform-specific needs (like basic IO operations) that require deep, OS- or processor-specific knowledge to implement.

**Key details:**

- Headers bundle function interfaces logically (e.g., `<stdio.h>` for input/output, `<math.h>` for numerics, `<time.h>` for time manipulation).
- Standard headers include `<stdbool.h>` for booleans, `<stdint.h>` for exact-width integer types, and `<stdlib.h>` for basic functions.
- C23 introduces new headers such as `<stdbit.h>` for bit operations and `<stdckdint.h>` for checked integer arithmetic.

**Pitfalls:**

- Failing to include the appropriate header can lead to undefined behavior or compilation errors because the compiler will lack the necessary interface descriptions for the functions used.

### 8.1.2. Interfaces

**What it is:**
Interfaces in the C library are primarily specified as functions, but implementations are permitted to realize them as **function-like macros** where appropriate. A function-like macro is a syntactic construct that resembles a function but is implemented via textual replacement.

**Why it matters / how it works:**
A macro replaces text directly during preprocessing. For instance, `putchar(A)` could be defined as `#define putchar(A) putc(A, stdout)`.

**Key details:**

- Function-like macros behave structurally like functions but are strictly textual replacements.

**Pitfalls:**

- Because macros substitute text, an argument passed to a macro might be evaluated multiple times if it appears multiple times in the replacement text.
- Passing an expression with side effects to a macro can lead to unpredictable state changes.

### 8.1.3. Error checking

**What it is:**
C library functions typically indicate failure by returning a special value, though the specific value varies depending on the function. The C standard also maintains a global state variable, `errno`, to track errors for certain functions.

**Why it matters / how it works:**
Proper error handling ensures that bugs are detected early and program integrity is maintained. Functions might return a null pointer, a special error code (like `EOF`), a nonzero value, or a special success code depending on their design.

> Takeaway 8.1.3 #2 _Check the return value of library functions for errors._

By verifying return values immediately, developers can intercept failures before they cause cascading issues in the abstract state machine.

> Takeaway 8.1.3 #3 _Fail fast, fail early, and fail often._

Immediate program failure upon encountering an error is frequently the most effective way to detect and rectify bugs early in the development lifecycle.

**Key details:**

- `fopen` returns a null pointer on failure.
- Functions like `puts`, `clock`, `mktime`, `strtod`, and `fclose` return a special error code.
- `fgetpos` and `fsetpos` return a nonzero value on failure.
- `thrd_create` returns a special success code.
- `perror` utilizes the `errno` state to provide diagnostic error messages.

**Pitfalls:**

- If a function fails but the program recovers, `errno` must be manually reset to `0`; otherwise, subsequent library function calls or error checks might misinterpret the stale error state.

### 8.1.4. Bounds-checking interfaces

**What it is:**
Many standard C functions are susceptible to **buffer overflow** vulnerabilities if invoked with inconsistent parameters. To address this, C provides optional bounds-checking interfaces specified in Annex K.

**Why it matters / how it works:**
These functions are designed to mitigate security bugs and exploits by verifying that passed arguments (like pointers) are valid and consistent.

> Takeaway 8.1.4 #1 _Identifier names terminating with_ `_s` _are reserved._

Because bounds-checking functions (e.g., `printf_s` replacing `printf`) use the `_s` suffix, programmers must not use this suffix for their own identifiers to avoid collisions.

**Key details:**

- Bounds-checking functions generally mirror standard functions but append `_s` to the name (e.g., `fopen_s`).
- If these functions detect an inconsistency, known as a **runtime constraint violation**, they typically terminate program execution after outputting a diagnostic message.

### 8.1.5. Platform preconditions

**What it is:**
While C prioritizes portability, code sometimes relies on specific execution platform characteristics. Preprocessor conditionals allow programs to assert platform properties at compile time.

> Takeaway 8.1.5 #2 _In a preprocessor conditional, only evaluate macros and integer literals._

> Takeaway 8.1.5 #3 _In a preprocessor conditional, unknown identifiers evaluate to 0._

These rules govern how `#if` and `#ifdef` statements evaluate conditions before the code is actually compiled.

**Recap:**

- Header files define the API for C library functions, abstracting platform-specific operations.
- Interfaces may be functions or function-like macros, which require care to avoid multiple evaluations of side effects.
- Return values must be checked constantly to fail fast, and `errno` must be managed carefully.
- Annex K provides `_s` suffixed functions to protect against buffer overflows.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdbool.h>
#include <errno.h>

// Demonstrating Takeaway 8.1.3 #2 and #3, and proper errno handling.
void puts_safe(char const s[static 1]) {
    static bool failed = false;
    // Check return value against EOF to handle potential output errors
    if (!failed && puts(s) == EOF) {
        perror("can't output to terminal:"); // Uses errno under the hood
        failed = true;
        errno = 0; // Reset error state so future calls aren't confused
    }
}

int main(void) {
    puts_safe("Hello, secure world!");
    return 0;
}

```

_Expected behavior:_ The string is printed to the terminal. If standard output is unexpectedly closed (causing `puts` to return `EOF`), it prints an error message and safely resets `errno`.

---

## 8.2. Integer arithmetic

**What it is:**
While standard operators handle most integer arithmetic, the C library provides additional functions in headers like `<stdbit.h>` for specialized bit-level operations and functions for cases requiring overflow detection.

**Why it matters / how it works:**
These functions allow for explicit and safe calculations, particularly in C23, which introduced new type-generic macros for bit manipulation. Hardware may implement these via specific instructions, but the standard library guarantees defined results for all arguments, relieving the programmer from handling edge cases.

**Key details:**

- Functions like `abs`, `labs`, and `llabs` calculate the absolute value `|x|`.
- Checked arithmetic functions like `ckd_add`, `ckd_sub`, and `ckd_mul` perform operations and yield an overflow flag.
- `<stdbit.h>` includes type-generic macros whose results are independent of the argument type, such as `stdc_bit_floor`, `stdc_bit_width`, `stdc_count_ones`, `stdc_has_single_bit`, `stdc_first_trailing_one`, and `stdc_first_trailing_zero`.
- Other functions depend on the width of the argument type, such as `stdc_bit_ceil`, `stdc_count_zeros`, and `stdc_leading_zeros`.

**Pitfalls:**

- The results of width-dependent functions (`stdc_count_zeros`, `stdc_leading_ones`, etc.) can be difficult for code readers to interpret across different platforms; they should be avoided if possible in favor of type-independent variants.

**Recap:**

- The C library extends basic arithmetic operators with specialized functions for absolute values, division, and checked operations.
- C23's `<stdbit.h>` provides extensive bit manipulation macros.
- Type-generic bit functions are preferred over width-dependent ones for readability and portability.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdbit.h> // Requires C23

int main(void) {
    unsigned int val = 0b01011000;

    // Type-independent macro usage
    int ones = stdc_count_ones(val);
    // stdc_first_trailing_zero returns 1 plus the index of the LS 0-bit
    int trail_zero = stdc_first_trailing_zero(val);

    printf("Value has %d ones.\n", ones);
    return 0;
}

```

_Expected behavior:_ Computes that `0b01011000` has 3 set bits.

---

## 8.3. Numerics

**What it is:**
Numerical functions are provided via `<math.h>`, but C offers type-generic macros in `<tgmath.h>` to simplify their usage.

**Why it matters / how it works:**
Instead of memorizing and calling `cosf` for floats, `cos` for doubles, and `cosl` for long doubles, `<tgmath.h>` dispatches a single macro invocation (e.g., `sin(x)`) to the appropriate function based on the argument's type, returning a value of that identical type.

**Key details:**

- Provides trigonometric functions (`acos`, `asin`, `atan2`), hyperbolic functions (`acosh`, `asinh`), and newly in C23, variants divided by $\pi$ (`acospi`, `asinpi`).
- Includes utility math like `fma` (floating-point multiply-add), `fmax`, `fminimum`, `fmod`, and rounding functions.
- `frexp` separates a floating point into its significand and exponent.

**Recap:**

- `<math.h>` provides the underlying math functions.
- `<tgmath.h>` provides type-generic macros that automatically select the correct function variant based on the passed type.

---

## 8.4. Input, output, and file manipulation

### 8.4.1. Unformatted text output

**What it is:**
The `<stdio.h>` header provides basic tools for output. `putchar` writes a single character, while `puts` writes a string followed by a newline. These operate on opaque standard streams like `stdout` and `stderr`.

**Why it matters / how it works:**
`stdout` is meant for standard output, while `stderr` is for urgent or error output; having both allows the program to separate standard results from diagnostics.

> Takeaway 8.4.1 #1 _Opaque types are specified through functional interfaces._

> Takeaway 8.4.1 #2 _Don't rely on implementation details of opaque types._

The `FILE` type representing streams is opaque; you interact with it entirely via functions without directly accessing its internal struct members.

> Takeaway 8.4.1 #3 `puts` _and_ `fputs` _differ in their end-of-line handling._

While `puts(s)` appends a newline character automatically to standard output, `fputs(s, stream)` writes the exact string without appending an additional newline.

### 8.4.2. Files and streams

**What it is:**
To interact with external files, they must be attached to the program via `fopen`, which links a file to a `FILE*` stream based on specific mode strings.

**Key details:**

- Base modes: `'r'` (read; file unmodified, starts at beginning), `'w'` (write; wipes file content, starts at beginning), `'a'` (append; file unmodified, starts at end).
- Modifiers: `'+'` (update; opens for both reading and writing), `'b'` (binary), `'x'` (exclusive; creates file for writing only if it doesn't exist).
- Bounds-checking versions `fopen_s` and `freopen_s` ensure valid pointer arguments.

### 8.4.3. Text output conversion & flushing

> Takeaway 8.4.3 #1 _Text input and output converts data._

> Takeaway 8.4.3 #2 _There are three commonly used conversion to encode end-of-line._ (Extracted from summary list).

**Key details:**

- Output streams are buffered. C library functions provide mechanisms (like `fflush` implicitly or explicitly) to force data out to the device.
- Filesystem manipulation functions include `remove` and `rename` to delete or rename files directly via the C library.

### 8.4.4. Formatted output

**What it is:**
`fprintf` functions identically to `printf` but accepts a target stream parameter, enabling formatted writes to files.

> Takeaway 8.4.4 #3 _Use the_ `"%b"` _or_ `"%x"` _formats to print bit patterns._

**Key details:**

- Hexadecimal (`%x`) and binary (`%b`) formats correspond to unsigned values and are optimal for viewing raw bit sets.
- Optional interfaces `printf_s` and `fprintf_s` verify that stream and format pointers are valid (though they do not fully validate the format specifiers against the argument list).

**Pitfalls:**

- Mixing formatted output streams connected to the same terminal without care can cause interleaved, garbled output.

### 8.4.5. Unformatted text input

**What it is:**
Input is performed using `fgetc` for single characters and `fgets` for strings. `stdin` is the standard stream connected to terminal input.

> Takeaway 8.4.5 #1 _Don't use_ `gets`. (From summary list).

The `gets` function is intrinsically unsafe regarding buffer overflows.

> Takeaway 8.4.5 #2 `fgetc` _returns_ `int` _to be able to encode a special error status,_ `EOF`, _in addition to all valid characters._

If `fgetc` returned a `char`, it would not be able to distinctively return `EOF` (usually `-1`) without overlapping with a valid character code.

> Takeaway 8.4.5 #3 _End-of-file can only be detected_ after _a failed read._

Receiving `EOF` does not explicitly mean the stream has reached its end; it could denote a read error. The `feof(stream)` function must be called _after_ an operation returns `EOF` to confirm the file's end marker was hit.

**Recap:**

- `puts`/`putchar` handle basic standard output, while `fputs`/`fgetc` offer finer control over streams.
- `FILE` pointers are opaque handlers for managing files via `fopen` using various mode/modifier combinations.
- `fgetc` returns `int` to allow returning `EOF`, which must be followed by `feof` to verify the end of the file.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    FILE* stream = fopen("test.txt", "r");
    if (!stream) {
        perror("Failed to open");
        return EXIT_FAILURE;
    }

    int val = fgetc(stream); // Returns int to accommodate EOF
    if (val == EOF) {
        if (feof(stream)) {
            puts("File is empty (EOF reached immediately).");
        } else {
            puts("A read error occurred.");
        }
    }

    fclose(stream);
    return EXIT_SUCCESS;
}

```

_Expected behavior:_ Attempts to read a character. Correctly differentiates between a genuine read error and reaching the end of the file using `feof`.

---

## 8.5. String processing and conversion

**What it is:**
The C library provides utilities in `<ctype.h>` for classifying and converting single characters, and utilities in `<stdlib.h>` and `<string.h>` for parsing strings into numbers and searching substrings.

**Why it matters / how it works:**
Strings in C are character arrays terminated by a null character. Because C operates heavily on raw bytes, these utilities provide standard, portable ways to interpret text.

**Key details:**

- `<ctype.h>` includes classifier functions like `isalnum`, `isalpha`, `isblank`, `isdigit`, `isspace`, and conversions like `toupper` and `tolower`.
- For historical reasons, classifier functions take `int` arguments and return `int`.
- `<stdlib.h>` functions like `strtoul`, `strtod`, and `strtoumax` parse strings into numbers. `strtoul` takes `base` as an argument (0, 2, 8, 10, or 16). Base 0 automatically interprets prefixes (`0x` for hex, `0b` for binary).
- `<string.h>` provides `strspn` (returns length of initial sequence consisting of specified characters) and `strcspn` (length of initial sequence _not_ consisting of specified characters).

> Takeaway 8.5 #1 _The interpretation of numerically encoded characters depends on the execution character set._ (From summary list).

### 8.5.1. Portability of string processing

> Takeaway 8.5.1 #1 _Don't use the string conversion functions to determine the boundaries of numbers._ (From summary list).

> Takeaway 8.5.1 #2 _Don't use the string conversion functions to scan numbers that originate from number literals._ (From summary list).

**Pitfalls:**

- String conversion functions have changed semantics across C standard versions and are not perfectly consistent with how C string literals for numbers are evaluated natively.

**Recap:**

- Use `<ctype.h>` for character-by-character analysis (e.g., `isdigit`).
- Use `strtoul`/`strtod` to convert user-input strings into variables.
- String conversions must be used carefully due to historical semantic shifts.

---

## 8.6. Time

**What it is:**
The `<time.h>` header provides functions to query and manipulate time. It handles both physical time (seconds/nanoseconds) and calendar time (structured for human interpretation).

**Key details:**

- `time_t` is generally used for physical time representation.
- `struct tm` holds structured calendar time (year, month, day, etc.).
- Functions include `timegm`, `gmtime_r`, and `localtime_r` to convert between raw physical time and structured calendar time.
- `mktime` converts a `struct tm` back to a `time_t`.

**Recap:**

- Time is bifurcated into physical time and calendar time.
- Thread-safe, reentrant functions ending in `_r` (like `localtime_r`) are provided to safely translate `time_t` into `struct tm`.

---

## 8.7. Runtime environment settings

**What it is:**
Standard C provides rudimentary interfaces to read the environment in which the program was launched. This is primarily done using `getenv` and internationalization parameters via `setlocale`.

**Why it matters / how it works:**
Environment variables dictate configurations (like paths and languages) dynamically at launch. `getenv("PATH")` fetches these key-value pairs from the OS.

**Key details:**

- `getenv` retrieves the value of an environment variable.
- `getenv_s` is a bounds-checked alternative that ensures the target buffer is only written to if the environment value fits.
- `setlocale` accepts categories like `LC_COLLATE` (string comparison), `LC_CTYPE` (character classification), `LC_TIME` (time formatting), and `LC_ALL` to adapt the program to local languages and conventions.

**Recap:**

- `getenv` and `getenv_s` interface with system variables.
- `setlocale` modifies internal state affecting IO and text classifications.

---

## 8.8. Program termination and assertions

**What it is:**
C offers multiple ways to halt a program intentionally or diagnostically.

> Takeaway 8.8 #1 _Regular program termination should use a_ `return` _from_ `main`.

A standard `return` cleanly unwinds the `main` function and passes the return code back to the environment.

> Takeaway 8.8 #2 _Use_ `exit` _from a function that may terminate the regular control flow._

If deep within a call stack and a fatal error occurs, `exit` can be called. However, using `exit` directly inside `main` is discouraged because a simple `return` suffices.

> Takeaway 8.8 #3 _Don't use functions other than_ `exit` _for program termination, unless you have to inhibit the execution of library cleanups._

Functions like `quick_exit`, `_Exit`, and `abort` terminate the program abruptly without executing all standard library cleanups (like flushing open streams or running functions registered with `atexit`).

> Takeaway 8.8 #4 _Use as many_ `asserts` _as you can to confirm runtime properties._ (From summary list).

> Takeaway 8.8 #5 _In production compilations, use_ `NDEBUG` _to switch off all_ `asserts`. (From summary list).

**Key details:**

- The `<assert.h>` header provides the `assert` macro, which evaluates a condition at runtime. If the condition is false, `assert` forcefully terminates the program (usually via `abort`) and prints diagnostic information.
- Defining the `NDEBUG` macro before including `<assert.h>` compiles assertions out entirely, making them zero-cost in production.

**Recap:**

- Always prefer returning from `main` to terminate cleanly.
- Use `exit` if deep in a function hierarchy, but avoid `abort` or `_Exit` unless bypassing library cleanup is explicitly required.
- Use `assert` liberally during development to codify invariants, disabling them for release.

**Code Demonstration:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

void perform_critical_task(int value) {
    // Assert invariant: value must be positive
    assert(value > 0);

    if (value == 999) {
        puts("Fatal condition hit, terminating from deep within stack.");
        exit(EXIT_FAILURE); // Proper use of exit outside of main
    }
}

int main(void) {
    perform_critical_task(10);
    // Regular termination uses return
    return EXIT_SUCCESS;
}

```

_Expected behavior:_ Exits successfully. If `perform_critical_task(-1)` were called, `assert` would trigger an abort. If `perform_critical_task(999)` were called, `exit` triggers a clean but premature failure shutdown.

---

## Summary

The C library acts as an essential platform abstraction layer, standardizing operations that require deep system knowledge, accessed through well-defined headers. It provides robust mechanisms for integer arithmetic, type-generic numerical processing, and comprehensive input/output manipulation using opaque stream interfaces. The library ensures text processing portability via character classification and string functions, whilst supplying structures and macros for manipulating system time and runtime environment properties. Finally, the library defines safe practices for error tracking via `errno` and program termination, emphasizing clean returns from `main` and the use of runtime `assert`s to catch logic errors early during development.

## Self-Check Questions

1. **What is the difference between a library function and a function-like macro?**
   _Answer:_ A function is an independent executable block, whereas a macro is a preprocessor construct that substitutes text directly into the code, which can risk evaluating side-effect arguments multiple times.

2. **Why should you reset `errno` to 0 after recovering from an error?**
   _Answer:_ Because subsequent library functions might check `errno`; if it is not cleared, they might read the old error state and behave unpredictably.

3. **What is the purpose of functions ending with `_s` in Annex K?**
   _Answer:_ They are bounds-checking interfaces designed to prevent buffer overflows and ensure parameter consistency.

4. **Why are `<stdbit.h>` type-generic macros preferred over width-dependent ones?**
   _Answer:_ Type-generic macros are independent of the exact width of the argument's type, making the code much easier for readers to interpret across platforms.

5. **How does `<tgmath.h>` simplify mathematical operations?**
   _Answer:_ It provides type-generic macros that automatically dispatch to the correct underlying function (e.g., `cos`, `cosf`, `cosl`) based on the type of the passed argument.

6. **Why does `fgetc` return an `int` rather than a `char`?**
   _Answer:_ So it can safely return the out-of-band `EOF` error/status value without confusing it with a valid character code.

7. **Is receiving `EOF` from a read function definitive proof that the end of the file was reached?**
   _Answer:_ No, `EOF` can also indicate a read error. The `feof` function must be called subsequently to confirm the end-of-file marker was actually reached.

8. **Why is `exit()` discouraged inside `main`?**
   _Answer:_ A standard `return` from `main` achieves the exact same clean termination without the unnecessary function call overhead, making it the preferred, idiomatic approach.

9. **What does the `NDEBUG` macro do in relation to assertions?**
   _Answer:_ Defining `NDEBUG` instructs the compiler to completely ignore and strip out all `assert` macros, neutralizing performance penalties in production builds.
