+++
title = "Modern C: The C memory model"
description = "A guide to the C23 memory model, covering object representations, unions, strict aliasing, void pointers, casts, effective types, and alignment."
date = 2026-09-28T04:47:32Z

[taxonomies]
tags = ["modernc"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

In **Chapter 12: The C memory model** of Jens Gustedt's *Modern C: A Guide to the C23 Standard*, the book explores C's abstract execution environment, object representations in memory, strict aliasing rules, and hardware alignment constraints.

Here is a detailed breakdown of the fundamental concepts and key takeaways from Chapter 12 across its seven main sections:


### 1. A Uniform Memory Model & Byte Representation (Section 12.1)
* **Objects as Byte Collections:** At the memory model level, all objects—regardless of type—are constructed from an assemblage of **bytes**.
* **The Atom of Storage:** **`sizeof(char)` is 1 by definition**. The C standard specifies **`unsigned char`** as the fundamental atomic building block for all object types in memory.
* **Byte Inspection:** Every object `A` can be viewed, inspected, and processed as an array of `unsigned char[sizeof A]`.
* **Character Pointer Exemption:** Pointers to character types (`unsigned char*`) are special because they are allowed to inspect and alias any object representation in memory.

#### Example

At the memory model level, all objects are constructed as an assemblage of **bytes**. By definition, `sizeof(char)` is 1. The standard designates **`unsigned char`** as the fundamental atomic unit of memory, allowing any object `A` to be viewed and inspected as an array of `unsigned char[sizeof A]`. Pointers to character types have special privileges allowing them to alias and inspect raw object memory.

```c
#include <stdio.h>
#include <stddef.h>

void demo_uniform_memory_model(void) {
    double val = 3.141592653589793;

    // Takeaway 12.1 #1: sizeof(char) is 1 by definition.
    // Takeaway 12.1 #2 & #5: Every object A can be viewed as unsigned char[sizeof A].
    // Takeaway 12.1 #3: Pointers to character types are special and can alias any object.
    unsigned char const* byte_view = (unsigned char const*)&val;

    printf("Object size: %zu bytes (using sizeof operator)\n", sizeof val);
    printf("Raw byte representation in memory: ");
    for (size_t i = 0; i < sizeof val; ++i) {
        printf("%02X ", byte_view[i]); // Inspecting individual representation bytes
    }
    printf("\n");
}
```

### 2. Unions & Object Overlaying (Section 12.2)
* **Overlaying Types:** A **`union`** overlays different type interpretations over the **exact same physical memory location** rather than collecting sequential fields like a `struct`.
* **Inspecting Byte Endianness:** Unions provide a safe mechanism to inspect the **object representation** and byte order (**endianness**) of arithmetic or pointer types.
* **Implementation-Defined Storage Order:** The in-memory order of representation digits (little-endian vs. big-endian) for arithmetic types is **implementation-defined** by the target platform.

#### Example

A **`union`** overlays different type interpretations over the **exact same physical memory location** rather than aggregating sequential fields like a `struct`. This makes unions the ideal tool to inspect the in-memory representation digits and byte ordering (**endianness**) of arithmetic types, which is implementation-defined by the platform.

```c
#include <stdio.h>
#include <stddef.h>
#include <stdbool.h>

// C23 header providing standard endianness macros
#if __has_include(<stdbit.h>)
  #include <stdbit.h>
#endif

// Defining a union to overlay an unsigned integer with a byte array
typedef union inspect_unsigned inspect_unsigned;
union inspect_unsigned {
    unsigned val;
    unsigned char bytes[sizeof(unsigned)]; // Overlays exact byte width of unsigned
};

void demo_unions_and_endianness(void) {
    inspect_unsigned twofold = { .val = 0xAABBCCDD };

    printf("Value: 0x%X\n", twofold.val);
    printf("In-memory byte layout (Implementation-defined endianness):\n");
    for (size_t i = 0; i < sizeof(unsigned); ++i) {
        printf("  bytes[%zu] = 0x%02X\n", i, twofold.bytes[i]);
    }

    // Checking C23 standard endianness macros
#ifdef __STDC_ENDIAN_NATIVE__
    if (__STDC_ENDIAN_NATIVE__ == __STDC_ENDIAN_LITTLE__) {
        puts("Platform is Little-Endian (Least significant byte stored first).");
    } else if (__STDC_ENDIAN_NATIVE__ == __STDC_ENDIAN_BIG__) {
        puts("Platform is Big-Endian (Most significant byte stored first).");
    }
#endif
}
```

### 3. Memory, State, and Aliasing Rules (Section 12.3)
* **Aliasing Defined:** **Aliasing** occurs when two or more pointers address the same object in memory. Uncontrolled aliasing prevents compiler optimization because writing through one pointer might silently alter values read through another.
* **Strict Aliasing Rule:** With the exclusion of character types, **only pointers of the same base type are allowed to alias**. The compiler assumes pointers of different base types (e.g., `double*` and `size_t*`) never point to the same memory location.
* **Avoiding the `&` Operator:** Minimizing the use of the address-of operator `&` prevents local variables from being inadvertently aliased, allowing the compiler to optimize register usage.

#### Example

**Aliasing** occurs when different pointers address the same object in memory. Under the **Strict Aliasing Rule**, with the exception of character pointers, **only pointers of the same base type may alias**. Compilers assume distinct pointer types never address the same location, enabling significant optimizations. Avoiding unnecessary address-of (`&`) operations protects local variables from inadvertent aliasing, allowing compilers to store them in CPU registers.

```c
#include <stdio.h>
#include <stddef.h>

// Strict Aliasing Rule: a and b have distinct non-character base types.
// The compiler ASSUMES writing through 'b' cannot modify the value pointed to by 'a'.
size_t compute_without_aliasing(size_t const* a, double* b) {
    size_t initial_a = *a;
    *b = 2.0 * (double)initial_a; // Writing through double* b

    // Compiler optimizes this to return 'initial_a' directly without re-reading *a from memory!
    return *a;
}

void demo_aliasing_and_state(void) {
    size_t x = 42;
    double y = 0.0;

    size_t res = compute_without_aliasing(&x, &y);
    printf("Result: %zu, y = %g\n", res, y);

    // Takeaway 12.3 #2: Avoid taking addresses (&) when not needed.
    // Keeping local variables un-aliased allows compilers to keep them inside hardware registers.
}
```

### 4. Pointers to Unspecific Objects (`void*`) (Section 12.4)
* **Untyped Memory References:** A **`void*`** represents a pointer to an unspecific object or generic storage instance.
* **Implicit Conversion:** Any object pointer converts implicitly to and from `void*`. Converting an object pointer to `void*` and back to its original type is an **identity operation** that preserves the original memory address.
* **Avoid `void*` in User Code:** Avoid using `void*` when possible because converting to `void*` strips away all type safety, leaving address access unmonitored by the compiler.

#### Example

A **`void*`** represents an untyped pointer to a raw storage instance. Any object pointer converts implicitly to and from `void*`. Converting an object pointer to `void*` and back to its original type is an **identity operation** that preserves the memory address. However, `void*` should be avoided in application code whenever possible because stripping away type information disables compiler safety checks.

```c
#include <stdio.h>
#include <stddef.h>

void demo_void_pointers(void) {
    double original_val = 99.9;
    double* typed_ptr = &original_val;

    // Takeaway 12.4 #1: Any object pointer converts implicitly to and from void*.
    void* untyped_ptr = typed_ptr; // Type information is stripped

    // Takeaway 12.4 #3: Converting back to the original type is the identity operation.
    double* restored_ptr = untyped_ptr;

    printf("Original: %g, Restored via void*: %g\n", *typed_ptr, *restored_ptr);

    // Takeaway 12.4 #4: Avoid void* in user code — it removes all type safety checks!
}
```

### 5. Explicit Conversions & Casts (Section 12.5)
* **Strict Conversion Limits:** Implicit pointer conversions are limited to adding qualifiers (e.g., adding `const`) or converting to/from `void*`.
* **The Danger of Casts:** **Don't use casts `(T*)`**. Explicit pointer casts strip away compiler warnings and type safety checks, risking invalid pointer access or runtime memory corruption.

#### Example

Explicit pointer casts `(T*)` force the compiler to bypass standard type checking. **Gustedt advises avoiding explicit casts** because they conceal dangerous bugs, such as reading past allocated boundaries or reinterpreting incompatible binary representations.

```c
#include <stdio.h>
#include <stddef.h>

void demo_explicit_casts(void) {
    float f = 37.0f;

    // SAFE IMPLICIT CONVERSIONS:
    double d = f;             // Value conversion float -> double
    float const* pfc = &f;    // Qualifier addition
    void* pv = &f;            // Conversion to void*

    // DANGEROUS CAST (Takeaway 12.5 #1: Don't use casts!):
    // Casting float* to double* tricks the compiler into treating a 4-byte object as an 8-byte double!
    // double* pd = (double*)&f; // DANGEROUS: Reading *pd reads garbage past 'f' memory!

    // The ONLY harmless explicit pointer cast is converting to unsigned char* for byte inspection:
    unsigned char* byte_ptr = (unsigned char*)&f; // Harmless byte-wise inspection cast

    printf("First byte of float 37.0f: 0x%02X (d = %g, const_ptr = %p)\n",
           byte_ptr, d, (void*)pfc);
}
```

### 6. Effective Types (Section 12.6)
* **Strict Effective Type Rule:** Objects in memory **must be accessed through their effective type or through a character pointer** (`unsigned char*`).
* **Immutable Declared Types:** The effective type of a declared variable or compound literal is determined permanently by its declaration. Attempting to read or write a declared variable (e.g., `unsigned char A`) through an incompatible pointer cast (e.g., `(unsigned*)A`) violates the effective type rule and results in **undefined behavior**.
* **Union Exemption:** Members of an object with an effective `union` type can be accessed at any time, provided the stored bit pattern represents a valid value for the accessed member type.

#### Example

An object's **effective type** determines how it can be legally accessed. Objects **must be accessed through their effective type or through a character pointer** (`unsigned char*`). The effective type of a declared variable or compound literal is immutable and fixed by its declaration. Attempting to access declared memory through an incompatible pointer cast causes **undefined behavior**.

```c
#include <stdio.h>
#include <stddef.h>

void demo_effective_types(void) {
    // Takeaway 12.6 #3: The effective type of a variable is its declared type (unsigned int).
    unsigned int x = 0x12345678;

    // LEGAL ACCESS (Takeaway 12.6 #1 & #4):
    // Accessing through declared type or character pointer.
    unsigned int* px = &x;                       // Access via declared type
    unsigned char* char_px = (unsigned char*)&x; // Access via character pointer

    printf("Access via declared type: 0x%X\n", *px);
    printf("Access via character pointer byte: 0x%02X\n", char_px);

    // ILLEGAL ACCESS (Takeaway 12.6 #4 -> Undefined Behavior):
    // Reinterpreting declared 'unsigned int x' through an incompatible 'float*' pointer
    // violates the effective type rule and results in undefined behavior!
    // float* bad_float_ptr = (float*)&x;
    // printf("%f\n", *bad_float_ptr); // UNDEFINED BEHAVIOR!
}
```

### 7. Memory Alignment & Word Boundaries (Section 12.7)
* **Hardware Word Boundaries:** Objects of non-character types cannot start at arbitrary byte boundaries; hardware architectures require them to start at aligned **word boundaries**.
* **Misalignment Crashes:** Forcing pointer conversions to an incompatible, misaligned byte offset (e.g., casting `&buf` to `complex double*`) leads to **misaligned memory access or runtime bus errors**.
* **C23 Alignment Keywords:** C23 standardizes **`alignof`** (retrieves a type's alignment requirement) and **`alignas`** (forces a specific byte alignment boundary during allocation).

#### Example

Non-character objects cannot start at arbitrary byte locations; hardware architectures require them to start at aligned **word boundaries**. Forcing a pointer access to a misaligned address triggers **bus errors** or severe execution penalties. C23 standardizes **`alignof`** to query alignment requirements and **`alignas`** to force alignment boundaries.

```c
#include <stdio.h>
#include <stddef.h>
#include <stdalign.h> // Provides alignof and alignas keywords/macros

void demo_alignment(void) {
    // Querying alignment requirements using C23 alignof:
    printf("Alignment requirement of char:   %zu byte\n", alignof(char));
    printf("Alignment requirement of double: %zu bytes\n", alignof(double));

    // Forcing specific alignment using C23 alignas:
    // Forces cache-line or 16-byte SIMD vector boundary alignment
    alignas(16) double vector_buffer = {1.0, 2.0, 3.0, 4.0};

    printf("Vector buffer address: %p (Is 16-byte aligned? %s)\n",
           (void*)vector_buffer,
           ((uintptr_t)vector_buffer % 16 == 0) ? "YES" : "NO");

    // MISALIGNMENT DANGER:
    // Forcing a double* pointer onto an odd byte offset (e.g., &buf) leads to misaligned access,
    // which causes a "Bus error" crash on architectures with strict alignment checks.
}
```