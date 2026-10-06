+++
title = "Modern C: The C memory model"
description = "A guide to the C23 memory model, covering object representations, unions, strict aliasing, void pointers, casts, effective types, and alignment."
date = 2026-09-28T04:47:32Z

[taxonomies]
tags = ["modernc", "object", "cast", "union", "alignment", "voidpointer", "unsignedchar"]

[extra]
math = true
# cover.image = "images/cover.png"
+++

## 12. The C memory model

- The C memory model provides an abstraction mapping objects to a virtual memory space.
- The memory model focuses heavily on the type of object a pointer addresses.
- **Takeaway 12 #1:** Pointer types with distinct base types are distinct.

```c
#include <stdio.h>

int main(void) {
    double x = 3.14;
    int y = 42;

    double* pd = &x;
    int* pi = &y;

    // Takeaway 12 #1: 'pd' and 'pi' are distinct types.
    // They cannot be freely assigned to each other without raising errors or requiring a cast.
    // pi = pd; // This would cause a compilation error.

    return 0;
}
```

## 12.1. A uniform memory model

- All objects in C can be treated as an assemblage of bytes on a fundamental level.
- **Takeaway 12.1 #1:** `sizeof(char)` is 1 by definition.
- **Takeaway 12.1 #2:** Every object A can be viewed as `unsigned char[sizeof A]`.
- **Takeaway 12.1 #3:** Pointers to character types are special because they can alias other types.
- **Takeaway 12.1 #4:** Use the type `char` for character and string data.
- **Takeaway 12.1 #5:** Use the type `unsigned char` as the atom of all object types.
- **Takeaway 12.1 #6:** The `sizeof` operator can be applied to objects and object types.
- **Takeaway 12.1 #7:** The size of all objects of type `T` is given by `sizeof(T)`.

```c
#include <stdio.h>

int main(void) {
    int value = 258;

    // Takeaway 12.1 #1 & #6: sizeof(char) is exactly 1.
    printf("Size of char: %zu\n", sizeof(char));

    // Takeaway 12.1 #7: Size of the int object 'value'
    printf("Size of int: %zu\n", sizeof(value));

    // Takeaway 12.1 #2 & #5: Treating object as unsigned char array
    unsigned char* byte_view = (unsigned char*)&value;
    printf("First byte of 'value': %u\n", byte_view[0]);

    return 0;
}
```

## 12.2. Unions

- Unions overlay different object types over the same object representation.
- **Takeaway 12.2 #1:** The in-memory order of the representation digits of an arithmetic type is implementation-defined (endianness).
- **Takeaway 12.2 #2:** On most architectures, `CHAR_BIT` is 8 and `UCHAR_MAX` is 255.

```c
#include <stdio.h>

// Example adapted from the book: endianness.c
typedef union unsignedInspect unsignedInspect;
union unsignedInspect {
    unsigned val;
    unsigned char bytes[sizeof(unsigned)];
};

int main(void) {
    // Overlays an unsigned integer and an array of characters
    unsignedInspect twofold = { .val = 0xAABBCCDD, };

    // Takeaway 12.2 #1: The output order here depends on the architecture's endianness
    printf("First byte: 0x%02X\n", twofold.bytes[0]);

    return 0;
}
```

## 12.3. Memory and state

- Aliasing happens when the same object is accessed through different pointers. This inhibits compiler optimizations because the abstract state becomes difficult to determine.
- **Takeaway 12.3 #1 (Aliasing):** With the exclusion of character types, only pointers of the same base type may alias.
- **Takeaway 12.3 #2:** Avoid the `&` operator to help prevent accidental aliasing.

```c
#include <stdio.h>
#include <stddef.h>

// Takeaway 12.3 #1: The compiler assumes 'a' and 'b' point to different objects
// because size_t* and double* are distinct and neither is a character type.
size_t blob(size_t const* a, double* b) {
    size_t myA = *a;
    *b = 2 * myA;
    return *a; // Compiler safely assumes *a is still equal to myA
}

int main(void) {
    size_t val1 = 10;
    double val2 = 0;

    blob(&val1, &val2);
    printf("val2 is now: %f\n", val2);

    return 0;
}
```

