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

The C memory model abstracts the physical memory into **virtual memory**, ensuring portability (you don't manage physical addresses) and safety (you can't access memory your process doesn't own).

**Key Takeaway #1:** Pointer types with distinct base types are distinct newly derived types.

```c
#include <stdio.h>

int main(void) {
    int x = 42;
    double y = 3.14;

    int *px = &x;
    double *py = &y;

    // px and py are distinct types because their base types (int vs double) differ.
    // px = py; // This would cause a compiler error or warning!

    return 0;
}

```

## 12.1 A uniform memory model

C simplifies objects by treating them as an assemblage of **bytes**.

- **Takeaway 12.1 #1:** `sizeof(char)` is `1` by definition. The character types (`char`, `unsigned char`, `signed char`) take up exactly 1 byte.
- **Takeaway 12.1 #7:** The size of all objects of type `T` is given by `sizeof(T)`.

```c
#include <stdio.h>

int main(void) {
    // Demonstrating the uniform size foundation in C
    printf("Size of char: %zu byte(s)\n", sizeof(char));

    double arr[5];
    // The size of the entire object (array) is calculated based on its type
    printf("Size of double[5]: %zu bytes\n", sizeof(arr));

    return 0;
}

```

## 12.2 Unions

`union` is the preferred tool to examine the individual bytes of objects. It overlays different object types over the exact same object representation (memory location).

```c
#include <stdio.h>

// Using a union to inspect the byte-level representation (endianness) of an integer
typedef union unsignedInspect unsignedInspect;
union unsignedInspect {
    unsigned val;
    unsigned char bytes[sizeof(unsigned)];
};

int main(void) {
    unsignedInspect twofold = { .val = 0xAABBCCDD };

    printf("value is 0x%.08X\n", twofold.val);

    // Prints the object memory byte-by-byte.
    // Depending on your CPU (Little-Endian vs Big-Endian), the order will differ!
    for (size_t i = 0; i < sizeof twofold.bytes; ++i) {
        printf("byte[%zu]: 0x%.02hhX\n", i, twofold.bytes[i]);
    }

    return 0;
}

```

## 12.3 Memory and state

The abstract state of a C program execution consists of the values of all its objects. Passing pointers to functions makes it difficult for the compiler to track state because pointers can modify objects behind the scenes (side effects/aliasing).

```c
#include <stdio.h>

// Because `b` is passed as a pointer, the compiler cannot guarantee
// the state of the object it points to won't change during the call.
double blub(double const* a, double* b) {
    double myA = *a;
    *b = 2 * myA; // State of the object pointed to by `b` is modified
    return *a;
}

int main(void) {
    double c = 35.0;
    double d = 3.5;

    printf("blub returns: %g\n", blub(&c, &d));
    // The compiler must read `d` from memory here, as `blub` modified it.
    printf("after blub, d is now %g\n", d);

    return 0;
}

```

## 12.4 Pointers to unspecific objects

Sometimes you need to hold an address without knowing its type yet. `void*` acts as a generic untyped pointer. It can accept any data pointer, but cannot be dereferenced directly without being cast to a specific type first.

```c
#include <stdio.h>

int main(void) {
    double value = 9.99;

    // Untyped pointer takes the address of a double
    void* ptr = &value;

    // To read the state/memory, we must cast it back to the proper typed pointer
    double* typed_ptr = (double*)ptr;
    printf("Value through untyped pointer: %g\n", *typed_ptr);

    return 0;
}

```

## 12.5 Explicit conversions

Casts (e.g., `(T)X`) explicitly convert a value to another type. Converting an object pointer to a pointer to a character type (`unsigned char*`) is explicitly allowed and mostly harmless for inspecting memory.

```c
#include <stdio.h>

int main(void) {
    unsigned val = 0xAABBCCDD;

    // Explicit conversion (cast) from unsigned* to unsigned char*
    // This allows us to inspect the raw bytes safely.
    unsigned char* valp = (unsigned char*)&val;

    for (size_t i = 0; i < sizeof(unsigned); ++i) {
        printf("byte[%zu]: 0x%.02hhX\n", i, valp[i]);
    }

    return 0;
}

```

## 12.6 Effective types

C heavily restricts how an object can be accessed to prevent undefined behavior and optimize safely.

- **Takeaway 12.6 #1:** Objects must be accessed through their **effective type** (their declared type) OR through a **pointer to a character type** (like `char*` or `unsigned char*`).

```c
#include <stdio.h>

int main(void) {
    float pi = 3.1415f;

    // VALID access: using the effective type
    float *p_float = &pi;

    // VALID access: using a character type
    unsigned char *p_char = (unsigned char *)&pi;

    // INVALID access (Violates strict aliasing):
    // int *p_int = (int *)&pi;
    // printf("%d", *p_int); // Undefined Behavior!

    printf("Effective type value: %f\n", *p_float);
    printf("First byte of float: 0x%02X\n", p_char[0]);

    return 0;
}

```

## 12.7 Alignment

Objects usually must start at specific byte boundaries (e.g., multiples of 4 or 8), known as **alignment**.
Forcing a pointer cast from a narrow type (like `unsigned char*`) to a wider type (like `double*` or `complex double*`) at an unaligned offset can cause severe crashes (Bus Errors).

```c
#include <stdio.h>
#include <complex.h>
#include <stdalign.h> // For alignof in older C standards (C23 uses just `alignof`)

typedef double complex cdbl;

int main(void) {
    // Overlay an array of complex doubles with a byte buffer
    union {
        cdbl val[2];
        unsigned char buf[sizeof(cdbl[2])];
    } toocomplex = {
        .val = { 0.5 + 0.5*I, 0.75 + 0.75*I }
    };

    printf("size/alignment of cdbl: %zu / %zu\n", sizeof(cdbl), alignof(cdbl));

    // WARNING: Code below demonstrates alignment danger!
    // In the book, casting to `cdbl*` at offset 4 causes a "Bus error"
    // because `cdbl` requires an alignment of 8 bytes, but 4 is not a multiple of 8.

    size_t offset = 8; // Change to 4 on many systems to trigger a crash!

    if (offset % alignof(cdbl) == 0) {
        cdbl* bp = (cdbl*)(&toocomplex.buf[offset]); // aligned and safe
        printf("Aligned access successful: %g + %gI\n", creal(*bp), cimag(*bp));
    } else {
        printf("Offset %zu is UNALIGNED! Accessing it could crash.\n", offset);
    }

    return 0;
}

```

