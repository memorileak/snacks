+++
title = "Modern C: Basic values and data"
description = "An overview of C23 values, types, literals, conversions, initialization, constants, and binary representations, with practical guidance for writing predictable code."
date = 2026-09-27

[taxonomies]
tags = ["modernc"]

# [extra]
# math = false
# cover.image = "images/cover.png"
+++

Here is a detailed breakdown of the most important concepts and key takeaways from **Chapter 5: Basic values and data** in Jens Gustedt's *Modern C: A Guide to the C23 Standard*.


### 1. The Abstract State Machine (Section 5.1)
* **Value-centric thinking:** C programs primarily reason about abstract mathematical values rather than concrete machine representations. All basic C values are either numbers or translate directly to numbers (e.g., characters, truth values, array positions).
* **Types determine behavior:** Every value has a statically determined type that dictates allowable operations, their results, and optimization opportunities.
* **The "As-if" Rule:** Compilers execute programs *as if* strictly following the abstract state machine. The compiler is permitted to reorder or optimize instructions freely as long as observable behavior (e.g., stored state in addressable memory or I/O calls like `printf`) remains identical.


### 2. Basic Types (Section 5.2)
* **Four Base Classes:** Base types belong to four fundamental classes: **unsigned integers**, **signed integers**, **real floating-point numbers**, and **complex floating-point numbers**.
* **Integer Promotion for Narrow Types:** Narrow integer types (`bool`, `char`, `signed char`, `unsigned char`, `short`, `unsigned short`) cannot undergo arithmetic directly; they are automatically *promoted* to `signed int` prior to arithmetic calculations.
* **Type Choice Best Practices:**
  * Use `size_t` for object sizes, cardinalities, array indices, and ordinal numbers.
  * Use `unsigned` for small non-negative quantities.
  * Use `signed` for small quantities that require negative values, and `ptrdiff_t` for signed pointer or index differences.
  * Use `double` for general floating-point calculations and `double complex` for complex numbers.


### 3. Specifying Values & Literals (Section 5.3)
* **Numerical Literals are Positive:** Literal values are strictly non-negative. A minus sign in front of a literal (e.g., `-42`) is a unary negation operator applied to a positive value, not part of the literal syntax itself.
* **Decimal Integer Literals:** Decimal literals default to the first signed integer type (`int`, `long`, `long long`) into which the value fits.
* **C23 Literals & Suffixes:**
  * Supports binary literals starting with `0b` or `0B` (e.g., `0b1010`).
  * Uses exact suffixes to force types: `u`/`U` for unsigned, `l`/`L` for long, `ll`/`LL` for long long, and C23's `wb`/`WB` for bit-precise `_BitInt(N)` literals.
  * Floating-point constants default to `double` unless given an `f`/`F` (`float`) or `l`/`L` (`long double`) suffix.
* **Complex Unit `I`:** Includes the standard macro `I` (from `<complex.h>`) representing the imaginary unit \\(\sqrt{-1}\\).


### 4. Implicit Conversions (Section 5.4)
* **Avoid Narrowing Conversions:** Converting a value to a narrower type can silently lose information or trigger implementation-defined behavior.
* **Dangers of Mixed Signedness:** Operations combining signed and unsigned values force conversion to unsigned types. For example, the comparison `-1 < 0U` evaluates to `false` because `-1` is converted to `UINT_MAX`.
* **Type Consistency:** Design types across expressions so that implicit conversions remain completely harmless and predictable.


### 5. Initializers (Section 5.5)
* **Initialize Everything:** All variables must be initialized upon definition to keep the abstract state machine in a valid, deterministic state.
* **C23 Universal Default Initializer `{}`:** The empty initializer `{}` is valid for all object types (including aggregate types and variable-length arrays), zeroing out all memory/fields.
* **Designated Initializers:** Aggregate structures and arrays should use designated initializers (e.g., ` = 1` or `.member = val`) for explicit and maintenance-safe initialization.


### 6. Named Constants (Section 5.6)
* **Distinguish Read-Only vs. Constants:** `const`-qualified variables define read-only objects in memory, not true compile-time constants.
* **Enumerations (`enum`):** Modern enumerations provide typed integer constants. C23 introduces fixed underlying type syntax for enumerations (e.g., `enum code : unsigned char`).
* **`constexpr` in C23:** Introduces true compile-time constant objects that are checked at compile time to ensure the initializer fits the declared type without value alteration.


### 7. Binary Representations (Section 5.7)
* **Unsigned Integers:** Represented via modular arithmetic modulo \\(2^p\\) (where \\(p\\) is precision). Unsigned integer arithmetic is strictly well-defined and safely wraps around on overflow.
* **Signed Integers & Two's Complement:** C23 strictly standardizes two's complement signed integer representation. Overflow in signed arithmetic is **undefined behavior** and must be avoided.
* **Bit Manipulation & Shifts:** Unsigned types should always be used for bitwise set operations (`&`, `|`, `^`, `~`) and shift operations (`<<`, `>>`).
* **Fixed-Width & Bit-Precise Integers:**
  * Exact-width integers (`int32_t`, `uint64_t`) from `<stdint.h>` provide guaranteed bit widths.
  * C23 introduces bit-precise integers `_BitInt(N)` and `unsigned _BitInt(N)` for arbitrary bit widths.
* **Floating-Point Realities:** Floating-point operations represent real number approximations. They are non-associative, non-commutative, and **must never be checked for exact equality (`==`)**.
