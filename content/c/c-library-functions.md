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

Chapter 8 of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, titled **"C library functions,"** covers the essential building blocks provided by the C standard library. The standard functionality is divided into the core language features and the **C library**, which provides platform abstraction, input/output, numerical processing, string manipulation, and runtime diagnostics.

Below is a detailed breakdown of the most important concepts, library header files, and key takeaways from Chapter 8, organized by section.


### 1. General Properties, Error Checking, & Preconditions (Section 8.1)

* **Platform Abstraction Layer:** The C library acts as an abstraction layer over platform-specific hardware and OS mechanics (such as terminal output, file handling, or system clocks) that would otherwise require low-level system code.
* **Function-Like Macros:** Many library interfaces are implemented as function-like macros for efficiency (e.g., `putchar(A)` mapped to `putc(A, stdout)`).
* **Error Signaling Conventions:** C library functions signal errors using standardized return values:
  * **Null pointers (`nullptr`):** Returned by stream creation functions like `fopen` on failure.
  * **Special error values:** Such as `EOF` (-1) returned by stream output functions like `puts` and `fputc`.
  * **`errno` & `perror`:** The global error tracking variable `errno` (from `<errno.h>`) records diagnostic codes, which `perror("msg")` prints as human-readable error messages.
* **Error Handling Takeaways:**
  * **Takeaway 8.1.3 #1:** *Failure is always an option*.
  * **Takeaway 8.1.3 #2:** *Check the return value of library functions for errors*.
  * **Takeaway 8.1.3 #3:** *Fail fast, fail early, and fail often*.
* **Bounds-Checking Interfaces (Annex K):** C11 introduced optional runtime bounds-checking functions (e.g., `printf_s`, `fopen_s`) guarded by `__STDC_LIB_EXT1__`.
  * **Takeaway 8.1.4 #1:** *Identifier names terminating with `_s` are reserved*.
* **Platform Preconditions & Preprocessor Conditionals:**
  * **Takeaway 8.1.5 #1:** *Missed preconditions for the execution platform must abort compilation* (using `#if` and `#error`).
  * **Takeaway 8.1.5 #2:** *In a preprocessor conditional, only evaluate macros and integer literals*.
  * **Takeaway 8.1.5 #3:** *In a preprocessor conditional, unknown identifiers evaluate to 0*.

#### Example

C library functions signal errors using special return values (such as `nullptr` or `EOF`) or by setting the global state `errno`. Gustedt emphasizes three golden rules: **failure is always an option**, **always check return values**, and **fail fast, fail early**.

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>

// Takeaway 8.1.5 #1: Missed platform preconditions must abort compilation
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ < 202311L)
  #warning "This code utilizes C23 library features!"
#endif

// Takeaway 8.1.4 #1: Identifiers terminating with _s are reserved (Annex K)
void demo_error_handling(void) {
    // Attempting to open a non-existent file
    FILE* stream = fopen("non_existent_file.txt", "r");

    // Takeaway 8.1.3 #2: Always check the return value of library functions
    if (!stream) {
        // perror() prints custom string + human-readable error based on 'errno'
        perror("Error opening file"); // e.g., "Error opening file: No such file or directory"

        // Reset errno after handling if recovering (Takeaway 8.1.3 #3)
        errno = 0;
    } else {
        fclose(stream);
    }
}
```

### 2. Integer Arithmetic & Bit Operations (Section 8.2)

* **Standard Arithmetic (`<stdlib.h>`):** Provides `abs`, `labs`, `llabs` for absolute values and `div`, `ldiv`, `lldiv` to compute integer quotient and remainder simultaneously.
* **C23 Checked Integer Arithmetic (`<stdckdint.h>`):** Introduces type-generic macros `ckd_add`, `ckd_sub`, and `ckd_mul`. They perform addition, subtraction, or multiplication while returning a boolean flag indicating if an arithmetic overflow occurred.
* **C23 Bit Operations (`<stdbit.h>`):** Standardizes bitwise inspection and manipulation for unsigned integer types:
  * **Type-Independent Macros:** `stdc_count_ones` (popcount), `stdc_bit_width`, `stdc_bit_floor`, `stdc_has_single_bit`, `stdc_first_trailing_one`, `stdc_first_trailing_zero`.
  * **Type-Width Dependent Macros:** `stdc_bit_ceil`, `stdc_count_zeros`, `stdc_leading_zeros`, `stdc_leading_ones`, `stdc_trailing_zeros`.

#### Example

C23 introduces `<stdckdint.h>` for overflow-safe checked integer arithmetic and `<stdbit.h>` for type-generic, platform-optimized bit manipulation.

```c
#include <stdio.h>
#include <stdbool.h>
#include <limits.h>

