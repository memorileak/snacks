+++
title = "Modern C: C library functions"
description = ""
date = 2026-09-28

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


### 2. Integer Arithmetic & Bit Operations (Section 8.2)

* **Standard Arithmetic (`<stdlib.h>`):** Provides `abs`, `labs`, `llabs` for absolute values and `div`, `ldiv`, `lldiv` to compute integer quotient and remainder simultaneously.
* **C23 Checked Integer Arithmetic (`<stdckdint.h>`):** Introduces type-generic macros `ckd_add`, `ckd_sub`, and `ckd_mul`. They perform addition, subtraction, or multiplication while returning a boolean flag indicating if an arithmetic overflow occurred.
* **C23 Bit Operations (`<stdbit.h>`):** Standardizes bitwise inspection and manipulation for unsigned integer types:
  * **Type-Independent Macros:** `stdc_count_ones` (popcount), `stdc_bit_width`, `stdc_bit_floor`, `stdc_has_single_bit`, `stdc_first_trailing_one`, `stdc_first_trailing_zero`.
  * **Type-Width Dependent Macros:** `stdc_bit_ceil`, `stdc_count_zeros`, `stdc_leading_zeros`, `stdc_leading_ones`, `stdc_trailing_zeros`.


### 3. Numerics & Floating-Point Mathematics (Section 8.3)

* **Type-Generic Math (`<tgmath.h>`):** Wraps functions from `<math.h>` and `<complex.h>` into type-generic macros that automatically dispatch calls based on whether the arguments are `float`, `double`, or `long double`.
* **Hardware Acceleration:** Functions like `sqrt`, `sin`, `fabs`, `fma` (fused multiply-add), and rounding utilities (`round`, `trunc`, `lround`, `llround`) bind directly to fast processor-specific instructions; re-implementing them manually in user code is discouraged.


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