## 12.4. Pointers to unspecific objects

- The `void*` type acts as a pointer into a generic storage instance, stripping away type information.
- **Takeaway 12.4 #1:** Any object pointer converts to and from `void*`.
- **Takeaway 12.4 #2:** An object has storage, type, and value.
- **Takeaway 12.4 #3:** Converting an object pointer to `void*` and then back to the same type is the identity operation.
- **Takeaway 12.4 #4:** Avoid `void*` when possible because it removes the compiler's ability to protect you with type checking.

```c
#include <stdio.h>

int main(void) {
    double original = 42.5;

    // Takeaway 12.4 #1: Implicit conversion to void*
    void* generic_ptr = &original;

    // Takeaway 12.4 #3: Converting back to double* restores the exact original pointer
    double* restored_ptr = generic_ptr;

    printf("Restored value: %f\n", *restored_ptr);

    return 0;
}
```

## 12.5. Explicit conversions

- Casts explicitly override the compiler's type system, which often masks poor design decisions.
- **Takeaway 12.5 #1:** Don't use casts.
- _Exception:_ You may cast an object pointer to a character pointer (like `unsigned char*`) to inspect its byte representation.

```c
#include <stdio.h>

int main(void) {
    float f = 37.0f;
    double a = f;        // Safe implicit conversion
    void* pv = &f;       // Safe implicit conversion to void*
    float* pfv = pv;     // Safe implicit conversion back from void*

    // Takeaway 12.5 #1: Avoid explicit casts. If you must inspect bytes:
    unsigned val = 0xAABBCCDD;
    unsigned char* valp = (unsigned char*)&val; // One of the rare acceptable casts

    for (size_t i = 0; i < sizeof(val); ++i) {
        printf("byte[%zu]: 0x%.02hhX\n", i, valp[i]);
    }

    return 0;
}
```

## 12.6. Effective types

- The effective type restricts how an object's memory can be accessed to prevent aliasing violations.
- **Takeaway 12.6 #1 (Effective type):** Objects must be accessed through their effective type or through a pointer to a character type.
- **Takeaway 12.6 #2:** Any member of an object that has an effective union type can be accessed at any time.
- **Takeaway 12.6 #3:** The effective type of a variable or compound literal is the type of its declaration.
- **Takeaway 12.6 #4:** Variables and compound literals must be accessed through their declared type or through a pointer to a character type.

```c
#include <stdio.h>

int main(void) {
    float myFloat = 3.14f; // Declared type is float

    // Takeaway 12.6 #1 & #4: Valid access via effective type
    float* fPtr = &myFloat;

    // Takeaway 12.6 #1 & #4: Valid access via character type pointer
    unsigned char* cPtr = (unsigned char*)&myFloat;

    // INVALID access (Violates strict aliasing/effective types)
    // int* iPtr = (int*)&myFloat;
    // printf("%d", *iPtr); // Undefined behavior

    return 0;
}
```

## 12.7. Alignment

- Objects typically start at specific byte positions (word boundaries) called alignments to optimize architecture operations.
- Keywords: `alignof` returns the alignment requirement of a type, and `alignas` forces allocation at a specified alignment.

```c
#include <stdio.h>
#include <stdalign.h> // Header for older standard compatibility

int main(void) {
    // 'alignof' returns the alignment of the specific type
    printf("Alignment of double: %zu\n", alignof(double));

    // 'alignas' forces custom alignment. Useful for vectorized operations.
    // Forcing an array of 4 floats to align to the total size of the array itself.
    alignas(sizeof(float[4])) float fvec[4] = {1.0, 2.0, 3.0, 4.0};

    printf("Custom alignment of fvec (address): %p\n", (void*)fvec);

    return 0;
}
```
