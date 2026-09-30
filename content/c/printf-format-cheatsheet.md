+++
title = "C `printf` format cheatsheet"
description = "A quick reference for the `printf` format specifiers in C."
date = 2026-09-30T14:49:54+00:00

[taxonomies]
tags = ["printf", "format"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 1. Standard Integers
The size of these types varies depending on your platform (32-bit vs. 64-bit). Using the correct length modifier prevents data truncation or corruption.

| Data Type | Signed Format | Unsigned Format | Safety Notes |
|---|---|---|---|
| `char` (Used as a number) | `%d` or `%i` | `%u` | Automatically promotes to `int` when passed to `printf`. |
| `short` | `%hd` | `%hu` | The `h` modifier explicitly tells the compiler it is a short integer. |
| `int` | `%d` or `%i` | `%u` | The default standard integer format. |
| `long` | `%ld` | `%lu` | The `l` modifier stands for long. |
| `long long` | `%lld` | `%llu` | The `ll` modifier is used for extra-large integers (guaranteed at least 64 bits). |

## 2. Fixed-Width Integers
When using `#include <stdint.h>`, these integers have a guaranteed, identical size across all architectures. To print them safely, C uses specific macros available under `#include <inttypes.h>`:

| Data Type | Signed Format | Unsigned Format | printf Syntax Example |
|---|---|---|---|
| `int8_t` / `uint8_t` | `PRId8` | `PRIu8` | `printf("Value: %" PRId8 "\n", var);` |
| `int16_t` / `uint16_t` | `PRId16` | `PRIu16` | `printf("Value: %" PRIu16 "\n", var);` |
| `int32_t` / `uint32_t` | `PRId32` | `PRIu32` | `printf("Value: %" PRId32 "\n", var);` |
| `int64_t` / `uint64_t` | `PRId64` | `PRIu64` | `printf("Value: %" PRIu64 "\n", var);` |

## 3. Floating-Point Numbers
An important rule for printf: float arguments are automatically converted (promoted) into double. Because of this, %f safely works for both types.

| Data Type | Standard Format | Scientific Format (e.g., 1.2e+03) | Key Differences |
|---|---|---|---|
| `float` | `%f` | `%e` or `%g` | `%g` automatically selects the shortest representation. |
| `double` | `%f` (or `%lf`) | `%e` or `%g` | While `%f` works for printf, the input function scanf strictly requires `%lf` for a double. |
| `long double` | `%Lf` | `%Le` or `%Lg` | Must use a capital `L` modifier. |

## 4. Characters, Strings, and Pointers
Handling text and memory addresses poses security risks if incorrect formatting is used.

| Data Type | Safe Format | Safety Notes |
|---|---|---|
| `char` (Character) | `%c` | Prints a single ASCII character. |
| `char*` or `char[]` (String) | `%s` | The string must end with a null character (`\0`). If missing, `printf` will read past bounds and crash. |
| **Any Pointer Type** (Address) | `%p` | Prints memory addresses in hexadecimal. Explicitly cast the argument to `(void*)` for full compliance. |

## 5. Memory Management & System Adaptive Types (C99+)
These specifiers ensure your code compiles and prints safely on both 32-bit and 64-bit systems without manual architecture adjustments:

| Data Type | Safe Format | Real-World Use Case |
|---|---|---|
| `size_t` | `%zu` | Used for physical object sizes, array capacities, or `sizeof` operator returns. |
| `ssize_t` | `%zd` | A signed version of `size_t` (commonly returned by file system operations like `read()` or `write()`). |
| `ptrdiff_t` | `%td` | Represents the distance or index difference between two pointers in the same array. |

## 💡 General Safety Rules

* **String Protection**: If a string might lack a `\0` terminator, cap the maximum printed width using `%.10s` (prints up to 10 characters only).
* **Literal Percent**: To display an actual `%` symbol on the screen, escape it by doubling it up: `%%`.


