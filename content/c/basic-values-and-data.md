+++
title = "Modern C: Basic values and data"
description = "An overview of C23 values, types, literals, conversions, initialization, constants, and binary representations, with practical guidance for writing predictable code."
date = 2026-09-27T17:21:19Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

Here is a detailed breakdown of the most important concepts and key takeaways from **Chapter 5: Basic values and data** in Jens Gustedt's *Modern C: A Guide to the C23 Standard*.


### 1. The Abstract State Machine (Section 5.1)
* **Value-centric thinking:** C programs primarily reason about abstract mathematical values rather than concrete machine representations. All basic C values are either numbers or translate directly to numbers (e.g., characters, truth values, array positions).
* **Types determine behavior:** Every value has a statically determined type that dictates allowable operations, their results, and optimization opportunities.
* **The "As-if" Rule:** Compilers execute programs *as if* strictly following the abstract state machine. The compiler is permitted to reorder or optimize instructions freely as long as observable behavior (e.g., stored state in addressable memory or I/O calls like `printf`) remains identical.

#### Example

C programs execute within an **abstract state machine** that reasons primarily about abstract mathematical values rather than concrete machine representations. Compilers are permitted to reorder or optimize operations *as long as* observable behavior (e.g., I/O via `printf` or volatile memory changes) remains completely identical.

```c
#include <stdio.h>

void demo_abstract_state_machine(void) {
    // The abstract state machine tracks mathematical values in addressable state.
    double x = 5.0;
    double y = 3.0;

    // The compiler can reorder or optimize instructions under the "as-if" rule.
    // As long as the observable result in x matches the state machine description,
    // the concrete machine code can execute this sequence however it sees fit.
    x = (x * 1.5) - y; 

    // Observable State Transition: Sending characters to stdout.
    printf("Observable state output: x = %g\n", x);
}
```

### 2. Basic Types (Section 5.2)
* **Four Base Classes:** Base types belong to four fundamental classes: **unsigned integers**, **signed integers**, **real floating-point numbers**, and **complex floating-point numbers**.
* **Integer Promotion for Narrow Types:** Narrow integer types (`bool`, `char`, `signed char`, `unsigned char`, `short`, `unsigned short`) cannot undergo arithmetic directly; they are automatically *promoted* to `signed int` prior to arithmetic calculations.
* **Type Choice Best Practices:**
    * Use `size_t` for object sizes, cardinalities, array indices, and ordinal numbers.
    * Use `unsigned` for small non-negative quantities.
    * Use `signed` for small quantities that require negative values, and `ptrdiff_t` for signed pointer or index differences.
    * Use `double` for general floating-point calculations and `double complex` for complex numbers.

#### Example

Basic C types belong to four classes: **unsigned integers**, **signed integers**, **real floating-point numbers**, and **complex floating-point numbers**. **Narrow integer types** (`bool`, `char`, `signed char`, `unsigned char`, `short`, `unsigned short`) cannot perform arithmetic directly; they undergo automatic **integer promotion** to `signed int` prior to evaluation.

```c
#include <stdio.h>
#include <stddef.h>  // size_t, ptrdiff_t
#include <stdbool.h> // bool (now built-in language keyword in C23)
#include <complex.h> // double complex

void demo_basic_types_and_promotions(void) {
    unsigned char uc1 = 200;
    unsigned char uc2 = 100;

    // INTEGER PROMOTION:
    // Narrow types (unsigned char) are promoted to 'signed int' BEFORE addition.
    // The intermediate addition yields signed int 300 without overflowing unsigned char.
    int promoted_sum = uc1 + uc2; 
    printf("Promoted sum (type signed int): %d\n", promoted_sum);

    // TYPE CHOICE BEST PRACTICES:
    size_t array_size = 100;            // Preferred for object sizes, counts, and indices
    unsigned int small_count = 42;      // Unsigned for small non-negative quantities
    ptrdiff_t index_diff = -5;          // Signed for differences between pointers/indices
    double pi = 3.1415926535;           // Preferred for real floating-point calculations
    double complex z = 1.0 + 2.0 * I;   // Standard complex floating-point number

    printf("size_t: %zu, ptrdiff_t: %td, complex: %g + %gi\n", 
           array_size, index_diff, creal(z), cimag(z));
}
```

### 3. Specifying Values & Literals (Section 5.3)
* **Numerical Literals are Positive:** Literal values are strictly non-negative. A minus sign in front of a literal (e.g., `-42`) is a unary negation operator applied to a positive value, not part of the literal syntax itself.
* **Decimal Integer Literals:** Decimal literals default to the first signed integer type (`int`, `long`, `long long`) into which the value fits.
* **C23 Literals & Suffixes:**
    * Supports binary literals starting with `0b` or `0B` (e.g., `0b1010`).
    * Uses exact suffixes to force types: `u`/`U` for unsigned, `l`/`L` for long, `ll`/`LL` for long long, and C23's `wb`/`WB` for bit-precise `_BitInt(N)` literals.
    * Floating-point constants default to `double` unless given an `f`/`F` (`float`) or `l`/`L` (`long double`) suffix.