// Included feature test checking for C23 headers
#if __has_include(<stdckdint.h>)
  #include <stdckdint.h> // ckd_add, ckd_sub, ckd_mul
#endif

#if __has_include(<stdbit.h>)
  #include <stdbit.h>    // stdc_count_ones, stdc_bit_width, stdc_has_single_bit
#endif

void demo_checked_arithmetic_and_bits(void) {
    // 1. CHECKED INTEGER ARITHMETIC (<stdckdint.h>):
    // Performs addition; returns 'true' if overflow occurred, storing wrap-around result in target.
    unsigned int res = 0;
    bool overflow = ckd_add(&res, UINT_MAX, 1U); // UINT_MAX + 1 overflows unsigned range

    printf("ckd_add overflow: %s, wrapped result: %u\n",
           overflow ? "true" : "false", res); // Output: true, 0

    // 2. C23 TYPE-GENERIC BIT OPERATIONS (<stdbit.h>):
    unsigned int mask = 0b0000'0000'1001'0100U; // Value 148 (3 bits set)

    // stdc_count_ones (popcount): Type-independent population count of 1-bits
    printf("1-bits count in %u: %unsigned\n", mask, stdc_count_ones(mask)); // 3

    // stdc_has_single_bit: Checks if value is a power of two
    printf("Is 16 a power of 2? %s\n", stdc_has_single_bit(16U) ? "yes" : "no");

    // stdc_bit_width: Computes minimum bit-width needed to store value (1 + floor(log2(x)))
    printf("Bit width needed for %u: %unsigned\n", mask, stdc_bit_width(mask));
}
```

### 3. Numerics & Floating-Point Mathematics (Section 8.3)

* **Type-Generic Math (`<tgmath.h>`):** Wraps functions from `<math.h>` and `<complex.h>` into type-generic macros that automatically dispatch calls based on whether the arguments are `float`, `double`, or `long double`.
* **Hardware Acceleration:** Functions like `sqrt`, `sin`, `fabs`, `fma` (fused multiply-add), and rounding utilities (`round`, `trunc`, `lround`, `llround`) bind directly to fast processor-specific instructions; re-implementing them manually in user code is discouraged.

#### Example

Using `<tgmath.h>` provides type-generic mathematical macros that automatically dispatch to the correct float, double, or long double implementations without manual function suffixes (e.g., `sin()` instead of `sinf()` or `sinl()`).

```c
#include <stdio.h>
#include <tgmath.h> // Wraps <math.h> and <complex.h> with type-generic macros

void demo_type_generic_math(void) {
  float f = 2.0f;
  double d = 2.0;

  // Automatic macro dispatch based on argument type:
  // Calling sqrt(f) dispatches to sqrtf(); sqrt(d) dispatches to sqrt()
  float res_f = sqrt(f);
  double res_d = sqrt(d);

  // Hardware-accelerated fused multiply-add: fma(x, y, z) computes (x * y) + z in one step
  double fma_res = fma(3.0, 4.0, 5.0); // 3*4 + 5 = 17.0

  printf("sqrt(2.0f) = %f, sqrt(2.0) = %g, fma = %g\n", res_f, res_d, fma_res);
}
```

### 4. Input, Output, and File Manipulation (Section 8.4)

* **Unformatted Text Output:**
  * `putchar(c)` outputs a single character to `stdout`.
  * `puts(s)` writes string `s` followed by an automatic newline `'\n'` to `stdout`.
  * `fputc(c, stream)` and `fputs(s, stream)` write to an explicit file stream; `fputs` **does not** automatically append a newline.
  * **Takeaway 8.4.1 #1 & #2:** `FILE` is an opaque type; do not rely on implementation details.
  * **Takeaway 8.4.1 #3:** `puts` and `fputs` differ in their end-of-line handling.
* **Files and Streams (`<stdio.h>`):**
  * Files are attached using `fopen(filename, mode)`. Common base modes are `"r"` (read), `"w"` (truncate/write), `"a"` (append), modified by `"+"` (update/rw), `"b"` (binary), or `"x"` (exclusive creation).
* **Formatted Output (`printf`, `fprintf`, `sprintf`, `snprintf`):**
  * `printf` writes to `stdout`; `fprintf` writes to an explicit `FILE*` stream (e.g., `stderr`).
  * **Format Specifiers:** `%zu` for `size_t`, `%td` for `ptrdiff_t`, `%d`/`%u` for signed/unsigned integers, `%b`/`%x` for binary/hexadecimal bit patterns, `%g` for floating-point, and `%a` for exact hex floating-point.
  * **Takeaway 8.4.4 #5:** *Using an inappropriate format specifier or modifier makes the behavior undefined*.
  * **Buffer Safety:** `sprintf` does not prevent buffer overflows; use `snprintf` (or `snprintf_s`) to bound the maximum written characters.
* **Unformatted Text Input:**
  * `fgetc(stream)` reads a single character and returns an `int` so it can represent the `EOF` error/end-of-file condition.
  * `fgets(buf, n, stream)` reads up to `n-1` characters safely into a buffer.
  * **Takeaway 8.4.5 #1:** *Don’t use `gets`* (removed from the standard because it cannot prevent buffer overflows).
  * **Takeaway 8.4.5 #3:** *End-of-file can only be detected after a failed read* (using `feof(stream)`).

#### Example

`<stdio.h>` provides file stream I/O. Gustedt highlights key formatting rules: format specifiers must strictly match argument types, and portable numeric conversions should use `%+d`, `%#X`, or `%a`.

