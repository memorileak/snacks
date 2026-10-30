+++
title = "Modern C: Basic values and data"
description = "An overview of C23 values, types, literals, conversions, initialization, constants, and binary representations, with practical guidance for writing predictable code."
date = 2026-09-27T17:21:19Z

[taxonomies]
tags = ["modernc", "type", "constant", "conversion", "initialization"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 5. Basic values and data

### Program Values and Representation

**What it is:**
In C, the primary focus of a program is manipulating _values_ rather than managing how those values are stored in memory (_representations_). Values are abstract, mathematical entities (like the number 12), while representations are how the physical hardware encodes those values (like a 32-bit binary sequence).

**Why it matters / how it works:**
The compiler acts as an intermediary, translating back and forth between abstract values and physical binary representations. As a programmer, you should ideally write code that reasons about values. This ensures your program remains portable across different hardware architectures (e.g., different endianness or word sizes) because the compiler handles the underlying representation details.

> Takeaway 5 #1: C programs primarily reason about values and not about their representation.

**Key details:**

- Values are abstract concepts (e.g., the number `0`).
- Representations are platform-specific encodings of values.
- Compilers manage the mapping between value and representation.

**Pitfalls:**

- Writing code that assumes a specific binary layout for a value (e.g., assuming a specific endianness when reading bytes of an integer) leads to non-portable and brittle code.

**Connections:**

- Connects to **Pointers and Memory (Chapter 11/12)**, where looking at the raw representation of an object via character pointers (`unsigned char*`) becomes possible but requires strict adherence to effective type rules.

**Recap:**

- Think in abstract mathematical values.
- Let the C compiler handle the physical representation.
- Portability relies on not making assumptions about memory layouts.

---

## 5.1. The abstract state machine

### 5.1.1. Values

**What it is:**
A value in C is an abstract entity that exists independently of your program's specific implementation or runtime memory. Fundamentally, all values in C are numbers or can be directly translated to numbers.

**Why it matters / how it works:**
By treating all values—whether they represent characters, truth values (`true`/`false`), or measurements—as numbers, C provides a unified, highly optimized way for processors (which are essentially giant calculators) to evaluate state transitions. The "abstract state machine" uses these values to determine the next state of the program.

> Takeaway 5.1.1 #1: All values are numbers or translate to numbers.

**Key details:**

- Characters (e.g., `'A'`) are just integer values (e.g., `65` in ASCII).
- Truth values are integers (`0` for false, `1` for true).
- Memory addresses (pointers) map to numerical spaces.

**Pitfalls:**

- Treating characters as strings or distinct semantic objects rather than underlying numbers can lead to confusion when doing character arithmetic (e.g., `'a' + 1`).

**Connections:**

- Ties into **Implicit Conversions (5.4)** where different types of numbers are seamlessly converted based on mathematical containment.

```c
// Code Demonstration: Values as Numbers
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    // Demonstration of Takeaway 5.1.1 #1
    char c = 'A';       // A character, but fundamentally a number (65 in ASCII)
    bool is_true = true; // A truth value, fundamentally the number 1 (C23 supports bool natively)

    // We can do arithmetic on these directly because they are numbers
    int result = c + is_true;

    printf("Result is %d\n", result);
    return EXIT_SUCCESS;
}
// Expected output: Result is 66
// What goes wrong if... you assume 'A' + '1' concatenates strings?
// In C, it adds the integer values of those characters, producing an unexpected numeric sum.
```

### 5.1.2. Types

**What it is:**
A type is a static property associated with every value in C. It defines both the meaning of the value and the operations that are permissible on it.

**Why it matters / how it works:**
Types allow the compiler to understand how to interpret a piece of data and what hardware instructions to generate. Because types are determined _statically_ (at compile time), the C compiler can rigorously check for errors before the program ever runs.

> Takeaway 5.1.2 #1: All values have a type that is statically determined.
> Takeaway 5.1.2 #2: Possible operations on a value are determined by its type.
> Takeaway 5.1.2 #3: A value’s type determines the results of all operations.

**Key details:**

- Static determination means the compiler knows the type before execution.
- The type dictates if you can add, shift, or dereference a value.
- Operations (like division) yield different results based on type (e.g., integer division truncates, floating-point division preserves fractions).

**Pitfalls:**

- Performing operations on incompatible types without casting, or misunderstanding integer division (e.g., `5 / 2` yields `2`, not `2.5`).

**Connections:**

- Ties into **Basic Types (5.2)** and **Derived Types (Chapter 6)**.

```c
// Code Demonstration: Types determine operations and results
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    // Takeaway 5.1.2 #3: Result depends on type
    int int_val_1 = 5;
    int int_val_2 = 2;
    double float_val_1 = 5.0;

    // Integer division truncates
    int result_int = int_val_1 / int_val_2;

    // Floating point division preserves fractions
    double result_float = float_val_1 / int_val_2; // int_val_2 is implicitly converted

    printf("Int division: %d\n", result_int);
    printf("Float division: %f\n", result_float);

    return EXIT_SUCCESS;
}
// Expected output:
// Int division: 2
// Float division: 2.500000
```

### 5.1.3. Binary representation and the abstract state machine

**What it is:**
The abstract state machine dictates _what_ a program should do based on values and types, while the _binary representation_ is how the specific target platform realizes those types in memory (e.g., how many bits make up an `int`, or the precision of a `double`).

**Why it matters / how it works:**
The C standard does not rigidly define the exact bit-width of every type. Instead, it provides minimum ranges. The exact binary representation on a specific platform determines the limits of computation (e.g., when an integer will overflow). Programs execute _as if_ they follow the abstract state machine, giving compilers massive freedom to optimize code (like skipping redundant calculations) as long as the _observable outputs_ match the abstract machine's rules.

> Takeaway 5.1.3 #1: A type’s binary representation determines the results of all operations.
> Takeaway 5.1.3 #3 (as-if): Programs execute _as if_ following the abstract state machine.

**Key details:**

- Implementations define exact representations (e.g., `SIZE_MAX`).
- Observable states (I/O, volatile variables) must be preserved.
- Compilers can optimize away intermediate steps if they don't affect observable states.

**Pitfalls:**

- Assuming `int` is exactly 32 bits on all platforms causes overflow bugs on platforms where it is 16 bits.
- Relying on unspecified timing behavior; optimizations can make code run faster by removing intermediate unobserved state computations entirely.

**Connections:**

- Ties into **Binary Representations (5.7)**.

**Recap for 5.1:**

- The C environment acts as an abstract state machine tracking values.
- Values are fundamentally numerical and are bound to static types.
- The "as-if" rule allows compilers to heavily optimize code, provided the final observable output remains correct.

---

## 5.2. Basic types

### Type Classifications

**What it is:**
C categorizes its base mathematical types into four main classes: unsigned integers, signed integers, real floating-point numbers, and complex floating-point numbers.

**Why it matters / how it works:**
Providing different classes of types allows programmers to express different mathematical domains efficiently. Hardware handles unsigned, signed, and floating-point math differently. By explicitly stating the type, the compiler picks the fastest and most correct hardware instruction.

**Key details:**

- **Unsigned Integers:** `bool` (or `_Bool`), `unsigned char`, `unsigned short`, `unsigned int`, `unsigned long`, `unsigned long long`. They cannot be negative and operate using modulo arithmetic.
- **Signed Integers:** `signed char`, `short`, `int`, `long`, `long long`. They can represent negative values.
- **Real Floating:** `float`, `double`, `long double`. For fractional approximations.
- **Complex Floating:** `float _Complex`, `double _Complex`, `long double _Complex`.
- **Narrow Types:** Types like `char` and `short` cannot be used directly in arithmetic; they are _promoted_ to `int` before computation.

**Pitfalls:**

- `char` can be signed or unsigned depending on the platform/compiler. Never assume its signedness.
- Using narrow types to save space in arithmetic calculations actually provides no benefit because they are promoted to `int` during calculation anyway.

**Connections:**

- Connects to **Implicit Conversions (5.4)** where narrow types are promoted.

```c
// Code Demonstration: Basic Types and Promotion
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    // Narrow types
    unsigned short a = 65000;
    unsigned short b = 65000;

    // a and b are implicitly promoted to 'signed int' for the arithmetic operation.
    // Assuming int is 32-bit, the addition won't overflow the int.
    int result = a + b;

    printf("Result: %d\n", result);
    return EXIT_SUCCESS;
}
// Expected output: Result: 130000
// What goes wrong if... we assigned the result back to an unsigned short?
// It would wrap around due to the limits of the binary representation of a 16-bit short.
```

**Recap for 5.2:**

- C has 18 base types divided into signed, unsigned, real floating, and complex.
- Narrow types (smaller than `int`) are automatically promoted to `int` during arithmetic.
- The exact precision/width of these types is implementation-defined, not strictly standardized (except for minimum bounds).

---

## 5.3. Specifying values

### Literals

**What it is:**
Literals are hardcoded, fixed values written directly into the source code (e.g., `0`, `3.14`, `'a'`).

**Why it matters / how it works:**
Literals must have a type, a value, and a binary representation just like variables. C deduces the type of a literal based on its format (prefix, suffix, and presence of decimal points). C23 introduces new conveniences like binary literals (`0b`) and digit separators (`'`).

> Takeaway 5.3 #A: Literals have value, type, and binary representations.

**Key details:**

- **Digit separators:** Introduced in C23, you can use `'` to make large numbers readable (e.g., `1'000'000`).
- **Binary literals:** C23 introduces `0b` or `0B` prefixes (e.g., `0b1010` is 10).
- **Hexadecimal literals:** `0x` or `0X` prefix. Hexadecimal floats exist too (e.g., `0x1.7aP-13`, where `P` specifies the exponent in base 2).
- **Suffixes:** Define the exact type. `U` for unsigned, `L` for long, `LL` for long long, `F` for float. C23 introduces `wb` or `WB` for `_BitInt`.
- **Zero:** `0` is inherently a `signed int`.

**Pitfalls:**

- Writing `3.14` deduces to a `double`. If you assign it to a `float` without an `F` suffix (`3.14F`), the compiler performs an implicit conversion, which may trigger a warning or lose precision.

**Connections:**

- Ties closely to **Implicit Conversions (5.4)**, as interacting literals of different types triggers type convergence.

### 5.3.1. Complex constants

**What it is:**
C provides standard ways to represent complex numbers if the platform supports them (checked via `__STDC_NO_COMPLEX__`).

**Why it matters / how it works:**
Mathematical modeling often requires imaginary numbers. C provides the `I` macro in `<complex.h>` to represent the imaginary unit (where $I^2 = -1$).

> Takeaway 5.3.1 #1: `I` is reserved for the imaginary unit.

**Key details:**

- Use `<complex.h>` to access `I`.
- Complex literals can be formed by multiplying `I` (e.g., `0.5 + 0.5 * I`).
- C23 standardizes complex literals (though the text notes compiler-specific `i` suffixes are common but historically non-standard).

**Recap for 5.3:**

- Literals specify raw values in code and have statically deduced types.
- Use C23 features like `'` separators and `0b` prefixes for readability.
- Append suffixes (`U`, `L`, `F`) to enforce specific types on literals.

---

## 5.4. Implicit conversions

### Type Promotion and Signedness

**What it is:**
When operators receive operands of different or narrow types, the C compiler automatically converts ("promotes") them to a common, wider type before evaluating the expression.

**Why it matters / how it works:**
Implicit conversion prevents the need for manual casting in every mixed-type math operation. However, mixing signed and unsigned integers triggers specific rules that can drastically alter the mathematical value of a variable.

> Takeaway 5.4 #1: Unary `-` and `+` have the type of their promoted operand.

**Key details:**

- **Narrow to Wide:** Narrow types (`short`, `char`) convert to `int` safely.
- **Mixed Signedness:** If an operation mixes `signed int` and `unsigned int`, the `signed int` is converted to `unsigned int`.
- If a negative `signed` value is converted to `unsigned`, it wraps around to a massive positive number due to modulo arithmetic.

**Pitfalls:**

- Comparing a negative signed integer to an unsigned integer (e.g., `-1 < 1U`). `-1` is converted to `unsigned`, becoming `UINT_MAX`. Thus, `UINT_MAX < 1U` evaluates to `false`!
- Assigning a value too big for a signed type triggers _implementation-defined_ behavior (unlike unsigned overflow, which is strictly defined as modulo wrap-around).

**Connections:**

- Essential for evaluating **Expressions (Chapter 4)** correctly and avoiding infinite loops in `for` loops counting downwards with unsigned types.

```c
// Code Demonstration: Implicit Conversions and Signedness
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    signed int a = -1;
    unsigned int b = 1U;

    // Takeaway 5.4 #1 and implicit conversion rules
    // a is implicitly converted to unsigned int!
    // -1 becomes a very large positive number (e.g., 4294967295)
    if (a < b) {
        printf("-1 is mathematically less than 1\n");
    } else {
        printf("Due to implicit conversion, -1 is seen as LARGER than 1U\n");
    }

    return EXIT_SUCCESS;
}
// Expected output: Due to implicit conversion, -1 is seen as LARGER than 1U
// What goes wrong if... you use unsigned for loop counters and check `i >= 0`?
// It implicitly stays unsigned, is always >= 0, and causes an infinite loop.
```

**Recap for 5.4:**

- Conversions happen automatically to maintain type consistency in expressions.
- Unary operators retain the promoted type of their operand.
- Mixing signed and unsigned types forces the signed operand into an unsigned representation, potentially mangling negative values.

---

## 5.5. Initializers

### Defining Initial State

**What it is:**
An initializer sets the very first value of an object at the point of its declaration. C provides precise syntaxes (like designated initializers and default initializers) for setting up complex data (arrays, structs).

**Why it matters / how it works:**
Uninitialized variables hold "garbage" values left over in memory, leading to undefined behavior if read. Explicit initialization ensures program determinism.

> Takeaway 5.5 #2: Use designated initializers for all aggregate data types.
> Takeaway 5.5 #3: `{ }` is a valid initializer for all objects.

**Key details:**

- **Designated initializers:** Use brackets for arrays (e.g., `[0] = 5.0`) or dots for structs to explicitly name the part being initialized. Unlisted elements default to `0`.
- **Default initializers:** C23 allows empty braces `{ }` to initialize _any_ object (scalar or aggregate) to its default zero state (equivalent to `{0}` in older standards, but universally valid now, even for Variable Length Arrays (VLAs)).

**Pitfalls:**

- Relying on positional array initializers (e.g., `int A[] = {1, 2};`) makes the code brittle. If the layout expectations change, the initializations silently map to the wrong indices.

**Connections:**

- Connects to **Derived Types (Chapter 6)** where arrays and structs are formally introduced.

```c
// Code Demonstration: Designated and Default Initializers
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    // Takeaway 5.5 #3: Empty braces zero-initialize everything
    double default_val = {};

    // Takeaway 5.5 #2: Designated initializers
    // Array of 5, but we only explicitly initialize index 1 and 4.
    // Missing elements ([0], [2], [3]) default to 0.0 automatically.
    double arr[5] = {
        [1] = 3.14,
        [4] = 2.71
    };

    printf("Default val: %f\n", default_val);
    printf("arr[0]: %f, arr[1]: %f\n", arr[0], arr[1]);

    return EXIT_SUCCESS;
}
// Expected output:
// Default val: 0.000000
// arr[0]: 0.000000, arr[1]: 3.140000
```

**Recap for 5.5:**

- Always initialize objects upon declaration.
- Use designated initializers (`[index] = val`) for arrays/structs to ensure code robustness.
- Use `{}` (C23) to zero-out any object trivially.

---

## 5.6. Named constants

### 5.6.1 - 5.6.5 Defining Intent

**What it is:**
Named constants are identifiers tied to fixed values that do not change. C provides multiple mechanisms to achieve this: `const` qualifiers, `constexpr` (C23), enumerations (`enum`), and macros (`#define`).

**Why it matters / how it works:**
"Magic numbers" (hardcoded literals scattered through code) make programs impossible to maintain. Naming constants centralizes the value, allowing for single-point updates and providing semantic context (e.g., `MAX_USERS` instead of `100`).

> Takeaway 5.6 #1: All constants with a particular meaning must be named.
> Takeaway 5.6 #2: All constants with different meanings must be distinguished.

**Key details:**

- **`const` qualifier:** Creates a _read-only_ variable. Its value might only be known at runtime (e.g., `const int size = get_size();`), but cannot be changed post-initialization.
- **`constexpr` (C23):** Guarantees the value is a strict compile-time constant. It is fundamentally immutable and evaluated by the compiler.
- **Enumerations (`enum`):** Traditional way to name small `int` constants.
  - _New in C23:_ Enums can now store values larger than `int`.
  > Takeaway 5.6.2 #4: If enumeration constants do not fit into `signed int`, if possible, the enumeration type is adjusted such that it can store all enumeration constants.
  - _New in C23:_ You can strictly specify the underlying type (e.g., `enum wider : long { ... }`).
- **Macros (`#define`):** Performs blind textual replacement before compilation. Useful for type-agnostic constants or wrapping compound literals, but lacks scope and type safety.

**Pitfalls:**

- Confusing `const` (read-only variable) with true compile-time constants (like `constexpr` or `#define`). Prior to C23, a `const int` could not be used to define an array size at file scope!
- Macro text replacement can cause priority of operations errors if not heavily parenthesized (e.g., `#define ADD(a, b) a + b` -> `ADD(1, 2) * 3` becomes `1 + 2 * 3 = 7`, not `9`).

**Connections:**

- Connects to **The Preprocessor (Chapter 8)**.

```c
// Code Demonstration: Named Constants and C23 Features
#include <stdio.h>
#include <stdlib.h>
#include <limits.h>

// Takeaway 5.6 #1: Meaningful names
// Macro textual replacement
#define MAX_BUFFER_SIZE 1024

// C23: Specifying underlying enum type explicitly
enum large_enum : unsigned long long {
    BIG_VAL = 0xFFFFFFFFFFFFFFFFULL,
    SMALL_VAL = 0
};

// C23: constexpr guarantees compile-time evaluation
constexpr double PI = 3.14159265358979;

int main(void) {
    const int read_only_val = 5; // Cannot be modified, but not a strict constant expression

    printf("PI: %f\n", PI);
    printf("Big enum val: %llu\n", BIG_VAL);

    return EXIT_SUCCESS;
}
// Expected output:
// PI: 3.141593
// Big enum val: 18446744073709551615
// What goes wrong if... you assign to a const or constexpr object? The compiler will throw an error preventing mutation.
```

**Recap for 5.6:**

- Eradicate magic numbers; name your constants.
- `const` makes an object read-only; `constexpr` makes it a true compile-time constant.
- Enums in C23 can be strictly typed and hold values larger than standard `int`.

---

## 5.7. Binary representations

### Bits and Booleans

**What it is:**
Binary representation is the specific bit-level layout of a type in memory. An unsigned type of precision $p$ utilizes bits $b_0$ (Least Significant Bit, LSB) through $b_{p-1}$ (Most Significant Bit, MSB).

**Why it matters / how it works:**
While you should program for _values_, systems programming often requires manipulating exact bits (e.g., hardware flags). Understanding how the bitwidth bounds a value (e.g., $2^p - 1$ is the max for an unsigned type) allows for safe bitwise operations (`<<`, `>>`, `&`, `|`).

**Key details:**

- **Boolean values:** `bool` is technically an unsigned integer type with precision 1 (capable of holding only 0 and 1). In C23, `bool`, `true`, and `false` are native keywords (no `<stdbool.h>` needed unless for backwards compatibility).
- **Extended Integers (C23):** C23 introduces exact bit-precise integer types (e.g., `_BitInt(N)`) and exposes 128-bit hardware types (`int128_t`), breaking the old `intmax_t` limitations.

**Pitfalls:**

- Left-shifting signed integers into the sign bit invokes undefined behavior in C. Always use unsigned integers for bit manipulations.

**Connections:**

- Ties into bit manipulation functions newly added in `<stdbit.h>`.

**Recap for 5.7:**

- Precision $p$ determines the maximum value ($2^p-1$ for unsigned).
- `bool` is inherently an unsigned type restricted to 0 and 1.
- C23 embraces modern hardware via exact-width and 128-bit types.

---

## Summary

Chapter 5 transitions the programmer's mindset from syntax execution to data management. It establishes that C programs manipulate abstract values, which are statically bound to types and translated by the compiler into hardware-specific binary representations. The "as-if" rule allows the abstract state machine to aggressively optimize these operations. The chapter breaks down C's complex type hierarchy (signed, unsigned, floating, complex), detailing how implicit conversions resolve type mismatches (often dangerously, particularly with signed/unsigned mixtures). Furthermore, it provides the modern C23 tools for initializing memory (`{}`) and defining intent through immutable named constants (`constexpr`, explicit `enum` types), moving away from error-prone macros. Ultimately, mastering this chapter ensures a developer writes deterministic, portable, and type-safe code.

---

## Self-Check Questions

1. **What is the difference between a value and a representation?**
   A value is an abstract mathematical concept (like the number 5), while a representation is the physical bit-pattern used by a specific machine to store that value.

2. **According to C's abstract state machine, what fundamental entity do all values translate to?**
   Numbers.

3. **What does the "as-if" rule allow the C compiler to do?**
   It allows the compiler to optimize and alter the execution of code however it wants, provided the _observable states_ (outputs, I/O) match what the strict abstract state machine would produce.

4. **What happens when you perform arithmetic on a narrow type like `unsigned short`?**
   The values are implicitly promoted to `signed int` (assuming `int` can hold all values of the short) before the calculation takes place.

5. **If you compare `-1` (a `signed int`) and `1U` (an `unsigned int`), what is the result and why?**
   The result is `false` (i.e., -1 is considered greater). The `-1` is implicitly converted to an `unsigned int`, wrapping around to a massive positive number (like `UINT_MAX`).

6. **What is the safest and most robust way to initialize specific elements in an array or struct?**
   Using designated initializers (e.g., `[index] = value` or `.member = value`).

7. **In C23, what does the initializer `{ }` do?**
   It acts as a default initializer, setting all scalar or aggregate elements of the object to their zero-state (0, 0.0, or null pointers).

8. **What is the difference between `const` and `constexpr` in C23?**
   `const` simply marks a variable as read-only (it might still be initialized at runtime), whereas `constexpr` mandates that the value is known and fixed exactly at compile time.

9. **Can an `enum` in C23 store a value larger than `INT_MAX`?**
   Yes. C23 allows enums to automatically adjust their underlying type to fit larger values, or you can explicitly specify the underlying type (e.g., `enum my_enum : unsigned long long`).

10. **Is `bool` a signed or unsigned type, and does it require a header in C23?**
    `bool` is considered an unsigned integer type with precision 1. In C23, it is a native keyword and no longer requires `<stdbool.h>`.