* **Complex Unit `I`:** Includes the standard macro `I` (from `<complex.h>`) representing the imaginary unit \\(\sqrt{-1}\\).

#### Example

Numerical literals are **always non-negative**; a leading minus sign is a unary negation operator applied to a positive value. C23 adds **digit separators (`'`)**, **binary literals (`0b`)**, and exact suffixes like **`wb`** for bit-precise types.

```c
#include <stdio.h>

void demo_literals_and_suffixes(void) {
    // 1. Numerical literals are strictly positive.
    // -42 is unary negation operator '-' applied to literal 42.
    int neg = -42; 

    // 2. C23 Digit Separator ('): Visual grouping for readability.
    long population = 8'000'000'000L;    // Grouping decimal thousands
    unsigned int mask = 0xAABB'CCDD;     // Grouping double bytes in hex

    // 3. C23 Binary Literals (0b / 0B):
    unsigned int byte_val = 0b1010'0101; // Binary literal representing 165

    // 4. Literal Suffixes:
    // 'u'/'U' = unsigned, 'l'/'L' = long, 'll'/'LL' = long long, 'wb'/'WB' = C23 _BitInt
    unsigned long long big_num = 123456789ULL; // Forces unsigned long long type

    // 5. Decimal integer literals are ALWAYS signed.
    // "First Fit" Rule: Decimal literals take the first signed type (int -> long -> long long) 
    // into which their value fits.
    printf("Binary literal: %u, Population: %ld\n", byte_val, population);
}
```

### 4. Implicit Conversions (Section 5.4)
* **Avoid Narrowing Conversions:** Converting a value to a narrower type can silently lose information or trigger implementation-defined behavior.
* **Dangers of Mixed Signedness:** Operations combining signed and unsigned values force conversion to unsigned types. For example, the comparison `-1 < 0U` evaluates to `false` because `-1` is converted to `UINT_MAX`.
* **Type Consistency:** Design types across expressions so that implicit conversions remain completely harmless and predictable.

#### Example

Implicit conversions that narrow types can silently lose information. Furthermore, mixing signed and unsigned operands forces conversion to unsigned types, creating dangerous comparison bugs.

```c
#include <stdio.h>
#include <stdbool.h>

void demo_implicit_conversions(void) {
    // Narrowing conversions convert according to modulus, risking data loss:
    unsigned short g = 0x8000'0000; // 2,147,483,648 converted modulo 2^16 -> yields 0!
    printf("Narrowing conversion result: %hu\n", g);

    // DANGERS OF MIXED SIGNEDNESS:
    signed int s_neg = -1;
    unsigned int u_zero = 0U;

    // DANGEROUS: -1 is implicitly converted to unsigned int (UINT_MAX, e.g. 4,294,967,295),
    // causing the expression (-1 < 0U) to evaluate to false!
    bool dangerous_cmp = (s_neg < u_zero); 
    printf("-1 < 0U evaluates to: %s (DANGEROUS MIXED SIGNEDNESS!)\n", 
           dangerous_cmp ? "true" : "false");

    // SAFE PRACTICE: Avoid operations with mixed signedness or explicitly convert.
    bool safe_cmp = (s_neg < (signed int)u_zero);
    printf("Safe comparison (-1 < (int)0U): %s\n", safe_cmp ? "true" : "false");
}
```

### 5. Initializers (Section 5.5)
* **Initialize Everything:** All variables must be initialized upon definition to keep the abstract state machine in a valid, deterministic state.
* **C23 Universal Default Initializer `{}`:** The empty initializer `{}` is valid for all object types (including aggregate types and variable-length arrays), zeroing out all memory/fields.
* **Designated Initializers:** Aggregate structures and arrays should use designated initializers (e.g., ` = 1` or `.member = val`) for explicit and maintenance-safe initialization.

#### Example

All variables should be explicitly initialized upon definition to keep the state machine valid. C23 introduces the **universal default initializer `{}`**, which cleanly zero-initializes any scalar, aggregate, or array object.

```c
#include <stdio.h>

void demo_initializers(void) {
    // C23 Universal Default Initializer {}: Valid for ALL object types.
    // Zeroes scalar values, floating-point numbers, pointers, and array elements.
    int x = {};             // Zeroes integer x to 0
    double d = {};          // Zeroes double d to 0.0
    double matrix = {};  // Zeroes all 5 elements

    // Designated Initializers for aggregates and arrays:
    double sparse_arr = {
         = 9.0,
        = 2.9,
        = 3.0e25 // Unspecified elements and are automatically zeroed
    };

    printf("x = %d, d = %g, sparse_arr = %g, sparse_arr = %g\n", 
           x, d, sparse_arr, sparse_arr);
}
```