```c
#include <stdio.h>
#include <stddef.h>

void demo_stdio_and_formatting(void) {
    // 1. UNFORMATTED TEXT OUTPUT:
    // puts(s) automatically appends a newline '\n'; fputs(s, stream) DOES NOT.
    puts("puts() adds newline automatically."); // Takeaway 8.4.1 #3
    fputs("fputs() requires manual newline!\n", stdout);

    // 2. FORMAT SPECIFIERS & MODIFIERS:
    size_t len = 255;
    ptrdiff_t diff = -10;
    double fp = 3.14159;

    // Specifiers: %zu for size_t, %td for ptrdiff_t, %g for general float
    // Takeaway 8.4.4 #1 & #5: Arguments must match format specifiers exactly!
    printf("size_t: %zu, ptrdiff_t: %td, double: %g\n", len, diff, fp);

    // Takeaway 8.4.4 #6: Use %#X (hex with prefix) and %a (hex float) for exact round-trips
    printf("Round-trip format: %#X, Hex float: %a\n", (unsigned int)len, fp);

    // 3. BOUNDED BUFFER PRINTING:
    // snprintf prevents buffer overflows by bounding output to buffer capacity
    char buf = {};
    snprintf(buf, sizeof buf, "Formatted value: %d", 42);
    puts(buf);
}
```

### 5. String Processing & Conversions (Section 8.5)

* **Character Classification (`<ctype.h>`):** Functions like `isalnum`, `isalpha`, `isdigit`, `isspace`, `islower`, `isupper` classify single characters, while `toupper` and `tolower` perform case conversions.
* **String-to-Number Conversions (`<stdlib.h>`):**
  * `strtod`, `strtof`, `strtold` convert text strings into floating-point numbers.
  * `strtoul`, `strtol`, `strtoull`, `strtoll` parse integer strings in bases `0`, `2`, `8`, `10`, or `16`.
  * Setting `base = 0` automatically interprets prefixes like `"0b"` (binary), `"0"` (octal), or `"0x"` (hexadecimal). Out-of-range values return `ULONG_MAX`/`LLONG_MAX` and set `errno` to `ERANGE`.
* **String Manipulation (`<string.h>`):**
  * `strlen(s)` computes string length up to the first `0` character.
  * `strcpy` / `memcpy` copy string or raw memory regions.
  * `strcmp` / `memcmp` compare strings or byte buffers lexicographically.
  * `strspn` returns length of matching initial characters; `strcspn` returns length of non-matching initial characters.
  * `strdup` / `strndup` allocate dynamic memory for a string copy.

#### Example

`<ctype.h>` provides character classifiers. `<stdlib.h>` conversion functions (`strtoul`, `strtod`) parse strings into numbers; setting `base = 0` automatically interprets prefixes like `"0b"` (binary), `"0x"` (hex), or `"0"` (octal).

```c
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <string.h>
#include <errno.h>

void demo_string_processing(void) {
    // 1. CHARACTER CLASSIFICATION (<ctype.h>):
    char c = 'a';
    if (islower(c)) {
        printf("'%c' upper-cased is '%c'\n", c, toupper(c));
    }

    // 2. STRING-TO-NUMBER CONVERSIONS (<stdlib.h>):
    // Base 0 automatically detects prefixes: "0b1010" -> binary, "0xFF" -> hex, "123" -> decimal
    char const* bin_str = "0b1010";
    char const* hex_str = "0x1A";

    unsigned long val1 = strtoul(bin_str, nullptr, 0); // Parsed as binary 10
    unsigned long val2 = strtoul(hex_str, nullptr, 0); // Parsed as hex 26

    printf("Parsed '%s' -> %lu, '%s' -> %lu\n", bin_str, val1, hex_str, val2);

    // 3. STRING SPAN FUNCTIONS (<string.h>):
    char const* text = "12345abc";
    // strspn returns length of initial segment containing ONLY characters from search set
    size_t num_digits = strspn(text, "0123456789"); // Yields 5 ('12345')
    printf("Initial digit span length: %zu\n", num_digits);
}
```

### 6. Time & Runtime Environment (Sections 8.6 & 8.7)

* **Time Processing (`<time.h>`):**
  * `clock()` returns CPU processor time used by the program.
  * `time()` returns simple calendar time in seconds since the Epoch.
  * `difftime(t1, t0)` calculates time differences in seconds as a `double`.
  * `timespec_get(&ts, TIME_UTC)` returns high-resolution time with nanosecond precision.
  * `strftime` formats date and time structures (`struct tm`) into custom text strings.
* **Environment & Localization:**
  * `getenv("NAME")` and `getenv_s` inspect platform environment variables.
  * `setlocale(category, locale)` configures locale-specific formatting (such as decimal points or character collating sequences) from `<locale.h>`.

#### Example

`<time.h>` provides calendar time (`time_t`, `struct tm`) and high-resolution nanosecond timestamps (`struct timespec` via `timespec_get`).

```c
#include <stdio.h>
#include <time.h>

void demo_time_processing(void) {
    // 1. CALENDAR TIME & FORMATTING:
    time_t now = time(nullptr); // Current timestamp in seconds since Epoch
    struct tm local_time = {};

    // Thread-safe conversion to local calendar time struct
    localtime_r(&now, &local_time);

    char time_str = {};
    // strftime formats calendar structs using standard specifiers (%Y-%m-%d %H:%M:%S)
    strftime(time_str, sizeof time_str, "%Y-%m-%d %H:%M:%S", &local_time);
    printf("Current Local Time: %s\n", time_str);

    // 2. HIGH-RESOLUTION NANOSECOND TIME (timespec_get):
    struct timespec ts = {};
    if (timespec_get(&ts, TIME_UTC) == TIME_UTC) { // Earth reference time
        printf("UTC Time: %ld sec, %ld nsec\n", (long)ts.tv_sec, ts.tv_nsec);
    }
}
```

### 7. Program Termination & Assertions (Section 8.8)

* **Program Termination Paths:**
  * **Takeaway 8.8 #1:** *Regular program termination should use a `return` from `main`*.
  * **Takeaway 8.8 #2:** *Use `exit` from a function that may terminate the regular control flow*.
  * `quick_exit(status)` terminates without running full library exit handlers (e.g., `atexit`).
  * `_Exit(status)` terminates immediately at the OS level.
  * `abort()` triggers abnormal program termination, raising `SIGABRT` without executing standard cleanup handlers.
* **Runtime Assertions (`<assert.h>`):**
  * The `assert(cond)` macro evaluates boolean conditions at runtime during development. If `cond` is false, it prints a diagnostic message containing file, function, and line numbers, then calls `abort()`.
  * **Takeaway 8.8 #4:** *Use as many `asserts` as you can to confirm runtime properties*.
  * **Takeaway 8.8 #5:** *In production compilations, use `NDEBUG` to switch off all `asserts`* (defined via compiler flag `-DNDEBUG`).

#### Example

Program execution should normally terminate via a `return` from `main()`. Functions terminating control flow early use `exit()`, while development preconditions are checked using `assert()`.

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h> // Provides runtime assert()

void validate_and_process(int factor) {
  // Takeaway 8.8 #4: Use as many asserts as possible to confirm runtime properties
  // In production builds, compiling with -DNDEBUG disables all assert() checks.
  assert(factor > 0 && "Factor must be positive!");

  if (factor > 100) {
    puts("Factor out of bounds! Exiting early.");
    // Takeaway 8.8 #2: Use exit() from nested functions terminating control flow
    exit(EXIT_FAILURE);
  }
}

void demo_environment_and_exit(void) {
  // Inspecting environment variables safely
  char const* path = getenv("PATH");
  if (path) {
    printf("System PATH environment variable is set.\n");
  }

  validate_and_process(5);
}
```