### 6. Named Constants (Section 5.6)
* **Distinguish Read-Only vs. Constants:** `const`-qualified variables define read-only objects in memory, not true compile-time constants.
* **Enumerations (`enum`):** Modern enumerations provide typed integer constants. C23 introduces fixed underlying type syntax for enumerations (e.g., `enum code : unsigned char`).
* **`constexpr` in C23:** Introduces true compile-time constant objects that are checked at compile time to ensure the initializer fits the declared type without value alteration.

#### Example

`const`-qualified variables define **read-only objects** in memory, not true compile-time constants. C23 introduces **`constexpr`** to declare true, type-checked compile-time constants, as well as explicit **underlying types for enumerations (`enum : type`)**.

```c
#include <stdio.h>

// C23 Fixed Underlying Type for Enumerations:
// Explicitly forces the underlying storage integer type (e.g., : unsigned char).
enum corvid : unsigned char {
    magpie,
    raven,
    jay,
    chough,
    corvid_num // Automatically takes value 4
};

void demo_named_constants(void) {
    // 1. Read-Only Object vs. True Constant:
    // 'const' defines a READ-ONLY object in memory (not usable in Integer Constant Expressions).
    const int read_only_var = 42; 

    // 2. C23 constexpr: True compile-time constant object.
    // Checked at compile-time: initializer MUST fit the target type exactly without data loss.
    constexpr double pi = 3.141'592'653'589'793; 
    
    // constexpr unsigned pi_flat = 3.14159; // COMPILER ERROR: fractional digits lost!

    // 3. Maintainable designated array initialization using enum constants:
    char const* const bird_names[corvid_num] = {
        [raven]  = "raven",
        [magpie] = "magpie",
        [jay]    = "jay",
        [chough] = "chough"
    };

    printf("Pi = %g, Total Corvids = %d, Bird 1 = %s\n", 
           pi, corvid_num, bird_names[magpie]);
}
```

### 7. Binary Representations (Section 5.7)
* **Unsigned Integers:** Represented via modular arithmetic modulo \\(2^p\\) (where \\(p\\) is precision). Unsigned integer arithmetic is strictly well-defined and safely wraps around on overflow.
* **Signed Integers & Two's Complement:** C23 strictly standardizes two's complement signed integer representation. Overflow in signed arithmetic is **undefined behavior** and must be avoided.
* **Bit Manipulation & Shifts:** Unsigned types should always be used for bitwise set operations (`&`, `|`, `^`, `~`) and shift operations (`<<`, `>>`).
* **Fixed-Width & Bit-Precise Integers:**
    * Exact-width integers (`int32_t`, `uint64_t`) from `<stdint.h>` provide guaranteed bit widths.
    * C23 introduces bit-precise integers `_BitInt(N)` and `unsigned _BitInt(N)` for arbitrary bit widths.
* **Floating-Point Realities:** Floating-point operations represent real number approximations. They are non-associative, non-commutative, and **must never be checked for exact equality (`==`)**.

#### Example

Unsigned integer arithmetic operates via modulo \\(2^p\\) and **never triggers undefined overflow**. Signed integer arithmetic strictly uses **two's complement** in C23; signed overflow causes **undefined behavior**. C23 also introduces **bit-precise integers (`_BitInt(N)`)**.

```c
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>

void demo_binary_representations(void) {
    // 1. Unsigned Integer Modular Arithmetic:
    // Unsigned arithmetic safely wraps around modulo 2^p.
    unsigned int u_max = UINT_MAX;
    printf("UINT_MAX + 1 = %u (Safe wrap-around to 0)\n", u_max + 1);

    // 2. Signed Integers & Two's Complement:
    // C23 strictly standardizes Two's Complement for signed integers.
    // Signed arithmetic overflow is UNDEFINED BEHAVIOR and must be avoided!
    // Note: Negation of INT_MIN overflows because |-INT_MIN| > INT_MAX!

    // 3. Bitwise Operations MUST ALWAYS use Unsigned Types:
    unsigned int a = 0b0000'0000'1111'0000U; // 240
    unsigned int b = 0b0000'0001'0001'1111U; // 287
    
    unsigned int bit_and = a & b; // Intersection {4} -> 16
    unsigned int bit_or  = a | b; // Union {0,1,2,3,4,5,6,7,8} -> 511
    printf("Bit AND: 0x%X, Bit OR: 0x%X\n", bit_and, bit_or);

    // 4. C23 Bit-Precise Integers _BitInt(N):
    // Declares exact bit-width integer types up to BITINT_MAXWIDTH.
    unsigned _BitInt(3) u3 = 7wbu; // 3-bit unsigned integer (holds values 0..7)
    signed _BitInt(4)   s4 = -3wb; // 4-bit signed two's complement integer

    printf("u3 = %u, s4 = %d\n", (unsigned int)u3, (int)s4);

    // 5. Floating-Point Equality Rule:
    // Real numbers are approximations; NEVER compare floating-point values for exact equality (==)!
    double f1 = 0.1 + 0.2;
    double f2 = 0.3;
    printf("0.1 + 0.2 == 0.3 evaluates to: %s (Never test float equality!)\n", 
           (f1 == f2) ? "true" : "false");
}
```

